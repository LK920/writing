# Testcontainers와 GitHub Actions을 활용한 통합 테스트 자동화 가이드

## 1. Testcontainers란?

Testcontainers는 통합 테스트를 위해 실제 서비스(예: 데이터베이스, 메시지 브로커)를 Docker 컨테이너로 실행하는 오픈소스 라이브러리입니다. 모킹이나 인메모리 데이터베이스 대신 실제 환경과 유사한 테스트 환경을 제공하여 신뢰성 높은 테스트를 가능하게 합니다.

### 비유
- Testcontainers는 마치 요리 실습실에서 실제 재료를 사용하는 것과 같습니다. 모의 재료(인메모리 DB) 대신 실제 재료(PostgreSQL 컨테이너)를 사용해 요리(테스트)를 준비하고, 실습이 끝나면 주방(컨테이너)을 자동으로 청소합니다.
- 예: 이커머스 프로젝트에서 `orders` 테이블을 테스트하려면, 인메모리 H2 DB 대신 실제 PostgreSQL 컨테이너를 띄워 테스트.

### Testcontainers의 주요 특징
- **실제 의존성 사용**: PostgreSQL, MySQL, Redis 등 실제 서비스를 컨테이너로 실행.
- **자동 관리**: 테스트 시작 시 컨테이너 생성, 종료 시 자동 삭제.
- **다양한 언어 지원**: Java, Python, Go, .NET 등.
- **CI/CD 통합**: GitHub Actions에서 Docker를 활용해 테스트 실행.

### 이커머스 프로젝트에서의 활용
- `orders`와 `users` 테이블 간의 `@ManyToOne` 관계 테스트.
- `coupons` API가 Redis 캐시와 통합된 동작 확인.
- `products` 조회 성능 테스트.

---

## 2. Testcontainers 사용법 (Spring Boot + PostgreSQL 예제)

Spring Boot와 Testcontainers를 사용해 이커머스 프로젝트의 `orders` 테이블을 테스트하는 방법을 설명합니다.

### 2.1. 전제 조건
- **Java**: JDK 17 이상.
- **Spring Boot**: 3.3.2 (2025년 기준 최신).
- **Docker**: 로컬에 설치(Docker Desktop 권장).
- **Maven**: 빌드 도구.
- **IDE**: IntelliJ 또는 Eclipse.

### 2.2. Maven 의존성 추가
`pom.xml`에 Testcontainers와 관련 의존성 추가:
```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
        <version>3.3.2</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <version>3.3.2</version>
        <scope>test</scope>
    </dependency>
    <!-- PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.4</version>
    </dependency>
    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.20.2</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <version>1.20.2</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>1.20.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 2.3. 엔티티와 리포지토리 설정
`Order` 엔티티와 `OrderRepository`를 정의:
```java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import java.math.BigDecimal;

@Entity
public class Order {
    @Id
    private Long id;
    private Long userId;
    private BigDecimal totalAmount;
    // Getters and Setters
}
```

```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface OrderRepository extends JpaRepository<Order, Long> {
}
```

### 2.4. Testcontainers로 통합 테스트 작성
PostgreSQL 컨테이너를 사용해 `OrderRepository` 테스트:
```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import java.math.BigDecimal;
import static org.assertj.core.api.Assertions.assertThat;

@Testcontainers
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class OrderRepositoryTest {

    @Container
    private static final PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("test_db")
            .withUsername("postgres")
            .withPassword("postgres");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldSaveAndFindOrder() {
        Order order = new Order();
        order.setId(1L);
        order.setUserId(100L);
        order.setTotalAmount(new BigDecimal("99.99"));

        orderRepository.save(order);

        Order found = orderRepository.findById(1L).orElse(null);
        assertThat(found).isNotNull();
        assertThat(found.getUserId()).isEqualTo(100L);
        assertThat(found.getTotalAmount()).isEqualTo(new BigDecimal("99.99"));
    }
}
```
- **설명**:
  - `@Testcontainers`: Testcontainers를 활성화.
  - `@Container`: PostgreSQL 컨테이너 정의.
  - `@DynamicPropertySource`: Spring Boot의 데이터소스 설정을 동적으로 구성.
  - 테스트: `Order`를 저장하고 조회하여 데이터베이스 동작 확인.

### 2.5. 로컬 테스트 실행
1. Docker Desktop 실행.
2. `mvn test`로 테스트 실행.
3. 콘솔에서 PostgreSQL 컨테이너 생성/삭제 로그 확인.

---

## 3. GitHub Actions에서 Testcontainers로 자동 테스트 설정

GitHub Actions에서 Testcontainers 테스트를 실행하려면 Docker가 설치된 환경(예: `ubuntu-latest`)이 필요합니다. 이커머스 프로젝트의 `OrderRepositoryTest`를 CI 파이프라인에서 실행하는 방법을 설명합니다.

### 3.1. GitHub Actions 워크플로우 설정
`.github/workflows/ci.yml` 파일을 생성:
```yaml
name: CI Build
on:
  push:
    branches: ["**"]
  pull_request:
    branches: ["main"]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Java 17
        uses: actions/setup-java@v4
        with:
          java-version: "17"
          distribution: "temurin"
          cache: "maven"
      - name: Build and Test with Maven
        run: mvn -B verify
```
- **설명**:
  - `runs-on: ubuntu-latest`: Docker가 사전 설치된 환경 사용.[](https://stackoverflow.com/questions/66174071/github-actions-for-running-test-with-testcontainers-and-gradle)
  - `actions/checkout@v4`: 소스 코드 체크아웃.
  - `actions/setup-java@v4`: Java 17 환경 설정.
  - `mvn -B verify`: Maven으로 빌드 및 테스트 실행.

### 3.2. Testcontainers Cloud (선택)
Testcontainers Cloud를 사용하면 로컬 Docker 대신 클라우드에서 컨테이너를 실행해 CI 속도를 향상시킬 수 있습니다.[](https://www.docker.com/blog/running-testcontainers-tests-using-github-actions/)
1. Testcontainers Cloud에서 무료 계정 생성.
2. 서비스 토큰(TC_CLOUD_TOKEN) 발급.
3. GitHub Secrets에 `TC_CLOUD_TOKEN` 추가:
   - GitHub 리포지토리 > **Settings** > **Secrets and variables** > **Actions** > **New repository secret**.
4. 워크플로우에 환경 변수 추가:
```yaml
      - name: Build and Test with Maven
        env:
          TC_CLOUD_TOKEN: ${{ secrets.TC_CLOUD_TOKEN }}
        run: mvn -B verify
```

### 3.3. 테스트 결과 확인
- GitHub 리포지토리의 **Actions** 탭에서 워크플로우 실행 결과 확인.
- Testcontainers Cloud 대시보드(선택)에서 컨테이너 실행 세부 정보 확인.[](https://www.docker.com/blog/running-testcontainers-tests-using-github-actions/)

---

## 4. 이커머스 프로젝트에서의 Testcontainers 적용 예시

### 4.1. 시나리오: 주문 생성 API 테스트
- **API**: `POST /orders`
- **목표**: `orders` 테이블에 주문 저장 및 `users`와의 관계 확인.
- **테스트 코드**:
```java
@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class OrderApiTest {

    @Container
    private static final PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("test_db")
            .withUsername("postgres")
            .withPassword("postgres");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrder() {
        String requestBody = """
            {
                "userId": 100,
                "totalAmount": 99.99
            }
            """;

        ResponseEntity<Order> response = restTemplate.postForEntity("/orders", requestBody, Order.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getUserId()).isEqualTo(100L);
    }
}
```
- **설명**: `TestRestTemplate`으로 `POST /orders` 호출, PostgreSQL 컨테이너에서 데이터 저장 확인.

### 4.2. 시나리오: 쿠폰 캐싱 테스트
- **API**: `GET /coupons/{code}`
- **목표**: Redis 컨테이너로 캐싱 동작 테스트.
- **테스트 코드**:
```java
@Testcontainers
@SpringBootTest
public class CouponCacheTest {

    @Container
    private static final GenericContainer<?> redis = new GenericContainer<>("redis:7")
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", redis::getFirstMappedPort);
    }

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Test
    void shouldCacheCoupon() {
        redisTemplate.opsForValue().set("coupon:TEST10", "10%");
        String discount = redisTemplate.opsForValue().get("coupon:TEST10");
        assertThat(discount).isEqualTo("10%");
    }
}
```
- **설명**: Redis 컨테이너로 쿠폰 캐시 저장/조회 테스트.

---

## 5. GitHub Actions에서 주의사항

- **Docker 지원**: GitHub Actions의 `ubuntu-latest` 러너는 Docker가 사전 설치되어 있음.[](https://stackoverflow.com/questions/66174071/github-actions-for-running-test-with-testcontainers-and-gradle)
- **Windows 러너 주의**: Windows 러너는 Linux 컨테이너를 지원하지 않으므로 `ubuntu-latest` 사용 권장.[](https://dotnet.testcontainers.org/cicd/)
- **컨테이너 정리**: Testcontainers는 테스트 후 컨테이너를 자동 삭제하지만, CI에서 남은 컨테이너를 정리하려면 스크립트 추가 가능:
```yaml
      - name: Clean up Docker containers
        if: always()
        run: docker system prune -f
```
- **네트워크 설정**: 컨테이너 간 통신이 필요하면 동일 네트워크 설정(예: Docker Compose).[](https://dev.to/sahanonp/setup-docker-for-integration-testing-in-github-action-39fn)

---

## 6. 장점과 단점

### 6.1. 장점
- **실제 환경 테스트**: 모킹 없이 실제 PostgreSQL, Redis 등 사용.
- **자동화**: 컨테이너 생성/삭제 자동화로 테스트 환경 관리 간소화.
- **CI/CD 통합**: GitHub Actions에서 쉽게 실행.[](https://www.docker.com/blog/running-testcontainers-tests-using-github-actions/)
- **재현성**: 로컬과 CI 환경에서 동일한 테스트 환경 제공.

### 6.2. 단점
- **Docker 의존성**: 로컬과 CI 모두 Docker 설치 필요.
- **리소스 사용**: 컨테이너 실행으로 CPU/메모리 사용량 증가.
- **설정 복잡성**: 초보자에게 컨테이너 설정이 다소 복잡할 수 있음.

---

## 7. 학습자 팁

- **시작하기**:
  - 로컬에서 `mvn test`로 PostgreSQL 컨테이너 테스트 실행.
  - Docker Desktop에서 컨테이너 생성/삭제 확인.
- **실습 예제**:
  - `POST /orders` API 테스트 후 `orders` 테이블 데이터 확인.
  - Redis 컨테이너로 `coupons` 캐싱 테스트:
    ```java
    @Test
    void shouldCacheMultipleCoupons() {
        redisTemplate.opsForValue().set("coupon:TEST20", "20%");
        assertThat(redisTemplate.opsForValue().get("coupon:TEST20")).isEqualTo("20%");
    }
    ```
- **도구**: Docker Desktop, IntelliJ의 Testcontainers 플러그인 활용.
- **추천 학습 경로**:
  1. 로컬에서 Testcontainers로 PostgreSQL 테스트 설정.
  2. GitHub Actions 워크플로우 추가 및 테스트 실행.
  3. Testcontainers Cloud로 CI 성능 최적화.
  4. Redis, Kafka 등 다른 모듈 테스트 추가.

---

## 8. 요약

Testcontainers는 실제 서비스를 Docker 컨테이너로 실행하여 통합 테스트를 지원하는 강력한 도구입니다. 이커머스 프로젝트에서는 `orders`, `coupons`, `products` 관련 API를 실제 PostgreSQL, Redis 환경에서 테스트할 수 있습니다. Spring Boot와 결합해 로컬 테스트를 설정하고, GitHub Actions로 CI 파이프라인에서 자동 실행 가능합니다. 초보자는 간단한 `OrderRepositoryTest`부터 시작해, Redis 캐싱, 다중 컨테이너 테스트로 확장해보세요.

### 추가 참고 자료
- Testcontainers 공식 사이트: https://testcontainers.org
- Testcontainers Java 문서: https://java.testcontainers.org
- GitHub Actions 문서: https://docs.github.com/en/actions
- Baeldung Testcontainers 가이드: https://www.baeldung.com/spring-boot-testcontainers-integration-test
- Testcontainers Cloud: https://testcontainers.cloud[](https://www.docker.com/blog/running-testcontainers-tests-using-github-actions/)