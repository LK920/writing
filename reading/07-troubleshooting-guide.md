# 🔧 트러블슈팅 가이드

## 📋 목차

1. [일반적인 문제 해결](#일반적인-문제-해결)
2. [동시성 관련 문제](#동시성-관련-문제)
3. [API 오류 해결](#api-오류-해결)
4. [메모리 및 성능 문제](#메모리-및-성능-문제)
5. [테스트 실행 문제](#테스트-실행-문제)
6. [빌드 및 배포 문제](#빌드-및-배포-문제)
7. [모니터링 및 로깅](#모니터링-및-로깅)
8. [디버깅 도구 및 기법](#디버깅-도구-및-기법)

## 일반적인 문제 해결

### 🚀 애플리케이션 시작 오류

#### 1. Port Already in Use
```bash
# 증상
***************************
APPLICATION FAILED TO START
***************************

Description:
Web server failed to start. Port 8080 was already in use.

# 해결 방법
# 1) 실행 중인 프로세스 확인 및 종료
lsof -ti:8080 | xargs kill -9

# 2) 다른 포트 사용
./gradlew bootRun --args='--server.port=8081'

# 3) application.yml 설정 변경
server:
  port: 8081
```

#### 2. Bean Creation Exception
```bash
# 증상
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: 
No qualifying bean of type 'kr.hhplus.be.server.domain.port.storage.UserRepositoryPort' available

# 원인: 구현체에 @Repository 어노테이션 누락

# 해결 방법
@Repository  // 이 어노테이션 추가
public class InMemoryUserRepository implements UserRepositoryPort {
    // 구현...
}
```

#### 3. Circular Dependency
```bash
# 증상
The dependencies of some of the beans in the application context form a cycle

# 원인: UseCase 간 순환 의존성

# 해결 방법
// ❌ 나쁜 예: UseCase 간 직접 의존
@Component
public class OrderUseCase {
    private final PaymentUseCase paymentUseCase;  // 순환 의존 가능성
}

// ✅ 좋은 예: 포트 인터페이스를 통한 의존
@Component  
public class OrderUseCase {
    private final PaymentPort paymentPort;  // 인터페이스 의존
}
```

### 🗃️ 데이터 관련 문제

#### 1. ConcurrentModificationException
```java
// 증상
java.util.ConcurrentModificationException
    at java.util.HashMap$HashIterator.nextNode(HashMap.java:1445)

// 원인: ConcurrentHashMap 대신 HashMap 사용

// 해결 방법
// ❌ 나쁜 예
private final Map<Long, User> users = new HashMap<>();

// ✅ 좋은 예  
private final Map<Long, User> users = new ConcurrentHashMap<>();
```

#### 2. NullPointerException
```java
// 증상
java.lang.NullPointerException
    at kr.hhplus.be.server.domain.usecase.balance.ChargeBalanceUseCase.execute

// 일반적인 원인들과 해결 방법

// 1) Optional 잘못 사용
// ❌ 나쁜 예
User user = userRepository.findById(userId).get();  // NPE 위험

// ✅ 좋은 예
User user = userRepository.findById(userId)
    .orElseThrow(() -> new UserException.NotFound());

// 2) Builder 패턴에서 필수 필드 누락
// ❌ 나쁜 예
Balance balance = Balance.builder()
    .amount(amount)
    // user 필드 누락
    .build();

// ✅ 좋은 예
Balance balance = Balance.builder()
    .user(user)
    .amount(amount)
    .build();
```

## 동시성 관련 문제

### ⚡ 락 관련 문제

#### 1. 데드락 상황
```java
// 증상: 애플리케이션이 멈춤, 로그에 데드락 관련 에러

// 원인: 여러 락을 다른 순서로 획득
// Thread 1: 락A → 락B
// Thread 2: 락B → 락A

// 해결 방법: 락 순서 통일
@Component
public class OrderUseCase {
    
    @Transactional
    public Order execute(Long userId, Map<Long, Integer> productQuantities) {
        // 락 키를 정렬하여 항상 같은 순서로 획득
        List<String> lockKeys = new ArrayList<>();
        lockKeys.add("balance-" + userId);
        lockKeys.add("order-creation-" + userId);
        lockKeys.sort(String::compareTo);  // 순서 통일
        
        // 순서대로 락 획득
        for (String lockKey : lockKeys) {
            if (!lockingPort.acquireLock(lockKey)) {
                // 이미 획득한 락들 해제
                releasePreviousLocks(lockKeys, lockKey);
                throw new ConcurrencyException.LockAcquisitionFailed();
            }
        }
        
        try {
            // 비즈니스 로직 수행
        } finally {
            // 역순으로 락 해제
            Collections.reverse(lockKeys);
            lockKeys.forEach(lockingPort::releaseLock);
        }
    }
}
```

#### 2. 락 타임아웃
```java
// 증상
kr.hhplus.be.server.domain.exception.ConcurrencyException$LockAcquisitionFailed

// 원인: 락 대기 시간 초과

// 해결 방법 1: 타임아웃 시간 조정
@Component
public class InMemoryLockingAdapter implements LockingPort {
    
    private static final Duration LOCK_TIMEOUT = Duration.ofSeconds(10);  // 타임아웃 증가
    
    @Override
    public boolean acquireLock(String lockKey) {
        return locks.computeIfAbsent(lockKey, k -> new ReentrantLock())
            .tryLock(LOCK_TIMEOUT.toMillis(), TimeUnit.MILLISECONDS);
    }
}

// 해결 방법 2: 재시도 로직 추가
@Component
public class ChargeBalanceUseCase {
    
    @Retryable(value = ConcurrencyException.class, maxAttempts = 3, delay = 1000)
    public Balance execute(Long userId, BigDecimal amount) {
        // 락 획득 및 비즈니스 로직
    }
}
```

#### 3. 메모리 누수 (락 미해제)
```java
// 증상: 메모리 사용량 계속 증가

// 원인: 락이 해제되지 않아 메모리에 계속 누적

// 해결 방법: try-finally 패턴 철저히 적용
@Component
public class BaseUseCase {
    
    protected <T> T executeWithLock(String lockKey, Supplier<T> operation) {
        if (!lockingPort.acquireLock(lockKey)) {
            throw new ConcurrencyException.LockAcquisitionFailed();
        }
        
        try {
            return operation.get();
        } finally {
            lockingPort.releaseLock(lockKey);  // 반드시 해제
        }
    }
}

// 사용 예시
public Balance execute(Long userId, BigDecimal amount) {
    return executeWithLock("balance-" + userId, () -> {
        // 비즈니스 로직
        return processBalanceCharge(userId, amount);
    });
}
```

### 🔄 동시성 테스트 실패

#### 1. 테스트 시간 초과
```java
// 증상: 동시성 테스트에서 timeout 발생

// 해결 방법: 적절한 대기 시간 설정
@Test
void concurrencyTest() throws InterruptedException {
    int numberOfThreads = 10;
    ExecutorService executor = Executors.newFixedThreadPool(numberOfThreads);
    CountDownLatch doneLatch = new CountDownLatch(numberOfThreads);
    
    // 작업 실행...
    
    // 충분한 대기 시간 설정 (기본값의 2-3배)
    boolean finished = doneLatch.await(30, TimeUnit.SECONDS);  // 10초 → 30초
    assertThat(finished).isTrue();
    
    executor.shutdown();
    boolean terminated = executor.awaitTermination(10, TimeUnit.SECONDS);
    assertThat(terminated).isTrue();
}
```

#### 2. 비결정적 테스트 결과
```java
// 증상: 같은 테스트가 때로는 성공, 때로는 실패

// 원인: 스레드 실행 순서에 의존

// 해결 방법: CountDownLatch로 동기화
@Test
void concurrentTest() throws InterruptedException {
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(numberOfThreads);
    
    for (int i = 0; i < numberOfThreads; i++) {
        executor.submit(() -> {
            try {
                startLatch.await();  // 모든 스레드가 동시에 시작
                // 테스트 로직
            } finally {
                doneLatch.countDown();
            }
        });
    }
    
    startLatch.countDown();  // 모든 스레드 동시 시작
    doneLatch.await();
}
```

## API 오류 해결

### 🌐 HTTP 관련 오류

#### 1. 404 Not Found
```bash
# 증상
{
  "code": "404",
  "message": "Not Found",
  "timestamp": "2024-01-01T12:00:00"
}

# 원인과 해결 방법

# 1) 잘못된 URL 경로
# 확인: @RequestMapping, @PostMapping 경로
@RestController
@RequestMapping("/api/balance")  # 경로 확인
public class BalanceController {
    
    @PostMapping("/charge")  # 실제 경로: /api/balance/charge
    public BalanceResponse chargeBalance() { }
}

# 2) HTTP 메서드 불일치
# POST로 정의된 API를 GET으로 호출
curl -X GET http://localhost:8080/api/balance/charge  # 잘못된 메서드

# 올바른 호출
curl -X POST http://localhost:8080/api/balance/charge \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "amount": 10000}'
```

#### 2. 400 Bad Request
```bash
# 증상  
{
  "code": "C001",
  "message": "입력값이 유효하지 않습니다",
  "timestamp": "2024-01-01T12:00:00"
}

# 원인과 해결 방법

# 1) JSON 형식 오류
# ❌ 잘못된 JSON
curl -X POST http://localhost:8080/api/balance/charge \
  -H "Content-Type: application/json" \
  -d '{userId: 1, amount: 10000}'  # 키에 따옴표 없음

# ✅ 올바른 JSON
curl -X POST http://localhost:8080/api/balance/charge \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "amount": 10000}'

# 2) 타입 불일치
# 요청: "amount": "10000" (문자열)
# 기대: "amount": 10000 (숫자)

# 3) 필수 필드 누락
{
  "userId": 1
  // amount 필드 누락
}
```

#### 3. 500 Internal Server Error
```bash
# 증상
{
  "code": "S500", 
  "message": "서버 내부 오류가 발생했습니다",
  "timestamp": "2024-01-01T12:00:00"
}

# 디버깅 방법

# 1) 로그 확인
tail -f logs/application.log

# 2) 스택 트레이스 분석
ERROR [http-nio-8080-exec-1] kr.hhplus.be.server.api.GlobalExceptionHandler : Unexpected exception occurred
java.lang.NullPointerException: Cannot invoke "getId()" because "user" is null
    at kr.hhplus.be.server.domain.usecase.balance.ChargeBalanceUseCase.execute(ChargeBalanceUseCase.java:45)

# 3) 단위 테스트로 재현
@Test
void reproduceIssue() {
    // 실패 상황을 단위 테스트로 재현
    assertThatThrownBy(() -> chargeBalanceUseCase.execute(null, amount))
        .isInstanceOf(NullPointerException.class);
}
```

### 🔍 디버깅 기법

#### 1. 로그 기반 디버깅
```java
// 로그 레벨 조정 (application.yml)
logging:
  level:
    kr.hhplus.be.server: DEBUG  # 프로젝트 패키지 DEBUG 레벨
    org.springframework.web: DEBUG  # Spring Web DEBUG
    org.springframework.transaction: DEBUG  # 트랜잭션 DEBUG

// 전략적 로그 추가
@Component
@RequiredArgsConstructor
@Slf4j
public class ChargeBalanceUseCase {
    
    public Balance execute(Long userId, BigDecimal amount) {
        log.debug("잔액 충전 시작: userId={}, amount={}", userId, amount);
        
        try {
            User user = userRepositoryPort.findById(userId)
                .orElseThrow(() -> new UserException.NotFound());
            log.debug("사용자 조회 완료: {}", user.getName());
            
            Balance balance = processCharge(user, amount);
            log.debug("잔액 충전 완료: 새 잔액={}", balance.getAmount());
            
            return balance;
        } catch (Exception e) {
            log.error("잔액 충전 실패: userId={}, amount={}, error={}", userId, amount, e.getMessage(), e);
            throw e;
        }
    }
}
```

#### 2. 조건부 중단점
```java
// IntelliJ IDEA에서 조건부 중단점 설정
// 중단점 우클릭 → More → Condition 입력
// userId.equals(1L) && amount.compareTo(BigDecimal.valueOf(50000)) > 0

public Balance execute(Long userId, BigDecimal amount) {
    // 특정 조건에서만 중단점 작동
    User user = userRepositoryPort.findById(userId)
        .orElseThrow(() -> new UserException.NotFound());
    
    return processCharge(user, amount);
}
```

## 메모리 및 성능 문제

### 💾 메모리 누수

#### 1. InMemory Repository 메모리 누수
```java
// 문제: 삭제된 데이터가 메모리에 남아있음

// 해결 방법: 명시적 정리 메서드 추가
@Repository
public class InMemoryUserRepository implements UserRepositoryPort {
    
    private final Map<Long, User> users = new ConcurrentHashMap<>();
    
    @Override
    public void deleteById(Long id) {
        users.remove(id);  // 메모리에서 완전 제거
    }
    
    // 테스트용 정리 메서드
    public void clear() {
        users.clear();
    }
    
    // 메모리 사용량 모니터링
    public int size() {
        return users.size();
    }
}
```

#### 2. 스레드 풀 정리
```java
// 문제: ExecutorService가 종료되지 않음

// 해결 방법: try-with-resources 또는 명시적 종료
public void concurrentProcess() {
    ExecutorService executor = Executors.newFixedThreadPool(10);
    
    try {
        // 작업 실행
        List<CompletableFuture<Void>> futures = new ArrayList<>();
        // ...
        
        // 모든 작업 완료 대기
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
    } finally {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### ⚡ 성능 문제

#### 1. 느린 API 응답
```bash
# 증상: API 응답 시간이 5초 이상

# 원인 분석 도구

# 1) 애플리케이션 로그에서 실행 시간 확인
2024-01-01 12:00:00.123 INFO  --- API 호출: BalanceController.chargeBalance - 응답시간: 5234ms

# 2) JVM 프로파일링
# JProfiler, VisualVM 등 사용

# 3) 스프링 액추에이터 활용
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  endpoint:
    metrics:
      enabled: true

# 메트릭 확인
curl http://localhost:8080/actuator/metrics/http.server.requests
```

#### 2. N+1 쿼리 문제 (향후 JPA 전환 시 대비)
```java
// 문제 상황: 주문 조회 시 각 상품을 개별 조회

// ❌ 나쁜 예
public OrderResponse getOrder(Long orderId) {
    Order order = orderRepository.findById(orderId);
    
    List<OrderItemResponse> items = order.getItems().stream()
        .map(item -> {
            Product product = productRepository.findById(item.getProductId());  // N+1 발생
            return OrderItemResponse.from(item, product);
        })
        .collect(Collectors.toList());
    
    return new OrderResponse(order, items);
}

// ✅ 좋은 예 (현재 InMemory 환경)
public OrderResponse getOrder(Long orderId) {
    Order order = orderRepository.findById(orderId);
    
    // 한번에 모든 상품 ID 수집
    Set<Long> productIds = order.getItems().stream()
        .map(OrderItem::getProductId)
        .collect(Collectors.toSet());
    
    // 배치로 상품 조회
    Map<Long, Product> productMap = productRepository.findByIds(productIds)
        .stream()
        .collect(Collectors.toMap(Product::getId, p -> p));
    
    List<OrderItemResponse> items = order.getItems().stream()
        .map(item -> OrderItemResponse.from(item, productMap.get(item.getProductId())))
        .collect(Collectors.toList());
    
    return new OrderResponse(order, items);
}
```

## 테스트 실행 문제

### 🧪 테스트 실패 원인

#### 1. 테스트 간 상태 공유
```java
// 문제: 테스트가 순서에 따라 성공/실패

// 원인: static 필드나 공유 리소스 사용
public class BalanceServiceTest {
    
    private static InMemoryBalanceRepository repository = new InMemoryBalanceRepository();  // 문제
    
    @Test
    void test1() {
        repository.save(testBalance);  // 데이터 저장
    }
    
    @Test  
    void test2() {
        // test1의 데이터가 남아있어서 예상과 다른 결과
    }
}

// 해결 방법: 각 테스트마다 새로운 인스턴스
public class BalanceServiceTest {
    
    private InMemoryBalanceRepository repository;
    
    @BeforeEach
    void setUp() {
        repository = new InMemoryBalanceRepository();  // 매번 새로운 인스턴스
    }
}
```

#### 2. Mock 설정 오류
```java
// 문제: Mock이 예상대로 동작하지 않음

// 원인 1: 잘못된 매처 사용
when(userRepository.findById(any())).thenReturn(Optional.of(user));
// 하지만 실제 호출: findById(1L)에서 다른 매처 사용

// 해결: 구체적인 값 또는 일관된 매처 사용
when(userRepository.findById(1L)).thenReturn(Optional.of(user));
// 또는
when(userRepository.findById(any(Long.class))).thenReturn(Optional.of(user));

// 원인 2: void 메서드 Mock 설정 오류
when(lockingPort.releaseLock(anyString())).thenReturn(void);  // 컴파일 에러

// 해결: doNothing() 사용
doNothing().when(lockingPort).releaseLock(anyString());
```

#### 3. 동시성 테스트 불안정
```java
// 문제: 동시성 테스트가 간헐적으로 실패

// 해결 방법: 더 안정적인 동기화
@Test
void concurrentTest() throws InterruptedException {
    int numberOfThreads = 10;
    ExecutorService executor = Executors.newFixedThreadPool(numberOfThreads);
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(numberOfThreads);
    
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger errorCount = new AtomicInteger(0);
    
    for (int i = 0; i < numberOfThreads; i++) {
        executor.submit(() -> {
            try {
                startLatch.await();  // 동시 시작 보장
                
                // 테스트 로직
                successCount.incrementAndGet();
                
            } catch (Exception e) {
                errorCount.incrementAndGet();
            } finally {
                doneLatch.countDown();
            }
        });
    }
    
    startLatch.countDown();
    
    // 충분한 대기 시간
    boolean finished = doneLatch.await(30, TimeUnit.SECONDS);
    assertThat(finished).isTrue();
    
    // 결과 검증
    assertThat(successCount.get() + errorCount.get()).isEqualTo(numberOfThreads);
    
    executor.shutdown();
}
```

## 빌드 및 배포 문제

### 🔨 Gradle 빌드 오류

#### 1. 의존성 충돌
```bash
# 증상
> Could not resolve all files for configuration ':compileClasspath'.
   > Could not resolve org.springframework:spring-web:5.3.21.
     Required by:
         project : > org.springframework.boot:spring-boot-starter-web:2.7.0

# 해결 방법: 의존성 트리 확인
./gradlew dependencies --configuration compileClasspath

# 특정 버전 강제 지정
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework', module: 'spring-web'
    }
    implementation 'org.springframework:spring-web:5.3.23'  // 특정 버전
}
```

#### 2. 테스트 실행 실패
```bash
# 증상
> Task :test FAILED

# 원인: 테스트 환경 설정 문제

# 해결 방법: 테스트 프로파일 설정
# src/test/resources/application-test.yml
spring:
  profiles:
    active: test
logging:
  level:
    kr.hhplus.be.server: DEBUG

# Gradle에서 테스트 프로파일 적용
test {
    systemProperty 'spring.profiles.active', 'test'
    jvmArgs '-Xmx2g'  // 메모리 부족 시
}
```

#### 3. 컴파일 오류
```bash
# 증상
error: cannot find symbol
  symbol:   method getMessage()
  location: variable errorCode of type ErrorCode

# 원인: ErrorCode enum에 getMessage() 메서드 없음

# 해결: 메서드 구현 확인
public enum ErrorCode {
    SUCCESS("S001", "성공");
    
    private final String code;
    private final String message;
    
    public String getMessage() {  // 이 메서드가 있는지 확인
        return message;
    }
}
```

### 🚀 배포 관련 문제

#### 1. 환경 변수 설정
```bash
# 프로덕션 환경 설정
export SPRING_PROFILES_ACTIVE=prod
export SERVER_PORT=8080
export LOG_LEVEL=INFO

# Docker 환경에서
docker run -e SPRING_PROFILES_ACTIVE=prod \
           -e SERVER_PORT=8080 \
           -p 8080:8080 \
           myapp:latest
```

#### 2. 헬스 체크 설정
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: when-authorized

# 헬스 체크 엔드포인트 테스트
curl http://localhost:8080/actuator/health
```

## 모니터링 및 로깅

### 📊 로그 분석

#### 1. 로그 레벨 조정
```yaml
# application.yml
logging:
  level:
    root: INFO
    kr.hhplus.be.server: DEBUG
    org.springframework.web: DEBUG
    org.springframework.transaction: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/application.log
    max-size: 100MB
    max-history: 30
```

#### 2. 구조화된 로깅
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class ChargeBalanceUseCase {
    
    public Balance execute(Long userId, BigDecimal amount) {
        // 구조화된 로그 (JSON 형태로 출력 가능)
        log.info("balance.charge.started", 
            kv("userId", userId),
            kv("amount", amount),
            kv("timestamp", Instant.now()));
        
        try {
            Balance result = processCharge(userId, amount);
            
            log.info("balance.charge.completed",
                kv("userId", userId),
                kv("newBalance", result.getAmount()),
                kv("duration", getDuration()));
            
            return result;
        } catch (Exception e) {
            log.error("balance.charge.failed",
                kv("userId", userId),
                kv("amount", amount),
                kv("error", e.getMessage()));
            throw e;
        }
    }
}
```

### 📈 메트릭 수집

#### 1. Spring Boot Actuator
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,info
  metrics:
    export:
      prometheus:
        enabled: true
  endpoint:
    metrics:
      enabled: true
```

#### 2. 커스텀 메트릭
```java
@Component
@RequiredArgsConstructor
public class MetricService {
    
    private final MeterRegistry meterRegistry;
    
    // 카운터 메트릭
    public void incrementBalanceChargeCount(String status) {
        Counter.builder("balance.charge.count")
            .tag("status", status)
            .register(meterRegistry)
            .increment();
    }
    
    // 타이머 메트릭
    public void recordBalanceChargeTime(Duration duration) {
        Timer.builder("balance.charge.duration")
            .register(meterRegistry)
            .record(duration);
    }
    
    // 게이지 메트릭 (현재 값)
    public void updateActiveUserCount(int count) {
        Gauge.builder("users.active.count")
            .register(meterRegistry, count, Number::doubleValue);
    }
}
```

## 디버깅 도구 및 기법

### 🔍 IntelliJ IDEA 디버깅

#### 1. 조건부 중단점
```java
// 중단점 설정 후 우클릭 → More
// Condition: userId.equals(1L) && amount.compareTo(new BigDecimal("50000")) > 0
// Log message to console: "잔액 충전: 사용자={userId}, 금액={amount}"

public Balance execute(Long userId, BigDecimal amount) {
    // 조건부 중단점이 여기에 설정됨
    User user = userRepository.findById(userId)
        .orElseThrow(() -> new UserException.NotFound());
        
    return processCharge(user, amount);
}
```

#### 2. 표현식 평가
```java
// 디버깅 중 Variables 패널에서 우클릭 → Evaluate Expression
// 평가할 표현식 입력:
// user.getName()
// balance.getAmount().compareTo(BigDecimal.valueOf(100000))
// order.getItems().stream().mapToInt(OrderItem::getQuantity).sum()
```

#### 3. Watch 변수 설정
```java
// Debug 창에서 Watches 탭 클릭
// 지속적으로 모니터링할 표현식 추가:
// userId
// amount.toString()
// balance != null ? balance.getAmount() : null
```

### 🛠️ 프로파일링

#### 1. JVM 메모리 분석
```bash
# 힙 덤프 생성
jmap -dump:live,format=b,file=heapdump.hprof <pid>

# 힙 덤프 분석 (Eclipse MAT 사용)
# 1. Eclipse MAT 다운로드
# 2. heapdump.hprof 파일 열기
# 3. Leak Suspects 확인
# 4. Dominator Tree에서 메모리 사용량 큰 객체 확인
```

#### 2. CPU 프로파일링
```bash
# async-profiler 사용
java -jar async-profiler.jar -d 60 -f profile.html <pid>

# 프로파일 결과 분석
# 1. CPU 사용률이 높은 메서드 확인
# 2. 호출 스택 분석
# 3. 핫스팟 메서드 최적화
```

### 🏥 장애 대응 절차

#### 1. 긴급 장애 대응
```bash
# 1단계: 현황 파악
curl http://localhost:8080/actuator/health  # 헬스 체크
tail -f logs/application.log                # 최근 로그 확인
top -p $(pgrep java)                        # CPU/메모리 사용률

# 2단계: 임시 조치
# 1) 애플리케이션 재시작
sudo systemctl restart myapp

# 2) 로드밸런서에서 인스턴스 제외
# 3) 데이터베이스 연결 풀 초기화

# 3단계: 근본 원인 분석
# 1) 스택 트레이스 분석
# 2) 로그 패턴 분석  
# 3) 메트릭 데이터 확인
```

#### 2. 장애 예방
```java
// 서킷 브레이커 패턴 (향후 확장 시)
@Component
public class CircuitBreakerService {
    
    private final CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("service");
    
    public String callExternalService() {
        return circuitBreaker.executeSupplier(() -> {
            // 외부 서비스 호출
            return externalServiceClient.call();
        });
    }
}

// 재시도 패턴
@Component
public class RetryService {
    
    @Retryable(value = {Exception.class}, maxAttempts = 3, delay = 1000)
    public void processWithRetry() {
        // 실패 가능성이 있는 작업
    }
    
    @Recover
    public void recover(Exception ex) {
        // 재시도 실패 시 복구 로직
        log.error("모든 재시도 실패: {}", ex.getMessage());
    }
}
```

---

이 트러블슈팅 가이드를 통해 개발 중 마주칠 수 있는 다양한 문제들을 효과적으로 해결할 수 있습니다. 문제가 발생했을 때는 단계적으로 접근하여 근본 원인을 파악하고, 임시 조치와 장기적 해결책을 모두 고려하는 것이 중요합니다.