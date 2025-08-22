
## 1. 영속성 컨텍스트란?
영속성 컨텍스트는 JPA가 엔티티를 관리하는 **논리적인 메모리 공간**입니다. EntityManager를 통해 엔티티를 다룰 때, 이 공간에서 엔티티의 상태를 추적하고 DB와 동기화합니다.

- **비유**: 영속성 컨텍스트는 **스마트 노트북**과 같아요. 노트북에 문서(엔티티)를 열어서 작업하고, 수정 사항을 추적한 뒤, 저장(`commit`)할 때 외부 저장소(DB)에 반영합니다.

### 질문
- 영속성 컨텍스트가 없으면 JPA가 어떻게 동작할까? 예를 들어, 엔티티 변경을 어떻게 DB에 반영해야 할까?

## 2. 더티 체킹(Dirty Checking)
더티 체킹은 영속성 컨텍스트가 **영속 상태(Managed)**인 엔티티의 변경 사항을 감지해서, 트랜잭션 커밋 시 자동으로 DB에 반영하는 기능입니다.

- **동작**:
  1. 엔티티가 영속 상태로 들어감(`em.find()`, `em.persist()`).
  2. 영속성 컨텍스트가 엔티티의 초기 상태(스냅샷)를 저장.
  3. 필드 변경 시(예: `user.setName("Bob")`), 변경을 감지.
  4. 트랜잭션 커밋 시, 스냅샷과 비교해 변경된 부분을 DB에 반영(UPDATE 쿼리).

### 예제
```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    public void setName(String name) { this.name = name; }
}

@Transactional
public void updateUser(EntityManager em, Long userId) {
    User user = em.find(User.class, userId); // Managed 상태
    user.setName("Bob"); // 더티 체킹으로 변경 감지
    // 커밋 시 자동 UPDATE
}
```

- **질문**: 더티 체킹이 동작하려면 엔티티가 어떤 상태여야 하나요? `em.detach(user)` 후 `setName`을 호출하면 DB에 반영될까요?

## 3. Managed 상태와 ID의 오해
**Managed 상태**는 엔티티가 영속성 컨텍스트에서 관리되는 상태를 의미합니다. **ID 유무와는 직접 관련이 없어요**.

- **Managed 상태의 조건**:
  - `em.persist()`, `em.find()`, `em.merge()`로 영속성 컨텍스트에 추가된 엔티티.
  - 이 상태에서 더티 체킹, 1차 캐시 같은 기능이 동작.
- **ID와의 관계**:
  - ID가 있는 엔티티도 영속성 컨텍스트에 추가되지 않으면 **비영속(Transient)** 또는 **분리(Detached)** 상태.
  - ID가 없는 엔티티도 `persist`로 추가되면 Managed 상태가 됨(예: `@GeneratedValue`로 ID 자동 생성).

### 예제
```java
User user = new User();
user.setId(1L); // ID 설정, 비영속 상태
em.persist(user); // Managed 상태, 새 엔티티로 간주

User foundUser = em.find(User.class, 1L); // Managed 상태
em.detach(foundUser); // ID 있음, Detached 상태
User mergedUser = em.merge(foundUser); // 다시 Managed 상태
```

- **질문**: ID가 있는 엔티티를 `em.persist()`하면 어떤 일이 일어날까? DB에 이미 같은 ID가 있다면?

## 4. 실습: Managed 상태와 더티 체킹 확인
다음 코드를 통해 Managed 상태와 더티 체킹을 확인해보세요.

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    public User() {}
    public User(String name) { this.name = name; }
    public void setName(String name) { this.name = name; }
}

@Transactional
public void testPersistenceContext(EntityManager em) {
    // 새로운 엔티티 저장
    User user = new User("Alice"); // 비영속
    em.persist(user); // Managed 상태, 커밋 시 INSERT

    // 조회 후 수정
    User foundUser = em.find(User.class, 1L); // Managed 상태
    foundUser.setName("Bob"); // 더티 체킹으로 변경 감지

    // 분리 후 수정
    em.detach(foundUser); // Detached 상태
    foundUser.setName("Charlie"); // 더티 체킹 안 됨
    em.merge(foundUser); // 다시 Managed 상태, 커밋 시 UPDATE
}
```

- **질문**: 위 코드에서 `foundUser.setName("Charlie")` 후 `em.merge()`를 호출하지 않으면 DB에 반영될까? 왜?

## 5. 메타인지 점검
- 영속성 컨텍스트와 더티 체킹 중 어떤 부분이 아직 헷갈리나요?
- Managed 상태를 "ID가 있는 상태"로 오해했던 이유는 뭘까? 이 자료로 오해가 풀렸나요?
- 네 프로젝트에서 Managed 상태와 더티 체킹을 활용할 수 있는 상황은 어떤 게 있을까?