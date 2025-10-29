---
aliases:
  - "Redis: 싱글스레드와 분산 환경 안전성 및 활용 사례"
---
## 1. Redis의 싱글스레드 아키텍처와 분산 환경 안전성

### 1.1 Redis의 싱글스레드란?

Redis는 **싱글스레드 이벤트 루프** 모델을 채택한 인메모리 데이터 저장소입니다. 이는 Redis가 단일 스레드에서 클라이언트 요청(예: GET, SET, INCR)을 처리하며, 비동기 I/O와 이벤트 루프를 통해 높은 처리량을 달성한다는 의미입니다. 주요 특징은 다음과 같습니다:

- **단일 스레드**: Redis는 주 작업(데이터 읽기/쓰기)을 단일 스레드에서 처리합니다. 이는 동시성 문제를 줄이고, 명령어가 **원자적**(atomic)으로 실행됨을 보장합니다.
- **이벤트 루프**: 클라이언트 요청을 큐에 저장하고, 이벤트 루프가 이를 순차적으로 처리하여 높은 성능을 유지합니다.
- **비차단 I/O**: 네트워크 I/O는 비차단 방식으로 처리되어 다중 클라이언트 요청을 효율적으로 관리합니다.

### 1.2 싱글스레드가 분산 환경에서 안전한 이유

코치가 언급한 "Redis는 싱글스레드라 분산환경에 안전하다"는 Redis의 아키텍처가 분산 환경에서 동시성 문제를 최소화하고 일관성을 유지하는 데 유리하다는 점을 강조합니다. 이를 쉽게 풀어 설명하면 다음과 같습니다:

1. **원자적 연산 보장**:
    
    - Redis의 명령어(예: `SET`, `INCR`, `GETSET`)는 단일 스레드에서 실행되므로, 명령어 실행 중 다른 요청에 의해 중단되지 않습니다.
    - 예: `INCR balance:user123`는 잔액 증가를 원자적으로 수행하여, 여러 클라이언트가 동시에 요청해도 레이스 컨디션(race condition)이 발생하지 않습니다.
    - e-커머스 과제에서 잔액 충전(`POST /balance/charge`)이나 쿠폰 발급(`POST /coupons/issue`)에서 `LockingPort`의 Redis 구현체(`RedissonLockAdapter`)가 이를 활용하여 동시성 제어를 안전하게 처리합니다.
2. **분산 환경에서의 일관성**:
    
    - 분산 환경에서 여러 서버가 Redis 인스턴스에 접근할 때, Redis의 싱글스레드 모델은 명령어 실행 순서를 명확히 보장합니다.
    - 예: 쿠폰 재고 감소(`DECR coupon:stock:coupon123`)는 여러 서버에서 동시에 호출되어도 Redis가 순차적으로 처리하여 재고 초과 발급을 방지합니다.
    - `LockingPort`의 `RedissonLockAdapter`는 Redis의 `SETNX`(Set if Not Exists)를 사용해 분산 락을 구현, 분산 환경에서 안전한 동시성 제어를 제공합니다.
3. **간단한 동기화 메커니즘**:
    
    - 싱글스레드 모델은 복잡한 락 관리(예: 멀티스레드에서의 뮤텍스) 없이도 동기화를 단순화합니다.
    - Redis는 내부적으로 락이 필요 없는 설계로, 분산 시스템에서 락 구현(예: Redisson의 분산 락)은 키 기반으로 동작하며, 이는 Redis의 원자성을 활용합니다.
4. **분산 환경에서의 확장성**:
    
    - Redis Cluster 또는 Sentinel을 사용하면 여러 노드에 데이터를 샤딩(sharding)하거나 복제(replication)하여 확장 가능합니다.
    - 싱글스레드 모델은 각 노드에서 독립적으로 동작하므로, 클러스터 내 노드 간 동기화 오버헤드가 적습니다.
    - e-커머스 과제에서 Redis Cluster는 상품 조회(`GET /products`) 캐싱(`CachePort`)이나 인기 상품 집계(`GET /products/popular`) 캐싱에 활용 가능.

**쉽게 말해**: Redis는 한 번에 하나의 명령어만 처리하므로, 여러 서버가 동시에 요청을 보내도 명령어가 꼬이지 않고 순서대로 실행됩니다. 이는 잔액 충전, 쿠폰 발급, 재고 관리 같은 동시성 문제가 중요한 상황에서 "안전하다"는 뜻입니다.

### 1.3 싱글스레드의 한계와 보완

- **한계**:
    - **CPU 집약적 작업**: 복잡한 연산(예: `SORT`, `SUNION`)은 단일 스레드를 차단(block)하여 성능 저하 가능.
    - **스케일 업 한계**: 단일 Redis 인스턴스는 CPU 코어 하나만 사용하므로, 높은 부하에서는 Redis Cluster나 다중 인스턴스로 확장 필요.
- **보완**:
    - **Redis Cluster**: 데이터 샤딩으로 처리량 증가.
    - **백그라운드 작업**: AOF(Append-Only File) 쓰기, RDB 스냅샷은 별도 스레드/프로세스에서 처리.
    - **비차단 명령**: e-커머스 과제에서는 주로 `SET`, `GET`, `INCR`, `DECR` 같은 빠른 명령어를 사용하므로 영향 적음.

## 2. Redis의 활용 사례

Redis는 단순한 메모리 캐시 이상으로 다양한 용도로 사용됩니다. e-커머스 과제 맥락에서 캐싱(`CachePort`), 락(`LockingPort`) 외의 활용 사례를 포함하여 정리합니다.

### 2.1 캐싱 (CachePort)

- **역할**: 자주 조회되는 데이터를 메모리에 저장하여 DB 부하 감소, 응답 시간 단축.
- **e-커머스 과제 적용**:
    - 잔액 조회(`GET /balance/{userId}`): `CachePort.checkCache("balance:{userId}")`로 캐시 조회.
    - 상품 목록 조회(`GET /products`): `checkCache("products")`.
    - 인기 상품 조회(`GET /products/popular`): `checkCache("popular_products")`.
- **Redis 명령어**: `SET`, `GET`, `EXPIRE`.
- **예시**:
    
    ```java
    redisTemplate.opsForValue().set("balance:user123", balance, 3600, TimeUnit.SECONDS);
    Balance balance = (Balance) redisTemplate.opsForValue().get("balance:user123");
    ```
    
- **장점**: 빠른 읽기/쓰기, TTL 설정으로 캐시 무효화 간단.

### 2.2 분산 락 (LockingPort)

- **역할**: 분산 환경에서 동시성 제어, 리소스(예: 잔액, 재고, 쿠폰) 보호.
- **e-커머스 과제 적용**:
    - 잔액 충전(`POST /balance/charge`): `LockingPort.acquireLock("user:user123")`.
    - 쿠폰 발급(`POST /coupons/issue`): `acquireLock("coupon:coupon123")`.
    - 주문 생성(`POST /orders`): `acquireLock("product:product123")`.
- **Redis 명령어**: `SETNX`, `EXPIRE`, Redisson 라이브러리 사용.
- **예시**:
    
    ```java
    RLock lock = redissonClient.getLock("lock:user123");
    lock.tryLock(5000, TimeUnit.MILLISECONDS);
    ```
    
- **장점**: 원자적 연산으로 안전한 락 관리, 분산 환경 지원.

### 2.3 메시지 큐/이벤트 스트리밍

- **역할**: 비동기 메시지 전송, 이벤트 발행/구독(pub/sub), 간단한 작업 큐.
- **e-커머스 과제 적용**:
    - 쿠폰 발급 후 알림: `CouponAppliedEvent`를 Redis Pub/Sub으로 사용자 알림 시스템에 전송.
    - 주문 데이터 전송: `OrderPaidEvent`를 Redis Stream으로 데이터 플랫폼에 전송.
- **Redis 기능**:
    - **Pub/Sub**: `PUBLISH`, `SUBSCRIBE`로 실시간 메시지 전송.
    - **Streams**: `XADD`, `XREAD`로 이벤트 스트리밍, Kafka/RabbitMQ 대체 가능.
- **예시**:
    
    ```java
    redisTemplate.convertAndSend("coupon.applied", new CouponAppliedEvent("coupon123", "user123"));
    // 또는 Stream
    redisTemplate.opsForStream().add("order.events", Map.of("event", new OrderPaidEvent("order123")));
    ```
    
- **장점**: 가벼운 메시지 큐, 설정 간단.
- **한계**: 대규모 스트리밍은 Kafka가 더 적합.

### 2.4 세션 관리

- **역할**: 사용자 세션 데이터를 저장, 빠른 세션 조회/업데이트.
- **e-커머스 과제 적용**:
    - 사용자 로그인 세션: `user:session:user123`에 토큰 저장.
    - 장바구니(요구사항 외): 임시 장바구니 데이터를 Redis에 저장.
- **Redis 명령어**: `SET`, `GET`, `EXPIRE`, `HASH`.
- **예시**:
    
    ```java
    redisTemplate.opsForHash().put("user:session:user123", "token", "abc123");
    redisTemplate.expire("user:session:user123", 24, TimeUnit.HOURS);
    ```
    
- **장점**: 빠른 세션 조회, TTL로 자동 만료.

### 2.5 리더보드/랭킹

- **역할**: 정렬된 데이터 집합으로 순위 관리(예: 인기 상품, 사용자 활동 랭킹).
- **e-커머스 과제 적용**:
    - 인기 상품 조회(`GET /products/popular`): `ZINCRBY`로 판매량 집계, `ZRANGE`로 상위 5개 조회.
- **Redis 명령어**: `ZADD`, `ZINCRBY`, `ZRANGE`.
- **예시**:
    
    ```java
    redisTemplate.opsForZSet().incrementScore("popular:products", "product123", 1);
    Set<String> topProducts = redisTemplate.opsForZSet().range("popular:products", 0, 4);
    ```
    
- **장점**: 빠른 정렬, 실시간 집계.

### 2.6 작업 큐

- **역할**: 비동기 작업 대기열(예: 배경 작업, 이메일 전송).
- **e-커머스 과제 적용**:
    - 결제 후 이메일 알림: Redis List로 작업 큐 관리.
- **Redis 명령어**: `LPUSH`, `RPOP`, `BRPOP`.
- **예시**:
    
    ```java
    redisTemplate.opsForList().leftPush("email.queue", "send:order123:user123");
    String task = redisTemplate.opsForList().rightPop("email.queue");
    ```
    
- **장점**: 간단한 큐 구현, 비동기 처리.
- **한계**: 복잡한 작업 큐는 RabbitMQ/Celery 권장.

### 2.7 카운터/메트릭

- **역할**: 실시간 카운터(예: 조회수, 클릭수) 관리.
- **e-커머스 과제 적용**:
    - 상품 조회수: `INCR product:views:product123`로 조회수 증가.
- **Redis 명령어**: `INCR`, `INCRBY`.
- **예시**:
    
    ```java
    redisTemplate.opsForValue().increment("product:views:product123");
    ```
    
- **장점**: 원자적 증가, 빠른 집계.

## 3. e-커머스 과제에서의 Redis 활용

- **캐싱 (`CachePort`)**: 잔액, 상품, 인기 상품 데이터를 캐시하여 DB 부하 감소.
- **분산 락 (`LockingPort`)**: 잔액 충전, 쿠폰 발급, 주문 생성 시 동시성 제어.
- **메시지 큐**: `CouponAppliedEvent`, `OrderPaidEvent`를 Pub/Sub 또는 Stream으로 전송.
- **리더보드**: 인기 상품 집계(`ZINCRBY`, `ZRANGE`).
- **카운터**: 상품 조회수, 이벤트 참여 횟수 관리.
- **세션 관리**: 사용자 세션 데이터 저장(요구사항 외, 확장 가능).

## 4. 구현 예시 (Redis CachePort, LockingPort)

### CachePort (Redis)

```java
@Component
public class RedisCacheAdapter implements CachePort {
    private final RedisTemplate<String, Object> redisTemplate;
    
    public RedisCacheAdapter(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    @Override
    public <T> Optional<T> checkCache(String key) {
        return Optional.ofNullable((T) redisTemplate.opsForValue().get(key));
    }
    
    @Override
    public <T> void cacheData(String key, T data, long ttlSeconds) {
        redisTemplate.opsForValue().set(key, data, ttlSeconds, TimeUnit.SECONDS);
    }
    
    @Override
    public void invalidateCache(String key) {
        redisTemplate.delete(key);
    }
}
```

### LockingPort (Redis with Redisson)

```java
@Component
public class RedissonLockAdapter implements LockingPort {
    private final RedissonClient redissonClient;
    
    public RedissonLockAdapter(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }
    
    @Override
    public boolean acquireLock(String key, long timeoutMs) {
        RLock lock = redissonClient.getLock("lock:" + key);
        try {
            return lock.tryLock(timeoutMs, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
    
    @Override
    public void releaseLock(String key) {
        RLock lock = redissonClient.getLock("lock:" + key);
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

## 5. 고려사항

- **성능 최적화**: `Pipelining` 또는 `BATCH`로 다중 명령 최적화.
- **영속성**: AOF/RDB 설정으로 데이터 손실 방지.
- **모니터링**: Redis 명령어 실행 시간, 메모리 사용량, 히트율 모니터링.
- **테스트**: `Testcontainers`로 Redis 통합 테스트, Mock 객체로 `CachePort`, `LockingPort` 테스트.
- **오버 엔지니어링 방지**: 소규모 시스템에서는 Redis Cluster 대신 단일 인스턴스 사용.

## 6. 결론

Redis의 싱글스레드 모델은 원자적 연산으로 분산 환경에서 안전한 동시성 제어를 제공하며, 캐싱, 락 외에도 메시지 큐, 리더보드, 세션 관리, 작업 큐, 카운터 등 다양한 용도로 활용됩니다. e-커머스 과제에서는 `CachePort`, `LockingPort`, 메시지 큐로 비즈니스 가치를 극대화하며, Kafka/RabbitMQ 대체 가능.