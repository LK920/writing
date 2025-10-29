---
tags:
  - study
  - architecture
  - domain
  - ddd
  - 도메인서비스
  - 애그리거트
  - 트랜잭션
  - 애그리거트잠금
  - 낙관적락
  - 비관적락
  - db락
  - jpa락
  - 동시성제어
  - race_condition
aliases:
---
 본 문은 <도메인 주도 개발 시작하기> 도서의 챕터 8(애그리거트 트랜잭션 관리)를 학습하고 정리한 내용이다.
# 애그리커트 트랜잭션 관리
## 8.1 애그리거트와 트랜잭션
 - 트랜잭션마다 리포지터리는 새로운 애그리거트 객체를 생성
 - 운영자 스레드와 고객 스레드는 같은 `주문` 애그리거트를 생성할 때 이때 각각 다른 객체를 생성
![[스크린샷 2025-09-25 21.27.57.png]]
- 두 스레드를 볼 경우 운영자는 기존 배송지 정보로 배송 상태를 변경, 그 사이 고객은 배송지를 변경
- 즉, 애그리거트의 일관성이 깨진다.(즉, 동시성 이슈 발생)
- 위 이슈 해결 방법
	- 운영자가 배송 상태 조회하고 변경하면 고객 접근 금지(낙관적락)
	- 운영자가 배송 상태 조회하고 변경 후 고객이 조회 후 수정하도록 처리(비관적락)
## 8.2 선점 잠금(비관적 락, Pessimistic Lock)
- 비관적 락 : 먼저 애그리거트를 구한 스레드가 애그리거트 사용이 끝날 때까지 다른 스레드가 해당 애그리거트를 수정하지 못하게 막는 방식
![[스크린샷 2025-09-25 21.46.59.png]]
- 스레드 1이 먼저 애그리거트를 점유하여 점유가 해제 될 때까지 스레드2가 애그리거트를 대기 후 접근가능
- 순차로 반드시 처리해야할 로직일 경우에 적합
- DBMS와 비관적락
	- 비관적 락은 일반적인 DBMS가 제공하는 행단위 잠금을 사용
	- `for update`와 같은 쿼리를 사용해 특정 레코드에 한 커넥션만 접근할 수 있는 잠금장치를 제공
- JPA EntityManager 사용시 `LockModeType`을 인자로 받는 `find()`메서드 제공
	```
	Order order = entityManager.find(
		Order.class, orderNo, LockModeType.PESSIMISTIC_WRITE);
	```
- 스프링 데이터 JPA 사용시 `@Lock` 사용
	```
	import org.springframework.data.jpa.repository.Lock;
	import javax.persistence.LockModeType;
	
	public interface MemberRepository extends Repository<Member, MemberId>{
	
		@Lock(LockModeType.PESSIMISTIC_WRITE)
		@Query("select m from Member m where m.id = :id")
		Optional<Member> findByIdForUpdate(@Param("id) MemberId memberId);
	}
	```

### 8.2.1 비관적 락과 교착 상태(deadLock)
- 교착 상태 : 둘 이상의 작업이 서로 상대가 점유한 자원을 기다리며 무한 대기하는 상태
![[스크린샷 2025-09-25 22.07.59.png]]
	1. 스레드 1 : A 애그리거트에 대한 선점 락 구함
	2. 스레드 2 : B 애그리거트에 대한 선점 락 구함
	3. 스레드 1 : B 애그리거트에 대한 선점 락 시도(B 애그리거트 락 풀릴때 까지 대기)
	4. 스레드 2 : A 애그리거트에 대한 선점 락 시도(A 애그리거트 락 풀릴때까지 대기) 
- 사용자 수가 많을 때 주로 발생
	-> 사용자 수가 많고 공유자원에 대해 동시에 접근할 때 발생
- JPA에서 잠금에 대한 최대 대기시간 지정하여 교착 상태 해결
	- `hints(javax.persistence.lock.timeout)`를 사용
	- 지정한 시간 이내에 잠금을 구하지 못하면 익셉션을 발생
	- 주의할 점 -> DBMS에 따라 적용여부 확인 필수(안되는 곳도 있음)
	```
	Map<String, Object> hints = new HashMap<>();
	hints.put("javax.persistence.lock.timeout", 2000); // 밀리 초
	Order order = entityManager.find(
		Order.class, orderNo, LockModeType.PESSIMISTIC_WRITE, hints);
	```
- 스프링 데이터 JPA에선 `@QueryHints`로 교착 상태 해결
```
import org.springframework.data.jpa.repository.QueryHints;
import javax.persistence.QueryHint;

public interface MemberRepository extends Repository<Member, MemberId> {
    Optional<Member> findById(MemberId memberId);
	
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({
		@QueryHint(name = "javax.persistence.lock.timeout", value = "3000")
    })
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdForUpdate(@Param("id") MemberId memberId);
    
    void save(Member member);
}
```
-> DBMS 별 교착 상태에 빠진 커넥션을 처리하는 방식이 다름
-> DBMS에 대해 JPA가 어떤 식으로 대기 시간을 처리하는지 반드시 확인 필요!

## 8.3 비선점 잠금(낙관적락, Optimistic)
- 낙관적 락 : 동시에 접근하는 것을 막는 대신 변경한 데이터를 실제 DBMS에 반영하는 시점에 변경 가능 여부를 확인하는 방식
	- 일종의 version 확인
	```
		UPDATE aggtable SET version = version + 1, colx = ?, coly = ?
		WHERE aggid = ? and version = 현재버전
	```
	- 수정할 애그리거트와 매핑되는 테이블의 버전 값이 현재 애그리거트의 버전과 동일한 경우에만 데이터 수정
	- ![[스크린샷 2025-09-25 22.58.09.png]]
- JPA는 버전을 이용한 낙관적 락 지원
	- `@Version`을 붙이고 매핑 되는 테이블에 저장할 칼럼을 추가
	```
		@Entity
		@Table(name = "purchase_order")
		@Access(AccessType.FIELD)
		public class Order {
			@EmbeddedId
			private OrderNo number;
			
			@Version
			private long version;
			...
		}
	```

### 8.3.1 강제 버전 증가
- 애그리거트 루트 이외에 다른 엔티티의 값만 변경 될 경우 -> jpa는 루트만 버전값만 갱신(루트와 연관된 엔티티 값은 신경 안씀)
- 애그리거트 관점으로 보면 문제
	- 애그리거트 내에 어떤 구성요소의 상태가 바뀌면 루트 애그리거트도 논리적으론 바뀐 것
- 해결방법 
	- jpa는 `EntityManager`의 `find()`메서드로 엔티티를 구할 때 강제로 버전 값을 증가시키는 잠금 모드를 지원
		- `LockModeType.OPTIMISTIC_FORCE_INCREMENT`를 사용하면 엔티티 상태 변경과는 상관없이 트랜잭션 종료 시점에 버전 값 증가
		- 루트 뿐만 아니라 다른 엔티티나 밸류가 변경되더라고 버전 값을 증가
	- 스프링 데이터 JPA는 `@Lock` 애너테이션을 이용해서 지정 가능

## 8.4 오프라인 선점 잠금(Offline Optimistic Lock)
- 여러 트랜잭션에 걸쳐 동시 변경을 막기 위한 것
- 예를 들어 글 수정 페이지를 들 수 있는데 누군가 수정 화면을 보고 있을 때 수정 화면 자체를 실행하지 못하도록 막아야지 해당 데이터의 일관성을 유지할 수 있다.
- 이때 필요한 것이 오프라인 선점 잠금방식이다.
### 8.4.1 오프라인 선점 잠금을 위한 LockManager 인터페이스와 관련 클래스
- 오프라인 선점 잠금 기능 
	- 잠금 시도
	- 잠금 확인
	- 잠금 해제
	- 잠금 유효시간 연장
```
public interface LockManager{
	// 잠금 시도 : type,id -> 필수
	LockId tryLock(String type, String id) throws LockException;
	// 잠금 여부 확인
	void checkLock(LockId lockId) throws LockException;
	// 잠금 해제
	void releaseLock(LockId lockId) throws LockException;
	// 잠금 유효 시간 연장
	void extendLockExpiration(LockId lockId, long inc) throws LockException;
}
```
- 잠금 시도 시
	- 서비스 
		- 오프라인 선점 잠금 시도
		- 기능 실행
	- 컨트롤러
		- 뷰로 자금 해제에 사용할 `LockId`를 모델에 추가
- 잠금 해제 시
	- 서비스
		- 잠금 확인
		- 기능 실행
		- 잠금 해제
	- 컨트롤러
		- 서비스 호출 시 잠금 ID를 함께 전달
### 8.4.2 DB를 이용한 LockManager 구현
- 잠금 정보를 DB에 저장하여 관리하는 방식
```

create table locks (
	`type` varchar(255),
	id varchar(255),
	lockdid varchar(255),
	expiration_time datetime,
	primary key (`type`, id)
) character set utf8;

```
- 방식은 위와 동일 하나 db에 접근하여 lock 상태여부를 체크하여 관리하는 방식

---
 lock을 이용해서 동시성 제어하는 부분은 꽤나 흥미 있게 읽었던 것 같다. 여러 프로젝트를 하면서 생각치 못한 부분이었기 때문이다. 
 트랜잭션 부분은 도메인 쪽으로만 생각해보는 것도 좋고 더 생각해볼 부분도 많은 것 같다. 가볍게 읽으면서 깊이있게 들어갈 수 있는 단초를 마련해주는 것 같다.