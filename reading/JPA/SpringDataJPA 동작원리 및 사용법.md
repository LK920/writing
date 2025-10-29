# Spring Data JPA 동작원리 및 사용법

## 📚 목차
1. [Spring Data JPA란?](#spring-data-jpa란)
2. [인터페이스 기반 동작 원리](#인터페이스-기반-동작-원리)
3. [프록시 패턴과 구현체 자동 생성](#프록시-패턴과-구현체-자동-생성)
4. [메서드 이름 기반 쿼리 생성](#메서드-이름-기반-쿼리-생성)
5. [기존 EntityManager vs Spring Data JPA](#기존-entitymanager-vs-spring-data-jpa)
6. [실전 사용법과 베스트 프랙티스](#실전-사용법과-베스트-프랙티스)
7. [트러블슈팅과 주의사항](#트러블슈팅과-주의사항)

## Spring Data JPA란?

Spring Data JPA는 **인터페이스만 정의하면 자동으로 구현체를 생성**해주는 마법 같은 기술입니다.

### 전통적인 Repository 구현 방식
```java
// 1. 인터페이스 정의
public interface UserRepository {
    User findById(Long id);
    User save(User user);
    List<User> findAll();
}

// 2. 구현체 작성 (개발자가 직접)
@Repository
public class UserRepositoryImpl implements UserRepository {
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public User findById(Long id) {
        return entityManager.find(User.class, id);
    }
    
    @Override
    public User save(User user) {
        if (user.getId() == null) {
            entityManager.persist(user);
            return user;
        } else {
            return entityManager.merge(user);
        }
    }
    
    @Override
    public List<User> findAll() {
        return entityManager.createQuery("SELECT u FROM User u", User.class)
            .getResultList();
    }
}
```

### Spring Data JPA 방식
```java
// 인터페이스만 정의하면 끝! 구현체는 Spring이 자동 생성
public interface UserRepository extends JpaRepository<User, Long> {
    // 기본 CRUD 메서드는 이미 제공됨 (save, findById, findAll, delete 등)
    
    // 커스텀 메서드만 추가로 정의
    List<User> findByName(String name);
    Optional<User> findByEmail(String email);
}
```

## 인터페이스 기반 동작 원리

### 1. Spring Bean 등록 과정

```java
@EnableJpaRepositories(basePackages = "com.example.repository")
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**동작 순서:**
1. Spring Boot 시작 시 `@EnableJpaRepositories` 스캔
2. `JpaRepository`를 상속한 인터페이스들을 찾음
3. 각 인터페이스에 대해 프록시 기반 구현체 생성
4. Spring Context에 Bean으로 등록

### 2. 프록시 생성 메커니즘

```java
// Spring이 내부적으로 생성하는 프록시 구현체 (개념적 예시)
public class UserRepositoryProxy implements UserRepository {
    private EntityManager entityManager;
    private JpaQueryExecutor queryExecutor;
    
    @Override
    public User findById(Long id) {
        // JpaRepository의 기본 구현 로직 실행
        return entityManager.find(User.class, id);
    }
    
    @Override
    public List<User> findByName(String name) {
        // 메서드 이름 분석 후 JPQL 생성 및 실행
        String jpql = "SELECT u FROM User u WHERE u.name = :name";
        return entityManager.createQuery(jpql, User.class)
            .setParameter("name", name)
            .getResultList();
    }
}
```

### 3. 런타임 프록시 생성 확인

```java
@Component
public class RepositoryAnalyzer {
    
    @Autowired
    private UserRepository userRepository;
    
    @PostConstruct
    public void analyzeRepository() {
        System.out.println("Repository 클래스: " + userRepository.getClass());
        // 출력: class com.sun.proxy.$Proxy123
        
        System.out.println("Is Proxy: " + AopUtils.isAopProxy(userRepository));
        // 출력: true
        
        // 실제 인터페이스들 확인
        Class<?>[] interfaces = userRepository.getClass().getInterfaces();
        for (Class<?> intf : interfaces) {
            System.out.println("구현 인터페이스: " + intf.getName());
        }
    }
}
```

## 프록시 패턴과 구현체 자동 생성

### 프록시 패턴이란?

```java
// 프록시 패턴의 개념
public interface Subject {
    void doSomething();
}

public class RealSubject implements Subject {
    @Override
    public void doSomething() {
        System.out.println("실제 작업 수행");
    }
}

public class ProxySubject implements Subject {
    private RealSubject realSubject;
    
    @Override
    public void doSomething() {
        // 추가 로직 (로깅, 권한 체크 등)
        System.out.println("프록시에서 전처리");
        
        if (realSubject == null) {
            realSubject = new RealSubject();
        }
        realSubject.doSomething();
        
        System.out.println("프록시에서 후처리");
    }
}
```

### Spring Data JPA의 프록시 동작

```java
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByAgeGreaterThan(int age);
}

// Spring이 생성하는 프록시 (의사코드)
public class UserRepositoryProxyImpl implements UserRepository {
    private final EntityManager entityManager;
    private final QueryMethodExecutor queryExecutor;
    
    // JpaRepository의 기본 메서드들
    @Override
    public User save(User entity) {
        // 저장 로직
        if (entity.getId() == null) {
            entityManager.persist(entity);
            return entity;
        } else {
            return entityManager.merge(entity);
        }
    }
    
    @Override
    public Optional<User> findById(Long id) {
        return Optional.ofNullable(entityManager.find(User.class, id));
    }
    
    // 커스텀 메서드 - 메서드명 분석 후 쿼리 생성
    @Override
    public List<User> findByAgeGreaterThan(int age) {
        // 1. 메서드명 파싱: "findBy" + "Age" + "GreaterThan"
        // 2. JPQL 생성: "SELECT u FROM User u WHERE u.age > :age"
        // 3. 쿼리 실행
        return entityManager.createQuery(
            "SELECT u FROM User u WHERE u.age > :age", User.class)
            .setParameter("age", age)
            .getResultList();
    }
}
```

## 메서드 이름 기반 쿼리 생성

### 지원되는 키워드들

| 키워드 | 예시 | 생성되는 JPQL |
|--------|------|---------------|
| `findBy` | `findByName(String name)` | `WHERE u.name = ?1` |
| `findByAnd` | `findByNameAndAge(String name, int age)` | `WHERE u.name = ?1 AND u.age = ?2` |
| `findByOr` | `findByNameOrEmail(String name, String email)` | `WHERE u.name = ?1 OR u.email = ?2` |
| `GreaterThan` | `findByAgeGreaterThan(int age)` | `WHERE u.age > ?1` |
| `LessThan` | `findByAgeLessThan(int age)` | `WHERE u.age < ?1` |
| `Between` | `findByAgeBetween(int start, int end)` | `WHERE u.age BETWEEN ?1 AND ?2` |
| `Like` | `findByNameLike(String pattern)` | `WHERE u.name LIKE ?1` |
| `In` | `findByAgeIn(Collection ages)` | `WHERE u.age IN ?1` |
| `IsNull` | `findByEmailIsNull()` | `WHERE u.email IS NULL` |
| `IsNotNull` | `findByEmailIsNotNull()` | `WHERE u.email IS NOT NULL` |
| `OrderBy` | `findByNameOrderByAgeDesc(String name)` | `WHERE u.name = ?1 ORDER BY u.age DESC` |

### 실제 사용 예시

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // 단순 조건
    List<User> findByName(String name);
    Optional<User> findByEmail(String email);
    
    // 복합 조건
    List<User> findByNameAndAge(String name, int age);
    List<User> findByNameOrEmail(String name, String email);
    
    // 비교 연산
    List<User> findByAgeGreaterThan(int age);
    List<User> findByAgeBetween(int startAge, int endAge);
    
    // 문자열 검색
    List<User> findByNameContaining(String keyword);
    List<User> findByNameStartingWith(String prefix);
    List<User> findByNameEndingWith(String suffix);
    
    // 정렬
    List<User> findByNameOrderByAgeDesc(String name);
    List<User> findByAgeGreaterThanOrderByNameAscAgeDesc(int age);
    
    // Null 체크
    List<User> findByEmailIsNull();
    List<User> findByEmailIsNotNull();
    
    // 컬렉션
    List<User> findByAgeIn(List<Integer> ages);
    List<User> findByAgeNotIn(List<Integer> ages);
    
    // 존재 여부
    boolean existsByEmail(String email);
    
    // 개수
    long countByAgeGreaterThan(int age);
    
    // 삭제
    void deleteByAge(int age);
    
    // 페이징
    List<User> findByName(String name, Pageable pageable);
    Page<User> findByAgeGreaterThan(int age, Pageable pageable);
}
```

### 메서드명 파싱 과정

```java
// 메서드명: findByNameAndAgeGreaterThanOrderByCreatedAtDesc
// 파싱 결과:
// - Subject: find
// - Predicate: By
// - Conditions: Name And Age GreaterThan  
// - OrderBy: CreatedAt Desc

// 생성되는 JPQL:
// SELECT u FROM User u 
// WHERE u.name = ?1 AND u.age > ?2 
// ORDER BY u.createdAt DESC
```

## 기존 EntityManager vs Spring Data JPA

### EntityManager 방식의 문제점

```java
@Repository
public class UserJpaRepository implements UserRepositoryPort {
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public Optional<User> findById(Long id) {
        try {
            User user = entityManager.find(User.class, id);
            return Optional.ofNullable(user);
        } catch (Exception e) {
            return Optional.empty();
        }
    }
    
    @Override
    public List<User> findByName(String name) {
        try {
            return entityManager.createQuery(
                "SELECT u FROM User u WHERE u.name = :name", User.class)
                .setParameter("name", name)
                .getResultList();
        } catch (Exception e) {
            return new ArrayList<>();
        }
    }
    
    @Override
    public User save(User user) {
        if (user.getId() == null) {
            entityManager.persist(user);
            return user;
        } else {
            return entityManager.merge(user);
        }
    }
    
    // 페이징 처리도 수동...
    @Override
    public List<User> findByNameWithPagination(String name, int limit, int offset) {
        return entityManager.createQuery(
            "SELECT u FROM User u WHERE u.name = :name", User.class)
            .setParameter("name", name)
            .setMaxResults(limit)
            .setFirstResult(offset)
            .getResultList();
    }
}
```

### Spring Data JPA 방식의 장점

```java
// 인터페이스만 정의
public interface UserRepositoryPort extends JpaRepository<User, Long> {
    // 기본 CRUD는 자동 제공
    // Optional<User> findById(Long id);  - 자동 제공
    // User save(User user);              - 자동 제공
    // void deleteById(Long id);          - 자동 제공
    // List<User> findAll();              - 자동 제공
    
    // 커스텀 메서드만 추가
    List<User> findByName(String name);
    
    // 페이징도 간단
    List<User> findByName(String name, Pageable pageable);
    Page<User> findByNameContaining(String keyword, Pageable pageable);
}
```

### 코드 비교

| 측면 | EntityManager | Spring Data JPA |
|------|---------------|-----------------|
| **코드량** | 많음 (보일러플레이트) | 적음 (인터페이스만) |
| **예외처리** | 수동 처리 필요 | 자동 처리 |
| **페이징** | 수동 구현 | Pageable 지원 |
| **정렬** | ORDER BY 직접 작성 | Sort 객체 지원 |
| **타입 안전성** | 런타임 체크 | 컴파일 타임 체크 |
| **테스트** | Mock 객체 필요 | @DataJpaTest 지원 |

## 실전 사용법과 베스트 프랙티스

### 1. 계층 구조 설계

```java
// Domain Layer - Port 인터페이스
public interface UserRepositoryPort extends JpaRepository<User, Long> {
    List<User> findByName(String name);
    Optional<User> findByEmail(String email);
}

// Service Layer - 비즈니스 로직
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepositoryPort userRepository;
    
    public User createUser(String name, String email) {
        // 비즈니스 로직
        if (userRepository.existsByEmail(email)) {
            throw new UserAlreadyExistsException();
        }
        
        User user = User.builder()
            .name(name)
            .email(email)
            .build();
            
        return userRepository.save(user);
    }
}
```

### 2. 복잡한 쿼리는 @Query 사용

```java
public interface UserRepositoryPort extends JpaRepository<User, Long> {
    // 간단한 쿼리는 메서드명으로
    List<User> findByName(String name);
    
    // 복잡한 쿼리는 @Query로
    @Query("SELECT u FROM User u WHERE u.age > :age AND u.active = true")
    List<User> findActiveUsersOlderThan(@Param("age") int age);
    
    // Native Query도 가능
    @Query(value = "SELECT * FROM users WHERE created_at > ?1", nativeQuery = true)
    List<User> findUsersCreatedAfter(LocalDateTime date);
}
```

### 3. 트랜잭션 관리

```java
@Service
@Transactional
@RequiredArgsConstructor
public class UserService {
    private final UserRepositoryPort userRepository;
    
    // 읽기 전용 트랜잭션
    @Transactional(readOnly = true)
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
    
    // 쓰기 트랜잭션
    @Transactional
    public User updateUser(Long id, String newName) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException());
        
        user.updateName(newName);
        
        return userRepository.save(user);
    }
}
```

### 4. 성능 최적화

```java
public interface UserRepositoryPort extends JpaRepository<User, Long> {
    // N+1 문제 해결 - @EntityGraph
    @EntityGraph(attributePaths = {"orders", "profile"})
    List<User> findByName(String name);
    
    // 특정 필드만 조회 - Projection
    @Query("SELECT u.name, u.email FROM User u WHERE u.active = true")
    List<UserSummary> findActiveUserSummaries();
    
    // 배치 처리 - @Modifying
    @Modifying
    @Query("UPDATE User u SET u.active = false WHERE u.lastLoginDate < :date")
    int deactivateInactiveUsers(@Param("date") LocalDateTime date);
}

// Projection 인터페이스
public interface UserSummary {
    String getName();
    String getEmail();
}
```

### 5. 테스트 작성

```java
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager testEntityManager;
    
    @Autowired
    private UserRepositoryPort userRepository;
    
    @Test
    void findByName_shouldReturnUser() {
        // Given
        User user = User.builder()
            .name("John")
            .email("john@example.com")
            .build();
        testEntityManager.persistAndFlush(user);
        
        // When
        List<User> result = userRepository.findByName("John");
        
        // Then
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("John");
    }
}
```

## 트러블슈팅과 주의사항

### 1. 메서드명 길이 제한
```java
// ❌ 너무 긴 메서드명
List<User> findByNameAndEmailAndAgeGreaterThanAndActiveAndCreatedAtBetween(...);

// ✅ @Query 사용
@Query("SELECT u FROM User u WHERE u.name = :name AND u.email = :email " +
       "AND u.age > :age AND u.active = true AND u.createdAt BETWEEN :start AND :end")
List<User> findUsersWithComplexConditions(@Param("name") String name, ...);
```

### 2. N+1 문제
```java
// ❌ N+1 문제 발생
List<User> users = userRepository.findAll();
for (User user : users) {
    user.getOrders().size(); // 각 User마다 추가 쿼리 실행
}

// ✅ Fetch Join 사용
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();

// ✅ @EntityGraph 사용
@EntityGraph(attributePaths = {"orders"})
List<User> findAll();
```

### 3. 타입 안전성
```java
// ❌ 잘못된 메서드명 (컴파일 에러 발생)
List<User> findByNaem(String name); // name 오타

// ❌ 존재하지 않는 필드 (런타임 에러)
List<User> findByNonExistentField(String value);

// ✅ IDE 자동완성 활용
List<User> findByName(String name);
```

### 4. 성능 주의사항
```java
// ❌ 모든 데이터 로드
List<User> findAll(); // 데이터가 많으면 OutOfMemoryError

// ✅ 페이징 사용
Page<User> findAll(Pageable pageable);

// ❌ EXISTS 대신 COUNT 사용
long count = userRepository.countByEmail(email);
boolean exists = count > 0;

// ✅ EXISTS 사용
boolean exists = userRepository.existsByEmail(email);
```

## 마무리

Spring Data JPA는 **인터페이스 기반의 프록시 패턴**을 사용하여 Repository 구현체를 자동 생성하는 혁신적인 기술입니다.

**핵심 포인트:**
- 인터페이스만 정의하면 Spring이 프록시 구현체 자동 생성
- 메서드명 기반 쿼리 생성으로 개발 생산성 향상
- Pageable, Sort 등 편의 기능 기본 제공
- @Query로 복잡한 쿼리도 처리 가능
- 성능과 N+1 문제에 주의하여 사용

이제 EntityManager 대신 Spring Data JPA를 활용해서 더 간결하고 효율적인 Repository를 구현해보세요!