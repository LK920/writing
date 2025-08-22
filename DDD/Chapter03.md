---
tags:
  - study
  - architecture
  - domain
  - ddd
  - aggregate
  - 애그리거트
  - aggregate_root
  - 애그리거트루트
aliases:
  - 도메인 아키텍처
---
 본 문은 <도메인 주도 개발 시작하기> 도서의 챕터3(애그리거트)를 학습하고 정리한 내용이다.
# 애그리거트(Aggregate)
## 3.1 애그리거트
### 애그리거트의 필요성
- 도메인 개체 모델이 복잡해지면 개별 구성요소 위주로 모델을 이해하게 되고 전반적인 구조나 큰 수준에서 도메인 간의 관계 파악이 어려워짐 -> 즉, 복잡해질수록 전체가 파악이 안됨
- 상위 수준에서 모델을 조망하기 위해 
### 애그리거트 예시
![[스크린샷 2025-08-05 오후 11.18.38.png]]
### 애그리거트의 장점
- 모델을 이해
- 일관성 관리
- 개발 시간 축소 -> 복잡도 낮아짐, 도메인 기능 확장/변경 시간 감소
### 애그리거트의 특징
- 한 애그리거트에 속한 객체는 유사하거나 동일한 라이프사이클을 가짐
- 애그리거트는 경계를 가짐
	- **도메인 규칙**에 따라 생성되는 구성요소가 한 애그리거트에 속할 가능성이 높음
	- 사용자의 **요구사항**에 따라 함께 변경되는 빈도가 높은 객체는 한 애그리거트에 속할 가능성이 높음

## 3.2 애그리거트 루트
### 애그리거트 루트란?
- 애그리거트는 여러 객체로 구성되기 때문에 한 객체만 상태가 정상이면 안된다. 즉, 일관되어야 함
- 애그리거트 루트 : 애그리거트에 속한 모든 객체의 일관성을 관리할 주체
- 예시) 주문 애그리거트에선 Order가 루트이며 OrderLine, ShippingInfo등은 간접/직접적으로 Order에 속함
### 3.2.1 도메인 규칙과 일관성
- 애그리거트 루트의 핵심 역할 = 애그리거트의 일관성 유지
- 이를 위해 애그리거트가 제공해야할 도메인 기능을 구현
	- 예시) 주문 애그리거트는 배송지변경, 상품 변경과 같은 기능 제공하고, 애그리거트 루트는 이 기능을 구현하는 메서드를 제공
- 도메인 기능은 정해진 도메인 규칙에 따라야 함
```
public class Order{
	
	// 애그리거트 루트는 도메인 규칙을 구현한 기능
	public void changeShippingInfo(ShippingInfo newShippingInfo){
		verifyNotYetShipped(); <- 도메인 기능을 구현하기위한 도메인 규칙
		setShippingInfo(newShippingInfo);
	}
	
	private void verifyNotYetShipped(){
		if(state != OrderState.PAYMENT_WAITING && state != OrderState.PREPARING)
			throw new IllegalStateException("already shipped");
	}
	...
}
```
* \*주의사항 : 애그리거트 외부에서 애그리거트에 속한 객체를 직접 변경해선 안됨
	* (논리적 데이터)일관성이 깨짐
	* -> 이를 해결하기 위해 상태확인 로직 추가 -> 매 번 중복 구현발생
```
ShippingInfo si = order.getShippingInfo();

// 주요 도메인 로직이 중복되는 문제
if(state != OrderState.PAYMENT_WAITING && state != OrderState.PREPARING){
	throw new IllegalStateException("already shipped");
}

si.setAddress(newAddress); <- 외부에서 직접 변경

```
- 애그리거트 루트를 통해서만 도메인 로직 구현
	- 단순 필드 변경하는 set 메서드를 public으로 만들지 않기
		- setXXX로 할 경우 도메인의 의미나 의도를 표현하지 못할 수 있음
		- 도메인 로직을 도메인 객체가 아닌 응용/표현 영역으로 분산 -> 유지 보수 어려워짐
	- value 타입은 불변
		- 애그리거트 외부에서 밸류 객체의 상태를 변경할 수 없음 -> 새로운 객체를 할당해서 값을 변경
### 3.2.2 애그리거트 루트의 기능 구현
- 애그리거트 내부의 다른 객체를 조합해서 기능을 완성
	- Order 내부의 주문 전체 금액을 계산하는 메서드 등
- 구성요소의 상태만 참조하지 않고 기능 실행을 위임
	- Order 구성요소 중 주문 목록 내부에서 별도의 총금액 계산하는 기능을 호출
- 외부에서 애그리거트 내부 상태 변경 시 -> 일관성 해침
	- 그런 기능을 조심해야함
- 팀 표준이나 구현 기술 제약으로 인해 애그리거트 루트의 기능 구현을 public 등으로 해야한다면 외부에서 실행할 수 없도록 패키지나 protected로 범위를 제한할 것
### 3.2.3 트랜잭션 범위
- **트랜잭션 범위는 작을 수록 좋다.**
	- 잠금 대상이 커질 수록 동시에 처리할 수 있는 트랜잭션 개수 감소 -> 전체적인 성능 감소
	- 한 트랜잭션 - 한 개의 애그리거트
		- -> 애그리거트 경계 넘나듦 X
- 애그리거트 간 독립적이어야 한다
	- 독립적이지 않을 경우 의존성 증가, 결합도 증가 -> 유지보수성 하락
- 부득이할 경우 한 애그리거트에서 다른 애그리거트를 직접 수정하지말고 응용 서비스에서 두 애그리거트를 수정하도록 구현
	- (부득이할 경우란) 팀 표준, 기술제약, UI 구현 편리성
```
public class ChangeService{
	// 두 개 이상의 애그리거트를 변경한다면
	// 별도의 서비스에서 변경할 것
	@Transactional
	public void changeShippingInfo(xxx id, xxx newShipping, ...){
		Order order = orderRepository.findById(id);
		if(order == null) throw new OrderNotFoundException();
		order.shipTo(newShipping); -> Order Aggregate
		
		if(useNewShippingAsMemberAddr){
			Member member = findMember(order.getOrderer());
			member.changeAddress(newShipping.getAddress()); -> Member Aggregate
		}
	}
}

```

## 3.3 Repository와 Aggregate
- 객체의 영속성을 처리하는 repository는 aggregate 단위로 존재
	- 예시)
	- Order, OrderLine은 물리적으로 별도의 DB 테이블에 저장
	- 그렇다고 OrderRepository, OrderLineRepository로 각가 만들지 않음
	- Order가 애그리거트 루트고 OrderLine은 그에 속하는 구성요소이므로 OrderRepository만 있음
- 애그리거트 전체를 저장소에서 영속화하기 때문
```
//리포지터리에 애그리거트를 저장하면 애그리거트 전체를 영속화
orderRepository.save(order);

-> Order 애그리거트와 관련된(속한) 구성요소에 매핑된 테이블에 데이터 저장됨
```
- 리포지터리 메서드는 완전한 애그리거트를 제공
```
Order order = orderRepository.findById(orderId); <- Order 구성요소 전체(완전한 Order)
-> 완전하지 않으면 기능 실행 중 에러 발생
```

## 3.4 ID를 이용한 애그리거트 참조
- 애그리거트 간 참조는 문제
	- 편한 탐색 오용
		- 한 애그리거트 내부에서 다른 애그리거트 접근시 일관성 이슈 발생
		- 애그리거트 간 의존 결합도 증가
	- 성능이슈
		- 애그리거트 간 참조될 경우 다양한 경우의 수 고려하여야 하는데 이 때 성능 이슈등 고민 포인트 증가
	- 확장 어려움
		- 단일 DBMS만 사용할 경우 문제 없음
		- 분산 DB 혹은 여러 종류 DB 사용시 다른 애그리거트를 참조 할 수 없어짐
- ID값을 이용하여 해당 문제 완화 
	- 애그리거트 간 경계가 명확
	- 복잡도 감소(결합가 낮아짐, 의존성 제거, 응집도 증가)
	- 다른 애그리거트 접근하여 수정하는 근원적 문제 방지
	- 애그리거트별 다른 구현 기술을 사용가능 -> 확장 쉬움
- ID값을 이용한 참조와 조회 성능
	- ID값을 참조 시 여러 애그리거트를 조회할 경우 조회 속도 문제 발생
		- N + 1 조회 문제
		- N + 1 조회 문제 해결은 객체 참조할 경우 해결되나 이는 다시 원점으로 돌아감
			- 애그리거트 간 참조 -> ID 참조 -> 애그리거트 간 참조... : 문제 해결X
	- 조회 전용 쿼리 사용
		- 쿼리가 복잡하거나 sql에 특화된 기능을 사용해야 한다면 조회를 위한 부분만 별도로 구현
			- JPQL 사용
			- myBatis 사용
			- 캐시 적용
## 3.5 애그리거트 간 집합 연관
### 1 : N, N : 1 관계
- 1 - N 연관을 구현하지 않음
- N - 1 연관을 구현
```
카테고리와 상품이 1:N 관계일 경우

// 1:N으로 구현시 N의 전체를 조회해야한다 -> 조회 성능 이슈 발생
public class Category{
	private Set<Product> products; // 다른 애그리거트에 대한 1:n 연관
	public List<Product> getProducts(int page, int size){
		List<Product> sortedProducts = sortById(products); 
		return sortedProducts.subList((page - 1) * size, page*size);
	}
}

- 구조: `Category` 엔티티 내부에 `Product` 컬렉션을 직접 보유.
- 문제점: 특정 페이지의 제품만 보고 싶어도,  **전체 `products` 컬렉션을 전부 로딩**해야 `subList` 가능.
- 조회 성능: N이 많아질수록 메모리와 쿼리 부하 급증.
- 쿼리 제어: JPA에서는 `@OneToMany` 컬렉션 페이징이 제약이 많고, 실무에서는 보통 fetch join 불가 + 메모리 페이징으로 바뀌는 경우 많음.


// N:1로 구현시 1에 대한 N을 조회 -> 조회 조건이 생겨 조회의 범위가 줄어듬

public class Product{
	private CategoryId categoryId;
	...
}
public class ProductListService{
	public Page<Product> getProductOfCategory(Long categoryId, int page, int size){
		Category category = categoryRepository.findById(categoryId);
		checkCategory(category); // 카테고리 확인
		List<Product> products = 
			productRepository.findByCategoryId(category.getId(), page, size);
		int totalCount = productRepository.countsByCategoryId(category.getId());
		return new Page(page, size, totalCount, products);
	}
}

- 구조: `Product`가 자신의 `categoryId`만 참조.
- 장점: `productRepository`에서 **조건 기반 쿼리 + 페이징** 가능 → DB가 필요한 데이터만 가져옴.
- 조회 성능: 카테고리 ID 조건을 걸어 **필요한 데이터만 로딩**.
- 유연성: 나중에 조회 조건(정렬, 필터링) 확장 가능.
```
### M : N 관계
- 양쪽 애그리거트에 컬렉션으로 연관을 만듦
- 양쪽 애그리거트 구현시 **요구사항**에 맞게 단방향 M:N 연관만 구현하면 됨
	- 예시 - 상품과 카테고리가 양방향일 경우 제품에 속한 모든 카테고리가 필요한 화면은 상세페이지에서만 노출된다는 요구사항일 경우 -> 실제 구현은 상품 - 카테고리 단방향으로 구현
```
// jpa 사용할 경우
@Entity
@Table(naem = "product")
public class Product{
	@EmbeddedId
	private ProductId id;
	@ElementCollection
	@CollectionTable(name = "product_category"
		joinColumns = @JoinColumn(name = "product_id"))
	private Set<categoryId> categoryIds;
	...
}

// 구현 시 지정한 카테고리에 속한 product 목록 조회
@Repository
public class JpaProductRepository implements ProductRepository{
	@PersistenceContext
	private EntityManager entityManager;
	
	@Override
	public List<Product> findByCategoryId(CategoryId catId, int page, int size){
		TypedQuery<Product> query = entityManager.createQuery(
			"select p from Product p " +
			"where :catId member of p.categoryIds order by p.id.id desc",
			Product.class 
		);
		query.setParameter("catId", catId);
		query.setFirstResult((page - 1) * size);
		query.setMaxResults(size);
		return query.getResultList();
	}
	...
}
```

## 3.6 애그리거트를 팩토리로 사용하기
- 애그리거트가 갖고 있는 데이터를 사용하여 다른 애그리거트를 생성이 필요할 경우 -> 팩토리 메서드를 구현
```
// Store 애그리거트가 Product애그리거트를 생성할 때
// 많은 정보를 알아야한다면 직접 생성하지않고 다른 팩토리에 위임하여 구현
public class Store{
	public Product createProduct(ProductId newProductId, ProductInfo pi){
		if(isBlocked()) throw new StoreBlockedException();
		return ProductFactory.create(newProductId.getId(), pi);
	}
}
```