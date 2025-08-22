# Redis Lua 스크립트 가이드: 왜 써야 하고 언제 필요한가

이 문서는 Redis의 Lua 스크립트가 무엇인지, 왜 사용해야 하는지, 어떤 상황에서 필수이고 어떤 때는 필요 없는지를 정리한 자료입니다. Spring 환경에서 RedisTemplate을 중심으로 초보자부터 현업 개발자까지 이해할 수 있도록 구성되었으며, 동시성 제어(락)와 캐싱에서의 활용을 강조합니다. 각 섹션은 개념 설명, 중요성, Spring 예시, 사고 확장 질문을 포함합니다.

---

## 📚 Lua 스크립트란?

**Lua 스크립트**는 Redis에서 Lua라는 경량 스크립팅 언어를 사용해 여러 Redis 명령어를 하나의 **원자적 작업**으로 실행하는 방법입니다. Redis는 단일 스레드로 동작하므로, Lua 스크립트는 실행 중 다른 클라이언트의 요청이 끼어들지 않도록 보장합니다. 이는 동시성 제어(락)와 캐싱에서 데이터 일관성과 성능을 보장하는 데 핵심적입니다.

- **핵심 특징**:
  - **원자성**: 여러 명령어를 하나의 작업으로 실행, 중간 간섭 방지.
  - **효율성**: 네트워크 호출을 줄여 성능 최적화.
  - **유연성**: 조건문, 반복문 등으로 복잡한 로직 처리.
- **Spring에서의 실행**:
  - **RedisTemplate**: Lua 스크립트를 직접 작성해 실행.
  - **Redisson**: 내부적으로 Lua를 사용해 락/캐싱 로Straight 처리.

**질문**:
- Redis에서 명령어를 하나씩 실행하면 어떤 문제가 생길 것 같아?
- Lua 스크립트가 "원자적"이라는 게 어떤 상황에서 중요할까?

---

## 🛠 Lua 스크립트를 왜 써야 하나?

Lua 스크립트를 사용하는 주된 이유는 **데이터 일관성**과 **성능 최적화**를 보장하기 위해서입니다. Redis는 빠르지만, 여러 명령어를 순차적으로 실행하면 동시성 이슈(Race Condition)나 네트워크 오버헤드가 발생할 수 있습니다. Lua 스크립트는 이를 해결합니다.

### 1. 원자성 보장
- **문제**: 여러 Redis 명령어를 따로 실행하면, 명령어 사이에 다른 클라이언트가 끼어들어 데이터 불일치 발생.
  - 예: `SETNX lock:user:123 value`로 락을 잡고, `EXPIRE lock:user:123 10`으로 타임아웃을 설정할 때, `SETNX` 후 `EXPIRE` 전에 서버가 크래시되면 락이 영구히 남아 데드락 발생.
- **Lua로 해결**: `SETNX`와 `EXPIRE`를 하나의 스크립트로 묶어 원자적으로 실행.
  ```lua
  if redis.call('SETNX', KEYS[1], ARGV[1]) == 1 then
      redis.call('EXPIRE', KEYS[1], ARGV[2])
      return 1
  else
      return 0
  end
  ```
- **Spring 예시** (RedisTemplate):
  ```java
  @Autowired
  private RedisTemplate<String, Object> redisTemplate;

  public boolean acquireLock(String lockKey, String lockValue, long timeout) {
      String script = "if redis.call('SETNX', KEYS[1], ARGV[1]) == 1 then " +
                      "redis.call('EXPIRE', KEYS[1], ARGV[2]) " +
                      "return 1 else return 0 end";
      RedisScript<Long> redisScript = RedisScript.of(script, Long.class);
      Long result = redisTemplate.execute(redisScript, List.of(lockKey), lockValue, String.valueOf(timeout));
      return result != null && result == 1;
  }
  ```
- **왜 중요?**: 원자성이 깨지면 데드락, 데이터 불일치 등 심각한 문제가 발생.

**질문**:
- `SETNX`와 `EXPIRE` 사이에 서버 크래시가 발생하면 어떤 문제가 생길까?
- Lua로 원자성을 보장하면 어떤 이점이 있을까?

### 2. 성능 최적화
- **문제**: 여러 Redis 명령어를 따로 호출하면 네트워크 왕복(RTT)이 늘어나 성능 저하.
  - 예: 락 획득(`SETNX`), 타임아웃 설정(`EXPIRE`), 값 확인(`GET`)을 따로 호출하면 3번의 네트워크 호출 필요.
- **Lua로 해결**: 모든 명령어를 하나의 스크립트로 묶어 단일 호출로 실행.
  - 위 락 획득 예시는 3번 호출 → 1번 호출로 줄어듦.
- **왜 중요?**: 대규모 트래픽 환경에서 네트워크 오버헤드는 시스템 성능에 큰 영향을 미침.

**질문**:
- 네트워크 호출이 많아지면 어떤 성능 문제가 생길까?
- Lua 스크립트로 호출을 줄이면 어떤 상황에서 특히 유리할까?

### 3. 복잡한 로직 처리
- **문제**: Redis 명령어로는 조건부 로직(예: "값이 특정 값일 때만 업데이트")을 구현하기 어려움.
  - 예: 락 해제 시 "내가 잡은 락인지 확인"하려면 `GET` 후 `DEL`을 호출해야 하지만, 중간에 다른 요청이 값을 변경할 수 있음.
- **Lua로 해결**: 조건문과 로직을 스크립트로 처리.
  ```lua
  if redis.call('GET', KEYS[1]) == ARGV[1] then
      redis.call('DEL', KEYS[1])
      return 1
  else
      return 0
  end
  ```
- **Spring 예시**:
  ```java
  public boolean releaseLock(String lockKey, String lockValue) {
      String script = "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
                      "redis.call('DEL', KEYS[1]) return 1 else return 0 end";
      RedisScript<Long> redisScript = RedisScript.of(script, Long.class);
      Long result = redisTemplate.execute(redisScript, List.of(lockKey), lockValue);
      return result != null && result == 1;
  }
  ```
- **왜 중요?**: 복잡한 로직을 Redis 서버에서 처리하면 클라이언트 코드가 간단해지고 안전성 향상.

**질문**:
- 락 해제 시 토큰을 확인하지 않으면 어떤 문제가 생길까?
- Lua로 복잡한 로직을 처리하면 클라이언트 코드에 어떤 이점이 있을까?

---

## 🚨 Lua 스크립트를 써야만 하는 상황

Lua 스크립트는 모든 상황에서 필수는 아니지만, 아래 경우에는 반드시 사용해야 해:

### 1. 동시성 제어가 필수적인 경우
- **상황**: 여러 클라이언트가 동시에 동일한 자원(재고, 잔액)을 수정.
  - 예: 재고 감소 로직에서 "재고 확인 → 감소 → 저장"을 따로 실행하면, 다른 요청이 끼어들어 재고가 음수가 될 수 있음.
  - **Lua 해결**:
    ```lua
    local stock = redis.call('GET', KEYS[1])
    if stock and tonumber(stock) > 0 then
        redis.call('DECR', KEYS[1])
        return 1
    else
        return 0
    end
    ```
  - **Spring 예시**:
    ```java
    public boolean decreaseStock(String stockKey) {
        String script = "local stock = redis.call('GET', KEYS[1]) " +
                       "if stock and tonumber(stock) > 0 then " +
                       "redis.call('DECR', KEYS[1]) return 1 else return 0 end";
        RedisScript<Long> redisScript = RedisScript.of(script, Long.class);
        Long result = redisTemplate.execute(redisScript, List.of(stockKey));
        return result != null && result == 1;
    }
    ```
- **왜 필수?**: Lua 없이 처리하면 Race Condition으로 데이터 불일치 발생.

**질문**:
- 재고 감소 시 Lua를 사용하지 않으면 어떤 문제가 생길까?
- 어떤 다른 시나리오에서 동시성 제어가 필요할까?

### 2. 락의 안전한 획득/해제
- **상황**: 락 설정(예: `SETNX` + `EXPIRE`)이나 해제(예: 토큰 확인 후 삭제) 시 원자성이 필요.
  - 예: `SETNX` 후 `EXPIRE` 전에 서버 크래시 → 락이 영구히 남음(데드락).
  - **Lua 해결**: 위 락 획득/해제 스크립트 참고.
- **왜 필수?**: 데드락은 시스템 전체를 멈추게 할 수 있음.

**질문**:
- 락이 영구히 남으면 어떤 문제가 생길까?
- Lua로 락 해제를 안전하게 처리하려면 어떤 로직이 필요할까?

### 3. 캐시 갱신의 데이터 일관성
- **상황**: 캐시 미스 시 DB 조회 후 캐시 저장 과정에서, 여러 요청이 동시에 DB를 조회하면 부하 증가(캐시 스탬피드).
  - **Lua 해결**:
    ```lua
    if redis.call('EXISTS', KEYS[1]) == 0 then
        redis.call('SET', KEYS[1], ARGV[1], 'EX', ARGV[2])
        return 1
    else
        return 0
    end
    ```
  - **Spring 예시**:
    ```java
    public boolean updateCacheIfMiss(String cacheKey, String value, long ttl) {
        String script = "if redis.call('EXISTS', KEYS[1]) == 0 then " +
                       "redis.call('SET', KEYS[1], ARGV[1], 'EX', ARGV[2]) " +
                       "return 1 else return 0 end";
        RedisScript<Long> redisScript = RedisScript.of(script, Long.class);
        Long result = redisTemplate.execute(redisScript, List.of(cacheKey), value, String.valueOf(ttl));
        return result != null && result == 1;
    }
    ```
- **왜 필수?**: 캐시 스탬피드는 DB에 과부하를 유발.

**질문**:
- 캐시 스탬피드가 발생하면 어떤 문제가 생길까?
- Lua로 캐시 갱신을 처리하면 어떤 이점이 있을까?

### 4. Redis Cluster 환경
- **상황**: Redis Cluster에서 여러 키를 다룰 때, 트랜잭션(`MULTI`/`EXEC`)은 키가 다른 슬롯에 있으면 동작하지 않음.
  - **Lua 해결**: Lua 스크립트는 단일 노드에서 실행되므로 여러 키를 원자적으로 처리.
- **왜 필수?**: 클러스터 환경에서 트랜잭션 대신 Lua가 안전한 대안.

**질문**:
- Redis Cluster에서 Lua를 사용하지 않으면 어떤 문제가 생길까?
- 여러 키를 다룰 때 Lua가 유리한 이유는?

---

## 🚫 Lua 스크립트를 쓰지 않아도 되는 상황

Lua 스크립트는 강력하지만, 아래 경우에는 필요 없을 수 있어:

### 1. 단순한 작업
- **상황**: 단일 Redis 명령어(예: `SET`, `GET`)로 충분한 경우.
  - 예: 간단한 캐시 조회(`GET key`)나 설정(`SET key value`).
- **왜 필요 없나?**: 단일 명령어는 이미 원자적이어서 Lua의 추가 오버헤드가 불필요.

### 2. 낮은 동시성
- **상황**: 동시 요청이 적어서 Race Condition 위험이 낮은 경우.
  - 예: 소규모 프로젝트에서 단일 사용자 데이터 조회.
- **왜 필요 없나?**: 동시성 이슈가 없으면 단순한 Redis 명령어로 충분.

### 3. Redisson 사용
- **상황**: Redisson으로 고급 락(Redlock, Pub/Sub Lock)이나 캐싱을 처리.
  - 예: Redisson의 `RLock`은 내부적으로 Lua를 사용해 락 로직 최적화.
  - **Spring 예시**:
    ```java
    @Autowired
    private RedissonClient redissonClient;

    public void charge(Long userId) {
        RLock lock = redissonClient.getLock("lock:user:" + userId);
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
- **왜 필요 없나?**: Redisson이 Lua를 내부적으로 처리하므로 직접 작성할 필요 없음.

**질문**:
- 너의 프로젝트에서 Lua 없이 충분한 작업은 어떤 게 있을까?
- Redisson을 사용하면 Lua 스크립트를 안 써도 되는 이유는?

---

## 📈 Lua 스크립트 학습 로드맵 (5시간)

Lua 스크립트 초보자를 위해 Spring 중심의 5시간 학습 로드맵을 제안합니다.

### Day 1: Lua 기초와 RedisTemplate (2시간)
- **활동**:
  - Redis 공식 문서에서 Lua 문법(`redis.call`, `KEYS`, `ARGV`) 읽기.
  - RedisTemplate으로 `SETNX` + `EXPIRE` Lua 스크립트 구현.
- **자가진단**:
  - Lua 스크립트의 기본 구조를 설명할 수 있나? (1~5점)
- **질문**: Lua에서 `KEYS`와 `ARGV`의 역할은 뭘까?

### Day 2: Lua로 락과 캐싱 구현 (3시간)
- **활동**:
  - Lua로 Fencing Token 기반 락 구현(예: 락 해제 시 토큰 확인).
  - Lua로 캐시 갱신 스크립트 작성(예: 캐시 미스 시 값 설정).
  - TestContainer로 동시 요청 테스트.
- **자가진단**:
  - Lua로 락 획득/해제 플로우를 설명할 수 있나? (1~5점)
- **질문**: Lua로 캐시 갱신을 원자적으로 처리하면 어떤 이점이 있을까?

---

## 🧪 통합 테스트 예시

Lua 스크립트를 사용한 락 테스트 코드입니다.

```java
@Test
void testLuaLock() throws InterruptedException {
    String lockKey = "lock:user:1";
    String lockValue = UUID.randomUUID().toString();
    String script = "if redis.call('SETNX', KEYS[1], ARGV[1]) == 1 then " +
                    "redis.call('EXPIRE', KEYS[1], ARGV[2]) return 1 else return 0 end";
    RedisScript<Long> redisScript = RedisScript.of(script, Long.class);

    ExecutorService executor = Executors.newFixedThreadPool(2);
    CountDownLatch latch = new CountDownLatch(2);

    executor.submit(() -> {
        boolean locked = redisTemplate.execute(redisScript, List.of(lockKey), lockValue, "10") == 1;
        assertTrue(locked);
        latch.countDown();
    });
    executor.submit(() -> {
        boolean locked = redisTemplate.execute(redisScript, List.of(lockKey), "anotherValue", "10") == 1;
        assertFalse(locked); // 락 획득 실패
        latch.countDown();
    });

    latch.await(5, TimeUnit.SECONDS);
}
```

---

## 🔗 추가 참고 자료
- [Redis 공식 문서 - Lua 스크립트](https://redis.io/commands/eval): `EVAL`, `SCRIPT LOAD`, `EVALSHA`.
- [Spring Data Redis 문서](https://docs.spring.io/spring-data/redis/docs/current/reference/html/): RedisTemplate으로 Lua 실행.
- [Lua 공식 문서](https://www.lua.org/docs.html): Lua 문법 기초.

---

## 🧠 자가진단 질문
- Lua 스크립트가 Redis에서 원자성을 보장하는 방법을 설명할 수 있나? (1~5점)
- Lua 스크립트를 써야 하는 상황을 구체적으로 설명할 수 있나? (1~5점)
- Lua 스크립트를 쓰지 않아도 되는 상황을 구분할 수 있나? (1~5점)