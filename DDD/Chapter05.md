---
tags:
  - study
  - architecture
  - domain
  - ddd
  - aggregate
  - 애그리거트
  - repository
  - jpa
  - pageable
  - pagerequest
  - specification
  - paging
  - sort
  - subselect
aliases:
---
 본 문은 <도메인 주도 개발 시작하기> 도서의 챕터5(리포지터리와 모델구현)를 학습하고 정리한 내용이다.
# 스프링 데이터 JPA를 이용한 조회 기능
### 5.1 시작에 앞서
- CQRS(Command and Query Responsibility Segregation) : 명령 모델과 조회 모델을 분리하는 패턴
	- 명령 모델 : 상태를 변경하는 기능 구현
		- 회원 가입, 암호 변경 등 처럼 도메인 모델은 주로 명령 모델에 해당
	- 조회 모델 : 데이터를 조회하는 기능을 구현
		- jpa, mybatis, jdbcTemplate 등이 이에 해당 됨
## 5.2 검색을 위한 스펙(Specification)
- 스펙이란?
	- 다양한 검색 조건을 조합해야할 경우에 사용할 수 있는 것
	- 애그리거트가 특정 조건을 충족하는지 검사할 때 사용하는 인터페이스
	- `Specification` 패턴은 “조건을 캡슐화해서 재사용”하려는 목적
```
기본 형태

public interface Specification<T> {
	public boolean isSatisfiedBy(T agg);
}

스펙을 리포지터리에 사용 시 agg => 애그리거트루트, DAO에 사용하면 agg -> 검색결과 데이터 객체

```

- 스펙 사용 예시
	- Order 애그리거트 객체가 특정 고객의 주문인지 확인해야하는 경우를 예시
- 일반 적으로는
```
if(order.getOrdererId().getMemberId().getId().equals("member123")){
	// 로직 수행
}
```
- 스펙 사용 시
```
// 주문자 확인 스펙 클래스
public class OrdererSpec implements Specification<Order>{
	private String ordererId;
	
	public OrdererSpec(String ordererId){
		this.ordererId = ordererId;
	}
	
	@Override
	public boolean isSatisfiedBy(Order agg){
		return agg.getOrdererId().getMemberId().getId().equals(ordererId);
	}
}

// 스펙 사용
Specification<Order> spec = new OrdererSpec("member123"); <- 스펙 객체 생성
if (spec.isSatisfiedBy(order)) { // 로직 수행 } <- 간단히 구현 가능 + 필요시 조합까지 가능
```
### 5.3 스프링 데이터 jpa를 이용한 스펙 구현
- 스프링 데이터 JPA는 스펙 인터페이스 기본 제공
- 스펙 인터페이스를 구현한 클래스 예시
	```
	public class OrdererIdSpec implements Specification<OrderSummary> {
	
	    private String orderId;
	
	    public OrdererIdSpec(String ordererId) {
	        this.ordererId = ordererId;
	    }
	
	    @Override
	    public Predicate toPredicate(Root<OrderSummary> root, 
	                             CriteriaQuery<?> query, 
	                             CriteriaBuilder cb) {
	        return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
	    }
	}
	```
	- OrderIdSpec 클래스는 Specification\<OrderSummary> 타입을 구현하므로 OrderSummary에 대한 검색 조건
	- `toPredicate`는 ordererId 프로퍼티 값이 생성자로 전달받은 ordererId와 동일한지 비교
	- `Root<T> root` : 쿼리에서 `FROM T`를 의미. 즉 엔티티 루트.
	- `CriteriaQuery<?> query` : 쿼리 자체(select ...과 같은 최종 쿼리 본문)
	- `CriteriaBuilder cb` : 동적 쿼리 조건을 만들 때 쓰는 팩토리(`=`, `and`, `or` 같은 조건 생성기)
- 스펙 생성 기능을 별도 클래스에 모아서 관리
```
public class OrderSummarySpecs { 
	// 주문자 조회 스펙
	public static Specification<OrderSummary> ordererId(String ordererId) { 
		return (Root<OrderSummary> root, 
			CriteriaQuery<?> query, 
			CriteriaBuilder cb) -> 
			cb.equal(root.<String>get("ordererId"), ordererId); 
	}
	// 일정 기간 내 날짜 조회 스펙
	public static Specification<OrderSummary> orderDateBetween( 
		LocalDateTime from, LocalDateTime to) {
			return (Root<OrderSummary> root, 
				CriteriaQuery<?> query, 
				CriteriaBuilder cb) -> 
				cb.between(root.get(OrderSummary_.orderDate), from, to); 
	} 
}
```
### 5.4 리포지터리/dao에서 스펙 사용하기
- 스펙을 충족하는 엔티티를 검색하고 싶다면 `findAll()` 사용
```
// 스펙 객체를 생성하고
Specifiacation<OrderSummary> spec = new OrdererIdSpec("user12");
// findAll() 메서드를 이용해서 검색
List<OrderSummary> res = orderSummaryDao.findAll(spec);
```
### 5.5 스펙 조합
- 스펙 인터페이스가 제공하는 메서드
	- `and`
		- 스펙 A와 스펙 B를 동시에 만족하는 결과가 필요할 경우 사용
		- 스펙 A.and(스펙 B)
	```
		Specification<OrderSummary> spec1 = ...
		Specification<OrderSummary> spec2 = ...
		Specification<OrderSummary> spec3 = spec1.and(spec2);
	```
	- `or`
	- `not` : 부정
	- `where` : null 가능성이 있는 스펙과 조합시 사용
	```
	// null나올 가능성이 있는 스펙 과 그렇지 않은 스펙 조합시 사용
	Specification<OrderSummary> spec = 
		Specification.where(createNullableSpec()).and(createOtherSpec());
	```
### 5.6 정렬 지정
- 스피링 데이터 jpa 정렬 방법
	- 메서드 이름에 OrderBy를 사용해서 기준 지정
		- `findByXxxOrderByNumberDesc` -> Number 프로퍼티 값 정렬
	- Sort를 인자로 전달
		- `findByXxx(String id, Sort sort)` -> 검색 메서드에 마지막 인자에 sort를 지정
		- 사용방법
		```
		// sort 생성
		Sort sort = Sort.by("number").ascending();
		// 메서드 마지막에 인자 추가
		List<OrderSummary> res = orderSummaryDao.findByOrdererId("user1", sort);
		```
		- 정렬 조건 여러개 일 경우
			- `.and()`를 이용해서 sort 생성 가능
### 5.7 페이징 처리하기
- `Pageable` 타입 : find메서드에 파라미터로 추가하면 자동 페이징 처리
	- `findByName(String name, Pageable pageable)`
	- `PageRequest`클래스를 이용해서 `Pageable` 생성
		- `.of(a,b)` :  a는 페이지 번호(0부터 시작), b는 한 페이지의 개수
	```
	PageRequest page = PageRequest.of(1, 10);
	List<...> list = xxxDao.findByName("user%", page);
	```
- `Sort`필요 시 `PageRequest.of()`의 마지막 파라미터에 추가 
```
	Sort sort = Sort.by("name").descending();
	PageRequest page = PageRequest.of(1,10,sort);
	List<...> list = dao.findByName("user%", page);
```
- 전체 개수 구하기
	- Pageable을 사용하는 메서드의 리턴 타입이 Page일 경우 
		- 스프링 데이터JPA는 목록 조회 쿼리와 함께 Count 쿼리도 실행
	```
	PageRequest pageReq = PageRequest.of(2,3);
	Page<MemberData> page = memberDataDao.findByBlocked(false, pageReq);
	List<MemberData> content = page.getContent(); // 조회 결과 목록
	long totalElements = page.getTotalElemets(); // 조건에 해당하는 전체 개수
	int totalPages = page.getTotalPages(); // 전체 페이지 번호
	int number = page.getNumber(); // 현재 페이지 번호
	int numberOfElements = page.getNumberOfElements(); // 조회 결과 개수
	int size = page.getSize(); // 페이지 크기
	```
	- spec을 사용하는 find 메서드에서 사용 가능
		- 스펙 사용하는 findAll()일 경우에는 count 쿼리 실행
	- `findBy프로퍼티` 형식의 메서드에선 `Pageable` 타입을 사용하더라도 count 쿼리를 실행 안함
- 처음부터 N개 데이터 구하기
	- `findFirstN` + 프로퍼티 : 처음부터 N 개
	- `findTopN` + 프로퍼티 : 처음부터 N 개
### 5.8 스펙 조합을 위한 스펙 빌더 클래스
- 사용 이유 : 코드 가독성, 구조 단순해짐
```
// 스펙 빌더 클래스 사용 하지 않을 경우
Specification<MemberData> spec = Specification.where(null);
if(searchRequest.isOnlyNotBlocked()){
	spec = spec.and(MemberDataSpecs.nonBlocked());
}
if(StringUtils.hasText(searchRequest.getName())){
	spec = spec.and(MemberDataSpecs.nameLike(searchRequest.getName()));
}
List<MemberData> res = memberDataDao.findAll(spec, PageRequest.of(0,5));

// 스펙 빌더 사용 시 
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
	.ifTrue(searchRequest.isOnlyNotBlocked(),
		()->MemberDataSpecs.nonBlocked())
	.ifHasText(searchRequest.getName(),
		name -> MemberDataSpecs.nameLike(searchRequest.getName()))
	.toSpec();
List<MemberData> res = memberDataDao.findAll(spec, PageRequest.of(0,5));
```
- `ifTrue, ifHasText, and` 이외에 필요한 메서드를 추가해서 사용가능
```
public class SpecBuilder{
	// 추가 할 곳
}
```
### 5.9 동적 인스턴스 생성
- JPA는 쿼리 결과를 임의의 객체를 동적으로 생성할 수 있는 기능을 제공
- JPQL의 select 절에 new 키워드 뒤에 생성할 인스턴스의 완전한 클래스 이름을 지정하고 괄호 안에 인자를 지정
```
@Query(
	select new com.my.shop.order.query.dto.OrderView(
		number, id, name, state, name
	) from ~
)
List<OrderView> findOrderViews(String ordererId);

```
### 5.10 하이버네이트 @Subselect 사용
- `@Subselect`
	- 쿼리 결과를 `@Entity`로 매핑할 수 있는 기능
	- 뷰와 비슷한 역할
	- 조회 결과는 수정 불가
- `@Immutable`
	- 하이버네이트는 해당 엔티티의 매핑 필드/프로퍼티가 변경되도 DB에 반영하지 않고 무시
- `@Synchronize`
	- 해당 엔티티와 관련된 테이블 목록을 명시
	- 하이버네이트는 엔티티를 로딩하기 전에 지정한 테이블과 관련된 변경이 발생하면 플러시를 먼저 한다.
	- 따라서 하이버네이트에서 변경 내역이 발생하게 하면 관련 내역을 먼저 플러시 함 -> 변경 내역이 반영