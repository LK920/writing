# 동시성 제어와 락(Lock) 완전 정리

## 목차
- [1. 동시성 제어란?](#1-동시성-제어란)
- [2. 락(Lock)의 기본 개념](#2-락lock의-기본-개념)
- [3. 락의 종류와 분류](#3-락의-종류와-분류)
- [4. 현재 프로젝트의 락 구현 분석](#4-현재-프로젝트의-락-구현-분석)
- [5. 다양한 락 구현 방법과 예시](#5-다양한-락-구현-방법과-예시)
- [6. 락 선택 가이드](#6-락-선택-가이드)
- [7. 실제 구현 예시](#7-실제-구현-예시)
- [8. 성능 비교와 트레이드오프](#8-성능-비교와-트레이드오프)

## 1. 동시성 제어란?

### 동시성 문제 상황
여러 개의 스레드나 프로세스가 **공유 자원에 동시에 접근**할 때 발생하는 문제들:

#### 1.1 Race Condition (경합 상태)
```java
// 문제 상황: 두 스레드가 동시에 계좌 잔액을 수정
public class BankAccount {
    private int balance = 1000;
    
    public void withdraw(int amount) {
        if (balance >= amount) {          // Thread A가 조건 확인
            // Thread B가 여기서 끼어들어 잔액을 차감할 수 있음
            balance = balance - amount;   // Thread A가 차감 실행
        }
    }
}
```

#### 1.2 데이터 불일치
```java
// 상품 재고 관리에서 발생할 수 있는 문제
public class Product {
    private int stock = 10;
    
    public boolean purchase() {
        if (stock > 0) {        // Thread 1: stock = 10 확인
                               // Thread 2: stock = 10 확인  
            stock--;           // Thread 1: stock = 9
            return true;       // Thread 2: stock = 8 (실제로는 1개만 팔려야 함)
        }
        return false;
    }
}
```

### 동시성 제어의 필요성
- **데이터 무결성 보장**: 공유 자원의 일관성 유지
- **원자성(Atomicity) 보장**: 연산이 모두 성공하거나 모두 실패
- **가시성(Visibility) 보장**: 한 스레드의 변경이 다른 스레드에게 보임
- **순서성(Ordering) 보장**: 연산의 실행 순서 제어

## 2. 락(Lock)의 기본 개념

### 락이란?
**락(Lock)**은 공유 자원에 대한 접근을 제어하는 동시성 제어 메커니즘으로, 한 번에 하나의 스레드만 자원에 접근할 수 있도록 보장합니다.

### 락의 기본 동작
```
1. Lock Acquisition (락 획득)
   ↓
2. Critical Section (임계 영역 실행)
   ↓
3. Lock Release (락 해제)
```

### 중요한 락 속성들

#### 2.1 Mutual Exclusion (상호 배제)
동시에 하나의 스레드만 락을 가질 수 있음

#### 2.2 Progress (진행)
락을 기다리는 스레드가 있다면 언젠가는 락을 획득할 수 있어야 함

#### 2.3 Bounded Waiting (한정 대기)
락을 기다리는 시간이 무한하지 않아야 함

## 3. 락의 종류와 분류

### 3.1 범위에 따른 분류

#### Thread-Level Lock (스레드 수준 락)
- **범위**: 하나의 JVM 프로세스 내 스레드 간
- **용도**: 단일 애플리케이션 인스턴스 내 동시성 제어
- **예시**: `synchronized`, `ReentrantLock`, `Semaphore`

```java
// synchronized 예시
public synchronized void criticalMethod() {
    // 한 번에 하나의 스레드만 실행 가능
}

// ReentrantLock 예시  
private final ReentrantLock lock = new ReentrantLock();

public void safeCriticalSection() {
    lock.lock();
    try {
        // 임계 영역
    } finally {
        lock.unlock();
    }
}
```

#### Process-Level Lock (프로세스 수준 락)
- **범위**: 여러 프로세스 간
- **용도**: 서로 다른 애플리케이션 간 동시성 제어
- **예시**: 파일 락, 세마포어, 뮤텍스

#### Distributed Lock (분산 락)
- **범위**: 여러 서버/머신 간
- **용도**: 분산 시스템에서 전역적 동시성 제어
- **예시**: Redis 분산락, ZooKeeper 락, 데이터베이스 락

### 3.2 락 모드에 따른 분류

#### Exclusive Lock (배타적 락)
- **특징**: 하나의 스레드만 락을 보유 가능
- **용도**: 데이터 변경 작업
- **예시**: Write Lock

```java
// 배타적 락 예시
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock writeLock = rwLock.writeLock();

public void updateData() {
    writeLock.lock();
    try {
        // 데이터 변경 - 다른 모든 접근 차단
        data.modify();
    } finally {
        writeLock.unlock();
    }
}
```

#### Shared Lock (공유 락)
- **특징**: 여러 스레드가 동시에 락을 보유 가능
- **용도**: 데이터 읽기 작업
- **예시**: Read Lock

```java
// 공유 락 예시
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();

public Data readData() {
    readLock.lock();
    try {
        // 데이터 읽기 - 여러 스레드 동시 접근 가능
        return data.read();
    } finally {
        readLock.unlock();
    }
}
```

### 3.3 대기 방식에 따른 분류

#### Blocking Lock (블로킹 락)
- **특징**: 락을 획득할 때까지 스레드가 대기(블록)
- **장점**: 구현이 간단, CPU 사용량 적음
- **단점**: 컨텍스트 스위칭 오버헤드

```java
// 블로킹 락 예시
public synchronized void blockingMethod() {
    // 락을 획득할 때까지 스레드 블록
    performWork();
}
```

#### Non-Blocking Lock (논블로킹 락)
- **특징**: 락 획득에 실패하면 즉시 반환
- **장점**: 응답성이 좋음, 데드락 회피 가능
- **단점**: 구현 복잡, 스핀락의 경우 CPU 사용량 증가

```java
// 논블로킹 락 예시
ReentrantLock lock = new ReentrantLock();

public boolean tryWork() {
    if (lock.tryLock()) {
        try {
            performWork();
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false; // 락 획득 실패
}
```

#### Spin Lock (스핀 락)
- **특징**: 락을 획득할 때까지 계속 확인 (busy waiting)
- **장점**: 컨텍스트 스위칭 없음, 짧은 대기에 효율적
- **단점**: CPU 리소스 소모

```java
// 스핀락 구현 예시
public class SpinLock {
    private AtomicBoolean locked = new AtomicBoolean(false);
    
    public void lock() {
        while (!locked.compareAndSet(false, true)) {
            // 계속 확인 (스핀)
        }
    }
    
    public void unlock() {
        locked.set(false);
    }
}
```

### 3.4 재진입 가능성에 따른 분류

#### Reentrant Lock (재진입 가능 락)
- **특징**: 같은 스레드가 락을 여러 번 획득 가능
- **용도**: 재귀 호출이나 중첩된 메서드 호출

```java
public class ReentrantExample {
    private final ReentrantLock lock = new ReentrantLock();
    
    public void method1() {
        lock.lock();
        try {
            method2(); // 같은 스레드에서 락을 다시 요청
        } finally {
            lock.unlock();
        }
    }
    
    public void method2() {
        lock.lock(); // 성공! (재진입 가능)
        try {
            // 작업 수행
        } finally {
            lock.unlock();
        }
    }
}
```

#### Non-Reentrant Lock (재진입 불가능 락)
- **특징**: 같은 스레드라도 락을 중복 획득할 수 없음
- **주의**: 데드락 위험

## 4. 현재 프로젝트의 락 구현 분석

### 4.1 현재 구현체 분석

```java
@Component
public class InMemoryLockingAdapter implements LockingPort {
    
    private final Map<String, Thread> locks = new ConcurrentHashMap<>();
    
    @Override
    public boolean acquireLock(String key) {
        Thread currentThread = Thread.currentThread();
        Thread existingThread = locks.putIfAbsent(key, currentThread);
        return existingThread == null || existingThread == currentThread;
    }
    
    @Override
    public void releaseLock(String key) {
        locks.remove(key);
    }
    
    @Override
    public boolean isLocked(String key) {
        return locks.containsKey(key);
    }
}
```

### 4.2 현재 구현의 특징

#### 락 타입 분류
- **범위**: Thread-Level Lock (단일 JVM 내부)
- **모드**: Exclusive Lock (배타적 락)
- **대기 방식**: Non-Blocking Lock (tryLock 방식)
- **재진입성**: Reentrant (같은 스레드에서 재진입 가능)

#### 동작 원리
1. `ConcurrentHashMap`을 사용하여 락 상태 관리
2. Key-Value 형태로 락 이름과 소유 스레드 매핑
3. `putIfAbsent()` 원자적 연산으로 락 획득 시도
4. 같은 스레드면 재진입 허용

### 4.3 현재 구현의 장단점

#### 장점 ✅
- **구현 단순성**: 간단하고 이해하기 쉬운 구조
- **재진입 지원**: 같은 스레드에서 중복 락 획득 가능
- **논블로킹**: 락 획득 실패 시 즉시 반환으로 응답성 좋음
- **메모리 효율성**: 별도 외부 저장소 불필요

#### 단점 ❌
- **단일 JVM 제한**: 여러 서버 인스턴스 간 동시성 제어 불가능
- **장애 취약성**: 프로세스 종료 시 락이 자동 해제되지 않을 수 있음
- **확장성 제한**: 분산 환경에서 사용 불가
- **타임아웃 미지원**: 무한 대기 방지 메커니즘 없음

### 4.4 적합한 사용 사례

#### 현재 구현이 적합한 경우
- **단일 서버 환경**: 하나의 애플리케이션 인스턴스만 실행
- **프로토타입/개발**: 빠른 개발과 테스트
- **사용자별 격리**: 사용자 ID 기반 독립적 락 관리

#### 현재 구현이 부적합한 경우
- **다중 서버 환경**: 로드밸런서 뒤의 여러 인스턴스
- **고가용성 요구**: 서버 장애 시에도 락 상태 유지 필요
- **긴 수행 시간**: 몇 분 이상의 긴 작업에 대한 락

## 5. 다양한 락 구현 방법과 예시

### 5.1 데이터베이스 락

#### Pessimistic Lock (비관적 락)
데이터를 읽을 때부터 락을 걸어 다른 트랜잭션의 접근을 차단

```java
// JPA에서 비관적 락 사용
@Repository
public class ProductRepository {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Product findByIdWithLock(@Param("id") Long id);
}

@Service
public class OrderService {
    
    @Transactional
    public void processOrder(Long productId, int quantity) {
        // 상품을 락과 함께 조회 (다른 트랜잭션 차단)
        Product product = productRepository.findByIdWithLock(productId);
        
        if (product.getStock() >= quantity) {
            product.decreaseStock(quantity);
            productRepository.save(product);
        }
    }
}
```

**장점**: 데이터 일관성 강하게 보장  
**단점**: 성능 저하, 데드락 위험, 동시성 낮음

#### Optimistic Lock (낙관적 락)
데이터 변경 시점에 충돌 검사

```java
@Entity
public class Product {
    @Id
    private Long id;
    
    @Version  // 낙관적 락을 위한 버전 필드
    private Long version;
    
    private int stock;
    
    public void decreaseStock(int quantity) {
        if (this.stock < quantity) {
            throw new InsufficientStockException();
        }
        this.stock -= quantity;
    }
}

@Service
public class OrderService {
    
    @Transactional
    public void processOrder(Long productId, int quantity) {
        Product product = productRepository.findById(productId);
        product.decreaseStock(quantity);
        
        try {
            productRepository.save(product); // 버전 충돌 시 예외 발생
        } catch (OptimisticLockingFailureException e) {
            throw new ConcurrentModificationException("재고가 다른 주문에 의해 변경됨");
        }
    }
}
```

**장점**: 높은 동시성, 데드락 없음  
**단점**: 충돌 시 재시도 로직 필요, 충돌 빈발 시 성능 저하

### 5.2 Redis 분산 락

#### 단순 Redis 락
```java
@Component
public class RedisDistributedLock {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    public boolean acquireLock(String key, String value, long expireTime) {
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, value, Duration.ofMillis(expireTime));
        return Boolean.TRUE.equals(result);
    }
    
    public void releaseLock(String key, String value) {
        // Lua 스크립트로 원자적 해제 (자신이 설정한 락만 해제)
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        
        redisTemplate.execute(new DefaultRedisScript<>(script, Long.class), 
                             Arrays.asList(key), value);
    }
}
```

#### Redisson 분산 락 (고급)
```java
@Component
public class RedissonDistributedLock {
    
    @Autowired
    private RedissonClient redissonClient;
    
    public <T> T executeWithLock(String lockKey, long waitTime, long leaseTime, 
                                TimeUnit unit, Supplier<T> task) {
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            boolean acquired = lock.tryLock(waitTime, leaseTime, unit);
            if (!acquired) {
                throw new LockAcquisitionException("락 획득 실패: " + lockKey);
            }
            
            return task.get();
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("락 획득 중 인터럽트 발생", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}

// 사용 예시
@Service
public class CouponService {
    
    @Autowired
    private RedissonDistributedLock distributedLock;
    
    public void issueCoupon(Long userId, Long couponId) {
        String lockKey = "coupon-issue:" + couponId;
        
        distributedLock.executeWithLock(lockKey, 3, 10, TimeUnit.SECONDS, () -> {
            // 쿠폰 발급 로직
            Coupon coupon = couponRepository.findById(couponId);
            if (coupon.canIssue()) {
                coupon.issue(userId);
                couponRepository.save(coupon);
            }
            return null;
        });
    }
}
```

**장점**: 진정한 분산 락, 타임아웃 지원, 자동 갱신  
**단점**: Redis 의존성, 네트워크 지연, 복잡한 설정

### 5.3 ZooKeeper 분산 락

```java
@Component
public class ZookeeperDistributedLock {
    
    private final CuratorFramework client;
    
    public ZookeeperDistributedLock(CuratorFramework client) {
        this.client = client;
    }
    
    public void executeWithLock(String lockPath, Runnable task) {
        InterProcessMutex lock = new InterProcessMutex(client, lockPath);
        
        try {
            if (lock.acquire(30, TimeUnit.SECONDS)) {
                task.run();
            } else {
                throw new LockAcquisitionException("ZooKeeper 락 획득 실패");
            }
        } catch (Exception e) {
            throw new RuntimeException("ZooKeeper 락 처리 중 오류", e);
        } finally {
            try {
                lock.release();
            } catch (Exception e) {
                log.error("ZooKeeper 락 해제 실패", e);
            }
        }
    }
}
```

**장점**: 강한 일관성, 자동 락 해제, 순서 보장  
**단점**: 복잡한 설정, ZooKeeper 클러스터 필요, 상대적 느림

### 5.4 파일 기반 락

```java
@Component
public class FileLock {
    
    public void executeWithFileLock(String lockFilePath, Runnable task) {
        File lockFile = new File(lockFilePath);
        
        try (RandomAccessFile file = new RandomAccessFile(lockFile, "rw");
             FileChannel channel = file.getChannel();
             java.nio.channels.FileLock lock = channel.tryLock()) {
            
            if (lock == null) {
                throw new LockAcquisitionException("파일 락 획득 실패");
            }
            
            task.run();
            
        } catch (IOException e) {
            throw new RuntimeException("파일 락 처리 중 오류", e);
        }
    }
}
```

## 6. 락 선택 가이드

### 6.1 선택 기준 매트릭스

| 구분 | Thread Lock | Database Lock | Redis Lock | ZooKeeper Lock |
|------|-------------|---------------|------------|----------------|
| **범위** | 단일 JVM | 단일 DB | 다중 서버 | 다중 서버 |
| **성능** | 매우 높음 | 낮음 | 높음 | 중간 |
| **복잡도** | 낮음 | 낮음 | 중간 | 높음 |
| **일관성** | 중간 | 높음 | 중간 | 매우 높음 |
| **가용성** | 낮음 | 중간 | 높음 | 높음 |
| **타임아웃** | 지원 | 지원 | 지원 | 지원 |
| **자동해제** | 불안정 | 안정 | 안정 | 매우 안정 |

### 6.2 시나리오별 권장사항

#### 단일 서버 환경
```
권장: Thread-Level Lock (ReentrantLock, synchronized)
이유: 간단하고 빠름, 외부 의존성 없음
```

#### 마이크로서비스 환경
```
권장: Redis 분산 락 (Redisson)
이유: 높은 성능, 쉬운 구현, 타임아웃 지원
```

#### 금융/결제 시스템
```
권장: Database Pessimistic Lock + ZooKeeper
이유: 강한 일관성, 신뢰성, 감사 추적 가능
```

#### 대용량 트래픽
```
권장: Redis 분산 락 + 낙관적 락 조합
이유: 높은 동시성, 확장성
```

## 7. 실제 구현 예시

### 7.1 현재 프로젝트 개선안 - Redis 분산 락

```java
// 개선된 LockingPort 인터페이스
public interface LockingPort {
    boolean acquireLock(String key);
    boolean acquireLock(String key, long waitTime, long leaseTime, TimeUnit unit);
    void releaseLock(String key);
    boolean isLocked(String key);
}

// Redis 분산 락 구현체
@Component
@Profile("!local") // local 환경이 아닐 때만 활성화
public class RedisDistributedLockingAdapter implements LockingPort {
    
    private final RedissonClient redissonClient;
    private final Map<String, RLock> activeLocks = new ConcurrentHashMap<>();
    
    public RedisDistributedLockingAdapter(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }
    
    @Override
    public boolean acquireLock(String key) {
        return acquireLock(key, 0, 30, TimeUnit.SECONDS);
    }
    
    @Override
    public boolean acquireLock(String key, long waitTime, long leaseTime, TimeUnit unit) {
        RLock lock = redissonClient.getLock("lock:" + key);
        
        try {
            boolean acquired = lock.tryLock(waitTime, leaseTime, unit);
            if (acquired) {
                activeLocks.put(key, lock);
            }
            return acquired;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
    
    @Override
    public void releaseLock(String key) {
        RLock lock = activeLocks.remove(key);
        if (lock != null && lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
    
    @Override
    public boolean isLocked(String key) {
        RLock lock = redissonClient.getLock("lock:" + key);
        return lock.isLocked();
    }
}

// 기존 InMemory 구현체 (로컬 환경용)
@Component
@Profile("local")
public class InMemoryLockingAdapter implements LockingPort {
    // 기존 구현 유지
    
    @Override
    public boolean acquireLock(String key, long waitTime, long leaseTime, TimeUnit unit) {
        // 단순화: 타임아웃 무시하고 즉시 시도
        return acquireLock(key);
    }
}
```

### 7.2 하이브리드 락 전략

```java
@Component
public class HybridLockingStrategy {
    
    private final LockingPort distributedLock;
    private final Map<String, ReentrantLock> localLocks = new ConcurrentHashMap<>();
    
    /**
     * 2단계 락: 로컬 락 + 분산 락
     * 1. 먼저 로컬 락으로 동일 인스턴스 내 경합 방지
     * 2. 분산 락으로 다른 인스턴스와의 경합 방지
     */
    public <T> T executeWithHybridLock(String key, Supplier<T> task) {
        // 1단계: 로컬 락
        ReentrantLock localLock = localLocks.computeIfAbsent(key, k -> new ReentrantLock());
        
        localLock.lock();
        try {
            // 2단계: 분산 락
            if (!distributedLock.acquireLock(key, 5, 30, TimeUnit.SECONDS)) {
                throw new LockAcquisitionException("분산 락 획득 실패: " + key);
            }
            
            try {
                return task.get();
            } finally {
                distributedLock.releaseLock(key);
            }
            
        } finally {
            localLock.unlock();
        }
    }
}
```

### 7.3 락 데코레이터 패턴

```java
// 락 기능을 추가하는 어노테이션
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    String key();
    long waitTime() default 5;
    long leaseTime() default 30;
    TimeUnit unit() default TimeUnit.SECONDS;
}

// AOP를 이용한 락 처리
@Aspect
@Component
public class DistributedLockAspect {
    
    private final LockingPort lockingPort;
    
    @Around("@annotation(distributedLock)")
    public Object handleDistributedLock(ProceedingJoinPoint joinPoint, 
                                       DistributedLock distributedLock) throws Throwable {
        
        String lockKey = generateLockKey(distributedLock.key(), joinPoint);
        
        boolean acquired = lockingPort.acquireLock(
            lockKey, 
            distributedLock.waitTime(), 
            distributedLock.leaseTime(), 
            distributedLock.unit()
        );
        
        if (!acquired) {
            throw new LockAcquisitionException("락 획득 실패: " + lockKey);
        }
        
        try {
            return joinPoint.proceed();
        } finally {
            lockingPort.releaseLock(lockKey);
        }
    }
    
    private String generateLockKey(String template, ProceedingJoinPoint joinPoint) {
        // SpEL을 사용하여 동적 키 생성
        // 예: "coupon-issue:#{args[1]}" -> "coupon-issue:123"
        return SpelExpressionParser.parseExpression(template, joinPoint);
    }
}

// 사용 예시
@Service
public class CouponService {
    
    @DistributedLock(key = "coupon-issue:#{args[1]}", waitTime = 3, leaseTime = 10)
    public void issueCoupon(Long userId, Long couponId) {
        // 락이 자동으로 적용됨
        Coupon coupon = couponRepository.findById(couponId);
        coupon.issue(userId);
        couponRepository.save(coupon);
    }
}
```

## 8. 성능 비교와 트레이드오프

### 8.1 벤치마크 결과 (참고용)

| 락 타입 | TPS (초당 처리량) | 평균 지연시간 | 메모리 사용량 | CPU 사용률 |
|---------|------------------|---------------|---------------|------------|
| Thread Lock | 50,000 | 0.1ms | 낮음 | 낮음 |
| Redis Lock | 5,000 | 2ms | 중간 | 중간 |
| DB Pessimistic | 1,000 | 10ms | 낮음 | 높음 |
| ZooKeeper | 2,000 | 5ms | 중간 | 중간 |

### 8.2 트레이드오프 분석

#### 성능 vs 일관성
```
높은 성능: Thread Lock > Redis Lock > ZooKeeper > Database Lock
강한 일관성: Database Lock > ZooKeeper > Redis Lock > Thread Lock
```

#### 단순성 vs 기능성
```
구현 단순: Thread Lock > Database Lock > Redis Lock > ZooKeeper
풍부한 기능: ZooKeeper > Redis Lock > Database Lock > Thread Lock
```

#### 비용 vs 확장성
```
낮은 비용: Thread Lock > Database Lock > Redis Lock > ZooKeeper
높은 확장성: Redis Lock > ZooKeeper > Database Lock > Thread Lock
```

### 8.3 실제 사용 권장사항

#### 프로젝트 규모별 권장
```
개인/소규모 프로젝트: Thread Lock (현재 구현)
중소규모 서비스: Redis 분산 락
대규모 엔터프라이즈: ZooKeeper + Database Lock 조합
```

#### 도메인별 권장
```
사용자 인증: Thread Lock (사용자별 독립)
상품 재고: Redis 분산 락 (서버간 동기화 필요)
금융 거래: Database Pessimistic Lock (강한 일관성)
쿠폰 발급: Redis 분산 락 + 낙관적 락 (높은 동시성)
```

## 결론

현재 프로젝트의 `InMemoryLockingAdapter`는 **Thread-Level Lock**으로, 분산락이 아닌 **단일 JVM 내 스레드 간 동시성 제어**를 담당합니다. 

### 현재 구현의 적절성
- ✅ **개발/테스트 환경**: 매우 적합
- ✅ **단일 서버 운영**: 적합  
- ❌ **다중 서버 환경**: 부적합 (분산락 필요)

### 향후 확장 방향
1. **프로덕션 환경**: Redis 분산락으로 교체
2. **하이브리드 전략**: 로컬락 + 분산락 조합
3. **도메인별 최적화**: 요구사항에 따른 락 전략 선택

이 문서를 통해 락의 전반적인 이해와 함께, 프로젝트의 요구사항에 맞는 최적의 락 전략을 선택할 수 있을 것입니다.