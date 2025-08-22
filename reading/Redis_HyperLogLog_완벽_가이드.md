# Redis HyperLogLog 완벽 가이드

## 📚 목차
1. [HyperLogLog란 무엇인가?](#1-hyperloglog란-무엇인가)
2. [왜 HyperLogLog가 필요한가?](#2-왜-hyperloglog가-필요한가)
3. [HyperLogLog의 작동 원리](#3-hyperloglog의-작동-원리)
4. [Redis HyperLogLog 명령어](#4-redis-hyperloglog-명령어)
5. [실전 활용 예제](#5-실전-활용-예제)
6. [장단점 및 주의사항](#6-장단점-및-주의사항)
7. [다른 자료구조와의 비교](#7-다른-자료구조와의-비교)

---

## 1. HyperLogLog란 무엇인가?

### 📊 개념 정의

**HyperLogLog**는 **확률적 자료구조(Probabilistic Data Structure)**로, 대용량 데이터셋에서 **고유한 원소의 개수(Cardinality)**를 추정하는 알고리즘입니다.

쉽게 말해서:
- **"우리 사이트에 오늘 방문한 고유 사용자가 몇 명이지?"** 같은 질문에 답할 때 사용
- 정확한 수가 아닌 **근사치**를 제공 (오차율 0.81%)
- 메모리를 **극도로 적게** 사용 (최대 12KB)

### 🎯 핵심 특징

```
일반 Set 방식: 1억 개 고유값 저장 → 약 400MB 메모리 필요
HyperLogLog: 1억 개 고유값 카운팅 → 단 12KB 메모리만 필요!
```

---

## 2. 왜 HyperLogLog가 필요한가?

### 🔴 **문제 상황: 일반적인 방법의 한계**

#### 시나리오: 일별 순 방문자(UV) 집계

```java
// ❌ 방법 1: Set 사용 (메모리 폭발!)
Set<String> uniqueVisitors = new HashSet<>();
uniqueVisitors.add("user1");
uniqueVisitors.add("user2");
// ... 1000만 명 추가
int count = uniqueVisitors.size(); // 메모리: 약 400MB~1GB
```

```java
// ❌ 방법 2: DB Count Distinct (성능 저하!)
SELECT COUNT(DISTINCT user_id) FROM visits WHERE date = '2025-08-19';
// 1000만 건 조회 시 수초~수십초 소요
```

### 🟢 **HyperLogLog 솔루션**

```java
// ✅ HyperLogLog 사용 (메모리 효율 + 고성능!)
RHyperLogLog<String> uniqueVisitors = redissonClient.getHyperLogLog("uv:2025-08-19");
uniqueVisitors.add("user1");
uniqueVisitors.add("user2");
// ... 1000만 명 추가
long count = uniqueVisitors.count(); // 메모리: 단 12KB, 속도: O(1)
```

### 📊 **비교표**

| 방식 | 메모리 사용량 (1000만 고유값) | 조회 속도 | 정확도 | 사용 사례 |
|------|----------------------------|----------|--------|----------|
| HashSet | 400MB ~ 1GB | O(1) | 100% | 정확한 값 필요 시 |
| DB COUNT DISTINCT | 0 (DB 저장) | O(N) | 100% | 소규모 데이터 |
| **HyperLogLog** | **12KB** | **O(1)** | **99.19%** | **대규모 근사치** |

---

## 3. HyperLogLog의 작동 원리

### 🧮 **수학적 원리 (간단 버전)**

HyperLogLog는 **"동전 던지기"** 원리를 활용합니다:

1. **해시 함수로 변환**: 각 원소를 이진수로 변환
2. **선행 0의 개수 세기**: 이진수에서 처음 1이 나올 때까지 0의 개수
3. **확률적 추정**: 선행 0이 많을수록 고유값이 많다고 추정

```
예시:
user1 → hash → 00000101... → 선행 0이 5개
user2 → hash → 00110010... → 선행 0이 2개
user3 → hash → 00000001... → 선행 0이 7개

최대 선행 0의 개수가 7이면 → 약 2^7 = 128개의 고유값이 있을 것으로 추정
```

### 🔧 **실제 구현 원리**

```java
// 개념적 구현 (실제는 더 복잡)
public class SimpleHyperLogLog {
    private int[] buckets = new int[16384]; // 2^14 개의 버킷
    
    public void add(String value) {
        long hash = hash(value);
        int bucketIndex = (int)(hash & 0x3FFF); // 하위 14비트로 버킷 선택
        int leadingZeros = countLeadingZeros(hash >> 14); // 나머지 비트에서 선행 0 카운트
        
        // 해당 버킷의 최댓값 업데이트
        buckets[bucketIndex] = Math.max(buckets[bucketIndex], leadingZeros);
    }
    
    public long count() {
        // 조화 평균을 사용한 추정
        double sum = 0;
        for (int bucket : buckets) {
            sum += Math.pow(2, -bucket);
        }
        
        double estimate = Math.pow(buckets.length, 2) / sum;
        return Math.round(estimate * correctionFactor()); // 보정 계수 적용
    }
}
```

### 📐 **정확도 보장 메커니즘**

- **16,384개의 버킷** 사용 (Redis 기본값)
- 각 버킷은 **6비트** 저장 (최대 64개의 선행 0 기록 가능)
- 총 메모리: 16,384 × 6 bits = 12KB
- **표준 오차**: ±0.81% (신뢰도 65%)

---

## 4. Redis HyperLogLog 명령어

### 📝 **기본 명령어**

#### PFADD - 원소 추가
```bash
# 단일 원소 추가
PFADD visitors:2025-08-19 user123
# (integer) 1  # 추가됨

# 여러 원소 한번에 추가
PFADD visitors:2025-08-19 user456 user789 user123
# (integer) 1  # 새로운 원소가 추가됨 (user123은 중복이므로 무시)
```

#### PFCOUNT - 카운트 조회
```bash
# 단일 HyperLogLog 카운트
PFCOUNT visitors:2025-08-19
# (integer) 3

# 여러 HyperLogLog 합집합 카운트
PFCOUNT visitors:2025-08-19 visitors:2025-08-20
# (integer) 5  # 두 날짜의 순 방문자 수
```

#### PFMERGE - 병합
```bash
# 여러 HyperLogLog를 하나로 병합
PFMERGE visitors:2025-08-weekly visitors:2025-08-19 visitors:2025-08-20 visitors:2025-08-21
# OK
```

### 🔬 **고급 사용법**

```bash
# 디버그 정보 확인
PFDEBUG GETREG visitors:2025-08-19 0
# 특정 레지스터(버킷) 값 확인

# 내부 인코딩 확인
DEBUG OBJECT visitors:2025-08-19
# encoding:raw serializedlength:151
```

---

## 5. 실전 활용 예제

### 🌐 **예제 1: 웹사이트 순 방문자(UV) 추적**

```java
@Service
public class UniqueVisitorService {
    
    private final RedissonClient redissonClient;
    private final KeyGenerator keyGenerator;
    
    // 방문자 기록
    public void recordVisit(String userId, String page) {
        // 일별 전체 UV
        String dailyKey = keyGenerator.generateCustomKey("uv", "daily", LocalDate.now().toString());
        RHyperLogLog<String> dailyUV = redissonClient.getHyperLogLog(dailyKey);
        dailyUV.add(userId);
        dailyUV.expire(Duration.ofDays(30));
        
        // 페이지별 UV
        String pageKey = keyGenerator.generateCustomKey("uv", "page", page + ":" + LocalDate.now());
        RHyperLogLog<String> pageUV = redissonClient.getHyperLogLog(pageKey);
        pageUV.add(userId);
        pageUV.expire(Duration.ofDays(7));
    }
    
    // UV 조회
    public UniqueVisitorStats getStats(LocalDate date) {
        String dailyKey = keyGenerator.generateCustomKey("uv", "daily", date.toString());
        RHyperLogLog<String> dailyUV = redissonClient.getHyperLogLog(dailyKey);
        
        return UniqueVisitorStats.builder()
                .date(date)
                .uniqueVisitors(dailyUV.count())
                .build();
    }
    
    // 주간 UV 집계
    public long getWeeklyUV(LocalDate startDate) {
        List<String> dailyKeys = new ArrayList<>();
        for (int i = 0; i < 7; i++) {
            String key = keyGenerator.generateCustomKey("uv", "daily", 
                                                       startDate.plusDays(i).toString());
            dailyKeys.add(key);
        }
        
        // 여러 HyperLogLog 병합하여 카운트
        String weeklyKey = keyGenerator.generateCustomKey("uv", "weekly", startDate.toString());
        RHyperLogLog<String> weeklyUV = redissonClient.getHyperLogLog(weeklyKey);
        
        // 각 일별 HLL을 주간 HLL에 병합
        for (String dailyKey : dailyKeys) {
            RHyperLogLog<String> dailyUV = redissonClient.getHyperLogLog(dailyKey);
            weeklyUV.mergeWith(dailyUV.getName());
        }
        
        return weeklyUV.count();
    }
}
```

### 🔍 **예제 2: 검색어 고유 사용자 추적**

```java
@Service
public class SearchAnalyticsService {
    
    // 검색어별 고유 사용자 추적
    public void recordSearch(String keyword, String userId) {
        // 시간별 검색 사용자
        String hourlyKey = String.format("search:hourly:%s:%s", 
                                        keyword.toLowerCase(), 
                                        LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMddHH")));
        
        RHyperLogLog<String> hourlySearchUsers = redissonClient.getHyperLogLog(hourlyKey);
        hourlySearchUsers.add(userId);
        hourlySearchUsers.expire(Duration.ofHours(24));
    }
    
    // 인기 검색어 분석 (고유 사용자 수 기준)
    public List<SearchKeywordStats> getPopularKeywords(int topN) {
        List<SearchKeywordStats> stats = new ArrayList<>();
        
        // 미리 저장된 키워드 목록에서
        Set<String> keywords = getTrackedKeywords();
        
        for (String keyword : keywords) {
            String hourlyKey = String.format("search:hourly:%s:%s", 
                                            keyword, 
                                            LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMddHH")));
            
            RHyperLogLog<String> searchUsers = redissonClient.getHyperLogLog(hourlyKey);
            long uniqueUsers = searchUsers.count();
            
            if (uniqueUsers > 0) {
                stats.add(SearchKeywordStats.builder()
                        .keyword(keyword)
                        .uniqueUsers(uniqueUsers)
                        .build());
            }
        }
        
        // 고유 사용자 수로 정렬
        return stats.stream()
                .sorted((a, b) -> Long.compare(b.getUniqueUsers(), a.getUniqueUsers()))
                .limit(topN)
                .collect(Collectors.toList());
    }
}
```

### 📱 **예제 3: 앱 기능별 사용자 추적**

```java
@Service
public class FeatureUsageTrackingService {
    
    // 기능 사용 추적
    public void trackFeatureUsage(String featureName, String userId) {
        // 일별 기능 사용자
        String dailyKey = String.format("feature:daily:%s:%s", 
                                       featureName, 
                                       LocalDate.now());
        
        // 월별 기능 사용자
        String monthlyKey = String.format("feature:monthly:%s:%s", 
                                         featureName, 
                                         YearMonth.now());
        
        RHyperLogLog<String> dailyUsers = redissonClient.getHyperLogLog(dailyKey);
        RHyperLogLog<String> monthlyUsers = redissonClient.getHyperLogLog(monthlyKey);
        
        dailyUsers.add(userId);
        monthlyUsers.add(userId);
        
        // TTL 설정
        dailyUsers.expire(Duration.ofDays(7));
        monthlyUsers.expire(Duration.ofDays(90));
    }
    
    // A/B 테스트 그룹별 고유 사용자 수
    public ABTestStats getABTestStats(String testName) {
        String groupAKey = String.format("abtest:%s:groupA", testName);
        String groupBKey = String.format("abtest:%s:groupB", testName);
        
        RHyperLogLog<String> groupA = redissonClient.getHyperLogLog(groupAKey);
        RHyperLogLog<String> groupB = redissonClient.getHyperLogLog(groupBKey);
        
        return ABTestStats.builder()
                .testName(testName)
                .groupAUsers(groupA.count())
                .groupBUsers(groupB.count())
                .conversionRateA(calculateConversionRate(groupA))
                .conversionRateB(calculateConversionRate(groupB))
                .build();
    }
}
```

### 🛒 **예제 4: 이커머스 분석**

```java
@Service
public class EcommerceAnalyticsService {
    
    // 상품 조회 고유 사용자
    public void trackProductView(Long productId, String userId) {
        // 일별 상품 조회자
        String dailyKey = String.format("product:viewers:daily:%d:%s", 
                                       productId, 
                                       LocalDate.now());
        
        RHyperLogLog<String> viewers = redissonClient.getHyperLogLog(dailyKey);
        viewers.add(userId);
        viewers.expire(Duration.ofDays(30));
    }
    
    // 카테고리별 고유 구매자
    public void trackCategoryPurchaser(String category, String userId) {
        String monthlyKey = String.format("category:purchasers:%s:%s", 
                                         category, 
                                         YearMonth.now());
        
        RHyperLogLog<String> purchasers = redissonClient.getHyperLogLog(monthlyKey);
        purchasers.add(userId);
    }
    
    // 전환율 분석 (조회자 대비 구매자)
    public ConversionStats getProductConversionStats(Long productId, LocalDate date) {
        String viewerKey = String.format("product:viewers:daily:%d:%s", productId, date);
        String buyerKey = String.format("product:buyers:daily:%d:%s", productId, date);
        
        RHyperLogLog<String> viewers = redissonClient.getHyperLogLog(viewerKey);
        RHyperLogLog<String> buyers = redissonClient.getHyperLogLog(buyerKey);
        
        long uniqueViewers = viewers.count();
        long uniqueBuyers = buyers.count();
        
        return ConversionStats.builder()
                .productId(productId)
                .date(date)
                .uniqueViewers(uniqueViewers)
                .uniqueBuyers(uniqueBuyers)
                .conversionRate(uniqueViewers > 0 ? 
                              (double) uniqueBuyers / uniqueViewers * 100 : 0)
                .build();
    }
    
    // 장바구니 이탈률 분석
    public CartAbandonmentStats getCartAbandonmentStats(LocalDate date) {
        String cartAddKey = String.format("cart:added:users:%s", date);
        String checkoutKey = String.format("cart:checkout:users:%s", date);
        
        RHyperLogLog<String> cartUsers = redissonClient.getHyperLogLog(cartAddKey);
        RHyperLogLog<String> checkoutUsers = redissonClient.getHyperLogLog(checkoutKey);
        
        long cartAdded = cartUsers.count();
        long checkedOut = checkoutUsers.count();
        
        return CartAbandonmentStats.builder()
                .date(date)
                .usersAddedToCart(cartAdded)
                .usersCheckedOut(checkedOut)
                .abandonmentRate(cartAdded > 0 ? 
                               (double)(cartAdded - checkedOut) / cartAdded * 100 : 0)
                .build();
    }
}
```

### 🎮 **예제 5: 게임/앱 DAU, MAU 추적**

```java
@Service
public class UserActivityMetricsService {
    
    // 일일 활성 사용자 (DAU) 기록
    public void recordDailyActiveUser(String userId) {
        String dauKey = String.format("dau:%s", LocalDate.now());
        RHyperLogLog<String> dau = redissonClient.getHyperLogLog(dauKey);
        dau.add(userId);
        dau.expire(Duration.ofDays(35)); // 월간 통계를 위해 35일 보관
    }
    
    // 월간 활성 사용자 (MAU) 계산
    public long calculateMAU() {
        LocalDate today = LocalDate.now();
        List<String> dauKeys = new ArrayList<>();
        
        // 최근 30일간의 DAU 키 수집
        for (int i = 0; i < 30; i++) {
            String dauKey = String.format("dau:%s", today.minusDays(i));
            dauKeys.add(dauKey);
        }
        
        // 임시 MAU 키로 병합
        String mauKey = String.format("mau:temp:%s", UUID.randomUUID());
        RHyperLogLog<String> mau = redissonClient.getHyperLogLog(mauKey);
        
        // 모든 DAU를 MAU로 병합
        for (String dauKey : dauKeys) {
            RHyperLogLog<String> dau = redissonClient.getHyperLogLog(dauKey);
            if (dau.isExists()) {
                mau.mergeWith(dau.getName());
            }
        }
        
        long mauCount = mau.count();
        
        // 임시 키 삭제
        mau.delete();
        
        return mauCount;
    }
    
    // Stickiness (DAU/MAU) 계산
    public double calculateStickiness() {
        String dauKey = String.format("dau:%s", LocalDate.now());
        RHyperLogLog<String> dau = redissonClient.getHyperLogLog(dauKey);
        
        long dauCount = dau.count();
        long mauCount = calculateMAU();
        
        return mauCount > 0 ? (double) dauCount / mauCount : 0;
    }
}
```

---

## 6. 장단점 및 주의사항

### ✅ **장점**

#### 1. **메모리 효율성**
```
Set 방식: O(N) 메모리 - N이 증가하면 선형적으로 증가
HyperLogLog: O(1) 메모리 - 항상 12KB 고정
```

#### 2. **속도**
```
모든 연산이 O(1) - 상수 시간
병합 연산도 O(1)
```

#### 3. **확장성**
```
10억 개 고유값도 12KB로 처리
여러 HLL 병합 가능 (일별 → 주별 → 월별)
```

### ❌ **단점 및 제한사항**

#### 1. **근사치만 제공**
```java
// 실제: 1000개
// HyperLogLog: 992개 또는 1008개 (±0.81% 오차)

// 작은 수에서는 오차가 더 클 수 있음
// 실제: 10개
// HyperLogLog: 9개 또는 11개 (±10% 오차)
```

#### 2. **원소 조회 불가**
```java
// ❌ 불가능
boolean contains = hyperLogLog.contains("user123"); // 이런 메서드 없음

// ❌ 불가능  
Set<String> allUsers = hyperLogLog.getAllElements(); // 이런 메서드 없음

// ✅ 가능한 것은 오직 카운트
long count = hyperLogLog.count();
```

#### 3. **삭제 불가**
```java
// ❌ 특정 원소 삭제 불가
hyperLogLog.remove("user123"); // 이런 메서드 없음

// ✅ 전체 삭제만 가능
hyperLogLog.delete();
```

### ⚠️ **사용 시 주의사항**

#### 1. **작은 데이터셋에서는 비효율적**
```java
// 100개 미만의 고유값이면 Set이 더 효율적
if (expectedUniqueCount < 100) {
    // Set 사용
    RSet<String> users = redissonClient.getSet(key);
} else {
    // HyperLogLog 사용
    RHyperLogLog<String> users = redissonClient.getHyperLogLog(key);
}
```

#### 2. **정확한 수가 필요한 경우 사용 금지**
```java
// ❌ 금융 관련 통계
// ❌ 과금 관련 집계
// ❌ 법적 요구사항이 있는 통계

// ✅ 대시보드 통계
// ✅ 트렌드 분석
// ✅ 대략적인 사용자 수 파악
```

#### 3. **병합 시 오차 누적**
```java
// 여러 HLL을 병합하면 오차가 누적될 수 있음
// 하지만 여전히 0.81% 수준 유지
```

---

## 7. 다른 자료구조와의 비교

### 📊 **비교 매트릭스**

| 기준 | Set | Bitmap | HyperLogLog | Bloom Filter |
|------|-----|--------|-------------|--------------|
| **용도** | 정확한 고유값 저장 | ID 기반 존재 여부 | 고유값 개수 추정 | 존재 여부 확인 |
| **메모리** | O(N) | O(max_id) | O(1) - 12KB | O(N) 설정 가능 |
| **정확도** | 100% | 100% | 99.19% | False Positive 있음 |
| **원소 조회** | 가능 | 가능 | 불가능 | 가능 (있을 수도) |
| **원소 삭제** | 가능 | 가능 | 불가능 | 불가능 |
| **속도** | O(1) | O(1) | O(1) | O(k) |

### 🔄 **사용 케이스별 선택 가이드**

```java
public DataStructure chooseOptimalStructure(UseCase useCase) {
    
    switch (useCase.getRequirement()) {
        
        case EXACT_COUNT_SMALL:
            // 정확한 수 + 작은 데이터셋 (< 10,000)
            return new RedisSet();
            
        case EXACT_COUNT_LARGE:
            // 정확한 수 + 큰 데이터셋
            return new RedisSetWithPaging(); // 또는 DB
            
        case APPROXIMATE_COUNT:
            // 근사치 허용 + 큰 데이터셋
            return new HyperLogLog();
            
        case EXISTENCE_CHECK:
            // 존재 여부만 확인
            return new BloomFilter();
            
        case USER_ID_BASED:
            // 사용자 ID가 연속적인 정수
            return new Bitmap();
            
        default:
            return new RedisSet();
    }
}
```

### 🏗️ **하이브리드 접근법**

```java
@Service
public class HybridUniqueCounterService {
    
    // 작은 수에서는 Set, 큰 수에서는 HyperLogLog로 자동 전환
    public void addUser(String key, String userId) {
        RSet<String> set = redissonClient.getSet(key + ":set");
        
        if (set.size() < 1000) {
            // 1000개 미만이면 Set 사용 (정확한 수)
            set.add(userId);
        } else if (set.size() == 1000) {
            // 1000개 도달 시 HyperLogLog로 마이그레이션
            RHyperLogLog<String> hll = redissonClient.getHyperLogLog(key + ":hll");
            
            // 기존 Set 데이터를 HLL로 이관
            for (String existingUser : set) {
                hll.add(existingUser);
            }
            hll.add(userId);
            
            // Set 삭제
            set.delete();
        } else {
            // 1000개 초과면 HyperLogLog 사용
            RHyperLogLog<String> hll = redissonClient.getHyperLogLog(key + ":hll");
            hll.add(userId);
        }
    }
    
    public long getCount(String key) {
        RSet<String> set = redissonClient.getSet(key + ":set");
        
        if (set.isExists()) {
            return set.size(); // 정확한 수
        } else {
            RHyperLogLog<String> hll = redissonClient.getHyperLogLog(key + ":hll");
            return hll.count(); // 근사치
        }
    }
}
```

---

## 🎓 핵심 요약

### 📌 **HyperLogLog는 이럴 때 사용하세요**

✅ **적합한 경우**:
- 순 방문자(UV) 집계
- DAU/MAU 계산
- 대규모 이벤트의 참여자 수 추정
- 검색어별 고유 사용자 수
- A/B 테스트 그룹 크기 측정
- 실시간 대시보드 통계

❌ **부적합한 경우**:
- 정확한 수가 필요한 경우
- 원소를 조회해야 하는 경우
- 원소를 삭제해야 하는 경우
- 100개 미만의 작은 데이터셋
- 금융/회계 관련 통계

### 🚀 **실전 팁**

1. **키 네이밍 전략을 명확히**
   ```
   pattern: {domain}:{metric}:{period}:{date}
   예: uv:daily:2025-08-19
   ```

2. **TTL을 항상 설정**
   ```java
   hyperLogLog.expire(Duration.ofDays(30));
   ```

3. **병합은 계획적으로**
   ```java
   // 일별 → 주별 → 월별 순서로 계층적 병합
   ```

4. **모니터링 추가**
   ```java
   // 메모리 사용량과 정확도를 주기적으로 체크
   ```

5. **Fallback 전략 수립**
   ```java
   // Redis 장애 시 DB COUNT DISTINCT로 폴백
   ```

### 💡 **기억하세요!**

> HyperLogLog = **"얼마나 많은가?"**에 대한 답 (카운트)
> Set = **"누가 있는가?"**에 대한 답 (멤버십)
> 
> 큰 규모에서 "대략 몇 명?"이면 HyperLogLog
> "정확히 누구?"가 필요하면 Set

---

## 📚 참고 자료

- [Redis HyperLogLog 공식 문서](https://redis.io/docs/data-types/hyperloglogs/)
- [HyperLogLog 원논문 (Flajolet et al., 2007)](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)
- [Google의 HyperLogLog++ 개선 논문](https://research.google/pubs/pub40671/)
- [Redis HyperLogLog 내부 구현](https://github.com/redis/redis/blob/unstable/src/hyperloglog.c)

이제 HyperLogLog를 활용하여 대규모 카디널리티 추정 문제를 효율적으로 해결할 수 있습니다! 🎉