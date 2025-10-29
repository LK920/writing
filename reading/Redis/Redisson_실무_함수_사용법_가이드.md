# Redisson 실무 함수 사용법 완벽 가이드

## 📋 목차
1. [실무 사용 빈도별 함수 분류](#1-실무-사용-빈도별-함수-분류)
2. [최고 빈도 함수 (매일 사용)](#2-최고-빈도-함수-매일-사용)
3. [고빈도 함수 (주 2-3회 사용)](#3-고빈도-함수-주-2-3회-사용)
4. [중간 빈도 함수 (월 2-3회 사용)](#4-중간 빈도-함수-월-2-3회-사용)
5. [저빈도 함수 (특수 상황)](#5-저빈도-함수-특수-상황)
6. [실무 패턴별 함수 조합](#6-실무-패턴별-함수-조합)

---

## 1. 실무 사용 빈도별 함수 분류

### 🔥 최고 빈도 (매일 사용 - TOP 10)
1. **RBucket** - 기본 캐싱 (95% 프로젝트)
2. **RAtomicLong** - 카운터/재고 관리 (80% 프로젝트)
3. **RLock** - 분산 락 (75% 프로젝트)
4. **RMap** - 구조화 데이터 (70% 프로젝트)
5. **RScoredSortedSet** - 랭킹 시스템 (60% 프로젝트)
6. **RBatch** - 성능 최적화 (90% 프로젝트)
7. **RSet** - 중복 제거 (50% 프로젝트)
8. **expire()** - TTL 관리 (100% 프로젝트)
9. **RTransaction** - 원자적 연산 (40% 프로젝트)
10. **RRateLimiter** - API 제한 (30% 프로젝트)

### ⚡ 고빈도 (주 2-3회 사용)
- **RList/RQueue** - 메시지 큐
- **RTopic** - 실시간 알림
- **RHyperLogLog** - 유니크 카운팅
- **RBuckets** - 대량 캐시 처리

### 🔧 중간 빈도 (월 2-3회 사용)
- **RBitSet** - 활동 추적
- **RStream** - 이벤트 로그
- **RScheduledExecutorService** - 작업 스케줄링
- **RReadWriteLock** - 읽기/쓰기 분리

### 🎯 저빈도 (특수 상황)
- **RMultiLock** - 다중 리소스 락
- **RReliableTopic** - 신뢰성 메시징
- **RBoundedBlockingQueue** - 크기 제한 큐
- **RPriorityQueue** - 우선순위 처리

---

## 2. 최고 빈도 함수 (매일 사용)

### 🔑 2.1 RBucket - 기본 캐싱 (사용률 95%)

#### 🎯 작동방식
- Redis STRING 타입의 고수준 래퍼
- 단일 키-값 저장/조회 최적화
- 자동 직렬화/역직렬화 지원

#### 💡 사용이유
- **가장 기본적인 캐싱 패턴** (세션, 설정값, 임시 데이터)
- **Spring @Cacheable 대체** (더 세밀한 제어)
- **DB 부하 감소** (조회 결과 캐싱)

#### 📖 핵심 사용법
```java
@Service
public class UserCacheService {
    
    private final RedissonClient redissonClient;
    
    // 1. 기본 설정/조회 (가장 많이 사용)
    public void cacheUser(Long userId, User user) {
        RBucket<User> bucket = redissonClient.getBucket("user:" + userId);
        bucket.set(user, Duration.ofHours(1));  // 1시간 TTL
    }
    
    public User getUser(Long userId) {
        RBucket<User> bucket = redissonClient.getBucket("user:" + userId);
        return bucket.get();  // null이면 캐시 미스
    }
    
    // 2. 조건부 설정 (중복 처리 방지)
    public boolean setUserIfAbsent(Long userId, User user) {
        RBucket<User> bucket = redissonClient.getBucket("user:" + userId);
        return bucket.trySet(user, Duration.ofMinutes(30));  // 없을 때만 설정
    }
    
    // 3. 원자적 업데이트 (동시성 안전)
    public User updateUserCache(Long userId, User newUser) {
        RBucket<User> bucket = redissonClient.getBucket("user:" + userId);
        return bucket.getAndSet(newUser);  // 기존값 반환 후 새값 설정
    }
    
    // 4. TTL 체크 (캐시 만료 확인)
    public long getUserCacheTTL(Long userId) {
        RBucket<User> bucket = redissonClient.getBucket("user:" + userId);
        return bucket.remainTimeToLive();  // 밀리초 단위
    }
    
    // 5. 실무 패턴: 캐시 우선 조회
    public User getUserWithCache(Long userId) {
        RBucket<User> bucket = redissonClient.getBucket("user:" + userId);
        User cachedUser = bucket.get();
        
        if (cachedUser != null) {
            return cachedUser;  // 캐시 히트
        }
        
        // 캐시 미스 - DB 조회 후 캐싱
        User dbUser = userRepository.findById(userId);
        if (dbUser != null) {
            bucket.set(dbUser, Duration.ofHours(1));
        }
        return dbUser;
    }
}
```

### 🔢 2.2 RAtomicLong - 카운터/재고 관리 (사용률 80%)

#### 🎯 작동방식
- Redis INCR/DECR 명령어의 고수준 래퍼
- 내부적으로 Lua Script 사용하여 원자성 보장
- 동시성 환경에서 Race Condition 방지

#### 💡 사용이유
- **재고 관리** (상품 수량, 쿠폰 발급 수)
- **조회수/방문자 카운팅** (실시간 집계)
- **Rate Limiting** (요청 횟수 제한)

#### 📖 핵심 사용법
```java
@Service
public class CounterService {
    
    private final RedissonClient redissonClient;
    
    // 1. 기본 증감 연산 (가장 많이 사용)
    public long incrementViewCount(Long productId) {
        RAtomicLong viewCount = redissonClient.getAtomicLong("product:" + productId + ":views");
        long newCount = viewCount.incrementAndGet();  // 1 증가 후 반환
        
        // TTL 설정 (매일 초기화)
        viewCount.expire(Duration.ofDays(1));
        return newCount;
    }
    
    // 2. 재고 감소 (동시성 안전)
    public boolean decreaseStock(Long productId, int quantity) {
        RAtomicLong stock = redissonClient.getAtomicLong("stock:" + productId);
        
        // 원자적 감소 및 체크
        long newStock = stock.addAndGet(-quantity);
        if (newStock < 0) {
            // 재고 부족 - 롤백
            stock.addAndGet(quantity);
            return false;
        }
        return true;
    }
    
    // 3. 조건부 연산 (CAS - Compare And Swap)
    public boolean decreaseStockSafely(Long productId, int quantity) {
        RAtomicLong stock = redissonClient.getAtomicLong("stock:" + productId);
        
        while (true) {
            long currentStock = stock.get();
            if (currentStock < quantity) {
                return false;  // 재고 부족
            }
            
            long newStock = currentStock - quantity;
            if (stock.compareAndSet(currentStock, newStock)) {
                return true;  // 성공
            }
            // CAS 실패 시 재시도
        }
    }
    
    // 4. 초기값 설정 (키가 없을 때만)
    public void initializeStock(Long productId, int initialStock) {
        RAtomicLong stock = redissonClient.getAtomicLong("stock:" + productId);
        stock.trySet(initialStock);  // 키가 없을 때만 설정
    }
    
    // 5. 실무 패턴: 일일 카운터 리셋
    @Scheduled(cron = "0 0 0 * * *")  // 매일 자정
    public void resetDailyCounters() {
        String today = LocalDate.now().toString();
        
        // 새로운 일일 카운터 초기화
        RAtomicLong dailyOrders = redissonClient.getAtomicLong("daily:orders:" + today);
        dailyOrders.set(0);
        dailyOrders.expire(Duration.ofDays(7));  // 7일간 보관
    }
}
```

### 🔒 2.3 RLock - 분산 락 (사용률 75%)

#### 🎯 작동방식
- Redis SET NX EX 명령어 + Lua Script 조합
- 자동 락 갱신 (Watchdog 메커니즘)
- 재진입 가능 (Reentrant Lock)

#### 💡 사용이유
- **중복 처리 방지** (쿠폰 발급, 결제 처리)
- **동시성 제어** (재고 업데이트, 순서 보장)
- **크리티컬 섹션 보호** (데이터 일관성)

#### 📖 핵심 사용법
```java
@Service
public class LockService {
    
    private final RedissonClient redissonClient;
    
    // 1. 기본 락 사용 (가장 많이 사용)
    public String processPayment(Long orderId) {
        RLock lock = redissonClient.getLock("payment:lock:" + orderId);
        
        try {
            // 5초 대기, 30초 후 자동 해제
            if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {
                return doPaymentProcess(orderId);
            } else {
                return "결제 처리 중입니다. 잠시 후 시도해주세요.";
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "결제 처리 중 오류가 발생했습니다.";
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();  // 현재 스레드가 소유한 락만 해제
            }
        }
    }
    
    // 2. 즉시 락 획득 (대기 없음)
    public boolean tryImmediateLock(String resourceId) {
        RLock lock = redissonClient.getLock("resource:lock:" + resourceId);
        
        try {
            if (lock.tryLock()) {  // 즉시 시도
                processResource(resourceId);
                return true;
            }
            return false;  // 락 획득 실패
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
    
    // 3. 재진입 락 테스트
    public void reentrantExample(String key) {
        RLock lock = redissonClient.getLock("reentrant:" + key);
        
        try {
            lock.lock();
            processLevel1();
            processLevel2();  // 같은 락을 다시 획득 가능
        } finally {
            lock.unlock();
        }
    }
    
    private void processLevel2() {
        RLock lock = redissonClient.getLock("reentrant:" + key);  // 같은 키
        try {
            lock.lock();  // 재진입 성공
            // 중첩 처리
        } finally {
            lock.unlock();
        }
    }
    
    // 4. 실무 패턴: 쿠폰 발급 락
    public CouponResult issueCoupon(Long couponId, Long userId) {
        String lockKey = "coupon:issue:" + couponId;
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
                // 중복 발급 체크
                if (isAlreadyIssued(couponId, userId)) {
                    return CouponResult.duplicate();
                }
                
                // 수량 체크 및 발급
                return processCouponIssue(couponId, userId);
            }
            return CouponResult.busy();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CouponResult.error();
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
    
    // 5. 공정한 락 (FIFO 순서 보장)
    public void processFairQueue(String queueId) {
        RLock fairLock = redissonClient.getFairLock("fair:queue:" + queueId);
        
        try {
            fairLock.lock();  // 요청 순서대로 락 획득
            processQueueItem(queueId);
        } finally {
            fairLock.unlock();
        }
    }
}
```

### 🗂 2.4 RMap - 구조화 데이터 (사용률 70%)

#### 🎯 작동방식
- Redis HASH 타입의 고수준 래퍼
- Java Map 인터페이스 완전 구현
- 필드별 개별 조작 가능

#### 💡 사용이유
- **사용자 프로필** (여러 속성을 하나로 관리)
- **설정 관리** (키-값 쌍의 구조화 저장)
- **세션 데이터** (복잡한 세션 정보)

#### 📖 핵심 사용법
```java
@Service
public class ProfileService {
    
    private final RedissonClient redissonClient;
    
    // 1. 기본 필드 조작 (가장 많이 사용)
    public void updateUserProfile(Long userId, String field, String value) {
        RMap<String, String> profile = redissonClient.getMap("profile:" + userId);
        profile.put(field, value);
        profile.expire(Duration.ofHours(2));
    }
    
    public String getUserField(Long userId, String field) {
        RMap<String, String> profile = redissonClient.getMap("profile:" + userId);
        return profile.get(field);
    }
    
    // 2. 대량 필드 처리 (성능 최적화)
    public void setUserProfile(Long userId, Map<String, String> profileData) {
        RMap<String, String> profile = redissonClient.getMap("profile:" + userId);
        profile.putAll(profileData);  // 한 번에 여러 필드 설정
        profile.expire(Duration.ofHours(6));
    }
    
    public Map<String, String> getUserProfile(Long userId, Set<String> fields) {
        RMap<String, String> profile = redissonClient.getMap("profile:" + userId);
        return profile.getAll(fields);  // 지정된 필드만 조회
    }
    
    // 3. 조건부 업데이트 (동시성 안전)
    public boolean updateIfExists(Long userId, String field, String newValue) {
        RMap<String, String> profile = redissonClient.getMap("profile:" + userId);
        String oldValue = profile.replace(field, newValue);  // 키가 존재할 때만 교체
        return oldValue != null;
    }
    
    public boolean setIfAbsent(Long userId, String field, String value) {
        RMap<String, String> profile = redissonClient.getMap("profile:" + userId);
        return profile.putIfAbsent(field, value) == null;  // 키가 없을 때만 설정
    }
    
    // 4. 객체 직렬화 (고급 사용)
    public void cacheUserObject(Long userId, UserProfile userProfile) {
        RMap<String, UserProfile> objectMap = redissonClient.getMap("user:objects");
        objectMap.put(userId.toString(), userProfile);
        objectMap.expire(Duration.ofHours(1));
    }
    
    // 5. 실무 패턴: 세션 관리
    public class SessionManager {
        
        public void createSession(String sessionId, Long userId) {
            RMap<String, String> session = redissonClient.getMap("session:" + sessionId);
            session.put("userId", userId.toString());
            session.put("createdAt", LocalDateTime.now().toString());
            session.put("lastAccess", LocalDateTime.now().toString());
            session.expire(Duration.ofMinutes(30));  // 30분 세션
        }
        
        public void updateLastAccess(String sessionId) {
            RMap<String, String> session = redissonClient.getMap("session:" + sessionId);
            if (session.isExists()) {
                session.put("lastAccess", LocalDateTime.now().toString());
                session.expire(Duration.ofMinutes(30));  // TTL 갱신
            }
        }
        
        public Long getUserFromSession(String sessionId) {
            RMap<String, String> session = redissonClient.getMap("session:" + sessionId);
            String userIdStr = session.get("userId");
            return userIdStr != null ? Long.parseLong(userIdStr) : null;
        }
    }
}
```

### 📊 2.5 RScoredSortedSet - 랭킹 시스템 (사용률 60%)

#### 🎯 작동방식
- Redis SORTED SET 타입의 고수준 래퍼
- 점수(Score)에 따른 자동 정렬
- O(log N) 시간복잡도로 빠른 조회

#### 💡 사용이유
- **실시간 랭킹** (게임 순위, 상품 인기도)
- **시간 기반 정렬** (최신 순, 인기 순)
- **범위 조회** (상위 N개, 특정 점수 범위)

#### 📖 핵심 사용법
```java
@Service
public class RankingService {
    
    private final RedissonClient redissonClient;
    
    // 1. 기본 점수 조작 (가장 많이 사용)
    public void updateProductScore(Long productId, double score) {
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet("product:ranking");
        ranking.add(score, "product:" + productId);
        ranking.expire(Duration.ofDays(1));
    }
    
    public void increaseProductScore(Long productId, double increment) {
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet("product:ranking");
        ranking.addScore("product:" + productId, increment);  // 기존 점수에 추가
    }
    
    // 2. 랭킹 조회 (실무 핵심)
    public List<ProductRank> getTopProducts(int limit) {
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet("product:ranking");
        
        Collection<ScoredEntry<String>> topEntries = ranking.entryRangeReversed(0, limit - 1);
        
        return topEntries.stream()
            .map(entry -> new ProductRank(
                extractProductId(entry.getValue()),
                entry.getScore().longValue(),
                ranking.revRank(entry.getValue()) + 1  // 순위 (1부터 시작)
            ))
            .collect(Collectors.toList());
    }
    
    // 3. 특정 항목 순위 조회
    public Long getProductRank(Long productId) {
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet("product:ranking");
        Integer rank = ranking.revRank("product:" + productId);
        return rank != null ? rank + 1 : null;  // 1부터 시작하는 순위
    }
    
    public Double getProductScore(Long productId) {
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet("product:ranking");
        return ranking.getScore("product:" + productId);
    }
    
    // 4. 점수 범위 조회
    public List<String> getProductsByScoreRange(double minScore, double maxScore) {
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet("product:ranking");
        return new ArrayList<>(ranking.valueRange(minScore, true, maxScore, true));
    }
    
    // 5. 실무 패턴: 시간 기반 랭킹
    @EventListener
    public void onProductPurchased(ProductPurchasedEvent event) {
        String dailyKey = "ranking:daily:" + LocalDate.now();
        String weeklyKey = "ranking:weekly:" + getWeekOfYear();
        
        RScoredSortedSet<String> dailyRanking = redissonClient.getScoredSortedSet(dailyKey);
        RScoredSortedSet<String> weeklyRanking = redissonClient.getScoredSortedSet(weeklyKey);
        
        String productKey = "product:" + event.getProductId();
        double scoreIncrement = event.getQuantity() * event.getPrice();
        
        // 일일 및 주간 랭킹 동시 업데이트
        dailyRanking.addScore(productKey, scoreIncrement);
        weeklyRanking.addScore(productKey, scoreIncrement);
        
        // TTL 설정
        dailyRanking.expire(Duration.ofDays(7));
        weeklyRanking.expire(Duration.ofDays(30));
    }
    
    // 6. 배치 업데이트 (성능 최적화)
    public void batchUpdateRanking(Map<Long, Double> productScores) {
        RBatch batch = redissonClient.createBatch();
        RScoredSortedSetAsync<String> rankingAsync = batch.getScoredSortedSet("product:ranking");
        
        productScores.forEach((productId, score) -> {
            rankingAsync.addScoreAsync("product:" + productId, score);
        });
        
        batch.execute();  // 모든 업데이트를 한 번에 실행
    }
}
```

---

## 3. 고빈도 함수 (주 2-3회 사용)

### 🚀 3.1 RBatch - 성능 최적화 (사용률 90%)

#### 🎯 작동방식
- 여러 Redis 명령어를 한 번의 네트워크 호출로 실행
- 내부적으로 Pipeline 패턴 사용
- 원자성 옵션 제공 (MULTI/EXEC)

#### 💡 사용이유
- **네트워크 지연 최소화** (RTT 감소)
- **대량 데이터 처리** (배치 작업)
- **성능 최적화** (3-5배 속도 향상)

#### 📖 핵심 사용법
```java
@Service
public class BatchService {
    
    private final RedissonClient redissonClient;
    
    // 1. 기본 배치 처리 (가장 많이 사용)
    public void batchUpdateUserStats(Map<Long, UserStats> userStatsMap) {
        RBatch batch = redissonClient.createBatch();
        
        userStatsMap.forEach((userId, stats) -> {
            RMapAsync<String, String> userMap = batch.getMap("user:" + userId);
            userMap.putAsync("loginCount", String.valueOf(stats.getLoginCount()));
            userMap.putAsync("lastLogin", stats.getLastLogin().toString());
            userMap.expireAsync(Duration.ofDays(30));
        });
        
        BatchResult<?> result = batch.execute();  // 한 번에 실행
        log.info("배치 처리 완료: {} 건", userStatsMap.size());
    }
    
    // 2. 원자적 배치 (트랜잭션)
    public void atomicBatchTransfer(Long fromUserId, Long toUserId, int amount) {
        BatchOptions options = BatchOptions.defaults()
            .executionMode(BatchOptions.ExecutionMode.REDIS_WRITE_ATOMIC);  // 원자성 보장
        
        RBatch batch = redissonClient.createBatch(options);
        
        RAtomicLongAsync fromBalance = batch.getAtomicLong("balance:" + fromUserId);
        RAtomicLongAsync toBalance = batch.getAtomicLong("balance:" + toUserId);
        
        fromBalance.addAndGetAsync(-amount);
        toBalance.addAndGetAsync(amount);
        
        try {
            BatchResult<?> result = batch.execute();
            log.info("잔액 이체 완료: {} -> {}, 금액: {}", fromUserId, toUserId, amount);
        } catch (Exception e) {
            log.error("잔액 이체 실패", e);
            throw new TransferFailedException("이체 처리 중 오류 발생");
        }
    }
    
    // 3. 조회 배치 (여러 데이터 한 번에 가져오기)
    public Map<Long, User> batchGetUsers(List<Long> userIds) {
        RBatch batch = redissonClient.createBatch();
        
        List<RBucketAsync<User>> buckets = userIds.stream()
            .map(userId -> batch.getBucket("user:" + userId))
            .collect(Collectors.toList());
        
        BatchResult<?> result = batch.execute();
        List<?> results = result.getResponses();
        
        Map<Long, User> userMap = new HashMap<>();
        for (int i = 0; i < userIds.size(); i++) {
            User user = (User) results.get(i);
            if (user != null) {
                userMap.put(userIds.get(i), user);
            }
        }
        
        return userMap;
    }
    
    // 4. 실무 패턴: 랭킹 배치 업데이트
    @Scheduled(fixedRate = 300000)  // 5분마다
    public void batchUpdateRankings() {
        List<ProductScore> scores = getRecentProductScores();
        
        RBatch batch = redissonClient.createBatch();
        RScoredSortedSetAsync<String> dailyRanking = batch.getScoredSortedSet("ranking:daily");
        RScoredSortedSetAsync<String> weeklyRanking = batch.getScoredSortedSet("ranking:weekly");
        
        scores.forEach(score -> {
            String productKey = "product:" + score.getProductId();
            dailyRanking.addScoreAsync(productKey, score.getScore());
            weeklyRanking.addScoreAsync(productKey, score.getScore());
        });
        
        batch.execute();
        log.info("랭킹 배치 업데이트 완료: {} 건", scores.size());
    }
}
```

### 📧 3.2 RTopic - 실시간 알림 (사용률 40%)

#### 🎯 작동방식
- Redis PUB/SUB 명령어의 고수준 래퍼
- 실시간 메시지 발행/구독
- 여러 구독자에게 동시 전달

#### 💡 사용이유
- **실시간 알림** (주문 상태, 채팅 메시지)
- **이벤트 브로드캐스팅** (시스템 공지)
- **마이크로서비스 통신** (서비스 간 이벤트)

#### 📖 핵심 사용법
```java
@Service
public class NotificationService {
    
    private final RedissonClient redissonClient;
    private final RTopic orderTopic;
    private final RTopic chatTopic;
    
    public NotificationService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
        this.orderTopic = redissonClient.getTopic("notifications:orders");
        this.chatTopic = redissonClient.getTopic("chat:messages");
        
        // 구독자 등록
        initializeListeners();
    }
    
    // 1. 기본 메시지 발행
    public void sendOrderNotification(OrderEvent orderEvent) {
        orderTopic.publish(orderEvent);  // 모든 구독자에게 전송
        log.info("주문 알림 발송: {}", orderEvent.getOrderId());
    }
    
    public void sendChatMessage(ChatMessage message) {
        String roomTopic = "chat:room:" + message.getRoomId();
        RTopic roomChat = redissonClient.getTopic(roomTopic);
        roomChat.publish(message);
    }
    
    // 2. 리스너 초기화 (구독자 설정)
    private void initializeListeners() {
        // 주문 알림 리스너
        orderTopic.addListener(OrderEvent.class, (channel, orderEvent) -> {
            switch (orderEvent.getStatus()) {
                case "COMPLETED":
                    sendEmailNotification(orderEvent);
                    sendPushNotification(orderEvent);
                    break;
                case "CANCELLED":
                    sendCancellationNotification(orderEvent);
                    break;
            }
        });
        
        // 채팅 메시지 리스너
        chatTopic.addListener(ChatMessage.class, (channel, message) -> {
            // WebSocket으로 클라이언트에 전송
            webSocketService.sendToRoom(message.getRoomId(), message);
            
            // 오프라인 사용자를 위한 푸시 알림
            if (isUserOffline(message.getReceiverId())) {
                sendPushNotification(message);
            }
        });
    }
    
    // 3. 패턴 구독 (여러 채널 동시 구독)
    @PostConstruct
    public void subscribeToUserNotifications() {
        RPatternTopic userPattern = redissonClient.getPatternTopic("user:*:notifications");
        
        userPattern.addListener(String.class, (pattern, channel, message) -> {
            Long userId = extractUserIdFromChannel(channel);
            sendPersonalNotification(userId, message);
        });
    }
    
    // 4. 실무 패턴: 주문 상태 업데이트
    @EventListener
    public void handleOrderStatusChange(OrderStatusChangedEvent event) {
        // 특정 사용자에게만 알림
        String userTopic = "user:" + event.getUserId() + ":notifications";
        RTopic personalTopic = redissonClient.getTopic(userTopic);
        
        OrderNotification notification = OrderNotification.builder()
            .orderId(event.getOrderId())
            .status(event.getNewStatus())
            .message(getStatusMessage(event.getNewStatus()))
            .timestamp(LocalDateTime.now())
            .build();
        
        personalTopic.publish(notification);
    }
    
    // 5. 관리자 브로드캐스트
    public void sendAdminBroadcast(String message) {
        RTopic adminTopic = redissonClient.getTopic("admin:broadcast");
        
        AdminMessage adminMessage = AdminMessage.builder()
            .message(message)
            .sender("SYSTEM")
            .timestamp(LocalDateTime.now())
            .priority("HIGH")
            .build();
        
        adminTopic.publish(adminMessage);
        log.info("관리자 브로드캐스트 발송: {}", message);
    }
}
```

---

## 4. 중간 빈도 함수 (월 2-3회 사용)

### 📊 4.1 RHyperLogLog - 유니크 카운팅 (사용률 25%)

#### 🎯 작동방식
- 확률적 데이터 구조로 대용량 유니크 카운팅
- 메모리 사용량 고정 (12KB)
- 약 0.81% 오차율

#### 💡 사용이유
- **대용량 유니크 방문자** (DAU/MAU 측정)
- **메모리 효율적 카운팅** (수억 개 아이템도 12KB)
- **실시간 집계** (정확도보다 속도 우선)

#### 📖 핵심 사용법
```java
@Service
public class AnalyticsService {
    
    private final RedissonClient redissonClient;
    
    // 1. 일별 유니크 방문자 추적
    public void trackDailyVisitor(String userId) {
        String todayKey = "unique:visitors:" + LocalDate.now();
        RHyperLogLog<String> dailyVisitors = redissonClient.getHyperLogLog(todayKey);
        
        dailyVisitors.add(userId);
        dailyVisitors.expire(Duration.ofDays(30));  // 30일간 보관
    }
    
    // 2. 유니크 방문자 수 조회
    public long getDailyUniqueVisitors(LocalDate date) {
        String key = "unique:visitors:" + date;
        RHyperLogLog<String> visitors = redissonClient.getHyperLogLog(key);
        return visitors.count();  // 근사값 반환
    }
    
    // 3. 월별 유니크 방문자 병합
    public long getMonthlyUniqueVisitors(int year, int month) {
        String monthlyKey = "unique:visitors:monthly:" + year + "-" + month;
        RHyperLogLog<String> monthlyVisitors = redissonClient.getHyperLogLog(monthlyKey);
        
        // 해당 월의 모든 일별 데이터를 병합
        YearMonth yearMonth = YearMonth.of(year, month);
        for (int day = 1; day <= yearMonth.lengthOfMonth(); day++) {
            String dailyKey = "unique:visitors:" + LocalDate.of(year, month, day);
            monthlyVisitors.mergeWith(dailyKey);
        }
        
        monthlyVisitors.expire(Duration.ofDays(365));
        return monthlyVisitors.count();
    }
    
    // 4. 실무 패턴: 상품별 유니크 조회자
    public void trackProductView(Long productId, String userId) {
        String productKey = "product:" + productId + ":unique_viewers";
        RHyperLogLog<String> productViewers = redissonClient.getHyperLogLog(productKey);
        
        productViewers.add(userId);
        productViewers.expire(Duration.ofDays(7));
    }
    
    public ProductAnalytics getProductAnalytics(Long productId) {
        String productKey = "product:" + productId + ":unique_viewers";
        RHyperLogLog<String> productViewers = redissonClient.getHyperLogLog(productKey);
        
        return ProductAnalytics.builder()
            .productId(productId)
            .uniqueViewers(productViewers.count())
            .lastUpdated(LocalDateTime.now())
            .build();
    }
}
```

### 🎯 4.2 RBitSet - 활동 추적 (사용률 20%)

#### 🎯 작동방식
- 비트 배열로 메모리 효율적 저장
- 각 비트는 특정 이벤트/상태를 나타냄
- 비트 연산으로 빠른 집합 연산

#### 💡 사용이유
- **사용자 활동 패턴** (일별 접속, 기능 사용)
- **A/B 테스트 그룹** (사용자 세그멘테이션)
- **대용량 boolean 데이터** (메모리 효율성)

#### 📖 핵심 사용법
```java
@Service
public class UserActivityService {
    
    private final RedissonClient redissonClient;
    
    // 1. 일별 활동 기록
    public void markUserActiveToday(Long userId) {
        String yearKey = "user:" + userId + ":active_days:" + LocalDate.now().getYear();
        RBitSet activeDays = redissonClient.getBitSet(yearKey);
        
        int dayOfYear = LocalDate.now().getDayOfYear();
        activeDays.set(dayOfYear, true);  // 해당 일자 비트를 1로 설정
        activeDays.expire(Duration.ofDays(400));  // 다음해까지 보관
    }
    
    // 2. 월별 활동일 수 조회
    public long getMonthlyActiveDays(Long userId, int year, int month) {
        String yearKey = "user:" + userId + ":active_days:" + year;
        RBitSet activeDays = redissonClient.getBitSet(yearKey);
        
        YearMonth yearMonth = YearMonth.of(year, month);
        LocalDate startDate = yearMonth.atDay(1);
        LocalDate endDate = yearMonth.atEndOfMonth();
        
        long count = 0;
        for (int day = startDate.getDayOfYear(); day <= endDate.getDayOfYear(); day++) {
            if (activeDays.get(day)) {
                count++;
            }
        }
        return count;
    }
    
    // 3. 사용자 간 공통 활동일 찾기
    public Set<Integer> getCommonActiveDays(Long userId1, Long userId2, int year) {
        RBitSet user1Days = redissonClient.getBitSet("user:" + userId1 + ":active_days:" + year);
        RBitSet user2Days = redissonClient.getBitSet("user:" + userId2 + ":active_days:" + year);
        
        // 임시 BitSet으로 교집합 계산
        String tempKey = "temp:common:" + System.currentTimeMillis();
        RBitSet commonDays = redissonClient.getBitSet(tempKey);
        
        commonDays.or(user1Days);   // user1의 모든 활동일 복사
        commonDays.and(user2Days);  // user2와 공통인 날만 남김
        
        Set<Integer> result = commonDays.asBitSet().stream()
            .boxed()
            .collect(Collectors.toSet());
        
        commonDays.delete();  // 임시 데이터 정리
        return result;
    }
    
    // 4. 실무 패턴: 기능 사용 추적
    public void trackFeatureUsage(Long userId, String feature) {
        String featureKey = "user:" + userId + ":features:" + LocalDate.now().getYear();
        RBitSet featureUsage = redissonClient.getBitSet(featureKey);
        
        int featureBit = getFeatureBitPosition(feature);
        featureUsage.set(featureBit, true);
        featureUsage.expire(Duration.ofDays(365));
    }
    
    private int getFeatureBitPosition(String feature) {
        // 기능별 비트 위치 매핑
        return switch (feature) {
            case "SEARCH" -> 0;
            case "CART" -> 1;
            case "WISHLIST" -> 2;
            case "REVIEW" -> 3;
            case "PAYMENT" -> 4;
            default -> 999;  // 기타 기능
        };
    }
    
    // 5. A/B 테스트 그룹 관리
    public void assignUserToABTest(Long userId, String testName, String group) {
        String testKey = "ab_test:" + testName + ":group_" + group;
        RBitSet testGroup = redissonClient.getBitSet(testKey);
        
        testGroup.set(userId.intValue(), true);  // 사용자를 해당 그룹에 추가
        testGroup.expire(Duration.ofDays(30));
    }
    
    public boolean isUserInABTestGroup(Long userId, String testName, String group) {
        String testKey = "ab_test:" + testName + ":group_" + group;
        RBitSet testGroup = redissonClient.getBitSet(testKey);
        
        return testGroup.get(userId.intValue());
    }
}
```

---

## 5. 저빈도 함수 (특수 상황)

### 🔒 5.1 RReadWriteLock - 읽기/쓰기 분리 (사용률 15%)

#### 🎯 작동방식
- 읽기 락: 동시 여러 스레드 허용
- 쓰기 락: 독점적 접근 (한 번에 하나만)
- 읽기 우선 또는 쓰기 우선 정책

#### 💡 사용이유
- **설정 관리** (읽기 많음, 쓰기 적음)
- **캐시 갱신** (조회 빈번, 업데이트 가끔)
- **성능 최적화** (읽기 동시성 향상)

#### 📖 핵심 사용법
```java
@Service
public class ConfigurationService {
    
    private final RedissonClient redissonClient;
    private final RReadWriteLock configLock;
    
    public ConfigurationService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
        this.configLock = redissonClient.getReadWriteLock("config:lock");
    }
    
    // 1. 설정 읽기 (동시 접근 허용)
    public String getConfiguration(String key) {
        RLock readLock = configLock.readLock();
        
        try {
            readLock.lock();  // 읽기 락 획득
            
            RMap<String, String> configMap = redissonClient.getMap("system:config");
            return configMap.get(key);
            
        } finally {
            readLock.unlock();
        }
    }
    
    // 2. 설정 쓰기 (독점 접근)
    public void updateConfiguration(String key, String value) {
        RLock writeLock = configLock.writeLock();
        
        try {
            if (writeLock.tryLock(5, TimeUnit.SECONDS)) {
                RMap<String, String> configMap = redissonClient.getMap("system:config");
                configMap.put(key, value);
                
                // 설정 변경 이벤트 발송
                notifyConfigurationChange(key, value);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (writeLock.isHeldByCurrentThread()) {
                writeLock.unlock();
            }
        }
    }
    
    // 3. 실무 패턴: 캐시 갱신
    public List<Product> getProductCache() {
        RLock readLock = configLock.readLock();
        
        try {
            readLock.lock();
            
            RBucket<List<Product>> cache = redissonClient.getBucket("product:cache");
            List<Product> products = cache.get();
            
            if (products == null) {
                // 읽기 락 해제 후 쓰기 락으로 업그레이드
                readLock.unlock();
                return refreshProductCache();
            }
            
            return products;
            
        } finally {
            if (readLock.isHeldByCurrentThread()) {
                readLock.unlock();
            }
        }
    }
    
    private List<Product> refreshProductCache() {
        RLock writeLock = configLock.writeLock();
        
        try {
            if (writeLock.tryLock(10, TimeUnit.SECONDS)) {
                // 다시 체크 (다른 스레드가 이미 갱신했을 수 있음)
                RBucket<List<Product>> cache = redissonClient.getBucket("product:cache");
                List<Product> products = cache.get();
                
                if (products == null) {
                    products = productRepository.findAll();
                    cache.set(products, Duration.ofMinutes(10));
                }
                
                return products;
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (writeLock.isHeldByCurrentThread()) {
                writeLock.unlock();
            }
        }
        
        // 락 획득 실패 시 DB에서 직접 조회
        return productRepository.findAll();
    }
}
```

---

## 6. 실무 패턴별 함수 조합

### 🛒 6.1 전자상거래 패턴

#### 장바구니 관리
```java
@Service
public class CartService {
    
    // RMap (장바구니 아이템) + RAtomicLong (수량) + RLock (동시성)
    public void addToCart(Long userId, Long productId, int quantity) {
        RLock cartLock = redissonClient.getLock("cart:lock:" + userId);
        
        try {
            if (cartLock.tryLock(3, 10, TimeUnit.SECONDS)) {
                RMap<String, Integer> cart = redissonClient.getMap("cart:" + userId);
                String productKey = "product:" + productId;
                
                int currentQuantity = cart.getOrDefault(productKey, 0);
                cart.put(productKey, currentQuantity + quantity);
                cart.expire(Duration.ofDays(7));
            }
        } finally {
            if (cartLock.isHeldByCurrentThread()) {
                cartLock.unlock();
            }
        }
    }
}
```

#### 재고 관리
```java
@Service
public class StockService {
    
    // RAtomicLong (재고 수량) + RBatch (대량 처리) + RTopic (재고 알림)
    public boolean decreaseStock(Long productId, int quantity) {
        RAtomicLong stock = redissonClient.getAtomicLong("stock:" + productId);
        RLock stockLock = redissonClient.getLock("stock:lock:" + productId);
        
        try {
            if (stockLock.tryLock(2, 5, TimeUnit.SECONDS)) {
                long currentStock = stock.get();
                if (currentStock >= quantity) {
                    stock.addAndGet(-quantity);
                    
                    // 재고 부족 알림
                    if (stock.get() < 10) {
                        RTopic stockAlert = redissonClient.getTopic("stock:alert");
                        stockAlert.publish(new StockAlertEvent(productId, stock.get()));
                    }
                    
                    return true;
                }
                return false;
            }
        } finally {
            if (stockLock.isHeldByCurrentThread()) {
                stockLock.unlock();
            }
        }
        
        return false;
    }
}
```

### 🎮 6.2 게임/랭킹 패턴

#### 실시간 리더보드
```java
@Service
public class LeaderboardService {
    
    // RScoredSortedSet (랭킹) + RBatch (배치 업데이트) + RBucket (유저 정보)
    public void updatePlayerScore(Long playerId, long score) {
        RBatch batch = redissonClient.createBatch();
        
        // 랭킹 업데이트
        RScoredSortedSetAsync<String> ranking = batch.getScoredSortedSet("game:ranking");
        ranking.addScoreAsync("player:" + playerId, score);
        
        // 유저 정보 캐싱
        RBucketAsync<PlayerInfo> playerBucket = batch.getBucket("player:info:" + playerId);
        PlayerInfo playerInfo = getPlayerInfo(playerId);
        playerBucket.setAsync(playerInfo, Duration.ofHours(1));
        
        batch.execute();
    }
    
    public List<PlayerRank> getTopPlayers(int limit) {
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet("game:ranking");
        Collection<ScoredEntry<String>> topEntries = ranking.entryRangeReversed(0, limit - 1);
        
        return topEntries.stream()
            .map(entry -> {
                Long playerId = extractPlayerId(entry.getValue());
                RBucket<PlayerInfo> playerBucket = redissonClient.getBucket("player:info:" + playerId);
                PlayerInfo info = playerBucket.get();
                
                return PlayerRank.builder()
                    .playerId(playerId)
                    .score(entry.getScore().longValue())
                    .rank(ranking.revRank(entry.getValue()) + 1)
                    .playerInfo(info)
                    .build();
            })
            .collect(Collectors.toList());
    }
}
```

### 📊 6.3 분석/모니터링 패턴

#### 실시간 대시보드
```java
@Service
public class DashboardService {
    
    // RAtomicLong (카운터) + RHyperLogLog (유니크) + RMap (지표) + RTopic (알림)
    public void recordApiCall(String endpoint, String userId) {
        RBatch batch = redissonClient.createBatch();
        
        String today = LocalDate.now().toString();
        
        // API 호출 수 카운팅
        RAtomicLongAsync totalCalls = batch.getAtomicLong("api:calls:" + endpoint + ":" + today);
        totalCalls.incrementAndGetAsync();
        
        // 유니크 사용자 추적
        RHyperLogLogAsync<String> uniqueUsers = batch.getHyperLogLog("api:unique_users:" + endpoint + ":" + today);
        uniqueUsers.addAsync(userId);
        
        batch.execute();
        
        // 임계치 초과 시 알림
        checkThresholdAndAlert(endpoint, today);
    }
    
    public DashboardMetrics getDashboardMetrics(String date) {
        RBatch batch = redissonClient.createBatch();
        
        // 여러 지표를 한 번에 조회
        List<RAtomicLongAsync> apiCalls = API_ENDPOINTS.stream()
            .map(endpoint -> batch.getAtomicLong("api:calls:" + endpoint + ":" + date))
            .collect(Collectors.toList());
        
        List<RHyperLogLogAsync<String>> uniqueUsers = API_ENDPOINTS.stream()
            .map(endpoint -> batch.getHyperLogLog("api:unique_users:" + endpoint + ":" + date))
            .collect(Collectors.toList());
        
        BatchResult<?> result = batch.execute();
        
        return buildDashboardMetrics(result.getResponses());
    }
}
```

이 가이드는 실무에서 가장 자주 사용되는 Redisson 함수들을 빈도 순으로 정리했습니다. 각 함수의 작동방식, 사용이유, 핵심 사용법을 통해 실무에 바로 적용할 수 있는 실용적인 정보를 제공합니다.