---
tags:
  - study
  - architecture
  - domain
  - ddd
  - aggregate
  - dip
  - entity
  - value
  - 애그리거트
  - layered
aliases:
  - 도메인 아키텍처
---
 본 문은 <도메인 주도 개발 시작하기> 도서의 챕터2(아키텍처 개요)를 학습하고 정리한 내용이다.
# 아키텍처 개요
## 아키텍처
### 4개의 영역
- 표현 영역 : HTTP 요청/응답 및 응용 영역에 수신
- 응용 영역 : 시스템이 사용자에게 제공해야할 기능 구현(직접 수행보단 도메인 로직 수행)
- 도메인 영역 : 도메인 모델 구현(핵심 로직 구현)
- 인프라스트럭처 영역 : 구현 기술에 대한 것(외부 연결점 + 구현점)

## 계층 구조 아키텍처(Layered Archtecture)
```
상위 계층에서 하위 계층으로의 의존만 존재

|-- 표현 계층 --|
	|
	↓
|-- 응용 계층 --|
	|
	↓
|-- 도메인 계층 --|
	|
	↓
|-- 인프라스트럭처 --|

```
- 응용에서 도메인에 의존하기도 하지만 유연하게 응용에서 인프라스트럭처 계층에 의존할 때도 있다.
- 장점
	- 구조를 쉽게 구현가능
- 단점
	- 테스트 어려움
	- 기능 확장이 어려움
	- 이유 : 의존성이 높아서 하위 계층을 구현이 강요됨

## DIP
- DIP : 의존성 역전 규칙
- 저수준 모듈이 고수준 모듈에 의존하도록 바꿈 -> 어떻게? 추상화한 인터페이스를 이용하여
- 고수준 모듈에선 저수준 모듈이 어떻게 구현했는지가 궁금한게 아닌 제대로 정보가 들어오는 게 중요
![[스크린샷 2025-08-05 오후 1.03.47.png]]
- 주의사항
	- DIP를 적용할 때 하위 기능을 추상화한 인터페이스는 고수준 모듈 관점에서 도출
	- 패키지 구성시에도 고수준 모듈에 위치
		- 도메인/응용 위치 = 고수준
		- 인프라스트럭처 = 저수준
	- 반드시 DIP를 적용할 필요는 없다(필수는 아님)

## 도메인 영역의 주요 구성요소
- 도메인 영역은 도메인의 핵심 모델을 구현

| 요소      | 설명                                                                              |
| ------- | ------------------------------------------------------------------------------- |
| 엔티티     | 고유 식별자를 갖는 객체로 자신의 라이프 사이클을 가짐<br>ex) Order, Member, Product 처럼 도메인의 고유 개념      |
| 밸류      | 개념적으로 하나인 값(고유식별자 x)<br>ex) Address, Money와 같은 타입                               |
| 애그리거트   | 엔티티와 밸류 객체를 개념적으로 하나로 묶은 것<br>ex) Order , OrderLine, Orderer를 주문 애그리거트로 묶을 수 있음 |
| 라포지터리   | 도메인 모델의 영속성 처리                                                                  |
| 도메인 서비스 | 특정 엔티티에 속하지 않은 도메인 로직<br>ex) 할인금액 계산 -> 상품, 쿠폰, 회원등급 등 다양한 조건으로 구현              |
### 엔티티와 밸류

 - 도메인 엔티티
	 - 데이터 + 도메인 기능
 - DB 엔티티
	 - 데이터만
- 밸류
	- 불변으로 구현
	- 엔티티의 밸류 타입 데이터 변경 시 객체 자체를 완전히 교체할 것

```
주문 도메인 엔티티
public class Order{

	// 주문 도메인 모델의 데이터
	private OrderNo number;
	private Orderer orderer;
	private ShippingInfo shippingInfo;
	...
	
	//도메인 엔티티는 기능도 함께 제공
	public void changeShippingInfo(ShippingInfo newShippingInfo){
		...
		setShippingInfo(newShippingInfo);
	}
	
	// 밸류 타입의 데이터를 변경할 때는 새로운 객체로 교체
	private void setShippingInfo(ShippingInfo newShippingInfo){
		if(newShippingInfo == null) throw new IllegalArgumentException();
		this.shippingInfo = newShippingInfo;
	}

}

// 주문자 밸류
// 밸류는 불변으로 구현할 것
public class Orderer{
	private String name;
	private String email;
}

```

### 애그리거트
- 애그리거트란?
	- 관련 객체를 하나로 묶은 군집
	- 도메인 모델도 개별 객체뿐만 아니라 상위 수준에서 모델을 볼 수 있어야 전체 모델의 관계와 개별 모델을 이해하는데 도움이된다.
	- 애그리거트는 루트 엔티티(=애그리거트 루트)를 갖는다![[스크린샷 2025-08-05 오후 1.38.31.png]]
### 라포지터리
- 구현을 위한 도메인 모델
- 도메인 객체를 영속화하는 데 필요한 기능을 추상화한 것으로 고수준 모듈에 해당
```

public interface OrderRepository{
	Order findByNumber(OrderNumber number);
	void save(Order order);
	void delete(Order order);
}

```
- 대상을 찾고 저장하는 단위가 애그리거트 루트 기준

## 요청 처리 흐름
- 표현 영역
	- 전송한 데이터 형식이 올바른지 검사
	- 응용 서비스 기능 실행
	- 요청/응답 시 형식 변경(응용 계층에서 사용하거나 응답을 내보낼시 통일되게 하기 위해)
- 응용 영역
	- 도메인 모델을 이용해서 기능을 구현
	- 트랜잭션 관리

## 인프라스트럭처 개요
- 표현/응용/도메인 영역을 지원
- 다른 영역에서 필요로 하는 프레임워크, 구현기술, 보조 기능을 지원
- 예를 들어 db, rest api, kafka 등 외부 연결점 

## 모듈 구성
- 패키지 구조의 정답은 없음
- 도메인이 크면 하위 도메인으로 나누고 각 하위 도메인 마다 별도의 패키지를 구성![[스크린샷 2025-08-05 오후 1.53.45.png]]
- 애그리거트, 모델, 라포지터리는 같은 패키지에 위치
```
...
com.myshop
|- order
	|- domain
		|- Order(애그리거트 루트)
		|- OrderLine
		|- Orderer
		|- OrderRepository
		...
```

- 도메인이 복잡하면 도메인 모델과 도메인 서비스를 별도 패키지 구성도 가능
```
com.myshop
|- order
	|- domain
		|- order (애그리거트 위치)
		|- service ( 도메인 서비스 위치)
```

- 응용 서비스도 도메인별 패키지 구분 예시
```
com.myshop
|- catalog
	|- application
		|- product
		|- category

```