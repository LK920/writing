
## 1. 영속성 컨텍스트란?

영속성 컨텍스트(Persistence Context)는 JPA(Java Persistence API)에서 엔티티 객체를 관리하는 **논리적인 메모리 공간**입니다. EntityManager를 통해 엔티티를 생성, 조회, 수정, 삭제할 때, 이 공간에서 엔티티의 상태를 추적하고 관리합니다. 쉽게 말해, 영속성 컨텍스트는 **엔티티의 생명주기를 관리하는 중간 관리자** 역할을 합니다.

- **비유**: 영속성 컨텍스트를 "임시 작업 공간"으로 생각해보세요. EntityManager가 이 공간에서 엔티티를 관리하고, 트랜잭션이 커밋될 때 이 작업 결과를 데이터베이스(DB)에 반영합니다.

### 질문 (메타인지 촉진)
- 영속성 컨텍스트가 왜 필요한 걸까요? 만약 JPA가 영속성 컨텍스트 없이 동작한다면, 어떤 문제가 생길 것 같나요?

---

## 2. 영속성 컨텍스트의 주요 역할

영속성 컨텍스트는 JPA의 핵심 기능들을 가능하게 하는 중요한 역할을 합니다. 아래는 그 주요 역할들입니다.

### 2.1. 1차 캐시 (First-Level Cache)
- **설명**: 영속성 컨텍스트는 엔티티를 메모리에 저장하여 동일 트랜잭션 내에서 같은 엔티티를 반복 조회할 때 DB에 접근하지 않도록 합니다. 이를 **1차 캐시**라고 부릅니다.
- **동작**:
  - EntityManager의 `find()` 메서드로 엔티티를 조회하면, 영속성 컨텍스트에 해당 엔티티가 캐싱됩니다.
  - 동일 트랜잭션 내에서 같은 엔티티를 다시 조회하면 DB 쿼리 없이 캐시에서 반환됩니다.
- **장점**: DB 접근을 줄여 성능을 향상시킵니다.

#### 예제
```java
EntityManager em = entityManagerFactory.createEntityManager();
EntityTransaction tx = em.getTransaction();
tx.begin();

User user1 = em.find(User.class, 1L); // DB에서 조회, 캐시에 저장
User user2 = em.find(User.class, 1L); // 캐시에서 반환 (DB 쿼리 없음)

tx.commit();
em.close();
```

- **질문**: 위 코드에서 두 번째 `em.find()` 호출 시 DB 쿼리가 실행될까? 왜 그런 결과가 나올까?

### 2.2. 더티 체킹 (Dirty Checking)
- **설명**: 영속성 컨텍스트는 영속 상태(Managed)인 엔티티의 변경 사항을 자동으로 감지합니다. 트랜잭션 커밋 시, 변경된 엔티티를 DB에 반영(UPDATE 쿼리 실행)합니다.
- **동작**:
  - 영속성 컨텍스트는 엔티티의 초기 상태(스냅샷)를 저장합니다.
  - 엔티티의 필드가 변경되면, 커밋 시 스냅샷과 비교해 변경된 부분만 DB에 반영합니다.
- **장점**: 개발자가 명시적으로 UPDATE 쿼리를 작성할 필요가 없습니다.

#### 예제
```java
EntityManager em = entityManagerFactory.createEntityManager();
EntityTransaction tx = em.getTransaction();
tx.begin();

User user = em.find(User.class, 1L); // 영속 상태
user.setName("Bob"); // 필드 변경
// 명시적 merge() 호출 없이도 커밋 시 자동 UPDATE
tx.commit(); // UPDATE 쿼리 실행: UPDATE user SET name = 'Bob' WHERE id = 1
em.close();
```

- **질문**: 더티 체킹이 없다면, 엔티티의 변경 사항을 DB에 반영하려면 어떤 추가 작업이 필요할까?

### 2.3. 쓰기 지연 (Write Behind)
- **설명**: 영속성 컨텍스트는 엔티티의 변경(`persist`, `merge`, `remove`)을 즉시 DB에 반영하지 않고, 트랜잭션 커밋 시 한꺼번에 SQL을 실행합니다. 이를 **쓰기 지연**이라고 합니다.
- **장점**: 여러 변경을 모아서 한 번에 DB에 반영하므로 성능이 최적화됩니다.

#### 예제
```java
EntityManager em = entityManagerFactory.createEntityManager();
EntityTransaction tx = em.getTransaction();
tx.begin();

User user1 = new User("Alice");
User user2 = new User("Bob");
em.persist(user1); // INSERT 쿼리 지연
em.persist(user2); // INSERT 쿼리 지연
tx.commit(); // 커밋 시 두 INSERT 쿼리 한꺼번에 실행
em.close();
```

- **질문**: 쓰기 지연이 없다면, `persist` 호출마다 어떤 일이 일어날까? 이게 성능에 어떤 영향을 줄까?

### 2.4. 엔티티 상태 관리
- **설명**: 영속성 컨텍스트는 엔티티의 상태를 관리합니다. 엔티티는 다음 네 가지 상태 중 하나에 속합니다:
  1. **비영속(Transient)**: 새로 생성된 엔티티로, 영속성 컨텍스트나 DB와 연결되지 않은 상태.
  2. **영속(Managed)**: 영속성 컨텍스트에 의해 관리되는 상태. `persist`나 `find`로 들어간 엔티티.
  3. **분리(Detached)**: 영속성 컨텍스트에서 관리되지 않게 된 상태(예: `em.detach()` 또는 트랜잭션 종료).
  4. **삭제(Removed)**: 영속성 컨텍스트와 DB에서 삭제 예정인 상태(`em.remove()` 호출).
- **동작**:
  - `persist`: 비영속 → 영속 상태로 전환.
  - `merge`: 분리 또는 비영속 → 영속 상태로 병합.
  - `remove`: 영속 → 삭제 상태로 전환.

#### 예제
```java
EntityManager em = entityManagerFactory.createEntityManager();
EntityTransaction tx = em.getTransaction();
tx.begin();

User user = new User("Alice"); // 비영속 상태
em.persist(user); // 영속 상태
user.setName("Bob"); // 더티 체킹으로 변경 감지
em.detach(user); // 분리 상태
em.merge(user); // 다시 영속 상태로 병합
em.remove(user); // 삭제 상태
tx.commit(); // DB에 반영
em.close();
```

- **질문**: `detach`한 엔티티를 수정한 후 `merge`하지 않고 트랜잭션을 커밋하면 어떤 결과가 나올까?

---

## 3. `persist`와 `merge`의 차이
영속성 컨텍스트에서 엔티티를 관리하는 두 가지 주요 메서드인 `persist`와 `merge`의 차이를 정리합니다.

- **`persist(Object entity)`**:
  - **용도**: 새로운 엔티티를 영속성 컨텍스트에 추가하여 영속 상태로 만듭니다.
  - **결과**: 트랜잭션 커밋 시 DB에 `INSERT` 쿼리가 실행됩니다.
  - **상태**: 비영속 → 영속.
  - **예시**:
    ```java
    User user = new User("Alice"); // 비영속
    em.persist(user); // 영속, 커밋 시 INSERT
    ```

- **`merge(Object entity)`**:
  - **용도**: 분리 상태(Detached) 또는 비영속 상태의 엔티티를 영속성 컨텍스트에 병합하여 영속 상태로 만듭니다.
  - **결과**: DB에서 해당 엔티티를 조회한 후, 입력된 엔티티의 데이터를 복사하여 영속 상태로 관리. 커밋 시 `UPDATE` 쿼리가 실행됩니다.
  - **상태**: 분리/비영속 → 영속.
  - **주의**: `merge`는 원본 객체를 수정하지 않고, 새로 영속 상태인 객체를 반환합니다.
  - **예시**:
    ```java
    User detachedUser = new User(1L, "Bob"); // 분리 상태
    User managedUser = em.merge(detachedUser); // 영속 상태로 병합, 커밋 시 UPDATE
    ```

### 질문 (응용)
- 다음 상황에서 `persist`와 `merge` 중 무엇을 사용해야 할까?
  1. DB에 없는 새로운 `User` 객체를 저장.
  2. 외부 API에서 받아온 `User` 데이터를 DB와 동기화.
  3. 이미 영속 상태인 엔티티를 수정 (예: `find`로 조회 후 필드 변경).

---

## 4. 실습: 영속성 컨텍스트 동작 확인
다음은 `User` 엔티티를 사용해 영속성 컨텍스트의 동작을 확인하는 간단한 코드입니다.

### User 엔티티
```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;

@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    public User() {}
    public User(String name) { this.name = name; }
    public void setName(String name) { this.name = name; }
    // Getters and Setters
}
```

### 영속성 컨텍스트 테스트
```java
EntityManager em = entityManagerFactory.createEntityManager();
EntityTransaction tx = em.getTransaction();
tx.begin();

// 1차 캐시 테스트
User user1 = em.find(User.class, 1L); // DB 조회
User user2 = em.find(User.class, 1L); // 캐시에서 반환

// 더티 체킹 테스트
user1.setName("Bob"); // 필드 변경, 커밋 시 자동 UPDATE

// 쓰기 지연 테스트
User newUser = new User("Alice");
em.persist(newUser); // INSERT 지연
tx.commit(); // 모든 변경 사항 DB에 반영
em.close();
```

- **질문**: 위 코드에서 `user1.setName("Bob")` 후 별도로 `merge`를 호출하지 않았는데, 왜 DB에 반영될까?

---

## 5. 메타인지 점검
- 영속성 컨텍스트의 어떤 역할(1차 캐시, 더티 체킹, 쓰기 지연, 상태 관리)이 가장 이해하기 어려웠나요?
- 위 예제를 보고, 네 프로젝트에서 비슷하게 EntityManager를 사용할 수 있는 상황은 어떤 게 있을까?
- 추가로 궁금한 점이나 더 깊게 파고들고 싶은 부분이 있나요?
