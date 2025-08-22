# JPA Repository 및 EntityManager 메서드 가이드 (persist 포함)

이 문서는 Spring Data JPA의 `JpaRepository` 인터페이스와 JPA `EntityManager`에서 제공하는 주요 메서드들을 사용 빈도 기준으로 정리한 가이드입니다. 특히, `persist` 메서드를 포함하여 기본 CRUD 작업, 쿼리 메서드, 고급 기능, 그리고 Repository Adapter 구현 방법을 상세히 설명합니다. 예시는 `User`와 `Balance` 엔티티를 기반으로 합니다.

---

## 1. JPA Repository 및 EntityManager 개요

- **Spring Data JPA (`JpaRepository`)**:
  - `JpaRepository`는 Spring Data JPA가 제공하는 인터페이스로, CRUD 작업과 쿼리 메서드를 자동으로 구현합니다.
  - 엔티티를 데이터베이스와 매핑하며, 개발자가 직접 SQL을 작성하지 않아도 됩니다.
  - 내부적으로 `EntityManager`를 사용합니다.
- **JPA `EntityManager`**:
  - JPA의 핵심 API로, 영속성 컨텍스트(Persistence Context)를 관리하며 엔티티의 생명주기를 제어합니다.
  - `persist`, `merge`, `remove`, `find` 등 저수준 메서드를 제공합니다.
  - `JpaRepository`의 메서드들은 `EntityManager`의 기능을 기반으로 동작합니다.

---

## 2. `EntityManager`의 `persist` 메서드 (사용 빈도: 중간)

`persist`는 `EntityManager`의 메서드로, 엔티티를 영속성 컨텍스트에 추가하여 데이터베이스에 저장할 준비를 합니다. 주로 새로운 엔티티를 생성할 때 사용됩니다.

### 2.1. 주요 특징
- **역할**: 엔티티를 영속성 컨텍스트에 추가하여 "영속" 상태로 만듭니다. 트랜잭션이 커밋될 때 데이터베이스에 `INSERT` 쿼리가 실행됩니다.
- **사용 시점**:
  - 새로운 엔티티를 데이터베이스에 저장할 때.
  - `JpaRepository`의 `save` 메서드 대신 저수준 제어가 필요할 때.
- **특징**:
  - 이미 영속 상태인 엔티티에 대해 `persist`를 호출하면 무시됩니다.
  - 엔티티가 비영속(Detached) 상태라면 예외(`IllegalStateException`)가 발생할 수 있습니다.
  - 연관된 엔티티도 `cascade = CascadeType.PERSIST`로 설정하면 함께 저장됩니다.

### 2.2. 사용 예시
```java
@PersistenceContext
private EntityManager entityManager;

@Transactional
public void createUser() {
    User user = new User();
    user.setName("김철수");
    entityManager.persist(user); // 영속성 컨텍스트에 추가
    // 트랜잭션 커밋 시 INSERT 쿼리 실행
}
```
- **설명**:
  - `persist`는 엔티티를 영속성 컨텍스트에 추가하며, 트랜잭션 커밋 시 데이터베이스에 저장됩니다.
  - `@Transactional`이 필요하며, 트랜잭션 범위 내에서 실행해야 합니다.

### 2.3. `persist` vs `JpaRepository.save`
- **`persist`**:
  - 새로운 엔티티를 저장(`INSERT`)하는 데 특화.
  - 이미 영속 상태인 엔티티는 무시.
  - 비영속 상태 엔티티에 대해 예외 발생 가능.
- **`save`**:
  - 새로운 엔티티는 `INSERT`, 기존 엔티티는 `UPDATE`를 수행.
  - `EntityManager`의 `persist` 또는 `merge`를 내부적으로 호출.
  - 더 유연하고 사용이 간편함.
- **사용 빈도**:
  - `JpaRepository.save`가 더 자주 사용되며, `persist`는 저수준 제어가 필요한 경우(예: 복잡한 트랜잭션 관리)에 사용됩니다.

---

## 3. 기본 CRUD 메서드 (사용 빈도: 매우 높음)

`JpaRepository`는 기본 CRUD(Create, Read, Update, Delete) 메서드를 제공하며, 가장 자주 사용됩니다. `persist`와 달리 별도의 구현 없이 바로 호출 가능합니다.

### 3.1. Create (생성)
- **메서드**:
  - `<S extends T> S save(S entity)`: 엔티티를 저장하거나 업데이트.
  - `<S extends T> List<S> saveAll(Iterable<S> entities)`: 여러 엔티티를 일괄 저장.
- **사용 예시**:
  ```java
  @Repository
  public interface UserRepository extends JpaRepository<User, Long> {
  }

  @Autowired
  private UserRepository userRepository;

  public void createUser() {
      User user = new User();
      user.setName("김철수");
      userRepository.save(user); // 단일 저장

      List<User> users = Arrays.asList(new User("이영희"), new User("박민수"));
      userRepository.saveAll(users); // 일괄 저장
  }
  ```
- **설명**:
  - `save`: 엔티티의 ID가 없으면 `INSERT`, 있으면 `UPDATE`.
  - `saveAll`: 대량 데이터 삽입 시 효율적.

### 3.2. Read (조회)
- **메서드**:
  - `Optional<T> findById(ID id)`: ID로 엔티티 조회.
  - `List<T> findAll()`: 모든 엔티티 조회.
  - `List<T> findAllById(Iterable<ID> ids)`: 여러 ID로 엔티티 조회.
- **사용 예시**:
  ```java
  public void readUser() {
      Optional<User> user = userRepository.findById(1L);
      user.ifPresent(u -> System.out.println(u.getName()));

      List<User> allUsers = userRepository.findAll();
      List<User> usersByIds = userRepository.findAllById(Arrays.asList(1L, 2L));
  }
  ```
- **설명**:
  - `findById`: `Optional` 반환으로 null 안전.
  - `findAll`: 대량 조회 시 성능 주의.

### 3.3. Update (수정)
- **메서드**:
  - `<S extends T> S save(S entity)`: 엔티티 저장/업데이트 (Create와 동일).
- **사용 예시**:
  ```java
  public void updateUser() {
      Optional<User> user = userRepository.findById(1L);
      user.ifPresent(u -> {
          u.setName("김철수_updated");
          userRepository.save(u); // 수정된 엔티티 저장
      });
  }
  ```
- **설명**:
  - 영속성 컨텍스트가 변경을 감지하여 `UPDATE` 쿼리 실행.
  - 별도의 `update` 메서드 없음.

### 3.4. Delete (삭제)
- **메서드**:
  - `void deleteById(ID id)`: ID로 엔티티 삭제.
  - `void delete(T entity)`: 주어진 엔티티 삭제.
  - `void deleteAll()`: 모든 엔티티 삭제.
  - `void deleteAll(Iterable<? extends T> entities)`: 여러 엔티티 일괄 삭제.
- **사용 예시**:
  ```java
  public void deleteUser() {
      userRepository.deleteById(1L);
      Optional<User> user = userRepository.findById(2L);
      user.ifPresent(userRepository::delete);
      userRepository.deleteAll();
  }
  ```
- **설명**:
  - `deleteById`: ID로 삭제, 엔티티 없으면 예외 가능.
  - `deleteAll`: 대량 삭제 시 성능 주의.

---

## 4. `EntityManager`의 기타 주요 메서드 (사용 빈도: 중간~낮음)

`persist` 외에도 `EntityManager`는 다양한 메서드를 제공합니다. `JpaRepository` 내부에서 이 메서드들을 활용합니다.

### 4.1. `merge` (사용 빈도: 중간)
- **역할**: 비영속(Detached) 상태의 엔티티를 영속성 컨텍스트에 병합하여 업데이트.
- **사용 예시**:
  ```java
  @Transactional
  public void updateUser() {
      User user = new User();
      user.setId(1L);
      user.setName("김철수_updated");
      entityManager.merge(user); // 영속성 컨텍스트에 병합, UPDATE 쿼리 실행
  }
  ```
- **설명**:
  - `merge`는 기존 엔티티를 업데이트하며, 없으면 새로 생성하지 않음(예외 발생 가능).
  - `JpaRepository.save`는 내부적으로 `persist` 또는 `merge`를 호출.

### 4.2. `remove` (사용 빈도: 중간)
- **역할**: 영속 상태의 엔티티를 삭제.
- **사용 예시**:
  ```java
  @Transactional
  public void deleteUser() {
      User user = entityManager.find(User.class, 1L);
      if (user != null) {
          entityManager.remove(user); // DELETE 쿼리 실행
      }
  }
  ```
- **설명**:
  - `remove`는 `deleteById`와 유사하지만, 영속 상태의 엔티티를 직접 삭제.

### 4.3. `find` (사용 빈도: 중간)
- **역할**: ID로 엔티티 조회.
- **사용 예시**:
  ```java
  public User findUser() {
      return entityManager.find(User.class, 1L);
  }
  ```
- **설명**:
  - `findById`와 유사하지만, `EntityManager`를 직접 사용.
  - 반환값은 `Optional`이 아닌 직접 엔티티(없으면 `null`).

### 4.4. `flush` (사용 빈도: 낮음)
- **역할**: 영속성 컨텍스트의 변경 사항을 데이터베이스에 즉시 동기화.
- **사용 예시**:
  ```java
  @Transactional
  public void saveAndFlush() {
      User user = new User();
      user.setName("김철수");
      entityManager.persist(user);
      entityManager.flush(); // 즉시 INSERT 쿼리 실행
  }
  ```
- **설명**:
  - 트랜잭션 커밋 전 강제로 DB에 반영.
  - 테스트나 특정 시나리오에서 사용.

### 4.5. `detach` (사용 빈도: 낮음)
- **역할**: 엔티티를 영속성 컨텍스트에서 분리하여 비영속 상태로 전환.
- **사용 예시**:
  ```java
  @Transactional
  public void detachUser() {
      User user = entityManager.find(User.class, 1L);
      entityManager.detach(user); // 비영속 상태로 전환
  }
  ```
- **설명**:
  - 더 이상 변경 추적 안 함.

---

## 5. 메서드 이름 규칙 기반 쿼리 (사용 빈도: 높음)

`JpaRepository`는 메서드 이름으로 쿼리를 자동 생성합니다. 특정 조건, 정렬, 제한 등을 처리할 때 유용합니다.

### 5.1. 주요 키워드
- `findBy`, `existsBy`, `countBy`, `deleteBy`.
- 연산자: `And`, `Or`, `Is`, `Like`, `In`, `Between`, `GreaterThan`, `LessThan`.
- 정렬: `OrderBy`, `Asc`, `Desc`.
- 제한: `First`, `Top`.

### 5.2. 사용 예시
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByName(String name);
    List<User> findByNameAndAge(String name, int age);
    List<User> findByAgeGreaterThan(int age);
    List<User> findTop10ByOrderByAgeDesc();
    boolean existsByName(String name);
    long countByAge(int age);
    void deleteByName(String name);
}
```
- **설명**:
  - `findByName`: `WHERE name = ?`.
  - `findByAgeGreaterThan`: `WHERE age > ?`.
  - `findTop10ByOrderByAgeDesc`: 상위 10명, 나이 내림차순.

---

## 6. 페이징 및 정렬 (사용 빈도: 중간)

페이징과 정렬은 대량 데이터 처리에 유용합니다.

### 6.1. 주요 메서드
- `Page<T> findAll(Pageable pageable)`: 페이징된 조회.
- `List<T> findAll(Sort sort)`: 정렬된 조회.
- 커스텀: `findBy[필드](..., Pageable pageable)`.

### 6.2. 사용 예시
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByAgeGreaterThan(int age, Pageable pageable);
}

@Autowired
private UserRepository userRepository;

public void pageUsers() {
    Pageable pageable = PageRequest.of(0, 10, Sort.by("age").descending());
    Page<User> userPage = userRepository.findByAgeGreaterThan(20, pageable);
    System.out.println("Total Pages: " + userPage.getTotalPages());
}
```
- **설명**:
  - `Pageable`: 페이지 번호, 크기, 정렬 지정.
  - `Page`: 결과, 총 페이지 수 등 제공.

---

## 7. 커스텀 쿼리 (@Query, 사용 빈도: 중간)

복잡한 쿼리는 `@Query`로 JPQL 또는 네이티브 SQL 작성.

### 7.1. 사용 예시
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name%")
    List<User> findUsersByNameLike(@Param("name") String name);

    @Query(value = "SELECT * FROM user WHERE age > ?1", nativeQuery = true)
    List<User> findUsersByAgeGreaterThan(int age);

    @Modifying
    @Query("UPDATE User u SET u.age = :age WHERE u.id = :id")
    int updateUserAge(@Param("id") Long id, @Param("age") int age);
}
```
- **설명**:
  - JPQL: 엔티티 기반 쿼리.
  - 네이티브 SQL: 직접 SQL 작성.
  - `@Modifying`: DML 쿼리에 사용.

---

## 8. 고급 기능 (사용 빈도: 낮음)

### 8.1. 벌크 연산
- **예시**:
  ```java
  @Modifying(clearAutomatically = true)
  @Query("UPDATE User u SET u.age = u.age + 1 WHERE u.age < :age")
  int incrementAgeForYoungUsers(@Param("age") int age);
  ```
- **설명**:
  - 영속성 컨텍스트 무시, `clearAutomatically` 권장.

### 8.2. Specification
- **예시**:
  ```java
  public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
  }

  public Specification<User> nameContains(String name) {
      return (root, query, cb) -> cb.like(root.get("name"), "%" + name + "%");
  }
  ```
- **설명**: 동적 쿼리 생성.

### 8.3. 엔티티 그래프
- **예시**:
  ```java
  @EntityGraph(attributePaths = {"balance"})
  Optional<User> findById(Long id);
  ```
- **설명**: 연관 엔티티 즉시 로드.

---

## 9. Repository Adapter 구현 예시

`User`와 `Balance` 엔티티를 기반으로 Repository Adapter를 구현합니다.

### 9.1. 엔티티 정의
```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToOne
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

### 9.2. Repository 인터페이스
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByName(String name);
    @Query("SELECT u FROM User u WHERE u.balance.amount > :amount")
    List<User> findByBalanceAmountGreaterThan(@Param("amount") BigDecimal amount);
}
```

### 9.3. EntityManager를 사용한 Repository Adapter
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
        entityManager.persist(user); // persist로 저장
        return user;
    }

    public Optional<User> findUserById(Long id) {
        return userRepository.findById(id);
    }

    public List<User> findUsersByBalanceAmount(BigDecimal amount) {
        return userRepository.findByBalanceAmountGreaterThan(amount);
    }

    @Transactional
    public void deleteUser(Long id) {
        User user = entityManager.find(User.class, id);
        if (user != null) {
            entityManager.remove(user); // remove로 삭제
        }
    }
}
```
- **설명**:
  - `persist`와 `remove`를 사용해 저수준 제어.
  - `JpaRepository` 메서드와 조합하여 도메인 로직 분리.

---

## 10. 주의사항 및 팁

- **persist 사용 시**:
  - `@Transactional` 필수.
  - 새로운 엔티티에만 사용, 기존 엔티티는 `merge` 사용.
- **N+1 문제**:
  - 연관 엔티티 조회 시 `@EntityGraph` 또는 `JOIN FETCH` 사용.
- **성능 최적화**:
  - 대량 데이터: `saveAll`, `deleteAll`.
  - 페이징: `Pageable` 사용.
- **테스트**:
  - 테스트 컨테이너로 실제 DB 환경 테스트.
  - 데이터 정리로 테스트 안정성 확보.

---

## 11. 결론
- **`persist`**: 새로운 엔티티 저장, `EntityManager` 사용, 중간 빈도.
- **기본 CRUD**: `save`, `findById`, `findAll`, `deleteById`가 가장 자주 사용.
- **쿼리 메서드**: 메서드 이름 규칙으로 간단한 조건 처리.
- **고급 기능**: 페이징, 커스텀 쿼리, Specification 등 특정 상황에 유용.
- Repository Adapter 구현 시 `JpaRepository`와 `EntityManager`를 적절히 조합하여 도메인 로직과 데이터 액세스를 분리하세요.