# Redis 심화 학습 자료: 동시성 제어와 캐싱 전략

이 문서는 Spring 환경에서 Redis를 활용한 동시성 제어(락)와 캐싱 전략의 심화 학습을 위한 자료입니다. Redis의 기본 개념(`SETNX`, TTL)을 이해한 초보자부터 현업 개발자까지를 대상으로, Spring의 **RedisTemplate**과 **Redisson**을 비교하고, 락과 캐싱의 심화 키워드를 중심으로 구조화했습니다. 각 키워드마다 개념, 중요성, 학습 포인트, 사고 확장 질문을 포함하며, 실습 중심 학습을 위한 가이드를 제공합니다.

---

## 🛠 Spring에서 RedisTemplate과 Redisson 비교

Spring Boot 프로젝트에서 Redis를 사용할 때, **RedisTemplate**과 **Redisson**은 주요 클라이언트 도구로, 각각의 강점과 사용 시나리오가 다릅니다. 두 도구를 함께 사용할 수도 있으며, 아래는 비교와 활용법입니다.

### RedisTemplate
- **설명**: Spring Data Redis에서 제공하는 경량 클라이언트. Redis 명령어를 직접 호출하거나 Spring Cache와 통합해 사용.
- **장점**:
  - 경량: Spring Boot에 기본 통합, 추가 의존성 최소화.
  - 유연성: `SETNX`, `HSET`, `LPUSH` 등 Redis 명령어 직접 호출 가능.
  - Spring Cache 통합: `@Cacheable`, `@CacheEvict`로 캐싱 간편.
- **단점**:
  - 고급 락(Redlock, Pub/Sub Lock)은 수동 구현 필요.
  - Redis Cluster 설정이 수동적.
  - 복잡한 락 로직(예: Fencing Token) 구현이 번거로움.
- **사용 예시** (캐싱):
  ```java
  @Autowired
  private StringRedisTemplate redisTemplate;

  @Cacheable(value = "popularItems", key = "'daily'")
  public List<String> getPopularItems() {
      return List.of("item1", "item2"); // DB 조회
  }
  ```
- **추천 시나리오**:
  - 간단한 캐싱 작업(예: 조회 데이터 캐싱).
  - Redis 데이터 구조(List, Hash)를 세밀히 제어.
  - 소규모 프로젝트에서 경량 구현.

### Redisson
- **설명**: 고급 Redis 클라이언트로, Redlock, Pub/Sub Lock, 캐시 관리 등 내장 기능 제공.
- **장점**:
  - 고급 락: Redlock, Fair Lock, ReadWriteLock 지원.
  - Redis Cluster: 키 분배 자동화, 클러스터 환경 최적화.
  - Pub/Sub: 락 해제 이벤트 구독으로 효율적 대기.
- **단점**:
  - 추가 의존성으로 애플리케이션 크기 증가.
  - Spring Cache와의 통합이 RedisTemplate보다 복잡.
  - 초기 설정 및 학습 곡선이 약간 높음.
- **사용 예시** (분산락):
  ```java
  @Autowired
  private RedissonClient redissonClient;

  public void placeOrder(Long userId) {
      RLock lock = redissonClient.getLock("orderLock:" + userId);
      try {
          if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
              // 비즈니스 로직
          }
      } catch (InterruptedException e) {
          throw new RuntimeException("Lock interrupted", e);
      } finally {
          if (lock.isHeldByCurrentThread()) {
              lock.unlock();
          }
      }
  }
  ```
- **추천 시나리오**:
  - 복잡한 분산락(Redlock, Pub/Sub Lock) 구현.
  - 대규모 트래픽, Redis Cluster 환경.
  - 고급 캐시 관리(예: Write-Through, Eviction).

### 함께 사용하는 경우
- **왜 함께 사용?**:
  - **RedisTemplate**: Spring Cache로 간단한 캐싱 처리.
  - **Redisson**: 복잡한 락 로직이나 Redis Cluster 환경에서 안정성 제공.
  - 예: 캐싱은 RedisTemplate(`@Cacheable`), 락은 Redisson(Redlock)으로 분리.
- **주의점**:
  - **키 충돌 방지**: 캐시 키(`cache:`)와 락 키(`lock:`) 네임스페이스 구분.
  - **연결 풀 관리**: Redis 연결 설정을 RedisTemplate과 Redisson에서 일관되게.
  - **복잡도**: 두 도구의 API를 모두 익혀야 함.

**질문**:
- RedisTemplate과 Redisson을 함께 사용할 때, 어떤 작업을 어느 도구에 맡기고 싶어?
- 두 도구의 키 충돌을 방지하려면 어떤 전략을 사용할 수 있을까?

---

## 📚 동시성 제어 (Distributed Lock) 심화 키워드

Redis의 분산락은 대규모 트래픽에서 데이터 일관성을 보장하는 핵심 기술입니다. 아래는 심화 학습을 위한 키워드와 접근법입니다.

### 1. Redlock 알고리즘
- **왜 중요?**: 단일 Redis 인스턴스의 `SETNX`는 단일 장애점(SPOF) 문제가 있음. Redlock은 다수 노드에서 과반수 락을 획득해 신뢰성 향상.
- **학습 포인트**:
  - Redlock 동작: 다수 노드에서 `SETNX` 호출 후 과반수 성공 확인.
  - Redisson으로 Redlock 구현.
  - 네트워크 지연, 클럭 드리프트 문제 분석.
- **질문**: Redlock이 단일 `SETNX`보다 어떤 상황에서 유리할까? 단점은?

### 2. Fencing Token
- **왜 중요?**: 락 해제 후 다른 클라이언트가 오래된 데이터로 작업하지 않도록 고유 토큰으로 요청 구분.
- **학습 포인트**:
  - `SETNX`로 락 획득 시 토큰 생성 및 검증.
  - RedisTemplate에서 Fencing Token 구현.
- **질문**: Fencing Token이 없으면 어떤 동시성 문제가 발생할까?

### 3. Lua 스크립트로 원자성 보장
- **왜 중요?**: `SETNX`와 `EXPIRE`를 별도로 호출하면 원자성이 깨질 수 있음. Lua 스크립트로 단일 명령 처리.
- **학습 포인트**:
  - Redis `EVAL`로 락 획득/해제 스크립트 작성.
  - Spring RedisTemplate에서 Lua 실행.
- **질문**: Lua 스크립트 없이 락 로직을 구현하면 어떤 위험이 있을까?

### 4. Pub/Sub 기반 락
- **왜 중요?**: Spin Lock의 CPU 낭비를 줄이고, 락 해제 이벤트를 구독해 효율적 대기.
- **학습 포인트**:
  - Redis Pub/Sub으로 락 해제 알림 구현.
  - Redisson의 Pub/Sub Lock 활용.
- **질문**: Pub/Sub 락이 Spin Lock보다 적합한 시나리오는?

### 5. Lock Granularity
- **왜 중요?**: 락 범위를 최적화해 병목 감소(예: `userLock:<userId>` vs `accountLock:<accountId>`).
- **학습 포인트**:
  - 적절한 락 키 설계.
  - 락 범위에 따른 성능/안정성 트레이드오프.
- **질문**: 락 범위를 너무 넓게 설정하면 어떤 문제가 생길까?

---

## 📚 캐싱 전략 심화 키워드

Redis 캐싱은 조회 성능을 최적화하고 DB 부하를 줄이는 핵심 기술입니다. 아래는 심화 학습을 위한 키워드입니다.

### 1. Cache-Aside (Lazy Loading)
- **왜 중요?**: 캐시 미스 시 DB에서 데이터를 가져와 캐시에 저장. Spring `@Cacheable`로 쉽게 구현.
- **학습 포인트**:
  - RedisTemplate으로 Cache-Aside 구현.
  - 캐시 미스 시 DB 부하 관리.
- **질문**: 캐시 미스가 자주 발생하면 어떤 문제를 유발할까?

### 2. Write-Through
- **왜 중요?**: 캐시와 DB를 동시에 업데이트해 데이터 일관성 보장.
- **학습 포인트**:
  - Redisson으로 Write-Through 구현.
  - 성능 오버헤드와 일관성 트레이드오프.
- **질문**: Write-Through가 Cache-Aside보다 적합한 시나리오는?

### 3. Cache Stampede 방지
- **왜 중요?**: 캐시 만료 시 다수 요청이 DB로 몰리는 현상 방지.
- **학습 포인트**:
  - TTL에 랜덤 Jitter 추가.
  - Spring 스케줄러로 조기 갱신(Early Refresh).
- **질문**: 캐시 스탬피드를 방지하지 않으면 어떤 성능 문제가 생길까?

### 4. Cache Eviction Policies
- **왜 중요?**: 메모리 부족 시 캐시 데이터를 효율적으로 제거.
- **학습 포인트**:
  - Redis의 LRU, LFU 정책(`MAXMEMORY-POLICY`).
  - Spring에서 캐시 크기 관리.
- **질문**: LRU와 LFU 중 어떤 상황에서 어떤 정책이 유리할까?

### 5. Redis Cluster와 캐싱
- **왜 중요?**: 대규모 트래픽에서 단일 Redis 인스턴스의 한계를 극복.
- **학습 포인트**:
  - Redis Cluster의 슬롯 기반 키 분배.
  - Redisson으로 클러스터 환경 최적화.
- **질문**: Redis Cluster에서 캐시 키가 잘못 분배되면 어떤 문제가 발생할까?

---

## 📈 학습 로드맵 (10시간)

### Day 1: RedisTemplate으로 캐싱과 기본 락 (3시간)
- **활동**:
  - RedisTemplate 설정 및 `@Cacheable`로 Cache-Aside 구현.
  - `SETNX`로 간단한 락 구현.
  - Lua 스크립트로 원자성 보장.
- **자가진단**:
  - RedisTemplate으로 캐싱과 락 플로우를 설명할 수 있나? (1~5점)
- **질문**: `SETNX`와 `EXPIRE`를 별도로 호출하면 어떤 문제가 생길까?

### Day 2: Redisson으로 고급 락 (3시간)
- **활동**:
  - Redisson 설정 및 Redlock 구현.
  - Pub/Sub Lock으로 락 대기 최적화.
  - TestContainer로 동시 요청 테스트.
- **자가진단**:
  - Redlock의 동작 원리를 설명할 수 있나? (1~5점)
- **질문**: Redisson의 Redlock이 RedisTemplate의 `SETNX`보다 편리한 점은?

### Day 3: RedisTemplate + Redisson 통합 (4시간)
- **활동**:
  - RedisTemplate으로 캐싱, Redisson으로 락을 결합한 주문 시스템 구현.
  - 키 네임스페이스 분리(예: `cache:`, `lock:`).
  - Redis `INFO`로 캐시 히트율과 락 성능 분석.
- **자가진단**:
  - 두 도구를 함께 사용하는 이유를 설명할 수 있나? (1~5점)
- **질문**: 캐시 갱신 시 Redisson 락을 사용하는 플로우를 설계해볼래?

---

## 🧪 통합 테스트 예시
RedisTemplate과 Redisson을 함께 사용하는 주문 시스템의 동시성 테스트입니다.

```java
@Test
void testConcurrentOrderWithCacheAndLock() throws InterruptedException {
    ExecutorService executor = Executors.newFixedThreadPool(2);
    CountDownLatch latch = new CountDownLatch(2);

    executor.submit(() -> {
        orderService.placeOrder(1L, 1L); // Redisson 락
        latch.countDown();
    });
    executor.submit(() -> {
        itemService.getPopularItems(); // RedisTemplate 캐싱
        latch.countDown();
    });

    latch.await(10, TimeUnit.SECONDS);
    assertEquals(1, redisTemplate.keys("cache:*").size()); // 캐시 확인
}
```

---

## 🔗 추가 참고 자료
- [Redis 공식 문서](https://redis.io/documentation): `SETNX`, Lua 스크립트, Redlock, Pub/Sub.
- [Spring Data Redis 문서](https://docs.spring.io/spring-data/redis/docs/current/reference/html/): RedisTemplate, Spring Cache.
- [Redisson Wiki](https://github.com/redisson/redisson/wiki): Redlock, Pub/Sub Lock.
- [Redis 분산락 가이드](https://notspoon.tistory.com/48).

---

## 🧠 자가진단 질문
- RedisTemplate과 Redisson의 역할 분리를 설명할 수 있나요? (1~5점)
- 캐시 스탬피드 방지 전략을 설계할 수 있나요? (1~5점)
- Redlock과 Lua 스크립트의 장점을 비교할 수 있나요? (1~5점)