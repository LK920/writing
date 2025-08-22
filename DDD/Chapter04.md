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
  - attributeConverter
  - attributeOverride
  - entity매핑
aliases:
---
 본 문은 <도메인 주도 개발 시작하기> 도서의 챕터4(리포지터리와 모델구현)를 학습하고 정리한 내용이다.
# 리포지터리와 모델 구현
## 4.1 JPA를 이용한 리포지터리 구현
	애그리거트를 어떤 저장소에 저장하느냐에 따라 리포지터리를 구현하는 방법이 다르기에 주로 사용하는 JPA를 기준으로 구현할 예정이다.
### 4.1.1 모듈 위치
- 리포지터리는 도메인 영역, 구현체는 인프라스트럭처 영역에 위치 ![[스크린샷 2025-08-07 오후 2.25.03.png]]
### 4.1.2 리포지터리 기본 기능 구현
- 인터페이스는 애그리거트 루트를 기준으로 작성
	- Order 애그리거트면 Order만 작성(OrderLine, ShippingInfo 등은 작성하지 않음)
```
	public interface OrderRepository{
		// 조회 기능은 주로 findBy프로퍼티(프로퍼티 값)
		Order findById(OrderNo no); // null 리턴이 싫다면 Optional을 사용해도 됨
		void save(orderId);
	}
```
- 인터페이스 구현은 implement를 이용하여 구현하며 repository의 기능들을 구현체에서 구현
- JPA에선 애그리거트를 수정한 결과를 저장소(DB)에 반영하는 메서드를 추가할 필요는 없음(자동 반영)
- ID 외에 다른 조건으로 조회할 경우 JPA의 Criteria나 JPQL을 이용
## 4.2 스프링 데이터 jpa 기본 규칙
```
- 저장
	Order save(Order entity)
	void save(Order entity)
- 조회
	- 식별자를 이용한 조회
		Order findById(OrderNo id)
		Optional<Order> findById(OrderNo id)
	- 특정 프로퍼티를 이용한 조회
		List<Order> findByOrderer(Orderer orderer) -> Orderer을 갖는 Order 전체 목록
	- 중첩 프로퍼티를 이용한 조회
		Orderer 객체의 MemberId를 이용한 조회시
		List<Order> findByOrdererMemberId(MemberId memberId)
- 삭제
	void delete(Order order)
	void deleteById(OrderNo id)
```
## 4.3 매핑 구현
### 4.3.1 엔티티와 밸류 기본 매핑 구현
- 기본 규칙
	- 애그리거트 루트는 엔티티이므로 @Entity로 매핑 설정
	- 밸류는 `@Embeddable`로 매핑
	- 밸류 타입 프로퍼티는 `@Embedded`로 매핑
	- `@AttributeOverride`: VO 내부 속성들을 컬럼명으로 매핑할 때, **기본 속성명을 오버라이드**하여 **컬럼 이름 충돌**을 방지하거나 **명시적 네이밍**을 가능하게 함.
```
// 애그리거트 루트
@Entity
@Table(name = "purchase_order")
public class Order{
	@Embedded
	private Orderer orderer;
	...
}
// 밸류
@Embeddable
public class Orderer{
	@Embedded
	@AttributeOverrides(
		@AttributeOverride(name = "id", column = @Column(name = "orderer_id"))
	) -> MemberId 객체의 id를 orderer_id 칼럼을 참조, 즉 member_id = orderer_id는 같음
	private MemberId memberId;
	private String name;
}

@Embeddable
public class MemberId implements Serializable {
	@Column(name = "member_id")
	private String id;
}

```
- Orderer의 memberId는 Member 애그리거트의 id를 참조하는 걸 볼 수 있다.
- 정리 외 생각 포인트
	- 결과적으로 DB 테이블에는 `orderer_id`라는 컬럼이 생깁니다. 그런데 이 필드는 사실상 **Member 애그리거트의 ID**, 즉 `member_id`인 셈이죠. 이로 인해 **명칭 상의 혼란**이 발생할 수 있다
	- 왜 이렇게 설계하는가?
		- DDD 관점에서는 명확한 역할 구분이 중요
			- `Order` 애그리거트에서 바라본 `Member`는 “주문자(Orderer)”라는 역할입니다.
			- 따라서 도메인 용어로는 `Orderer`, 필드명도 `orderer.memberId`가 되는 게 타당.
		-  DB는 도메인의 역할보다는 구조를 따른다
			- DB는 `Orderer`라는 Value Object를 모르기 때문에 단일 테이블의 컬럼으로 flatten됩니다.
			- 그래서 JPA에서는 `orderer.memberId` → `orderer_id`와 같은 prefix를 붙여야 중복이나 충돌을 방지할 수 있음.
	- 어떠한 관점에서 명칭을 정할지는 선택의 문제인 듯하나 이후 이를 관리할 때 문서화 등이 잘 이루어져야 한다.
### 4.3.2 기본 생성자
- 앤티티와 밸류의 생성자는 객체를 생성할 때 필요한 것을 전달 받는다.
- 밸류는 불변이므로 기본생성자를 추가할 필요는 없다.
- 하지만, Jpa에서 엔티티와 밸류(@Embaddable) 매핑 시 기본생성자를 제공해야함
	- DB에서 데이터를 읽어와 매핑된 객체를 생성할 대 기본 생성자를 사용해서 객체를 생성하기 때문
```
@Embaddable
public class XXX{
	.....
	// jpa를 적용하기 위한 기본 생성자 추가
	protected xxx(){} -> protected는 다른 코드에서 사용 못하도록 막기 위해
}

```
### 4.3.3 필드 접근 방식 사용
- 필드 접근 방식(@Access(AccessType.FIELD)) -> 기능 중심
```
@Entity
@Access(AccessType.FIELD)
public class Order{
	...
	@Column(name = "state)
	@Enumerated(EnumType.STRING)
	private OrderState state;
	// cancel() 등 도메인 기능 구현하여 변경 처리
}
```
- 메서드 접근 방식(@Access(AccessType.PROPERTY)) -> 사용안하는게 좋음 : get/set메서드로 인해 캡슐화가 깨짐
```
@Entity
@Access(AccessType.PROPERTY)
public class Order{
	...
	@Column(name = "state)
	@Enumerated(EnumType.STRING)
	public OrderState getState(){
		return state;
	}
	public void setState(OrderState state){
		this.state = state;
	}
}
```
### 4.3.4 AttributeConverter를 이용한 밸류 매핑 처리
- 밸류 타입의 프로퍼티를 한 개 칼럼에 매핑 시 
```
public class length{            ---------> DB
	private int value; 1000               WIDTH VARCHAR(20) 1000mm
	private String unit; mm
}
```
- AttributeConverter는 밸류 타입과 칼럼 데이터간의 변환을 위해 사용
- 구현 방법
```
1. interface 작성
	public interface AttributeConverter<X,Y>{
		public Y convertToDatabaseColumn (X attribute);
		public X convertToEntityAttribute (Y dbData);
		// X는 밸류 타입
		// Y는 DB 타입
	}
2. 사용한 밸류에 맞는 컨버터 구현
	@Converter(autoApply = true) // false: 컨버터 직접지정, true : 모든 Money타입에 적용
	public class 돈_Converter implements AttributeConverter<Money, Integer>{
		@Override
		public Integer convertToDatabaseColumn(Money money){
			return money == null ? null : money.getValue();
		}
		@Override
		public Money convertToEntityAttribute(Integer value){
			return value == null ? null : new Money(value);
		}
	}
3. @Convert를 이용하여 적용
	public class Order{
		@Column(name = "total_amounts")
		@Convert(converter = 돈_Converter.class)//autoApply true일경우에는 없어도 됨
		private Money totalAmounts; 
	}

```
### 4.3.5 밸류 컬렉션 : 별도 테이블 매핑
- Order 엔티티는 한 개 이상의 OrderLine을 가질 수 있음 -> List\<OrderLine> :  밸류 컬렉션
- 밸류 컬렉션을 별도로 매핑이 필요한 경우
	- `@ElementCollection` : 컬렉션 객체임 명시
	- `@CollectionTable` : 컬렉션 객체를 연결한 테이블 지정 및 조인 칼럼 지정
		- `name` : 테이블 명
		- `joinColumns` : 칼럼명
			- @joinColumn(name = "칼럼명") <- 외래키
	- `OrderColumn` : List 타입일 경우 사용 -> 인덱스 값 자동 추가
		- `name` : 인덱스 추가할 칼럼 명 작성
```예시
public class Order {
	...
	@ElementCollection(fetch = FetchType.EAGER)
	@CollectionTable(name = "order_line",
					joinColomns = @JoinColumn(name = "order_number"))
	@OrderColumn(name = "line_idx")
	private List<OrderLine> orderLines;
	...
}
```
### 4.3.6 밸류 컬렉션 : 한 개의 컬럼 매핑
- 별도의 테이블이 아닌 한 개의 칼럼에 밸류 컬렉션을 저장해야할 때
- AttributeConverter를 이용하여 구현체를 사용하면 된다.
### 4.3.7 밸류를 이용한 ID 매핑
- 식별자 의미를 부각시키기 위해 사용
- `@Id`가 아닌 `@EmbeddedId`를 사용
- 식별자 클래스는 Serializable을 상속받아 사용
```
@Embeddable
public class OrderNo implements Serializable{
	@Column(name = "order_number")
	private String number;
}
```
 - 식별자에 기능을 추가할 수 있는 장점
	 - 예를 들어 주문번호가 1세대, 2세대로 구분해야할 경우
```
	...
	public boolean is2ndGeneration(){
		return number.startsWidth("N");
	}
	...
```
### 4.3.8 별도 테이블에 저장하는 밸류 매핑
- 우선 밸류를 별도로 구성해야하는지 확인
- 이때 라이프사이클이 다르다면 다른 애그리거트일 확률이 높음
- 밸류임이 판별 난다면 ID(식별자) 작성 주의
	- 밸류의 id는 맞지만 이는 엔티티와 연결하기 위함이지 별도의 식별자가 필요함을 의미 하지 않음
- 어떻게 구성해야 할까?
	- 밸류이니 `@Embeddable`로 매핑
	- 엔티티에선 `@SecondaryTable` 을 이용한다.
```
@Entity
@SecondaryTable(
	name = "article_content",
	pkJoinColumns = @PrimaryKeyJoinColumn(name = "id") <-밸류 테이블에서 조인할 컬럼
)
public class Article{
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id; <-- 엔티티의 식별자
	
	@AttributeOverrides([
		@AttributeOverride(...)
		@AttributeOverride(...)
	])
	@Embedded
	private ArticleContent content;
	...
}


pkJoinColumns = @PrimaryKeyJoinColumn(name = "id") 의미
보조테이블의 id 칼럼과 주 테이블의 id 칼럼을 기준으로 조인한다

```
- 문제점 
	- secondaryTable 사용 시 항상 보조 테이블까지 조회하게 됨 -> 성능 이슈
	- 이를 해결하기 위해 보조테이블을 엔티티로 매핑 후 지연 로딩 방식 사용
		- 이런 경우 좋은 방법은 아니다라고 책에선 언급
### 4.3.9 밸류 컬렉션을 @Entity로 매핑
- 팀규칙/ 구현기술 어떠한 이유로 밸류 컬렉션을 엔티티로 만들어야 할 경우
- 벨류 컬렉션을 엔티티로 만들 때 `@Inheritance`, `@DiscriminatorColumn`을 사용
## 4.4 애그리거트 로딩 전략
- **애그리거트 속한 모든 객체는 완전한 상태**여야 함
	- 상태를 변경하는 기능을 실행할 때 애그리거트 상태가 완전
	- 표현 영역에서 애그리거트의 상태 정보를 완전하게 보여줘야 함
- 연관 매핑 조회 방식
	- 즉시 로딩(default)
		- 모든 객체 조회 가능
		- 카타시안 조인 -> 결과 중복 발생
	- 지연 로딩
		- 원하는 일부 결과만 빠르게 조회 가능
		- 세부적인 결과 필요시 지연로딩을 이용하면 세부적인 결과도 조회가능
### 4.5 애그리거트의 영속성 전파
- 저장 메서드 시 애그리거트 루트에 속한 모든 객체가 저장
- 삭제 메서드 시 애그리거트 루트와 속한 모든 객체 삭제
- `@Embeddable`매핑은 함께 저장되고 삭제됨(cascade 속성 추가 안해도 됨)
- `@Entity` 타입에 대한 매핑은 cascade 속성 추가 필요

## 4.6 식별자 생성 기능
- 사용자가 직접 생성 -> 엔티티 생성시 별도 서비스로 구현
- 도메인 로직 생성 
	- 식별자 생성 서비스에 로직을 구현하여 생성
	- OrderIdService 클래스에서 구현 로직 생성
- DB를 이용한 일련번호 사용
	- 리포지터리에서 구현
	- `@GeneratedValue(strategy = "GeneratedType.IDENTITY")` 를 사용
	- 단, 저장 전 까지 식별자를 알 수 없음
## 4.7 도메인 구현과 DIP
 - 앞선 dip를 구현을 통해서 의존성을 낮춰야 한다고 했으나 이 장에서 구현한 도메인들은 jpa 어노테이션으로 인해 dip 원칙을 지키지 못하고 있다.
 - 그럼 완벽하게 분리하는 경우도 있지만 실무에서도 그렇고 구현기술을 변경할 일은 거의 없음
 - dip를 원칙적으로 지키지 않고 어느정도 타협을 하기도 함

JPA를 이용하여 도메인 모델을 매핑하고 구현하는 기술을 간단히 보여준 느낌, 그리고 약간의 실무적인 관점도 들어간 챕터였다.