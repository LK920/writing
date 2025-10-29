# LazyConnectionDataSourceProxy 학습 자료

이 자료는 Spring Framework의 `LazyConnectionDataSourceProxy`의 동작 방식, 사용 방법, 그리고 이로 해결할 수 있는 문제들을 체계적으로 설명합니다. `LazyConnectionDataSourceProxy`는 데이터베이스 연결을 지연(lazy) 방식으로 처리해 성능 최적화를 돕는 강력한 도구입니다. 이 자료를 통해 그 원리와 실제 적용 방법을 학습하고, 어떤 문제를 해결할 수 있는지 이해할 수 있습니다.

---

## 1. LazyConnectionDataSourceProxy란?

### 정의
- **LazyConnectionDataSourceProxy**는 Spring Framework의 `org.springframework.jdbc.datasource` 패키지에 포함된 클래스입니다.
- 실제 JDBC 연결(Connection)을 **필요한 시점에만** 가져오도록 프록시 패턴을 사용하는 DataSource 래퍼입니다.
- 기본 DataSource(예: HikariCP, BoneCP)를 감싸며, 트랜잭션 시작 시점이 아니라 실제 SQL 쿼리 실행 직전에 연결을 획득합니다.

### 핵심 개념
- **지연 연결(Lazy Fetching)**: 데이터베이스 연결은 리소스가 제한적이며 획득 비용이 큽니다. `LazyConnectionDataSourceProxy`는 트랜잭션 시작 시 연결을 바로 가져오지 않고, 실제로 데이터베이스 작업(예: `Statement` 생성)이 필요할 때까지 연결을 지연시킵니다.
- **프록시 패턴**: 실제 DataSource를 감싸는 프록시 객체로, `Connection` 요청을 가로채어 지연 로직을 적용합니다.
- **주요 목적**: 불필요한 연결 획득을 줄여 연결 풀(Connection Pool)의 효율성을 높이고, 트랜잭션 응답 시간을 단축합니다.

### 질문으로 확인
- 데이터베이스 연결을 "지연"시키는 게 왜 중요하다고 생각하나요?

---

## 2. 동작 방식

### 기본 동작 흐름
1. **트랜잭션 시작**:
   - `@Transactional` 메소드가 호출되면 Spring은 트랜잭션을 시작합니다.
   - 일반 DataSource는 트랜잭션 시작 즉시 연결을 획득하지만, `LazyConnectionDataSourceProxy`는 가상의 `ConnectionProxy` 객체를 반환합니다.
2. **연결 지연**:
   - `ConnectionProxy`는 실제 JDBC 연결을 아직 가져오지 않고, 트랜잭션 설정(예: `readOnly`, `transactionIsolation`)을 내부적으로 저장합니다.
   - `commit`, `rollback`, `setReadOnly` 같은 호출은 실제 연결 없이 처리됩니다.
3. **실제 연결 획득**:
   - SQL 실행을 위해 `Statement`, `PreparedStatement`, 또는 `CallableStatement`가 생성될 때 `getTargetConnection` 메소드를 통해 실제 JDBC 연결을 가져옵니다.
   - 저장된 트랜잭션 설정(예: `readOnly=true`)이 실제 연결에 적용됩니다.
4. **연결 반환**:
   - 트 Verma가 종료되면 연결이 풀에 반환됩니다.

### 코드로 이해하기
```java
@Service
@Transactional
public class LazyService {
    @Autowired
    private DataSource dataSource;

    public void lazyMethod() {
        // 트랜잭션 시작, LazyConnectionDataSourceProxy는 실제 연결을 아직 안 가져옴
        System.out.println("비즈니스 로직 시작");
        // 외부 서비스 호출 (예: 외부 API 호출, 500ms 소요)
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 실제 쿼리 실행 시점에 연결 획득
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement("SELECT * FROM product")) {
            ResultSet rs = stmt.executeQuery();
            while (rs.next()) {
                System.out.println("데이터 조회: " + rs.getString("name"));
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```
- **설명**:
  - `dataSource.getConnection()`은 `LazyConnectionDataSourceProxy`가 제공하는 `ConnectionProxy`를 반환.
  - 실제 연결은 `stmt.executeQuery()`가 호출될 때 획득됨.
  - 외부 API 호출(500ms) 동안 연결 풀이 점유되지 않음.

### 비유
- **비유**: `LazyConnectionDataSourceProxy`는 식당에서 주문서를 들고 있는 웨이터와 같습니다. 손님이 주문(트랜잭션 시작)을 해도 바로 요리(연결 획득)를 시작하지 않고, 손님이 음식을 달라고 요청(쿼리 실행)할 때만 주방에 전달합니다. 이렇게 하면 주방(연결 풀)의 부담을 줄입니다.

### Spring과의 연계
- `LazyConnectionDataSourceProxy`는 `DataSourceTransactionManager`나 `HibernateTransactionManager`와 잘 동작합니다.
- JTA(`JtaTransactionManager`) 환경에서는 이미 지연 연결이 기본이므로 추가적인 이점이 없음.[](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)

### 질문으로 확인
- 트랜잭션 시작과 실제 쿼리 실행 사이에 어떤 작업이 있다면 연결을 지연시키는 게 왜 유리할까?

---

## 3. 사용 방법

### 설정 방법
`LazyConnectionDataSourceProxy`를 사용하려면 실제 DataSource를 래핑하도록 Spring Bean을 설정합니다. 아래는 Spring Boot와 HikariCP를 사용한 예시입니다.

#### XML 설정
```xml
<bean id="hikariDataSource" class="com.zaxxer.hikari.HikariDataSource">
    <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/mydb"/>
    <property name="username" value="root"/>
    <property name="password" value="password"/>
    <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
</bean>

<bean id="dataSource" class="org.springframework.jdbc.datasource.LazyConnectionDataSourceProxy">
    <property name="targetDataSource" ref="hikariDataSource"/>
</bean>

<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

#### Java Config
```java
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("root");
        config.setPassword("password");
        config.setDriverClassName("com.mysql.cj.jdbc.Driver");
        HikariDataSource hikariDataSource = new HikariDataSource(config);
        return new LazyConnectionDataSourceProxy(hikariDataSource);
    }

    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

#### Spring Boot (application.properties)
Spring Boot에서는 `spring.datasource` 설정을 사용해 HikariCP를 기본적으로 구성한 후, `LazyConnectionDataSourceProxy`로 래핑합니다.
```java
@Configuration
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    public HikariDataSource hikariDataSource() {
        return new HikariDataSource();
    }

    @Bean
    public DataSource dataSource(HikariDataSource hikariDataSource) {
        return new LazyConnectionDataSourceProxy(hikariDataSource);
    }
}
```
- **참고**: Spring Boot에서 `DataSourceAutoConfiguration`을 비활성화하고 수동 설정해야 할 수 있음.[](https://github.com/spring-projects/spring-boot/issues/15480)

### 주요 메소드
- `checkDefaultConnectionProperties()`: 시작 시 연결 속성(예: auto-commit, isolation level)을 확인. 기본적으로 첫 연결 획득 시 설정되지만, 이 메소드를 호출하면 시작 시 강제로 확인.[](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)
- `getTargetConnection(Method)`: 실제 JDBC 연결을 가져오는 내부 메소드. 쿼리 실행 시 호출됨.[](https://github.com/spring-projects/spring-framework/blob/main/spring-jdbc/src/main/java/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.java)
- `setReadOnlyDataSource(DataSource)`: 읽기 전용 트랜잭션에 사용할 별도의 DataSource 지정 가능.[](https://docs.spring.io/spring-framework/docs/current-SNAPSHOT/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)

### 주의점
- **ConnectionProxy**: `LazyConnectionDataSourceProxy`는 `ConnectionProxy`를 반환하므로, `OracleConnection` 같은 네이티브 연결로 캐스팅 불가. `Wrapper.unwrap()` 사용 필요.[](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)
- **HikariCP 설정**: HikariCP의 `autoCommit`과 같은 설정이 `LazyConnectionDataSourceProxy`로 래핑되므로, `HikariConfig`에 명시적으로 설정해야 함.[](https://github.com/spring-projects/spring-boot/issues/15480)

### 질문으로 확인
- 위 설정에서 `LazyConnectionDataSourceProxy`를 DataSource로 래핑하는 이유는 뭘까?

---

## 4. 해결하는 문제들

`LazyConnectionDataSourceProxy`는 다양한 성능 및 리소스 문제를 해결합니다. 아래는 주요 문제와 해결 사례입니다.

### (1) 연결 풀 고갈
- **문제**: 트랜잭션 시작 시 즉시 연결을 획득하면, 연결 풀이 빠르게 고갈될 수 있음. 특히 고트래픽 애플리케이션에서 읽기 전용 트랜잭션이 많거나, 트랜잭션 내에 외부 서비스 호출(예: API 호출)이 포함된 경우.
- **해결**:
  - `LazyConnectionDataSourceProxy`는 쿼리가 실행될 때까지 연결을 지연시켜, 연결 풀 점유 시간을 최소화.
  - 예: 외부 API 호출(500ms) 후 쿼리를 실행하는 메소드에서, 연결은 API 호출 동안 점유되지 않음.[](https://vladmihalcea.com/lazyconnectiondatasourceproxy-spring-data-jpa/)
- **실제 사례**:
  - 전자상거래 시스템에서 상품 목록 조회 API가 Redis 캐시를 먼저 확인하고, 캐시에 없으면 DB 쿼리를 실행. 캐시 히트 시 연결을 전혀 사용하지 않음.[](https://eunajung01.tistory.com/171)

### (2) 트랜잭션 응답 시간 증가
- **문제**: 트랜잭션 시작부터 종료까지 연결을 점유하면, 외부 작업(예: 파일 업로드, 외부 API 호출)으로 인해 응답 시간이 길어짐.
- **해결**:
  - 연결을 쿼리 실행 직전에 획득해 트랜잭션 응답 시간을 단축.
  - 예: `@Transactional(readOnly=true)` 메소드에서 외부 환율 서비스 호출(750ms) 후 DB 조회 시, 연결은 조회 직전에 획득.[](https://vladmihalcea.com/lazyconnectiondatasourceproxy-spring-data-jpa/)
- **실제 사례**:
  - 금융 시스템에서 환율 변환 후 상품 조회 시, 환율 API 호출 동안 연결을 점유하지 않아 동시 트랜잭션 처리량 증가.[](https://vladmihalcea.com/lazyconnectiondatasourceproxy-spring-data-jpa/)

### (3) 2차 캐시 활용 시 비효율
- **문제**: Hibernate의 2차 캐시를 사용하면 DB 조회 없이 캐시에서 데이터를 가져올 수 있지만, 일반 DataSource는 트랜잭션 시작 시 연결을 획득해 불필요한 리소스 낭비.
- **해결**:
  - `LazyConnectionDataSourceProxy`는 2차 캐시 히트 시 연결을 전혀 획득하지 않음.
  - 예: Hibernate 2차 캐시로 상품 데이터를 조회하면, DB 연결 없이 데이터를 반환.[](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)[](https://kimcoder.tistory.com/550)
- **실제 사례**:
  - 전자상거래 시스템에서 자주 조회되는 상품 목록을 2차 캐시에 저장, 연결 획득 없이 응답 처리.[](https://kimcoder.tistory.com/550)

### (4) MySQL Replication에서 DataSource 분기 실패
- **문제**: MySQL Replication 환경에서 읽기 전용 트랜잭션(`@Transactional(readOnly=true)`)은 슬레이브 DB로 라우팅해야 하지만, 트랜잭션 시작 시 연결을 획득하면 `AbstractRoutingDataSource`의 라우팅 로직이 올바르게 동작하지 않을 수 있음.
- **해결**:
  - `LazyConnectionDataSourceProxy`는 쿼리 실행 시점에 `AbstractRoutingDataSource`의 `determineCurrentLookupKey()`를 호출해 올바른 DataSource(마스터/슬레이브)를 선택.
  - 예: `readOnly=true`인 트랜잭션은 슬레이브 DB로 라우팅.[](https://kookiencream.tistory.com/135)[](https://chagokx2.tistory.com/103)
- **실제 사례**:
  - 대규모 웹 서비스에서 읽기 트래픽을 슬레이브 DB로 분산, 연결 지연으로 라우팅 정확도 향상.[](https://chagokx2.tistory.com/103)

### (5) 불필요한 커밋/롤백 오버헤드
- **문제**: 쿼리가 실행되지 않은 트랜잭션에서 `commit`이나 `rollback` 호출은 불필요한 오버헤드를 유발.
- **해결**:
  - `LazyConnectionDataSourceProxy`는 쿼리가 실행되지 않으면 `commit`/`rollback`을 무시해 오버헤드 감소.[](https://docs.spring.io/spring-framework/docs/3.0.6.RELEASE_to_3.1.0.BUILD-SNAPSHOT/3.0.6.RELEASE/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)
- **실제 사례**:
  - 읽기 전용 트랜잭션에서 데이터 조회 없이 종료 시, 불필요한 롤백 호출 방지.[](https://docs.spring.io/spring-framework/docs/current-SNAPSHOT/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)

### 질문으로 확인
- 위 문제들 중 당신의 프로젝트에서 가장 심각할 것 같은 문제는 뭔가요?

---

## 5. 실제 사용 예시

### 예시 1: 읽기 전용 트랜잭션 최적화
```java
@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository productRepository;

    @Transactional(readOnly = true)
    public Product getProduct(Long productId) {
        // 외부 API 호출 (예: 500ms 소요)
        Thread.sleep(500);
        // DB 조회
        return productRepository.findById(productId).orElseThrow();
    }
}
```
- **문제**: 일반 DataSource는 트랜잭션 시작 시 연결을 획득, 외부 API 호출 동안 연결 점유.
- **해결**: `LazyConnectionDataSourceProxy`로 래핑하면, `findById` 호출 시에만 연결 획득.

### 예시 2: MySQL Replication
```java
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource routingDataSource() {
        AbstractRoutingDataSource routingDataSource = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "slave" : "master";
            }
        };
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("master", masterDataSource());
        targetDataSources.put("slave", slaveDataSource());
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        return new LazyConnectionDataSourceProxy(routingDataSource);
    }
}
```
- **설명**: 읽기 전용 트랜잭션은 슬레이브 DB로, 쓰기 트랜잭션은 마스터 DB로 라우팅. `LazyConnectionDataSourceProxy`가 연결 지연으로 라우팅 정확도 보장.[](https://chagokx2.tistory.com/103)

---

## 6. 장점과 한계

### 장점
- **연결 풀 효율성**: 불필요한 연결 점유 감소로 고트래픽 환경에서 성능 향상.[](https://github.com/spring-projects/spring-boot/issues/15480)
- **트랜잭션 응답 시간 단축**: 외부 작업 동안 연결 점유를 방지.[](https://vladmihalcea.com/lazyconnectiondatasourceproxy-spring-data-jpa/)
- **2차 캐시 활용**: DB 연결 없이 캐시 데이터 반환 가능.[](https://kimcoder.tistory.com/550)
- **MySQL Replication 호환**: `AbstractRoutingDataSource`와 함께 사용해 라우팅 정확도 향상.[](https://kookiencream.tistory.com/135)

### 한계
- **설정 복잡성**: Spring Boot에서 자동 설정을 비활성화하고 수동 설정 필요.[](https://github.com/spring-projects/spring-boot/issues/15480)
- **네이티브 연결 제한**: `ConnectionProxy` 반환으로 네이티브 연결 캐스팅 불가.[](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)
- **JTA 환경 비효율**: JTA는 이미 지연 연결을 지원하므로 추가 이점 없음.[](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)

---

## 7. 질문으로 이해도 확인
1. `LazyConnectionDataSourceProxy`가 일반 DataSource와 다른 점은 뭔가요?
2. 어떤 상황에서 `LazyConnectionDataSourceProxy`가 가장 큰 이점을 줄 것 같나요?
3. MySQL Replication 환경에서 `LazyConnectionDataSourceProxy`를 왜 사용하는 걸까?
4. 프로젝트에서 연결 풀 고갈 문제를 겪은 적이 있다면, 어떻게 해결했나요?

---

## 8. 추가 학습 제안
- **Spring 문서**: `LazyConnectionDataSourceProxy` 공식 문서 읽기.[](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/LazyConnectionDataSourceProxy.html)
- **실습**: Spring Boot 프로젝트에서 `LazyConnectionDataSourceProxy`를 설정하고, 쿼리 실행 전.Добавить

System: ### 질문으로 확인
1. `LazyConnectionDataSourceProxy`가 일반 DataSource와 다른 점은 뭔가요?
2. 어떤 상황에서 `LazyConnectionDataSourceProxy`가 가장 큰 이점을 줄 것 같나요?
3. MySQL Replication 환경에서 `LazyConnectionDataSourceProxy`를 왜 사용하는 걸까?
4. 프로젝트에서 연결 풀 고갈 문제를 겪은 적이 있다면, 어떻게 해결했나요?

---

이 자료로 `LazyConnectionDataSourceProxy`의 동작 방식과 사용 방법, 해결할 수 있는 문제들에 대해 충분히 이해가 되셨나요? 특정 부분(예: 설정 코드, MySQL Replication 사례, 성능 테스트)을 더 깊이 다루고 싶거나 추가 질문이 있으면 말씀해주세요!