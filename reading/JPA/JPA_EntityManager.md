# JPA EntityManager 가이드

이 문서는 JPA의 `EntityManager`에 대한 상세 가이드입니다. `EntityManager`의 역할, 주요 메서드, 사용 시나리오, Spring Data JPA의 `JpaRepository`와의 관계를 설명하며, Repository Adapter 구현 예시를 포함합니다. 예시는 `User`와 `Balance` 엔티티를 기반으로 하며, 초보자도 이해할 수 있도록 구조화했습니다.

---

## 1. EntityManager란?

`EntityManager`는 JPA에서 엔티티와 데이터베이스 간의 상호작용을 관리하는 핵심 인터페이스입니다. 엔티티의 생명주기를 관리하며, CRUD 작업, 쿼리 실행, 트랜잭션 관리 등을 처리합니다.

### 1.1. 주요 역할
- **영속성 컨텍스트 관리**: 엔티티를 영속성 컨텍스트(Persistence Context)에 추가, 조회, 수정, 삭제하여 상태를 관리합니다.
- **데이터베이스 연동**: 엔티티와 데이터베이스 간 매핑을 통해 SQL 쿼리를 생성하고 실행합니다.
- **트랜잭션 연계**: 트랜잭션 내에서 엔티티 변경 사항을 추적하고, 커밋 시 데이터베이스에 반영합니다.

### 1.2. 영속성 컨텍스트란?
- 영속성 컨텍스트는 엔티티의 상태를 관리하는 논리적 저장소입니다.
- **역할**:
  - 엔티티 캐싱: 조회한 엔티티를 캐싱하여 동일 트랜잭션 내에서 재사용.
  - 변경 감지(Dirty Checking): 엔티티 변경 시 자동으로 `UPDATE` 쿼리 생성.
  - 지연 로딩(Lazy Loading): 연관 엔티티를 필요 시 로드.
- **생명주기**: 트랜잭션 시작 시 생성되고, 커밋 또는 롤백 시 종료됩니다.

### 1.3. EntityManager와 JpaRepository의 관계
- Spring Data JPA의 `JpaRepository`는 `EntityManager`를 내부적으로 사용하여 CRUD 작업과 쿼리를 처리합니다.
- `JpaRepository`는 고수준 API로, `EntityManager`의 저수준 메서드(`persist`, `merge`, `remove` 등)를 추상화하여 사용 편의성을 높입니다.
- 예: `JpaRepository.save`는 내부적으로 `EntityManager.persist` 또는 `merge`를 호출.

---

## 2. EntityManager의 주요 메서드 (사용 빈도 기준)

`EntityManager`는 엔티티 관리와 데이터베이스 작업을 위한 다양한 메서드를 제공합니다. 아래는 사용 빈도에 따라 정리된 주요 메서드입니다.

### 2.1. 매우 자주 사용되는 메서드

#### 2.1.1. `persist(Object entity)` (사용 빈도: 높음)
- **역할**: 새로운 엔티티를 영속성 컨텍스트에 추가하여 영속 상태로 전환. 트랜잭션 커밋 시 `INSERT` 쿼리 실행.
- **특징**:
  - 새로운 엔티티에만 사용. 이미 영속 상태인 엔티티는 무시.
  - 비영속(Detached) 엔티티는 예외(`IllegalStateException`) 발생 가능.
  - `cascade = CascadeType.PERSIST`로 연관 엔티티도 저장 가능.
- **예시**:
  ```java
  @PersistenceContext
  private EntityManager entityManager;

  @Transactional
  public void createUser() {
      User user = new User();
      user.setName("김철수");
      entityManager.persist(user); // 영속성 컨텍스트에 추가
  }
  ```

#### 2.1.2. `find(Class<T> entityClass, Object primaryKey)` (사용 빈도: 높음)
- **역할**: ID로 엔티티를 조회. 영속성 컨텍스트에 캐싱.
- **특징**:
  - 엔티티가 없으면 `null` 반환.
  - `FetchType.LAZY`로 설정된 연관 엔티티는 지연 로딩.
- **예시**:
  ```java
  public User findUser(Long id) {
      return entityManager.find(User.class, id);
  }
  ```

#### 2.1.3. `merge(Object entity)` (사용 빈도: 높음)
- **역할**: 비영속 상태의 엔티티를 영속성 컨텍스트에 병합하여 업데이트. 트랜잭션 커밋 시 `UPDATE` 쿼리 실행.
- **특징**:
  - 기존 엔티티가 없으면 새로 생성하지 않고 예외 발생 가능.
  - 병합된 엔티티는 새로운 영속 엔티티로 반환.
- **예시**:
  ```java
  @Transactional
  public User updateUser(Long id, String newName) {
      User user = new User();
      user.setId(id);
      user.setName(newName);
      return entityManager.merge(user); // 병합 후 영속 엔티티 반환
  }
  ```

#### 2.1.4. `remove(Object entity)` (사용 빈도: 높음)
- **역할**: 영속 상태의 엔티티를 삭제. 트랜잭션 커밋 시 `DELETE` 쿼리 실행.
- **특징**:
  - 비영속 엔티티는 예외 발생.
  - `cascade = CascadeType.REMOVE`로 연관 엔티티도 삭제 가능.
- **예시**:
  ```java
  @Transactional
  public void deleteUser(Long id) {
      User user = entityManager.find(User.class, id);
      if (user != null) {
          entityManager.remove(user);
      }
  }
  ```

### 2.2. 중간 빈도로 사용되는 메서드

#### 2.2.1. `flush()` (사용 빈도: 중간)
- **역할**: 영속성 컨텍스트의 변경 사항을 즉시 데이터베이스에 동기화.
- **사용 시점**: 트랜잭션 커밋 전 강제로 쿼리 실행이 필요할 때(예: 테스트, 중간 상태 확인).
- **예시**:
  ```java
  @Transactional
  public void saveAndFlush() {
      User user = new User();
      user.setName("김철수");
      entityManager.persist(user);
      entityManager.flush(); // 즉시 INSERT 쿼리 실행
  }
  ```

#### 2.2.2. `detach(Object entity)` (사용 빈도: 중간)
- **역할**: 영속 상태의 엔티티를 비영속 상태로 전환, 변경 추적 중지.
- **사용 시점**: 특정 엔티티를 더 이상 관리하지 않으려 할 때.
- **예시**:
  ```java
  @Transactional
  public void detachUser(Long id) {
      User user = entityManager.find(User.class, id);
      entityManager.detach(user); // 비영속 상태로 전환
  }
  ```

#### 2.2.3. `createQuery(String jpql)` (사용 빈도: 중간)
- **역할**: JPQL 쿼리를 생성하여 복잡한 조회 수행.
- **예시**:
  ```java
  public List<User> findUsersByName(String name) {
      return entityManager.createQuery("SELECT u FROM User u WHERE u.name LIKE :name", User.class)
                         .setParameter("name", "%" + name + "%")
                         .getResultList();
  }
  ```

### 2.3. 낮은 빈도로 사용되는 메서드

#### 2.3.1. `refresh(Object entity)` (사용 빈도: 낮음)
- **역할**: 데이터베이스에서 엔티티를 다시 조회하여 영속성 컨텍스트를 갱신.
- **사용 시점**: 데이터베이스 상태와 영속성 컨텍스트를 동기화해야 할 때.
- **예시**:
  ```java
  @Transactional
  public void refreshUser(Long id) {
      User user = entityManager.find(User.class, id);
      entityManager.refresh(user); // DB에서 최신 데이터로 갱신
  }
  ```

#### 2.3.2. `clear()` (사용 빈도: 낮음)
- **역할**: 영속성 컨텍스트의 모든 엔티티를 비영속 상태로 전환.
- **사용 시점**: 대량 작업 후 메모리 정리 필요 시.
- **예시**:
  ```java
  @Transactional
  public void clearContext() {
      entityManager.clear();
  }
  ```

#### 2.3.3. `contains(Object entity)` (사용 빈도: 낮음)
- **역할**: 엔티티가 영속성 컨텍스트에 포함되어 있는지 확인.
- **예시**:
  ```java
  public boolean isManaged(User user) {
      return entityManager.contains(user);
  }
  ```

---

## 3. EntityManager와 JpaRepository의 관계

- **JpaRepository**:
  - 고수준 API로, `EntityManager`의 기능을 추상화.
  - 예: `save`는 `persist` 또는 `merge`를 호출, `findById`는 `find`를 호출.
  - 메서드 이름 규칙, 페이징, `@Query`로 간편한 쿼리 제공.
- **EntityManager**:
  - 저수준 API로, 세밀한 제어가 가능.
  - 복잡한 트랜잭션 관리, 동적 쿼리, 특정 생명주기 제어 시 사용.
- **사용 선택**:
  - 간단한 CRUD: `JpaRepository` 사용.
  - 복잡한 로직, 저수준 제어: `EntityManager` 사용.

---

## 4. EntityManager 사용 시나리오

### 4.1. 기본 CRUD
- **생성**: `persist`로 새 엔티티 저장.
- **조회**: `find` 또는 `createQuery`로 엔티티 조회.
- **수정**: `merge`로 비영속 엔티티 병합, 또는 영속 엔티티 수정 후 변경 감지.
- **삭제**: `remove`로 엔티티 삭제.

### 4.2. 복잡한 트랜잭션 관리
- 트랜잭션 내에서 여러 엔티티를 조작하고, `flush`로 중간 동기화.
- 예: 대량 데이터 삽입 후 중간 상태 확인.

### 4.3. 동적 쿼리
- JPQL 또는 네이티브 SQL로 런타임에 쿼리 생성.
- 예: 조건에 따라 동적으로 사용자 조회.

---

## 5. Repository Adapter 구현 예시

`User`와 `Balance` 엔티티를 기반으로 `EntityManager`와 `JpaRepository`를 혼합한 Repository Adapter를 구현합니다.

### 5.1. 엔티티 정의
```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "balance_id")
    private Balance balance;

    // Getter, Setter
}

@Entity
public class Balance {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal amount;

    // Getter, Setter
}
```

### 5.2. JpaRepository 인터페이스
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByName(String name);
    @Query("SELECT u FROM User u WHERE u.balance.amount > :amount")
    List<User> findByBalanceAmountGreaterThan(@Param("amount") BigDecimal amount);
}
```

### 5.3. Repository Adapter 구현
```java
@Component
public class UserRepositoryAdapter {
    private final UserRepository userRepository;
    private final EntityManager entityManager;

    @Autowired
    public UserRepositoryAdapter(UserRepository userRepository, EntityManager entityManager) {
        this.userRepository = userRepository;
        this.entityManager = entityManager;
    }

    @Transactional
    public User saveUser(User user) {
        entityManager.persist(user); // EntityManager로 저장
        return user;
    }

    public Optional<User> findUserById(Long id) {
        User user = entityManager.find(User.class, id); // EntityManager로 조회
        return Optional.ofNullable(user);
    }

    public List<User> findUsersByName(String name) {
        return userRepository.findByName(name); // JpaRepository 사용
    }

    public List<User> findUsersByBalanceAmount(BigDecimal amount) {
        return userRepository.findByBalanceAmountGreaterThan(amount);
    }

    @Transactional
    public void deleteUser(Long id) {
        User user = entityManager.find(User.class, id);
        if (user != null) {
            entityManager.remove(user); // EntityManager로 삭제
        }
    }

    @Transactional
    public void updateUserName(Long id, String newName) {
        User user = entityManager.find(User.class, id);
        if (user != null) {
            user.setName(newName); // 변경 감지로 자동 UPDATE
        }
    }

    @Transactional
    public void flushChanges() {
        entityManager.flush(); // 즉시 동기화
    }
}
```
- **설명**:
  - `persist`, `find`, `remove`를 사용해 저수준 제어.
  - `JpaRepository`의 고수준 메서드와 조합하여 도메인 로직 분리.
  - `@Transactional`으로 트랜잭션 관리.

---

## 6. 주의사항 및 팁

### 6.1. 트랜잭션 관리
- `persist`, `merge`, `remove`, `flush`는 `@Transactional` 내에서 실행해야 함.
- 트랜잭션 없이 호출 시 예외(`TransactionRequiredException`) 발생.

### 6.2. N+1 문제
- 연관 엔티티 조회 시 N+1 쿼리 발생 가능.
- 해결: `createQuery`에 `JOIN FETCH` 사용 또는 `@EntityGraph` 적용.
- 예:
  ```java
  public List<User> findUsersWithBalance() {
      return entityManager.createQuery("SELECT u FROM User u JOIN FETCH u.balance", User.class)
                         .getResultList();
  }
  ```

### 6.3. 성능 최적화
- **지연 로딩**: `fetch = FetchType.LAZY`로 설정.
- **벌크 연산**: `createQuery`와 `@Modifying`으로 대량 업데이트/삭제.
- **캐싱**: 영속성 컨텍스트 활용으로 반복 조회 감소.

### 6.4. 테스트
- 테스트 컨테이너(Testcontainers)로 실제 DB 환경 테스트.
- `flush`와 `clear`로 영속성 컨텍스트 상태 관리.
- 예:
  ```java
  @Test
  @Transactional
  public void testSaveUser() {
      User user = new User();
      user.setName("김철수");
      entityManager.persist(user);
      entityManager.flush();
      assertNotNull(entityManager.find(User.class, user.getId()));
  }
  ```

### 6.5. JpaRepository와의 조합
- 간단한 CRUD: `JpaRepository` 사용.
- 복잡한 쿼리, 트랜잭션 제어: `EntityManager` 사용.
- Repository Adapter에서 둘을 조합하여 도메인과 데이터 계층 분리.

---

## 7. 결론
- **EntityManager**:
  - JPA의 저수준 API로, 엔티티 생명주기와 데이터베이스 작업 관리.
  - 주요 메서드: `persist`, `find`, `merge`, `remove`, `flush`, `createQuery` 등.
- **JpaRepository**:
  - `EntityManager`를 추상화한 고수준 API.
  - `save`, `findById`, 쿼리 메서드 등으로 간편한 작업 가능.
- **사용 선택**:
  - 간단한 작업: `JpaRepository`로 생산성 향상.
  - 세밀한 제어: `EntityManager`로 직접 관리.
- Repository Adapter 구현 시, `EntityManager`와 `JpaRepository`를 적절히 조합하여 도메인 로직과 데이터 액세스를 분리하세요.