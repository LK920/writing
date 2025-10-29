---
tags:
  - study
  - architecture
  - domain
  - ddd
  - domainservice
  - 도메인서비스
aliases:
---
 본 문은 <도메인 주도 개발 시작하기> 도서의 챕터 7(도메인 서비스)를 학습하고 정리한 내용이다.
# 도메인 서비스
## 7.1 여러 애그리거트가 필요한 기능
 도메인 영역의 코드를 작성다하보면, 한 애그리거트로 기능을 구현할 수 없을 때가 있다.
	 대표적으로 `결제 로직`
	 - 결제 로직에 필요한 많은 애그리거트들
		 - 상품, 주문, 할인, 회원 등 -> 주체는 어떤 애그리거트인가?
	 - 만약 주문 애그리거트가 필요한 모든 데이터를 가진 뒤 계산을 하게되면?
		 - 코드가 복잡해짐
		 - 책임 구분 힘듦
		 - 다른 애그리거트들 의존하게 됨
 즉, 억지로 한 애그리거트에 넣을 경우 다양한 문제(`책임범위, 코드복잡, 외부의존 등`)가 발생한다.
 -> 도메인 기능을 별도 서비스로 구현이 필요!
## 7.2 도메인 서비스
- 도메인 서비스 : 도메인 영역에 위치한 도메인 로직을 표현할 때 사용
	- 계산 로직
	- 외부 시스템 연동이 필요한 도메인 로직

### 7.2.1 계산 로직과 도메인 서비스
- 응용 서비스는 응용 로직, 도메인 서비스는 도메인 로직!
- 도메인 서비스
	- 상태 없음
	- 로직만 구현
	- 도메인의 의미가 드러나는 용어 사용(타입, 메서드 이름)
		``` java
			// 도메인 서비스 예시(할인 계산 서비스)
			public class DiscountCalculationService{
				// 타입은 Money
				// 메서드 이름은 할인금액이란 의미가 잘 들어남
				public Money calculateDiscountAmounts(
					List<OrderLine> orderLines,
					List<Coupon> coupons,
					MemberGrade grade
				){
					Money couponDiscount = coupons.stream()
						.map(coupon -> calculateDiscount(coupon))
						.reduce(Money(0), (v1, v2) -> v1.add(v2));
					Money membershipDiscount = 
						calculateDiscount(orderer.getMember().getGrade());
						
					return couponDiscount.add(membershipDiscount);
				}
				
				private Money calculateDiscount(Coupon coupon){...}
				private Money calculateDiscount(MemberGrade grad){...}
			}
		```
- 도메인 서비스의 사용 주체는?
	- 애그리거트에 도메인 서비스를 전달하여 사용하는 방식
		- 예를 들어 `Order`의 금액 계산 기능에 전달하여 사용
		- ```java
		  // 주문 애그리거트
		  public class Order{
			  ...
			  public void calculateAmounts(
				  DiscountCalculationService disCalService, MemberGrade grade
			  ){
				  ...
				  // 할인 계산 서비스 호출하여 사용(인자로 도메인 서비스를 전달 받아 사용)
				  Money discountAmounts = 
					  disCalService.calculateDiscountAmounts(
						  this.orderLines, this.coupons, grade);
				  this.paymentAmounts = totalAmounts.minus(discountAmounts);
			  }
		  }
			  
		  ```
		  - 애그리거트에 `도메인 서비스`를 전달하는 것은 `응용 서비스`의 책임
			  - `OrderService`에 `도메인 서비스`를 주입하여 전달
			  - ```java
			    public class OrderService{
				    // 도메인 서비스를 전달하기 위해
				    private DiscountCalculationService disCalService;
				    ...
				    @Transactional
				    public OrderNo placeOrder(OrderRequest orderRequest){
					    OrderNo orderNo = orderRepository.nextId();
					    Order order = creatOrder(orderNo, orderRequest);
					    orderRepository.save(order);
					    return orderNo;
				    }
				    
				    private Order createOrder(
					    OrderNo orderNo, OrderRequest orderReq)
					{
						Member member = findMember(orderReq.getOrdererId());
						Order order = new Order(
							orderNo, orderReq.getOrderLines(),
							orderReq.getCoupons(), createOrderer(member),
							OrerReq.getShippingInfo());
						// 도메인 서비스를 전달하는 부분
						order.calculateAmounts(
							this.disCalService, member.getGrade());
						return order;
				    }
			    }
			    ```
			- 애그리거트에 직접 주입하지 않기 -> 직접 주입할 경우 의존하게 됨
			- note
				- 도메인 객체는 필드로 구성된 데이터와 메서드를 이용한 개념적으로 하나인 모델
	- 도메인 서비스의 기능을 실행할 때 애그리거트를 전달 하는 방식
		- 계좌 이체 기능
		- ```java
		  public class TransferService{
			  public void transfer(Account fromAcc, Account toAcc, Money amounts){
				  fromAcc.withdraw(amounts);
				  toAcc.credit(amounts);
			  }
		  }
		  /*
			  계좌 애그리거트의 상태를 도메인 서비스가 변경한게 아닌가라는 의문이 들 수 있지만
			  도메인 서비스가 직접 상태변경을 한 것이 아닌 간접적으로 한 것에 가깝다.
			  
			  즉, 도메인 서비스는 애그리거트의 도메인 기능을 호출해서 
			  상태 변경이 일어나도록 만들 수는 있지만, 상태 변경의 주체는 애그리거트라는 것이다.
		  */ 
		  ```
		  - 위의 계좌이체 도메인서비스에 계좌 애그리거트를 전달하여 이체 기능을 수행

NOTE. 역할 비교표

| 구분        | 애그리거트                                | 도메인 서비스                                       | 응용서비스                           |
| --------- | ------------------------------------ | --------------------------------------------- | ------------------------------- |
| 책임        | 자신의 상태와 일관성 유지                       | 여러 애그리거트를 조율하여 **도메인 규칙** 수행                  | 트랜잭션, 보안, 흐름제어 등 애플리케이션 실행 관리   |
| 상태 변경     | ✅ 직접 변경 (자기 자신만)                     | ❌ 직접 변경 안 함,  <br>✅ 애그리거트의 메서드를 호출해서 간접 변경 유발 | ❌ 상태 변경 안 함 (애그리거트/도메인 서비스 호출만) |
| 예시 메서드    | `withdraw(amount)`, `credit(amount)` | transfer(fromAcc, toAcc, amount)              | transferMoney(command)          |
| 도메인 규칙 처리 | 개별 애그리거트 내부 규칙                       | 여러 애그리거트에 걸친 규칙                               | 도메인 규칙 없음 (흐름만)                 |
| 트랜잭션 처리   | ❌ 없음                                 | ❌ 없음                                          | ✅ 있음                            |
| 비유        | “계좌” (돈을 빼고 넣는 건 계좌 스스로 함)           | “창구 직원” (여러 계좌에게 ‘넌 빼고, 넌 넣어’라고 지시)           | “은행 시스템” (요청을 받고 트랜잭션 시작/종료 관리) |
### 7.2.2 외부 시스템 연동과 도메인 서비스
- 외부 시스템이나 타 도메인과의 연동
### 7.2.3 도메인 서비스의 패키지 위치
- 도메인 구서요소와 동일한 패키지에 위치![[스크린샷 2025-08-30 오후 12.10.40.png]]
- 엔티티나 밸류등 명시적으로 구분을 원할 시
```
domain
|- model
|- service
|- repository
...
```
### 7.2.4 도메인 서비스의 인터페이스와 클래스
 -  도메인 서비스의 로직이 고정되지 않을 경우 인터페이스로 구현 후 구현한 클래스를 인프라 영역에 두기도 함
---
## 정리
### 도메인 서비스가 어떤 것인지 명시적으로 알 수 있었다. 
  - 도메인 서비스란 한 애그리거트로 기능을 구현하기 어려울 때 이를 조율하여 기능을 수행하기 위한 서비스의 역할을 한다. 책임의 범위는 어디까지 로직 수행이기에 애그리거트의 상태를 직접 변경해선 안된다.
### 도메인 서비스는 도메인 패키지 내에 위치 왜?
- 도메인 규칙을 구현하는 코드이므로 응집된 상태를 유지할 수 있고, 계층 구조가 깔끔해짐
- 애그리거트 여러 개를 엮는 규칙관리
- 순수 비즈니스 규칙을 구현