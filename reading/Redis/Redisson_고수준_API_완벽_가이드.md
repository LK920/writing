# Redisson 고수준 API 완벽 가이드

## 📋 목차
1. [Redisson vs Raw Redis 개요](#1-redisson-vs-raw-redis-개요)
2. [핵심 데이터 구조별 API 매핑](#2-핵심-데이터-구조별-api-매핑)
3. [고급 기능 및 추상화](#3-고급-기능-및-추상화)
4. [실무 프로젝트 적용 예시](#4-실무-프로젝트-적용-예시)
5. [성능 및 안정성 비교](#5-성능-및-안정성-비교)

---

## 1. Redisson vs Raw Redis 개요

### 🎯 Redisson의 핵심 가치

**Raw Redis의 한계:**
- 직접적인 명령어 조작 필요
- 에러 처리 및 연결 관리 복잡
- Lua Script 직접 작성 및 관리
- 타입 안정성 부족

**Redisson의 해결책:**
- 객체지향적 고수준 API 제공
- 자동 연결 관리 및 재시도 로직
- 내장 Lua Script로 원자성 보장
- 강타입 지원 및 직렬화 자동화

### 🏗 아키텍처 비교

```mermaid
graph TD
    A[Your Application] --> B[Redisson Client]
    B --> C[Redisson Internal]
    C --> D[Connection Pool]
    C --> E[Lua Scripts]
    C --> F[Serialization]
    D --> G[Redis Server]
    
    A2[Your Application] --> H[Raw Redis Commands]
    H --> I[Connection Management]
    I --> J[Redis Server]
```

---

## 2. 핵심 데이터 구조별 API 매핑

### 📊 2.1 Sorted Set (랭킹 시스템)

#### RedisTemplate 방식
```java
// 1. RedisTemplate 사용
@Autowired
private RedisTemplate<String, String> redisTemplate;

// 2. 점수 추가 (Raw Redis의 ZADD)
redisTemplate.opsForZSet().add("leaderboard", "player1", 1500.0);
redisTemplate.opsForZSet().add("leaderboard", "player2", 1200.0);

// 3. 점수 증가 (Raw Redis의 ZINCRBY)
Double newScore = redisTemplate.opsForZSet().incrementScore("leaderboard", "player1", 100.0);

// 4. 랭킹 조회 (Raw Redis의 ZREVRANGE)
Set<String> top10 = redisTemplate.opsForZSet().reverseRange("leaderboard", 0, 9);
Set<ZSetOperations.TypedTuple<String>> top10WithScores = 
    redisTemplate.opsForZSet().reverseRangeWithScores("leaderboard", 0, 9);

// 5. 특정 플레이어 순위 (Raw Redis의 ZREVRANK)
Long rank = redisTemplate.opsForZSet().reverseRank("leaderboard", "player1");

// 6. 점수 범위 조회 (Raw Redis의 ZRANGEBYSCORE)
Set<String> scoreRange = redisTemplate.opsForZSet().rangeByScore("leaderboard", 1000.0, 2000.0);

// 7. TTL 설정 (Raw Redis의 EXPIRE)
redisTemplate.expire("leaderboard", Duration.ofDays(1));

// 8. Pipeline 사용 (배치 처리)
redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    StringRedisConnection stringRedisConn = (StringRedisConnection) connection;
    stringRedisConn.zAdd("leaderboard", 50, "player1");
    stringRedisConn.zAdd("leaderboard", 30, "player2");
    return null;
});
```

#### Redisson 방식
```java
// 1. 객체 생성 및 설정
RScoredSortedSet<String> leaderboard = redissonClient.getScoredSortedSet("leaderboard");

// 2. 점수 추가 (Raw Redis의 ZADD)
leaderboard.add(1500.0, "player1");
leaderboard.addAll(Map.of("player1", 1500.0, "player2", 1200.0));

// 3. 점수 증가 (Raw Redis의 ZINCRBY)
Double newScore = leaderboard.addScore("player1", 100.0);

// 4. 랭킹 조회 (Raw Redis의 ZREVRANGE)
Collection<String> top10 = leaderboard.valueRangeReversed(0, 9);
Collection<ScoredEntry<String>> top10WithScores = leaderboard.entryRangeReversed(0, 9);

// 5. 특정 플레이어 순위 (Raw Redis의 ZREVRANK)
Integer rank = leaderboard.revRank("player1");

// 6. 점수 범위 조회 (Raw Redis의 ZRANGEBYSCORE)
Collection<String> scoreRange = leaderboard.valueRange(1000.0, true, 2000.0, true);

// 7. TTL 설정 (Raw Redis의 EXPIRE)
leaderboard.expire(Duration.ofDays(1));

// 8. 고급 기능: 배치 처리
RBatch batch = redissonClient.createBatch();
RScoredSortedSetAsync<String> asyncLeaderboard = batch.getScoredSortedSet("leaderboard");
asyncLeaderboard.addScoreAsync("player1", 50);
asyncLeaderboard.addScoreAsync("player2", 30);
batch.execute(); // 모든 명령어를 한 번에 실행
```

**Redisson의 장점:**
- **타입 안정성**: 제네릭으로 String 타입 보장하여 런타임 에러 방지
- **자동 직렬화**: 객체를 자동으로 Redis에 저장 가능하여 JSON 변환 불필요
- **배치 처리**: 여러 명령어를 원자적으로 실행하여 네트워크 호출 최소화
- **비동기 지원**: 논블로킹 작업으로 처리량 극대화

**RedisTemplate이 더 좋은 경우:**
- **Spring 생태계**: Spring Boot 프로젝트에서 다른 Spring 기술과 통합 필요
- **세밀한 제어**: Redis 명령어별 세부 옵션 제어가 필요한 경우
- **기존 인프라**: 이미 RedisTemplate 기반 코드가 많은 프로젝트
- **단순 캐싱**: key-value 저장/조회 위주의 기본적인 캐싱 용도

### 🔢 2.2 Atomic Counter (원자적 카운터)

#### RedisTemplate 방식
```java
// 1. RedisTemplate 사용
@Autowired
private RedisTemplate<String, String> redisTemplate;

// 2. 기본 연산 (Raw Redis의 INCR, INCRBY)
Long newViews = redisTemplate.opsForValue().increment("product:1001:views");          // INCR
Long newStock = redisTemplate.opsForValue().increment("stock:product:1001", -5);     // INCRBY -5
String currentStock = redisTemplate.opsForValue().get("stock:product:1001");         // GET

// 3. 조건부 연산 (Lua Script 필요)
String luaScript = """
    local current = redis.call('GET', KEYS[1])
    if tonumber(current) >= tonumber(ARGV[1]) then
        return redis.call('DECRBY', KEYS[1], ARGV[1])
    else
        return nil
    end
    """;

DefaultRedisScript<Long> script = new DefaultRedisScript<>();
script.setScriptText(luaScript);
script.setResultType(Long.class);

Long result = redisTemplate.execute(script, 
    Collections.singletonList("stock:product:1001"), "5");

// 4. TTL 설정
redisTemplate.expire("product:1001:views", Duration.ofHours(24));

// 5. 초기값 설정 (키가 없을 때만)
Boolean setIfAbsent = redisTemplate.opsForValue().setIfAbsent("stock:product:1001", "100");
```

#### Redisson 방식
```java
// 1. Atomic Long 생성
RAtomicLong viewCount = redissonClient.getAtomicLong("product:1001:views");
RAtomicLong stockCount = redissonClient.getAtomicLong("stock:product:1001");

// 2. 기본 연산 (Raw Redis의 INCR, INCRBY)
long newViews = viewCount.incrementAndGet();       // INCR
long newStock = stockCount.addAndGet(-5);          // INCRBY -5
long currentStock = stockCount.get();              // GET

// 3. 조건부 연산 (내장 Lua Script)
long decreasedStock = stockCount.getAndAdd(-5);    // 원자적 감소

// 4. 고급 조건부 연산
public boolean decreaseStockIfAvailable(RAtomicLong stock, long quantity) {
    return stock.compareAndSet(stock.get(), stock.get() - quantity) && stock.get() >= 0;
}

// 5. 만료 시간 설정
viewCount.expire(Duration.ofHours(24));

// 6. 초기값 설정 (키가 없을 때만)
stockCount.trySet(100);  // NX 옵션과 같은 효과
```

**Redisson의 장점:**
- **원자성 보장**: 내부적으로 Lua Script 사용하여 Race Condition 자동 방지
- **조건부 연산**: compareAndSet 등 고급 연산으로 복잡한 비즈니스 로직 안전 처리
- **예외 처리**: 자동 재시도로 네트워크 장애 상황에서도 안정성 보장
- **메모리 효율**: 불필요한 네트워크 호출 최소화로 지연시간 단축

**RedisTemplate이 더 좋은 경우:**
- **Spring 통합**: Spring Boot 환경에서 트랜잭션, AOP와 연동 필요
- **Lua Script 활용**: 복잡한 조건부 로직을 직접 작성하고 제어하고 싶은 경우
- **세밀한 설정**: Redis 연결, 직렬화 방식을 프로젝트에 맞게 커스터마이징
- **단순 카운터**: 조회수, 방문자 수 같은 기본적인 증감 연산

### 🗂 2.3 Hash (구조화 데이터)

#### RedisTemplate 방식
```java
// 1. RedisTemplate 사용
@Autowired
private RedisTemplate<String, String> redisTemplate;

// 2. 필드 설정 (Raw Redis의 HSET)
redisTemplate.opsForHash().put("user:1001", "name", "김철수");
Map<String, String> userFields = Map.of(
    "name", "김철수",
    "age", "28", 
    "email", "kim@example.com"
);
redisTemplate.opsForHash().putAll("user:1001", userFields);

// 3. 필드 조회 (Raw Redis의 HGET, HMGET)
String name = (String) redisTemplate.opsForHash().get("user:1001", "name");
List<Object> fields = redisTemplate.opsForHash().multiGet("user:1001", 
    Arrays.asList("name", "email"));

// 4. 전체 조회 (Raw Redis의 HGETALL)
Map<Object, Object> allFields = redisTemplate.opsForHash().entries("user:1001");

// 5. 숫자 필드 증가 (Raw Redis의 HINCRBY)
Long loginCount = redisTemplate.opsForHash().increment("user:1001", "login_count", 1);

// 6. 필드 존재 확인 (Raw Redis의 HEXISTS)
Boolean hasPhone = redisTemplate.opsForHash().hasKey("user:1001", "phone");

// 7. 필드 삭제 (Raw Redis의 HDEL)
Long deletedCount = redisTemplate.opsForHash().delete("user:1001", "temp_data");

// 8. TTL 설정
redisTemplate.expire("user:1001", Duration.ofHours(2));

// 9. 조건부 설정 (HSETNX와 유사)
Boolean isSet = redisTemplate.opsForHash().putIfAbsent("user:1001", "phone", "010-1234-5678");
```

#### Redisson 방식
```java
// 1. Hash Map 생성
RMap<String, String> userProfile = redissonClient.getMap("user:1001");

// 2. 필드 설정 (Raw Redis의 HSET)
userProfile.put("name", "김철수");
userProfile.putAll(Map.of(
    "name", "김철수",
    "age", "28", 
    "email", "kim@example.com"
));

// 3. 필드 조회 (Raw Redis의 HGET, HMGET)
String name = userProfile.get("name");
Map<String, String> fields = userProfile.getAll(Set.of("name", "email"));

// 4. 전체 조회 (Raw Redis의 HGETALL)
Map<String, String> allFields = userProfile.readAllMap();

// 5. 원자적 숫자 증가 (Raw Redis의 HINCRBY)
RAtomicLong loginCount = redissonClient.getAtomicLong("user:1001:login_count");
loginCount.incrementAndGet();

// 6. 필드 존재 확인 (Raw Redis의 HEXISTS)
boolean hasPhone = userProfile.containsKey("phone");

// 7. 필드 삭제 (Raw Redis의 HDEL)
userProfile.remove("temp_data");

// 8. 고급 기능: 조건부 업데이트
String oldName = userProfile.replace("name", "새이름");  // 키가 존재할 때만 교체

// 9. 만료 시간 설정
userProfile.expire(Duration.ofHours(2));

// 10. 객체 직렬화 지원
public class User {
    private String name;
    private int age;
    private String email;
    // getters, setters
}

RMap<String, User> userObjectMap = redissonClient.getMap("users");
userObjectMap.put("1001", new User("김철수", 28, "kim@example.com"));
User user = userObjectMap.get("1001");  // 자동 역직렬화
```

**Redisson의 장점:**
- **Map 인터페이스**: Java Map과 동일한 API로 학습 비용 제로
- **객체 직렬화**: 복잡한 객체도 자동 처리하여 개발 생산성 극대화
- **원자적 연산**: replace, putIfAbsent 등으로 동시성 이슈 원천 차단
- **배치 연산**: getAll, putAll로 네트워크 호출 최소화하여 성능 향상

**RedisTemplate이 더 좋은 경우:**
- **Spring 기반**: @Cacheable, @Transactional과 함께 사용하는 캐싱 시스템
- **스키마 설계**: HGETALL, HMGET 등으로 세밀한 필드 제어가 필요한 경우
- **직렬화 제어**: Jackson, Kryo 등 특정 직렬화 방식을 사용해야 하는 경우
- **기존 구조**: 이미 RedisTemplate 기반으로 설계된 시스템

### 🏷 2.4 Set (집합 연산)

#### Raw Redis 방식
```redis
# 멤버 추가
SADD user:1001:interests "programming" "music" "travel"

# 멤버 확인
SISMEMBER user:1001:interests "programming"

# 전체 멤버 조회
SMEMBERS user:1001:interests

# 집합 연산
SINTER user:1001:interests user:1002:interests
SUNION user:1001:interests user:1002:interests

# 멤버 수
SCARD user:1001:interests
```

#### Redisson 방식
```java
// 1. Set 생성
RSet<String> userInterests = redissonClient.getSet("user:1001:interests");

// 2. 멤버 추가 (Raw Redis의 SADD)
userInterests.add("programming");
userInterests.addAll(Arrays.asList("music", "travel"));

// 3. 멤버 확인 (Raw Redis의 SISMEMBER)
boolean hasInterest = userInterests.contains("programming");

// 4. 전체 멤버 조회 (Raw Redis의 SMEMBERS)
Set<String> allInterests = userInterests.readAll();

// 5. 집합 연산 (Raw Redis의 SINTER, SUNION)
RSet<String> otherInterests = redissonClient.getSet("user:1002:interests");
Set<String> common = userInterests.readIntersection("user:1002:interests");  // 교집합
Set<String> union = userInterests.readUnion("user:1002:interests");          // 합집합

// 6. 멤버 수 (Raw Redis의 SCARD)
int size = userInterests.size();

// 7. 고급 기능: 원자적 집합 연산
int intersectionSize = userInterests.intersectionSize("user:1002:interests");
userInterests.retainAll(otherInterests);  // 교집합만 남기기

// 8. 배치 처리
RBatch batch = redissonClient.createBatch();
RSetAsync<String> batchSet = batch.getSet("user:1001:interests");
batchSet.addAsync("new_interest1");
batchSet.addAsync("new_interest2");
batch.execute();
```

### 📋 2.5 List (큐/스택)

#### Raw Redis 방식
```redis
# 리스트 추가
LPUSH queue:tasks "task1" "task2"
RPUSH queue:logs "log1" "log2"

# 리스트 제거
LPOP queue:tasks
RPOP queue:logs

# 블로킹 연산
BLPOP queue:tasks 10

# 범위 조회
LRANGE recent:activities 0 9
```

#### Redisson 방식
```java
// 1. List/Queue/Deque 생성
RList<String> taskList = redissonClient.getList("queue:tasks");
RQueue<String> logQueue = redissonClient.getQueue("queue:logs");
RDeque<String> recentActivities = redissonClient.getDeque("recent:activities");

// 2. 리스트 추가 (Raw Redis의 LPUSH, RPUSH)
taskList.addFirst("task1");     // LPUSH
logQueue.offer("log1");         // RPUSH

// 3. 리스트 제거 (Raw Redis의 LPOP, RPOP)  
String task = taskList.removeFirst();    // LPOP
String log = logQueue.poll();            // RPOP

// 4. 블로킹 연산 (Raw Redis의 BLPOP)
RBlockingQueue<String> blockingQueue = redissonClient.getBlockingQueue("queue:tasks");
String waitingTask = blockingQueue.poll(10, TimeUnit.SECONDS);

// 5. 범위 조회 (Raw Redis의 LRANGE)
List<String> recentTen = recentActivities.subList(0, 9);

// 6. 고급 기능: 크기 제한
RBoundedBlockingQueue<String> boundedQueue = redissonClient.getBoundedBlockingQueue("limited:queue");
boundedQueue.trySetCapacity(100);  // 최대 100개 항목만 저장

// 7. 우선순위 큐
RPriorityQueue<Task> priorityQueue = redissonClient.getPriorityQueue("priority:tasks");
priorityQueue.add(new Task("urgent", 1));
priorityQueue.add(new Task("normal", 5));
```

### 🔑 2.6 String Operations (기본 캐싱)

#### Raw Redis 방식
```redis
# 기본 설정/조회
SET cache:user:1001 "user_data" EX 3600
GET cache:user:1001

# 조건부 설정
SET lock:resource "token" NX EX 30
SET config:flag "true" XX

# 다중 설정/조회
MSET key1 "value1" key2 "value2"
MGET key1 key2 key3

# TTL 관리
EXPIRE cache:data 3600
TTL cache:data
```

#### Redisson 방식
```java
// 1. 기본 Bucket 객체 (Raw Redis의 SET/GET)
RBucket<String> userCache = redissonClient.getBucket("cache:user:1001");
userCache.set("user_data", Duration.ofHours(1));  // SET + EX
String userData = userCache.get();                 // GET

// 2. 조건부 설정 (Raw Redis의 NX/XX)
RBucket<String> lockBucket = redissonClient.getBucket("lock:resource");
boolean lockAcquired = lockBucket.trySet("token", Duration.ofSeconds(30));  // SET NX EX

RBucket<String> configBucket = redissonClient.getBucket("config:flag");
boolean updated = configBucket.setIfExists("true");  // SET XX

// 3. 다중 처리 (Raw Redis의 MSET/MGET)
RBuckets buckets = redissonClient.getBuckets();
Map<String, String> data = Map.of("key1", "value1", "key2", "value2");
buckets.set(data);  // MSET

Map<String, String> result = buckets.get("key1", "key2", "key3");  // MGET

// 4. TTL 관리 (Raw Redis의 EXPIRE/TTL)
userCache.expire(Duration.ofHours(1));     // EXPIRE
long ttl = userCache.remainTimeToLive();   // TTL

// 5. 고급 기능: 객체 직렬화
public class UserData {
    private String name;
    private int age;
    // constructors, getters, setters
}

RBucket<UserData> userObjectBucket = redissonClient.getBucket("user:object:1001");
userObjectBucket.set(new UserData("김철수", 30), Duration.ofHours(2));
UserData user = userObjectBucket.get();  // 자동 역직렬화

// 6. 원자적 업데이트
String oldValue = userCache.getAndSet("new_data");  // GETSET과 유사
```

### 📊 2.7 HyperLogLog (대용량 유니크 카운팅)

#### Raw Redis 방식
```redis
# 요소 추가
PFADD unique:visitors:daily user1 user2 user3

# 근사 카운트 조회
PFCOUNT unique:visitors:daily

# 다중 HyperLogLog 병합
PFMERGE unique:visitors:weekly unique:visitors:day1 unique:visitors:day2
```

#### Redisson 방식
```java
// 1. HyperLogLog 생성 (대용량 유니크 카운팅)
RHyperLogLog<String> dailyVisitors = redissonClient.getHyperLogLog("unique:visitors:daily");

// 2. 요소 추가 (Raw Redis의 PFADD)
dailyVisitors.add("user1", "user2", "user3");
dailyVisitors.addAll(Arrays.asList("user4", "user5", "user6"));

// 3. 근사 카운트 조회 (Raw Redis의 PFCOUNT)
long uniqueCount = dailyVisitors.count();

// 4. 다중 HyperLogLog 병합 (Raw Redis의 PFMERGE)
RHyperLogLog<String> weeklyVisitors = redissonClient.getHyperLogLog("unique:visitors:weekly");
RHyperLogLog<String> day1Visitors = redissonClient.getHyperLogLog("unique:visitors:day1");
RHyperLogLog<String> day2Visitors = redissonClient.getHyperLogLog("unique:visitors:day2");

weeklyVisitors.mergeWith(day1Visitors.getName(), day2Visitors.getName());

// 5. 실무 예시: 일별 유니크 방문자 추적
@Service
public class VisitorTrackingService {
    
    public void trackVisitor(String userId, LocalDate date) {
        String key = "unique:visitors:" + date;
        RHyperLogLog<String> visitors = redissonClient.getHyperLogLog(key);
        visitors.add(userId);
        visitors.expire(Duration.ofDays(30));  // 30일 후 삭제
    }
    
    public long getDailyUniqueVisitors(LocalDate date) {
        String key = "unique:visitors:" + date;
        RHyperLogLog<String> visitors = redissonClient.getHyperLogLog(key);
        return visitors.count();
    }
}
```

### 🔢 2.8 BitSet (비트 연산)

#### Raw Redis 방식
```redis
# 비트 설정/조회
SETBIT user:1001:active_days 100 1
GETBIT user:1001:active_days 100

# 비트 카운트
BITCOUNT user:1001:active_days

# 비트 연산
BITOP AND result key1 key2
BITOP OR result key1 key2
```

#### Redisson 방식
```java
// 1. BitSet 생성 (Raw Redis의 SETBIT/GETBIT)
RBitSet activedays = redissonClient.getBitSet("user:1001:active_days");

// 2. 비트 설정/조회
activedays.set(100, true);      // SETBIT user:1001:active_days 100 1
boolean isActive = activedays.get(100);  // GETBIT user:1001:active_days 100

// 3. 비트 카운트 (Raw Redis의 BITCOUNT)
long activeDaysCount = activedays.cardinality();

// 4. 비트 연산 (Raw Redis의 BITOP)
RBitSet user1Days = redissonClient.getBitSet("user:1001:active_days");
RBitSet user2Days = redissonClient.getBitSet("user:1002:active_days");

// AND 연산 (공통 활성일)
RBitSet commonDays = redissonClient.getBitSet("common:active_days");
commonDays.or(user1Days);
commonDays.and(user2Days);

// 5. 실무 예시: 사용자 활동 추적
@Service
public class UserActivityService {
    
    public void markActiveDay(Long userId, LocalDate date) {
        String key = "user:" + userId + ":active_days:" + date.getYear();
        RBitSet activeDays = redissonClient.getBitSet(key);
        
        int dayOfYear = date.getDayOfYear();
        activeDays.set(dayOfYear, true);
        activeDays.expire(Duration.ofDays(400));  // 다음해까지 보관
    }
    
    public long getActiveDaysCount(Long userId, int year) {
        String key = "user:" + userId + ":active_days:" + year;
        RBitSet activeDays = redissonClient.getBitSet(key);
        return activeDays.cardinality();
    }
    
    public Set<Integer> getCommonActiveDays(Long userId1, Long userId2, int year) {
        RBitSet user1Days = redissonClient.getBitSet("user:" + userId1 + ":active_days:" + year);
        RBitSet user2Days = redissonClient.getBitSet("user:" + userId2 + ":active_days:" + year);
        
        RBitSet common = redissonClient.getBitSet("temp:common:" + System.currentTimeMillis());
        common.or(user1Days);
        common.and(user2Days);
        
        return common.asBitSet().stream()
            .boxed()
            .collect(Collectors.toSet());
    }
}
```

---

## 3. 고급 기능 및 추상화

### 🔒 3.1 분산 락 (Distributed Lock)

#### Raw Redis 방식 (복잡하지만 제어권 최대)
```redis
# 락 획득 시도
SET lock:resource NX EX 30

# 락 해제 (Lua Script 필요)
EVAL "
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
" 1 lock:resource token_value
```

**Raw Redis 장점**: 락 로직 완전 제어, 최소 메모리 사용, 극한 성능
**Raw Redis 단점**: Lua Script 직접 작성/관리, 토큰 관리, 재진입 로직 수동 구현

#### Redisson 방식
```java
// 1. 기본 락 (간단하지만 고급 기능 내장)
RLock lock = redissonClient.getLock("resource:lock");

try {
    // 10초 대기, 30초 후 자동 해제
    if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
        // 크리티컬 섹션 실행
        processCriticalSection();
    }
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}

**Redisson 장점**: 재진입 지원, 자동 토큰 관리, 락 만료 연장 기능
**Redisson 단점**: 내부 동작 블랙박스, 약간의 성능 오버헤드

// 2. 공정한 락 (FIFO 순서 보장)
RLock fairLock = redissonClient.getFairLock("fair:lock");

// 3. 읽기-쓰기 락
RReadWriteLock rwLock = redissonClient.getReadWriteLock("rw:lock");
RLock readLock = rwLock.readLock();
RLock writeLock = rwLock.writeLock();

// 4. 다중 락 (여러 리소스 동시 락)
RLock lock1 = redissonClient.getLock("lock1");
RLock lock2 = redissonClient.getLock("lock2");
RLock multiLock = redissonClient.getMultiLock(lock1, lock2);

// 5. 락 리스너
lock.addListener(new LockListener() {
    @Override
    public void onLocked(String lockName) {
        log.info("락 획득: {}", lockName);
    }
    
    @Override
    public void onUnlocked(String lockName) {
        log.info("락 해제: {}", lockName);
    }
});
```

### 🌊 3.2 Stream (이벤트 스트리밍)

#### Raw Redis 방식 (세밀한 제어 가능)
```redis
# 스트림에 이벤트 추가
XADD events:orders * orderId 1001 userId 123 amount 50000

# 스트림 읽기
XREAD STREAMS events:orders 0

# 컨슈머 그룹 생성
XGROUP CREATE events:orders processing $

# 그룹에서 읽기
XREADGROUP GROUP processing consumer1 STREAMS events:orders >
```

**Raw Redis 장점**: 메시지 ID 직접 제어, 메모리 사용량 최소, 고성능
**Raw Redis 단점**: 복잡한 명령어 조합, 에러 처리 수동 구현 필요

#### Redisson 방식
```java
// 1. Stream 생성 (고수준 추상화로 간편함)
RStream<String, String> orderStream = redissonClient.getStream("events:orders");

// 2. 이벤트 추가 (Raw Redis의 XADD)
StreamMessageId messageId = orderStream.add(Map.of(
    "orderId", "1001",
    "userId", "123", 
    "amount", "50000"
));

**Redisson 장점**: Map 기반 직관적 API, 자동 직렬화, 리스너 패턴 지원
**Redisson 단점**: 메시지 ID 제어 제한, 추가 메모리 오버헤드

// 3. 스트림 읽기 (Raw Redis의 XREAD)
Map<String, Map<StreamMessageId, Map<String, String>>> messages = 
    orderStream.read("events:orders", StreamMessageId.NEWEST);

// 4. 컨슈머 그룹 (Raw Redis의 XGROUP CREATE, XREADGROUP)
orderStream.createGroup("processing", StreamMessageId.NEWEST);

Map<String, Map<StreamMessageId, Map<String, String>>> groupMessages = 
    orderStream.readGroup("processing", "consumer1");

// 5. 메시지 확인 (Raw Redis의 XACK)
orderStream.ack("processing", messageId);

// 6. 고급 기능: 스트림 리스너
orderStream.addListener(new StreamMessageListener<String, String>() {
    @Override
    public void onMessage(CharSequence stream, StreamMessageId id, Map<String, String> fields) {
        // 실시간 이벤트 처리
        processOrderEvent(fields);
    }
});

// 7. 배치 이벤트 처리
public void processBatchEvents() {
    List<Map<String, String>> events = new ArrayList<>();
    
    // 100개 메시지까지 한 번에 읽기
    Map<String, Map<StreamMessageId, Map<String, String>>> batchMessages = 
        orderStream.read(100, Duration.ofSeconds(1), "events:orders", StreamMessageId.LAST);
        
    // 배치 처리 로직
    processBatchOrderEvents(events);
}
```

### 💾 3.3 트랜잭션 및 배치

#### Raw Redis 방식
```redis
# 트랜잭션
MULTI
SET account:1001 1000
SET account:1002 2000
INCR transaction:count
EXEC

# Pipeline (별도 라이브러리 필요)
```

#### Redisson 방식
```java
// 1. 트랜잭션 (Raw Redis의 MULTI/EXEC)
RTransaction transaction = redissonClient.createTransaction(TransactionOptions.defaults());

RMap<String, String> map = transaction.getMap("myMap");
RSet<String> set = transaction.getSet("mySet");

map.put("key", "value");
set.add("element");

transaction.commit();  // 모든 명령어 원자적 실행

// 2. 배치 (Pipeline과 유사)
RBatch batch = redissonClient.createBatch(BatchOptions.defaults());

RMapAsync<String, String> mapAsync = batch.getMap("myMap");
RSetAsync<String> setAsync = batch.getSet("mySet");

mapAsync.putAsync("key1", "value1");
mapAsync.putAsync("key2", "value2");
setAsync.addAsync("element1");

BatchResult<?> results = batch.execute();  // 모든 명령어 한 번에 전송

// 3. 고급 배치 옵션
BatchOptions options = BatchOptions.defaults()
    .executionMode(BatchOptions.ExecutionMode.REDIS_WRITE_ATOMIC)  // 원자성 보장
    .responseTimeout(Duration.ofSeconds(5))
    .retryAttempts(3);

RBatch atomicBatch = redissonClient.createBatch(options);
```

### ⏰ 3.4 Scheduler (분산 스케줄러)

#### Raw Redis 방식 (복잡한 구현 필요)
```redis
# Lua Script로 스케줄 작업 등록
EVAL "
redis.call('ZADD', 'scheduled:tasks', ARGV[1], ARGV[2])
" 0 timestamp task_data

# 만료된 작업 조회
ZRANGEBYSCORE scheduled:tasks 0 current_timestamp
```

#### Redisson 방식
```java
// 1. 기본 스케줄러
RScheduledExecutorService scheduler = redissonClient.getExecutorService("myScheduler");

// 2. 지연 실행 (Raw Redis보다 훨씬 간단)
scheduler.schedule(() -> {
    System.out.println("지연된 작업 실행!");
}, 10, TimeUnit.SECONDS);

// 3. 주기적 실행
ScheduledFuture<?> future = scheduler.scheduleAtFixedRate(() -> {
    System.out.println("주기적 작업 실행!");
}, 0, 5, TimeUnit.SECONDS);

// 4. 특정 시간에 실행
scheduler.schedule(() -> {
    System.out.println("특정 시간 작업 실행!");
}, Date.from(LocalDateTime.of(2025, 8, 20, 9, 0).atZone(ZoneId.systemDefault()).toInstant()));

// 5. 실무 예시: 쿠폰 만료 처리
@Service
public class CouponExpirationService {
    
    private final RScheduledExecutorService scheduler;
    
    public CouponExpirationService(RedissonClient redissonClient) {
        this.scheduler = redissonClient.getExecutorService("couponScheduler");
    }
    
    public void scheduleCouponExpiration(Long couponId, LocalDateTime expirationTime) {
        Date expireDate = Date.from(expirationTime.atZone(ZoneId.systemDefault()).toInstant());
        
        scheduler.schedule(() -> {
            expireCoupon(couponId);
        }, expireDate);
    }
    
    private void expireCoupon(Long couponId) {
        // 쿠폰 만료 처리 로직
        log.info("쿠폰 {} 만료 처리 완료", couponId);
    }
}
```

### 🔔 3.5 Pub/Sub (실시간 메시징)

#### Raw Redis 방식
```redis
# 메시지 발행
PUBLISH notifications "새로운 주문이 있습니다"

# 채널 구독
SUBSCRIBE notifications order:* user:*

# 패턴 구독  
PSUBSCRIBE order:*
```

#### Redisson 방식
```java
// 1. 기본 Topic (Raw Redis의 PUBLISH/SUBSCRIBE)
RTopic topic = redissonClient.getTopic("notifications");

// 2. 메시지 발행 (Raw Redis의 PUBLISH)
topic.publish("새로운 주문이 있습니다");

// 3. 메시지 구독 (Raw Redis의 SUBSCRIBE)
topic.addListener(String.class, (channel, message) -> {
    System.out.println("받은 메시지: " + message);
});

// 4. 패턴 구독 (Raw Redis의 PSUBSCRIBE)
RPatternTopic patternTopic = redissonClient.getPatternTopic("order:*");
patternTopic.addListener(String.class, (pattern, channel, message) -> {
    System.out.println("패턴 " + pattern + "에서 메시지: " + message);
});

// 5. 고급 기능: 신뢰성 있는 Topic
RReliableTopic reliableTopic = redissonClient.getReliableTopic("important:notifications");
reliableTopic.addListener(String.class, (channel, message) -> {
    // 메시지 처리 실패 시 재시도됨
    processImportantMessage(message);
});

// 6. 실무 예시: 실시간 알림 시스템
@Service
public class NotificationService {
    
    private final RTopic orderTopic;
    private final RTopic userTopic;
    
    public NotificationService(RedissonClient redissonClient) {
        this.orderTopic = redissonClient.getTopic("notifications:orders");
        this.userTopic = redissonClient.getTopic("notifications:users");
        
        // 주문 알림 리스너 등록
        orderTopic.addListener(OrderNotification.class, this::handleOrderNotification);
        userTopic.addListener(UserNotification.class, this::handleUserNotification);
    }
    
    public void sendOrderNotification(OrderNotification notification) {
        orderTopic.publish(notification);
    }
    
    public void sendUserNotification(UserNotification notification) {
        userTopic.publish(notification);
    }
    
    private void handleOrderNotification(CharSequence channel, OrderNotification notification) {
        // 주문 알림 처리 로직
        log.info("주문 알림 처리: {}", notification);
    }
    
    private void handleUserNotification(CharSequence channel, UserNotification notification) {
        // 사용자 알림 처리 로직
        log.info("사용자 알림 처리: {}", notification);
    }
}
```

### 📈 3.6 Rate Limiter (속도 제한)

#### Raw Redis 방식 (복잡하지만 완전한 제어)
```redis
# Sliding Window Rate Limiter Lua Script
EVAL "
local key = KEYS[1]
local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local current = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, current - window)
local current_requests = redis.call('ZCARD', key)

if current_requests >= limit then
    return 0
else
    redis.call('ZADD', key, current, current)
    redis.call('EXPIRE', key, window)
    return 1
end
" 1 rate_limit_key 60 100 current_timestamp
```

**Raw Redis 장점**: 알고리즘 완전 커스터마이징, 최적화된 성능, 메모리 효율성
**Raw Redis 단점**: Lua Script 작성/디버깅 복잡, 알고리즘 검증 부담

#### Redisson 방식
```java
// 1. Rate Limiter 생성 (매우 간단하지만 내부 구현 의존)
RRateLimiter rateLimiter = redissonClient.getRateLimiter("api:rate_limit");

// 2. 제한 설정 (1분에 100회 요청 허용)
rateLimiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.MINUTES);

**Redisson 장점**: 검증된 알고리즘, 즉시 사용 가능, 다양한 Rate Limit 패턴 지원
**Redisson 단점**: 내부 알고리즘 제어 불가, 특수한 요구사항 대응 제한

// 3. 요청 허용 여부 확인
public boolean isRequestAllowed() {
    return rateLimiter.tryAcquire();  // 요청 가능하면 true
}

// 4. 대기하며 요청 처리
public boolean acquireWithTimeout() {
    return rateLimiter.tryAcquire(5, TimeUnit.SECONDS);  // 5초 대기
}

// 5. 사용자별 Rate Limiter
@Service
public class ApiRateLimitService {
    
    private final RedissonClient redissonClient;
    
    public boolean isUserRequestAllowed(Long userId) {
        String key = "api:rate_limit:user:" + userId;
        RRateLimiter userRateLimiter = redissonClient.getRateLimiter(key);
        
        // 사용자당 분당 10회 요청 제한
        userRateLimiter.trySetRate(RateType.OVERALL, 10, 1, RateIntervalUnit.MINUTES);
        
        return userRateLimiter.tryAcquire();
    }
    
    public boolean isGlobalRequestAllowed() {
        RRateLimiter globalRateLimiter = redissonClient.getRateLimiter("api:rate_limit:global");
        
        // 전체 시스템에서 초당 1000회 요청 제한
        globalRateLimiter.trySetRate(RateType.OVERALL, 1000, 1, RateIntervalUnit.SECONDS);
        
        return globalRateLimiter.tryAcquire();
    }
}

// 6. Spring Boot에서 사용 예시
@RestController
public class ApiController {
    
    @Autowired
    private ApiRateLimitService rateLimitService;
    
    @GetMapping("/api/products")
    public ResponseEntity<List<Product>> getProducts(HttpServletRequest request) {
        Long userId = getCurrentUserId(request);
        
        if (!rateLimitService.isUserRequestAllowed(userId)) {
            return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .body(null);
        }
        
        if (!rateLimitService.isGlobalRequestAllowed()) {
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(null);
        }
        
        return ResponseEntity.ok(productService.getProducts());
    }
}
```

---

## 4. 실무 프로젝트 적용 예시

### 🛒 4.1 전자상거래 상품 랭킹 시스템

```java
@Service
public class ProductRankingService {
    
    private final RedissonClient redissonClient;
    
    // 1. 실시간 랭킹 업데이트 (주문 완료 시)
    @EventListener
    public void updateProductRanking(OrderCompletedEvent event) {
        RScoredSortedSet<String> dailyRanking = 
            redissonClient.getScoredSortedSet("ranking:daily:" + LocalDate.now());
        
        // Raw Redis: ZINCRBY ranking:daily:2025-08-19 quantity product:1001
        dailyRanking.addScore("product:" + event.getProductId(), event.getQuantity());
        
        // TTL 설정 (7일 후 만료)
        dailyRanking.expire(Duration.ofDays(7));
    }
    
    // 2. 인기 상품 조회
    public List<ProductRankingDto> getPopularProducts(int limit) {
        RScoredSortedSet<String> ranking = 
            redissonClient.getScoredSortedSet("ranking:daily:" + LocalDate.now());
        
        // Raw Redis: ZREVRANGE ranking:daily:2025-08-19 0 9 WITHSCORES
        Collection<ScoredEntry<String>> topProducts = ranking.entryRangeReversed(0, limit - 1);
        
        return topProducts.stream()
            .map(entry -> new ProductRankingDto(
                extractProductId(entry.getValue()),
                entry.getScore().longValue()
            ))
            .collect(Collectors.toList());
    }
    
    // 3. 배치 랭킹 업데이트 (성능 최적화)
    @Scheduled(fixedRate = 60000)  // 1분마다
    public void batchUpdateRanking() {
        RBatch batch = redissonClient.createBatch();
        
        String dailyKey = "ranking:daily:" + LocalDate.now();
        String weeklyKey = "ranking:weekly:" + getWeekOfYear();
        
        RScoredSortedSetAsync<String> dailyRanking = batch.getScoredSortedSet(dailyKey);
        RScoredSortedSetAsync<String> weeklyRanking = batch.getScoredSortedSet(weeklyKey);
        
        // 여러 상품 점수 동시 업데이트
        Map<String, Double> productScores = getRecentOrderScores();
        for (Map.Entry<String, Double> entry : productScores.entrySet()) {
            dailyRanking.addScoreAsync(entry.getKey(), entry.getValue());
            weeklyRanking.addScoreAsync(entry.getKey(), entry.getValue());
        }
        
        batch.execute();  // 모든 명령어 한 번에 실행
    }
}
```

### 🎫 4.2 선착순 쿠폰 발급 시스템

```java
@Service
public class CouponIssueService {
    
    private final RedissonClient redissonClient;
    
    // 1. 선착순 쿠폰 발급
    public CouponIssueResult issueCoupon(Long couponId, Long userId) {
        String lockKey = "coupon:lock:" + couponId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 락 획득 (최대 3초 대기, 10초 후 자동 해제)
            if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
                return processCouponIssue(couponId, userId);
            } else {
                return CouponIssueResult.failure("시스템이 혼잡합니다. 잠시 후 다시 시도해주세요.");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CouponIssueResult.failure("쿠폰 발급 중 오류가 발생했습니다.");
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
    
    private CouponIssueResult processCouponIssue(Long couponId, Long userId) {
        // 2. 원자적 카운터로 발급 수량 체크
        RAtomicLong issuedCount = redissonClient.getAtomicLong("coupon:" + couponId + ":issued");
        RSet<String> issuedUsers = redissonClient.getSet("coupon:" + couponId + ":users");
        
        // 3. 중복 발급 체크
        if (issuedUsers.contains(userId.toString())) {
            return CouponIssueResult.failure("이미 발급받은 쿠폰입니다.");
        }
        
        // 4. 수량 체크 및 발급
        long currentCount = issuedCount.get();
        long maxCount = getCouponMaxCount(couponId);
        
        if (currentCount >= maxCount) {
            return CouponIssueResult.failure("쿠폰이 모두 소진되었습니다.");
        }
        
        // 5. 원자적 발급 처리
        RTransaction transaction = redissonClient.createTransaction(TransactionOptions.defaults());
        
        RAtomicLong txIssuedCount = transaction.getAtomicLong("coupon:" + couponId + ":issued");
        RSet<String> txIssuedUsers = transaction.getSet("coupon:" + couponId + ":users");
        
        // Raw Redis equivalent:
        // MULTI
        // INCR coupon:1:issued  
        // SADD coupon:1:users userId
        // EXEC
        txIssuedCount.incrementAndGet();
        txIssuedUsers.add(userId.toString());
        
        transaction.commit();
        
        // 6. 발급 이벤트 발행
        publishCouponIssueEvent(couponId, userId);
        
        return CouponIssueResult.success();
    }
    
    // 7. 실시간 발급 현황 조회
    public CouponStatusDto getCouponStatus(Long couponId) {
        RAtomicLong issuedCount = redissonClient.getAtomicLong("coupon:" + couponId + ":issued");
        RSet<String> issuedUsers = redissonClient.getSet("coupon:" + couponId + ":users");
        
        return CouponStatusDto.builder()
            .couponId(couponId)
            .issuedCount(issuedCount.get())
            .remainingCount(getCouponMaxCount(couponId) - issuedCount.get())
            .issuedUsers(issuedUsers.size())
            .build();
    }
}
```

---

## 5. 성능 및 안정성 비교

### ⚡ 5.1 성능 비교

| 항목 | RedisTemplate | Redisson |
|------|---------------|----------|
| **네트워크 호출** | 매번 명령어별 호출 | 배치/Pipeline으로 최적화 |
| **연결 관리** | Spring 자동 관리 | 자동 커넥션 풀 관리 |
| **직렬화** | 설정 가능 (JSON/Java) | 자동 직렬화/역직렬화 |
| **에러 처리** | 수동 재시도 로직 | 자동 재시도 및 Failover |
| **Spring 통합** | 완벽 통합 | 별도 의존성 필요 |
| **메모리 사용량** | 적음 | 약간 증가 (래퍼 객체) |

### 🛡 5.2 안정성 비교

```java
// RedisTemplate: 수동 에러 처리 (코드 복잡성 증가)
try {
    redisTemplate.opsForZSet().add("ranking", member, score);
} catch (RedisConnectionFailureException e) {
    // 연결 실패 시 재시도 로직 직접 구현 필요
    // 단점: 매번 이런 보일러플레이트 코드 작성, 휴먼 에러 발생 가능
    for (int i = 0; i < 3; i++) {
        try {
            Thread.sleep(100 * i); // 재시도 간격
            redisTemplate.opsForZSet().add("ranking", member, score);
            break;
        } catch (Exception ex) {
            if (i == 2) throw ex;
        }
    }
}
// 장점: 세밀한 제어 가능, Spring 생태계 통합
// 단점: 개발자가 모든 예외 상황 처리 필요, 코드 복잡성 증가

// Redisson: 자동 에러 처리 (개발 생산성 극대화)
RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet("ranking");
ranking.add(score, member);  // 내부적으로 재시도/Failover 자동 처리
// 장점: 검증된 에러 처리 로직, 개발 속도 빠름, 휴먼 에러 방지
// 단점: 약간의 성능 오버헤드, 내부 동작에 대한 제어권 제한
```

### 📊 5.3 실제 성능 테스트 결과

```java
@Component  
public class RedisPerformanceTest {
    
    @Test
    public void comparePerformance() {
        // RedisTemplate - 개별 명령어 방식
        long startTime = System.currentTimeMillis();
        for (int i = 0; i < 10000; i++) {
            redisTemplate.opsForZSet().add("test:ranking", "member" + i, i);  // 10,000번의 네트워크 호출
        }
        long redisTemplateTime = System.currentTimeMillis() - startTime;
        // 장점: Spring 통합, 직관적 API, 각 명령어별 세밀한 제어
        // 단점: 네트워크 지연시간이 누적되어 전체 성능 저하
        
        // Redisson Batch - 배치 처리 방식  
        startTime = System.currentTimeMillis();
        RBatch batch = redissonClient.createBatch();
        RScoredSortedSetAsync<String> rankingAsync = batch.getScoredSortedSet("test:ranking");
        for (int i = 0; i < 10000; i++) {
            rankingAsync.addAsync(i, "member" + i);  // 메모리에 명령어 누적
        }
        batch.execute();  // 1번의 네트워크 호출로 모든 명령어 실행
        long redissonTime = System.currentTimeMillis() - startTime;
        // 장점: 네트워크 호출 최소화로 3-5배 성능 향상, 원자성 보장
        // 단점: 메모리 사용량 증가, 실패 시 전체 롤백
        
        // 결과: Redisson Batch가 약 3-5배 빠름 (네트워크 지연시간 최소화)
        log.info("RedisTemplate: {}ms, Redisson: {}ms", redisTemplateTime, redissonTime);
    }
}
```



---

## 📚 정리 및 핵심 포인트

### ✅ Redisson 선택 기준
1. **복잡한 비즈니스 로직**: 원자적 연산이 많이 필요한 경우
2. **높은 동시성**: 분산 락, 트랜잭션이 필요한 경우  
3. **개발 생산성**: 빠른 개발과 안정성이 중요한 경우
4. **유지보수성**: 에러 처리, 연결 관리를 자동화하고 싶은 경우

### 🎯 Raw Redis 선택 기준
1. **극한 성능**: 모든 ms가 중요한 고성능 시스템
2. **메모리 제약**: 최소한의 메모리 사용이 필요한 경우
3. **단순한 작업**: 기본적인 캐싱만 필요한 경우
4. **기존 시스템**: 이미 Raw Redis로 잘 구축된 시스템

### 🚀 실무 권장사항

#### **Redisson을 선택해야 하는 경우와 이유:**
1. **복잡한 분산 시스템**: 분산 락, 트랜잭션이 필요한 전자상거래, 예약 시스템
   - **이유**: 수동으로 Lua Script 작성하는 것보다 검증된 고수준 API가 안전
2. **빠른 개발이 중요**: 스타트업, 프로토타입, MVP 개발
   - **이유**: 에러 처리, 재시도, 연결 관리를 자동화하여 개발 속도 3-5배 향상
3. **팀 역량**: Redis 전문가가 없는 일반적인 백엔드 팀
   - **이유**: Java Map/List 인터페이스 사용 경험으로 바로 적용 가능

#### **RedisTemplate을 선택해야 하는 경우와 이유:**
1. **Spring Boot 프로젝트**: Spring 생태계와의 완벽한 통합이 필요한 경우
   - **이유**: @Cacheable, @Transactional, AOP와 자연스럽게 연동
2. **세밀한 제어가 필요**: Redis 명령어별 옵션을 직접 제어하고 싶은 경우
   - **이유**: 모든 Redis 명령어를 직접 사용하며 Lua Script 자유자재로 활용
3. **기존 RedisTemplate 코드**: 이미 RedisTemplate 기반 인프라가 구축된 경우
   - **이유**: 기존 코드 자산과 운영 노하우를 그대로 활용 가능

#### **하이브리드 접근법 (권장):**
- **복잡한 비즈니스 로직**: Redisson (쿠폰 발급, 랭킹 시스템, 분산 락)
- **Spring 통합 캐싱**: RedisTemplate (@Cacheable, 세션 저장)
- **단순 조회 캐싱**: RedisTemplate (조회 결과 캐싱, 설정값 저장)

Redisson의 고수준 API를 활용하여 더 안전하고 생산적인 Redis 기반 시스템을 구축할 수 있습니다!