# Redisson 사용법 학습 자료: 락과 캐싱 기능 중심

## 1. Redisson 개요
Redisson은 Java 기반의 Redis 클라이언트 라이브러리로, Redis의 기능을 더 쉽게, 안전하게 사용할 수 있게 해줍니다. 특히 분산 환경에서 락(Lock)과 캐싱(Caching) 기능이 실무적으로 인기 있어요. 왜냐하면 Redis의 기본 API는 저수준이라 실무에서 에러가 발생하기 쉽기 때문이죠. Redisson은 이러한 부분을 추상화해줘서 개발 속도를 높이고, 안정성을 더합니다.

### 질문으로 시작하기
- Redisson을 왜 사용하려고 하나요? Redis 직접 사용과 비교했을 때 어떤 문제가 있었나요?
- 실무에서 락이나 캐싱이 필요한 상황을 하나 떠올려보세요. 예를 들어, 동시성 제어나 데이터 일관성 유지 같은 거요.

---

## 2. Redisson의 락 기능: 실무적으로 자주 쓰이는 것들
Redisson의 락 기능은 분산 시스템에서 데이터 일관성을 유지하기 위해 필수적입니다. 단일 서버가 아닌 여러 서버에서 동시에 접근할 때 충돌을 방지하죠. 아래는 사용 빈도가 높은 기능들로, 왜 쓰는지(목적), 뭐가 좋아서(장점), 어떻게 쓰는지(사용법)를 중점으로 설명했어요.

### 2.1. RLock (Reentrant Lock)
- **왜 쓰는지**: 여러 스레드나 서버가 공유 자원(예: 데이터베이스 레코드)에 동시에 접근하지 못하게 막기 위해. 예를 들어, 재고 업데이트나 사용자 프로필 수정 시 중복 처리를 방지.
- **뭐가 좋아서**: Redis 기반이라 분산 환경에서 안전하고, 재진입 가능(reentrant)해서 같은 스레드가 여러 번 락을 걸어도 데드락이 안 생김. 자동 만료(TTL) 기능으로 락이 영원히 걸리지 않음.
- **어떻게 쓰는지**: 
  - RedissonClient를 통해 RLock 인스턴스를 얻음.
  - `lock()`으로 락 획득, `unlock()`으로 해제.
  - 예시 코드 (Spring Boot 환경):
    ```java
    @Autowired
    private RedissonClient redissonClient;

    public void updateInventory(String itemId) {
        RLock lock = redissonClient.getLock("inventory:" + itemId);
        try {
            lock.lock(10, TimeUnit.SECONDS);  // 10초 TTL로 락 획득
            // 재고 업데이트 로직
        } finally {
            lock.unlock();  // 반드시 해제
        }
    }
    ```
  - 팁: `tryLock()`으로 타임아웃 설정 가능. 실무에서 데드락 방지를 위해 TTL 필수.

### 심화 질문
- RLock을 쓰지 않고 Redis의 SETNX로 직접 구현하면 어떤 문제가 생길 수 있을까? Redisson의 장점이 여기서 어떻게 드러날까?

### 2.2. RFairLock (Fair Lock)
- **왜 쓰는지**: 락을 기다리는 순서대로 공평하게 획득해야 할 때. 예: 작업 큐 처리나 자원 할당 시 선착순 보장.
- **뭐가 좋아서**: FIFO(First-In-First-Out) 방식으로 공정함. Redisson이 내부적으로 큐를 관리해줘서 구현이 간단.
- **어떻게 쓰는지**: RLock과 유사하지만 `getFairLock()` 사용.
  - 예시:
    ```java
    RFairLock fairLock = redissonClient.getFairLock("fairResource");
    fairLock.lock();
    // 작업 수행
    fairLock.unlock();
    ```

### 메타인지 질문
- Fair Lock을 쓸 때 공정성이 왜 중요한지, 당신의 프로젝트에서 적용할 수 있는 사례를 생각해볼래요?

### 2.3. RReadWriteLock (Read/Write Lock)
- **왜 쓰는지**: 읽기 작업은 여러 개 동시에 허용하고, 쓰기 작업만 배타적으로 할 때. 예: 캐시 데이터 읽기/쓰기 분리.
- **뭐가 좋아서**: 읽기 성능을 극대화(동시 읽기 가능)하면서 쓰기 안전성 보장. Redisson이 다중 락을 자동 관리.
- **어떻게 쓰는지**:
  - `getReadWriteLock()`으로 얻음.
  - 읽기: `readLock().lock()`, 쓰기: `writeLock().lock()`.
  - 예시:
    ```java
    RReadWriteLock rwLock = redissonClient.getReadWriteLock("dataKey");
    // 읽기
    rwLock.readLock().lock();
    // 데이터 읽기
    rwLock.readLock().unlock();
    // 쓰기
    rwLock.writeLock().lock();
    // 데이터 쓰기
    rwLock.writeLock().unlock();
    ```

### 응용 질문
- Read/Write Lock을 캐싱과 결합하면 어떤 이점이 있을까? 실무 예시를 상상해볼래요?

### 2.4. RedLock (Multi-Lock)
- **왜 쓰는지**: 여러 Redis 인스턴스에서 락을 걸어 고가용성 보장. 단일 Redis 실패 시에도 안전.
- **뭐가 좋아서**: SPOF 피함. 다수결 원칙으로 안정성 높음.
- **어떻게 쓰는지**: `getRedLock()`으로 여러 RLock 결합.
  - 예시:
    ```java
    RLock lock1 = client1.getLock("lock1");
    RLock lock2 = client2.getLock("lock1");
    RedissonRedLock redLock = new RedissonRedLock(lock1, lock2);
    redLock.lock();
    // 작업
    redLock.unlock();
    ```

---

## 3. Redisson의 캐싱 기능: 실무적으로 자주 쓰이는 것들
Redisson의 캐싱은 Redis를 메모리 캐시처럼 사용해 DB 부하를 줄입니다. 실무에서 API 응답 속도 향상이나 세션 관리에 자주 씁니다.

### 3.1. RMapCache (Map Cache with Eviction)
- **왜 쓰는지**: 키-값 캐시에 만료 시간(TTL)을 설정해 메모리 관리. 예: 사용자 세션 저장.
- **뭐가 좋아서**: 자동 만료(eviction)로 메모리 누수 방지. Redisson이 동기/비동기 지원.
- **어떻게 쓰는지**:
  - `getMapCache()`으로 얻음.
  - `put(key, value, ttl)`으로 저장.
  - 예시:
    ```java
    RMapCache<String, Object> cache = redissonClient.getMapCache("userCache");
    cache.put("user:1", userData, 30, TimeUnit.MINUTES);  // 30분 만료
    Object data = cache.get("user:1");
    ```

### 심화 질문
- TTL 없이 캐시를 쓰면 어떤 문제가 생길까? Redisson의 eviction이 어떻게 도와줄까?

### 3.2. RLocalCachedMap (Local Cache + Redis Sync)
- **왜 쓰는지**: 로컬 JVM 캐시와 Redis를 동기화해 지연 시간 최소화. 예: 자주 읽는 데이터.
- **뭐가 좋아서**: 로컬 캐시로 빠름 + Redis로 일관성 유지. 무효화(invalidation) 자동.
- **어떻게 쓰는지**:
  - `getLocalCachedMap()` 사용.
  - 옵션으로 로컬 캐시 크기 설정.
  - 예시:
    ```java
    LocalCachedMapOptions options = LocalCachedMapOptions.defaults()
        .evictionPolicy(LocalCachedMapOptions.EvictionPolicy.LRU)
        .cacheSize(1000);
    RLocalCachedMap<String, String> localCache = redisson.getLocalCachedMap("localCache", options);
    localCache.put("key", "value");
    ```

### 메타인지 질문
- Local Cache를 쓸 때 동기화가 왜 중요한지, 당신의 경험에서 떠올려보세요.

### 3.3. RBucket (Simple Key-Value Cache)
- **왜 쓰는지**: 간단한 단일 키 캐싱. 예: 설정 값 저장.
- **뭐가 좋아서**: 최소 오버헤드. Redisson의 TTL 지원으로 편함.
- **어떻게 쓰는지**:
  - `getBucket()`으로 얻음.
  - 예시:
    ```java
    RBucket<String> bucket = redissonClient.getBucket("configKey");
    bucket.set("value", 1, TimeUnit.DAYS);  // 1일 만료
    ```

---

## 4. 실무 팁과 베스트 프랙티스
- **연결 설정**: RedissonConfig에서 클러스터 모드 설정. 예: `config.useClusterServers().addNodeAddress("redis://node1:6379");`
- **에러 핸들링**: 락 실패 시 재시도 로직 추가.
- **테스트**: JUnit으로 모킹 테스트.

### 실전 연습 문제
1. RLock을 사용해 은행 이체 로직을 설계해보세요. 왜 이 기능이 적합할까?
2. RMapCache로 API 응답 캐싱을 구현해보세요. TTL을 어떻게 설정할지 고민해보세요.

### 메타인지 점검
- 이 자료 중 가장 이해하기 어려운 기능은 뭐예요? 왜 그럴까요?
- Redisson을 실제 프로젝트에 적용하려면 어떤 준비가 필요할까?

---

## 5. 추가 학습 자료
- Redisson GitHub Wiki: https://github.com/redisson/redisson/wiki
- 실습: 로컬 Redis 띄우고 Spring Boot 프로젝트에서 테스트해보세요.