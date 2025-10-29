# Redis Sorted Set 완벽 가이드

## 📋 목차
1. [Redis Sorted Set 개념과 원리](#1-redis-sorted-set-개념과-원리)
2. [왜 Sorted Set을 사용하는가?](#2-왜-sorted-set을-사용하는가)
3. [Sorted Set vs 다른 데이터 구조](#3-sorted-set-vs-다른-데이터-구조)
4. [핵심 명령어와 사용법](#4-핵심-명령어와-사용법)
5. [실시간 랭킹 시스템 구현](#5-실시간-랭킹-시스템-구현)
6. [성능 최적화 팁](#6-성능-최적화-팁)
7. [실제 코드 예시](#7-실제-코드-예시)

---

## 1. Redis Sorted Set 개념과 원리

### 🎯 Sorted Set이란?

**Sorted Set**은 Redis의 데이터 구조 중 하나로, **값(member)과 점수(score)가 쌍으로 저장되는 정렬된 집합**입니다.

### 🏆 실생활 비유: 게임 리더보드

게임의 점수 리더보드를 생각해보세요:

```
🥇 플레이어A: 9,500점
🥈 플레이어B: 8,200점  
🥉 플레이어C: 7,800점
4위 플레이어D: 6,500점
5위 플레이어E: 5,100점
```

여기서:
- **member**(값): 플레이어명 (A, B, C, D, E)
- **score**(점수): 게임 점수 (9500, 8200, 7800, 6500, 5100)
- **정렬**: 점수 내림차순으로 자동 정렬

### 🔧 내부 구조와 원리

```
Redis Sorted Set 내부 구조:
┌─────────────────────────────────────┐
│ Skip List + Hash Table              │
├─────────────────────────────────────┤
│ member: "playerA"  score: 9500      │
│ member: "playerB"  score: 8200      │
│ member: "playerC"  score: 7800      │
│ member: "playerD"  score: 6500      │
│ member: "playerE"  score: 5100      │
└─────────────────────────────────────┘
```

**핵심 특징:**
- **자동 정렬**: score 기준으로 항상 정렬 상태 유지
- **중복 방지**: 같은 member는 하나만 존재 (score는 업데이트 가능)
- **빠른 조회**: O(log N) 시간복잡도로 조회/삽입/삭제

---

## 2. 왜 Sorted Set을 사용하는가?

### 🚫 기존 방식의 문제점

#### 문제 상황: 상품 인기 랭킹 구현

**일반적인 DB 방식:**
```sql
-- 매번 실행해야 하는 무거운 쿼리
SELECT p.*, SUM(oi.quantity) as total_orders
FROM products p
JOIN order_items oi ON p.id = oi.product_id  
JOIN orders o ON oi.order_id = o.id
WHERE o.created_at >= '2025-08-18'
GROUP BY p.id
ORDER BY total_orders DESC
LIMIT 10;
```

**문제점:**
1. **성능 이슈**: 매번 JOIN과 GROUP BY로 느린 실행
2. **DB 부하**: 많은 사용자가 동시에 랭킹을 조회하면 DB 과부하
3. **실시간성 부족**: 새 주문이 들어와도 즉시 반영되지 않음
4. **확장성 한계**: 데이터가 많아질수록 더욱 느려짐

### ✅ Sorted Set 방식의 해결책

```redis
# 실시간 상품 랭킹 저장
ZADD product_ranking:daily 150 "product:1"   # 150번 주문된 상품1
ZADD product_ranking:daily 89  "product:2"   # 89번 주문된 상품2
ZADD product_ranking:daily 67  "product:3"   # 67번 주문된 상품3

# 즉시 TOP 10 조회 (O(log N))
ZREVRANGE product_ranking:daily 0 9 WITHSCORES
```

**장점:**
1. **초고속 조회**: O(log N) 시간복잡도로 즉시 결과 반환
2. **실시간 업데이트**: 새 주문 시 즉시 랭킹 반영
3. **DB 부하 없음**: Redis에서 처리하므로 DB 부담 없음
4. **확장성**: 수백만 개 데이터도 빠르게 처리

### 📊 성능 비교

| 구분 | DB 방식 | Sorted Set 방식 |
|------|---------|-----------------|
| 조회 속도 | 500ms ~ 2초 | 1ms ~ 5ms |
| DB 부하 | 높음 | 없음 |
| 실시간성 | 낮음 | 높음 |
| 동시 사용자 처리 | 제한적 | 매우 높음 |

---

## 3. Sorted Set vs 다른 데이터 구조

### 🔄 비교표

| 데이터 구조 | 정렬 | 중복 허용 | 범위 조회 | 점수 기반 조회 | 사용 사례 |
|-------------|------|-----------|-----------|----------------|-----------|
| **Sorted Set** | ✅ 자동 | ❌ | ✅ 빠름 | ✅ | 랭킹, 리더보드 |
| List | ❌ | ✅ | ✅ 느림 | ❌ | 큐, 스택 |
| Set | ❌ | ❌ | ❌ | ❌ | 태그, 카테고리 |
| Hash | ❌ | ❌ | ❌ | ❌ | 객체 저장 |

### 🎮 상황별 선택 가이드

**Sorted Set을 선택해야 하는 경우:**
- ✅ 순위/랭킹이 필요한 경우
- ✅ 점수 기준 정렬이 필요한 경우  
- ✅ 범위 조회(상위 N개)가 필요한 경우
- ✅ 실시간 업데이트가 필요한 경우

**다른 구조를 선택해야 하는 경우:**
- ❌ 단순 목록만 필요 → List
- ❌ 중복 제거만 필요 → Set  
- ❌ 키-값 저장만 필요 → Hash

---

## 4. 핵심 명령어와 사용법

### 📝 기본 명령어

#### 4.1 데이터 추가/수정
```redis
# 기본 추가
ZADD ranking 100 "member1"
ZADD ranking 200 "member2" 150 "member3"  # 여러 개 동시 추가

# 조건부 추가
ZADD ranking NX 300 "member4"     # member4가 없을 때만 추가
ZADD ranking XX 250 "member1"     # member1이 있을 때만 점수 수정

# 점수 증가
ZINCRBY ranking 50 "member1"      # member1 점수에 50 추가
```

#### 4.2 데이터 조회
```redis
# 순위별 조회 (점수 오름차순)
ZRANGE ranking 0 2              # 1~3위 (member만)
ZRANGE ranking 0 2 WITHSCORES   # 1~3위 (member와 score)

# 순위별 조회 (점수 내림차순) - 랭킹에서 주로 사용
ZREVRANGE ranking 0 2           # 상위 3명
ZREVRANGE ranking 0 9           # 상위 10명

# 점수 범위로 조회
ZRANGEBYSCORE ranking 100 200   # 100점~200점 구간
ZREVRANGEBYSCORE ranking 200 100 # 200점~100점 구간 (내림차순)

# 개별 조회
ZSCORE ranking "member1"        # member1의 점수
ZRANK ranking "member1"         # member1의 순위 (오름차순)
ZREVRANK ranking "member1"      # member1의 순위 (내림차순)
```

#### 4.3 통계 정보
```redis
ZCARD ranking               # 전체 멤버 수
ZCOUNT ranking 100 200      # 100~200점 구간 멤버 수
```

### 🎯 실제 사용 예시

#### 상품 랭킹 시나리오
```redis
# 초기 데이터 설정
ZADD product_ranking:daily 45 "product:101"
ZADD product_ranking:daily 32 "product:102" 
ZADD product_ranking:daily 28 "product:103"
ZADD product_ranking:daily 15 "product:104"

# 새 주문 발생 시 랭킹 업데이트
ZINCRBY product_ranking:daily 3 "product:102"  # 상품102에 3개 주문 추가

# TOP 5 인기 상품 조회
ZREVRANGE product_ranking:daily 0 4 WITHSCORES
# 결과:
# 1) "product:101" 45점
# 2) "product:102" 35점 (32+3)
# 3) "product:103" 28점
# 4) "product:104" 15점

# 특정 상품의 순위 확인
ZREVRANK product_ranking:daily "product:102"   # 결과: 1 (2위, 0부터 시작)
```

---

## 5. 실시간 랭킹 시스템 구현

### 🏗 시스템 아키텍처

```
주문 완료 → 이벤트 발생 → 랭킹 업데이트 → 즉시 반영
    ↓
[Order Service] → [Event] → [Ranking Service] → [Redis Sorted Set]
                                    ↓
[Client] ← [Ranking API] ← [Cache] ← [랭킹 조회]
```

### 📊 랭킹 키 설계

```redis
# 기간별 랭킹 분리
ranking:daily:2025-08-18     # 일별 랭킹
ranking:weekly:2025-W33      # 주별 랭킹  
ranking:monthly:2025-08      # 월별 랭킹

# TTL 설정으로 자동 정리
EXPIRE ranking:daily:2025-08-18 604800    # 7일 후 삭제
EXPIRE ranking:weekly:2025-W33 2592000    # 30일 후 삭제
EXPIRE ranking:monthly:2025-08 31536000   # 365일 후 삭제
```

### 🔄 업데이트 로직

```java
// 주문 완료 시 랭킹 업데이트
public void updateProductRanking(Long productId, int quantity) {
    String today = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
    
    // 일별 랭킹 업데이트
    redisTemplate.opsForZSet()
        .incrementScore("ranking:daily:" + today, "product:" + productId, quantity);
    
    // 주별, 월별도 동일하게 업데이트
}
```

### 📈 조회 최적화

```java
// 캐시를 활용한 최적화된 조회
public List<Product> getTopProducts(int limit) {
    String cacheKey = "top_products:" + limit;
    
    // 1. 캐시에서 먼저 조회
    List<Product> cached = cacheService.get(cacheKey);
    if (cached != null) return cached;
    
    // 2. Redis Sorted Set에서 조회
    String today = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
    Set<String> productIds = redisTemplate.opsForZSet()
        .reverseRange("ranking:daily:" + today, 0, limit - 1);
    
    // 3. DB에서 상품 정보 조회
    List<Product> products = productRepository.findByIds(productIds);
    
    // 4. 결과 캐시 저장 (5분 TTL)
    cacheService.put(cacheKey, products, 300);
    
    return products;
}
```

---

## 6. 성능 최적화 팁

### ⚡ 최적화 전략

#### 6.1 배치 처리
```redis
# 개별 명령어 (비효율적)
ZADD ranking 100 "member1"
ZADD ranking 200 "member2"
ZADD ranking 300 "member3"

# 배치 처리 (효율적)
ZADD ranking 100 "member1" 200 "member2" 300 "member3"
```

#### 6.2 파이프라인 사용
```java
// Redisson 파이프라인 예시
RBatch batch = redissonClient.createBatch();
RScoredSortedSet<String> ranking = batch.getScoredSortedSet("ranking");

ranking.addAsync(100, "member1");
ranking.addAsync(200, "member2");
ranking.addAsync(300, "member3");

BatchResult<?> result = batch.execute();
```

#### 6.3 적절한 TTL 설정
```redis
# TTL로 메모리 관리
ZADD ranking:daily:2025-08-18 100 "product:1"
EXPIRE ranking:daily:2025-08-18 604800  # 7일 후 자동 삭제
```

### 📊 성능 모니터링

```redis
# Sorted Set 크기 확인
ZCARD ranking:daily:2025-08-18

# 메모리 사용량 확인  
MEMORY USAGE ranking:daily:2025-08-18

# 명령어 실행 시간 측정
--latency-history -i 1000 ZREVRANGE
```

---

## 7. 실제 코드 예시

### 🎯 완전한 상품 랭킹 시스템

#### 7.1 랭킹 업데이트 서비스
```java
@Service
@RequiredArgsConstructor
public class ProductRankingService {
    
    private final RedissonClient redissonClient;
    private static final String RANKING_PREFIX = "ranking:daily:";
    
    /**
     * 상품 주문량을 랭킹에 반영
     */
    public void updateProductRanking(Long productId, int quantity) {
        String today = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
        String rankingKey = RANKING_PREFIX + today;
        
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet(rankingKey);
        
        // 상품 점수 증가 (원자적 연산)
        ranking.addScore("product:" + productId, quantity);
        
        // TTL 설정 (7일)
        ranking.expire(Duration.ofDays(7));
        
        log.info("상품 랭킹 업데이트: productId={}, quantity={}", productId, quantity);
    }
    
    /**
     * TOP N 상품 조회
     */
    public List<Long> getTopProductIds(int limit) {
        String today = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
        String rankingKey = RANKING_PREFIX + today;
        
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet(rankingKey);
        
        // 상위 N개 조회 (점수 내림차순)
        Collection<String> topProducts = ranking.valueRangeReversed(0, limit - 1);
        
        return topProducts.stream()
                .map(key -> Long.parseLong(key.substring(8))) // "product:" 제거
                .collect(Collectors.toList());
    }
    
    /**
     * 특정 상품의 랭킹 정보 조회
     */
    public ProductRankingInfo getProductRankingInfo(Long productId) {
        String today = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
        String rankingKey = RANKING_PREFIX + today;
        String productKey = "product:" + productId;
        
        RScoredSortedSet<String> ranking = redissonClient.getScoredSortedSet(rankingKey);
        
        Double score = ranking.getScore(productKey);
        Integer rank = ranking.revRank(productKey); // 내림차순 순위
        
        return new ProductRankingInfo(
            productId, 
            rank != null ? rank.longValue() + 1 : -1L, // 1부터 시작하는 순위
            score != null ? score : 0.0
        );
    }
}
```

#### 7.2 이벤트 기반 업데이트
```java
@Component
@RequiredArgsConstructor
public class OrderEventHandler {
    
    private final ProductRankingService rankingService;
    
    /**
     * 주문 완료 시 상품 랭킹 업데이트
     */
    @EventListener
    @Async
    public void handleOrderCompleted(OrderCompletedEvent event) {
        for (OrderCompletedEvent.OrderItem item : event.getOrderItems()) {
            rankingService.updateProductRanking(
                item.getProductId(), 
                item.getQuantity()
            );
        }
        
        log.info("주문 완료 이벤트 처리: orderId={}, itemCount={}", 
                event.getOrderId(), event.getOrderItems().size());
    }
}
```

#### 7.3 랭킹 조회 API
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/ranking")
public class RankingController {
    
    private final ProductRankingService rankingService;
    private final ProductService productService;
    
    /**
     * 인기 상품 랭킹 조회
     */
    @GetMapping("/products/popular")
    public ResponseEntity<List<ProductRankingResponse>> getPopularProducts(
            @RequestParam(defaultValue = "10") int limit) {
        
        // 1. Redis에서 상위 상품 ID 조회
        List<Long> topProductIds = rankingService.getTopProductIds(limit);
        
        // 2. 상품 정보 조회
        List<Product> products = productService.getProductsByIds(topProductIds);
        
        // 3. 랭킹 정보와 조합
        List<ProductRankingResponse> response = products.stream()
                .map(product -> {
                    ProductRankingInfo rankingInfo = rankingService.getProductRankingInfo(product.getId());
                    return new ProductRankingResponse(product, rankingInfo);
                })
                .collect(Collectors.toList());
        
        return ResponseEntity.ok(response);
    }
}
```

### 🧪 테스트 코드 예시

```java
@SpringBootTest
@Testcontainers
class ProductRankingServiceTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    
    @Autowired
    private ProductRankingService rankingService;
    
    @Test
    void 상품_랭킹_업데이트_테스트() {
        // Given
        Long productId = 1L;
        int quantity = 5;
        
        // When
        rankingService.updateProductRanking(productId, quantity);
        
        // Then
        ProductRankingInfo rankingInfo = rankingService.getProductRankingInfo(productId);
        assertThat(rankingInfo.getScore()).isEqualTo(5.0);
        assertThat(rankingInfo.getRank()).isEqualTo(1L);
    }
    
    @Test
    void TOP_상품_조회_테스트() {
        // Given
        rankingService.updateProductRanking(1L, 10); // 1위
        rankingService.updateProductRanking(2L, 8);  // 2위  
        rankingService.updateProductRanking(3L, 5);  // 3위
        
        // When
        List<Long> topProducts = rankingService.getTopProductIds(2);
        
        // Then
        assertThat(topProducts).containsExactly(1L, 2L);
    }
}
```

---

## 📚 정리 및 핵심 포인트

### ✅ Redis Sorted Set을 사용해야 하는 이유
1. **초고속 성능**: O(log N) 시간복잡도로 빠른 조회
2. **실시간 반영**: 데이터 변경 시 즉시 순위 업데이트
3. **확장성**: 대용량 데이터도 효율적 처리
4. **메모리 효율성**: 적은 메모리로 많은 데이터 저장

### 🎯 적용 시 주의사항
1. **TTL 설정**: 메모리 관리를 위한 적절한 만료 시간
2. **키 네이밍**: 일관된 명명 규칙으로 관리 용이성 확보
3. **배치 처리**: 대량 업데이트 시 성능 최적화
4. **장애 대응**: Redis 장애 시 DB 폴백 전략 수립

### 🚀 이번 과제 적용 방안
1. **일별/주별/월별 랭킹**: 기간별 Sorted Set 분리 관리
2. **주문 완료 이벤트**: 실시간 랭킹 업데이트
3. **캐시 계층화**: 조회 성능 극대화
4. **모니터링**: 랭킹 시스템 안정성 확보

이제 Redis Sorted Set을 활용하여 고성능 실시간 상품 랭킹 시스템을 구축할 수 있습니다!