# Redis Hash 완벽 가이드

## 📋 목차
1. [Redis Hash 개념과 원리](#1-redis-hash-개념과-원리)
2. [왜 Hash를 사용하는가?](#2-왜-hash를-사용하는가)
3. [Hash vs 다른 저장 방식](#3-hash-vs-다른-저장-방식)
4. [핵심 명령어와 사용법](#4-핵심-명령어와-사용법)
5. [상품 메타데이터 캐싱 구현](#5-상품-메타데이터-캐싱-구현)
6. [성능 최적화 전략](#6-성능-최적화-전략)
7. [실제 코드 예시](#7-실제-코드-예시)

---

## 1. Redis Hash 개념과 원리

### 🎯 Hash란?

**Redis Hash**는 **필드-값 쌍의 컬렉션을 저장하는 데이터 구조**로, 하나의 키 안에 여러 개의 필드와 값을 저장할 수 있습니다.

### 🏠 실생활 비유: 사용자 프로필 카드

개인 신분증을 생각해보세요:

```
┌─────────────────────────┐
│     사용자 프로필        │
├─────────────────────────┤
│ 이름: 김철수             │
│ 나이: 28                │
│ 이메일: kim@email.com   │
│ 전화번호: 010-1234-5678 │
│ 주소: 서울시 강남구      │
└─────────────────────────┘
```

Redis Hash로 표현하면:
```redis
HSET user:1001 name "김철수"
HSET user:1001 age "28"  
HSET user:1001 email "kim@email.com"
HSET user:1001 phone "010-1234-5678"
HSET user:1001 address "서울시 강남구"
```

여기서:
- **Hash Key**: `user:1001` (신분증 자체)
- **Field**: `name`, `age`, `email`, `phone`, `address` (각 항목명)
- **Value**: `"김철수"`, `"28"`, ... (각 항목값)

### 🔧 내부 구조와 원리

```
Redis Hash 내부 구조:
┌─────────────────────────────────────┐
│ Hash Key: "product:1001"            │
├─────────────────────────────────────┤
│ Field: "name"    → Value: "iPhone"  │
│ Field: "price"   → Value: "1200000" │
│ Field: "stock"   → Value: "50"      │
│ Field: "category"→ Value: "phone"   │
│ Field: "brand"   → Value: "Apple"   │
└─────────────────────────────────────┘
```

**핵심 특징:**
- **구조적 저장**: 관련된 데이터를 하나의 키에 그룹화
- **부분 조회**: 전체가 아닌 특정 필드만 조회 가능
- **원자적 연산**: 필드별 개별 수정 가능
- **메모리 효율**: 작은 Hash는 압축 저장

---

## 2. 왜 Hash를 사용하는가?

### 🚫 기존 방식의 문제점

#### 문제 상황: 상품 정보 캐싱

**개별 String 키 방식:**
```redis
SET product:1001:name "iPhone 15 Pro"
SET product:1001:price "1200000"
SET product:1001:stock "50"
SET product:1001:category "smartphone"
SET product:1001:brand "Apple"
SET product:1001:description "최신 아이폰"
```

**문제점:**
1. **키 관리 복잡**: 상품 하나당 여러 개의 키 생성
2. **네트워크 오버헤드**: 정보 조회 시 여러 번의 Redis 호출 필요
3. **원자성 부족**: 상품 정보 전체 업데이트 시 일관성 문제
4. **메모리 낭비**: 각 키마다 메타데이터 오버헤드

**JSON String 방식:**
```redis
SET product:1001 '{"name":"iPhone 15 Pro","price":1200000,"stock":50,...}'
```

**문제점:**
1. **부분 업데이트 불가**: 재고만 변경해도 전체 JSON 교체 필요
2. **파싱 오버헤드**: 매번 JSON 직렬화/역직렬화 비용
3. **타입 안정성 부족**: 모든 값이 문자열로 저장
4. **쿼리 불가**: 특정 필드만 조회 불가능

### ✅ Hash 방식의 해결책

```redis
# 한 번의 명령어로 여러 필드 설정
HMSET product:1001 
  name "iPhone 15 Pro"
  price 1200000
  stock 50
  category "smartphone"
  brand "Apple"
  description "최신 아이폰"

# 필요한 필드만 선택적 조회
HMGET product:1001 name price stock

# 개별 필드 원자적 업데이트
HINCRBY product:1001 stock -1  # 재고 1 감소
```

**장점:**
1. **그룹화된 관리**: 관련 데이터를 논리적으로 묶어 관리
2. **효율적 조회**: 필요한 필드만 선택적으로 가져오기
3. **원자적 업데이트**: 개별 필드 수정 시 원자성 보장
4. **메모리 효율**: 키 오버헤드 최소화

### 📊 성능 비교

| 구분 | 개별 String | JSON String | Hash |
|------|-------------|-------------|------|
| 키 관리 | 복잡 | 단순 | 단순 |
| 부분 조회 | 여러 요청 | 불가능 | 1회 요청 |
| 부분 업데이트 | 가능 | 불가능 | 가능 |
| 메모리 효율성 | 낮음 | 보통 | 높음 |
| 네트워크 비용 | 높음 | 보통 | 낮음 |

---

## 3. Hash vs 다른 저장 방식

### 🔄 상황별 비교

#### 상품 정보 저장 예시

**1. 개별 String 키**
```redis
SET product:1:name "노트북"
SET product:1:price "1500000"  
SET product:1:stock "10"

# 조회 시
GET product:1:name
GET product:1:price  
GET product:1:stock
# → 3번의 네트워크 호출
```

**2. JSON String**
```redis
SET product:1 '{"name":"노트북","price":1500000,"stock":10}'

# 조회 시
GET product:1
# → JSON 파싱 필요, 부분 조회 불가
```

**3. Hash (권장)**
```redis
HMSET product:1 name "노트북" price 1500000 stock 10

# 조회 시
HMGET product:1 name price stock
# → 1번의 네트워크 호출, 필요한 필드만 조회
```

### 📋 선택 가이드

| 상황 | 권장 방식 | 이유 |
|------|-----------|------|
| 간단한 키-값 저장 | String | 오버헤드 최소 |
| 구조화된 객체 저장 | **Hash** | 필드별 접근 가능 |
| 복잡한 중첩 구조 | JSON String | 복잡한 구조 표현 |
| 대용량 텍스트 | String | 단순함 |
| 빈번한 부분 업데이트 | **Hash** | 원자적 필드 수정 |

---

## 4. 핵심 명령어와 사용법

### 📝 기본 명령어

#### 4.1 데이터 설정
```redis
# 단일 필드 설정
HSET product:1 name "iPhone"
HSET product:1 price 1200000

# 다중 필드 설정
HMSET product:1 name "iPhone" price 1200000 stock 50

# 조건부 설정 (필드가 없을 때만)
HSETNX product:1 created_at "2025-08-18"
```

#### 4.2 데이터 조회
```redis
# 단일 필드 조회
HGET product:1 name

# 다중 필드 조회
HMGET product:1 name price stock

# 전체 필드 조회
HGETALL product:1

# 모든 필드명만 조회
HKEYS product:1

# 모든 값만 조회  
HVALS product:1
```

#### 4.3 데이터 수정
```redis
# 숫자 증가/감소
HINCRBY product:1 stock -1      # 재고 1 감소
HINCRBYFLOAT product:1 rating 0.5 # 평점 0.5 증가

# 필드 삭제
HDEL product:1 temporary_field

# 필드 존재 확인
HEXISTS product:1 name          # 1 또는 0 반환

# 필드 개수 확인
HLEN product:1
```

### 🎯 실제 사용 예시

#### 상품 재고 관리 시나리오
```redis
# 초기 상품 정보 설정
HMSET product:101
  name "MacBook Pro"
  price 2500000
  stock 20
  category "laptop"
  brand "Apple"
  created_at "2025-08-18T10:00:00Z"

# 주문 발생 시 재고 감소
HINCRBY product:101 stock -2    # 2개 주문 → 재고 18개

# 상품 정보 조회 (이름, 가격, 재고만)
HMGET product:101 name price stock
# 결과:
# 1) "MacBook Pro"
# 2) "2500000"  
# 3) "18"

# 재고 상태 확인
HGET product:101 stock          # 결과: "18"

# 상품 전체 정보 조회
HGETALL product:101
# 결과:
# 1) "name"
# 2) "MacBook Pro"
# 3) "price"
# 4) "2500000"
# 5) "stock"
# 6) "18"
# ... (모든 필드-값 쌍)
```

---

## 5. 상품 메타데이터 캐싱 구현

### 🏗 시스템 아키텍처

```
[Client] → [API] → [Service] → [Cache Layer] → [Database]
                      ↓           ↓
                   [Hash Cache] [MySQL]
                      ↓
              [Redis Hash Store]
```

### 📊 캐싱 전략 설계

#### 5.1 캐시 키 구조
```redis
# 상품 기본 정보
product:meta:{productId}

# 상품 통계 정보  
product:stats:{productId}

# 상품 캐시 메타데이터
product:cache:{productId}
```

#### 5.2 필드 설계
```redis
# 상품 기본 정보 Hash
HMSET product:meta:101
  id "101"
  name "iPhone 15 Pro"
  price "1200000"
  stock "50"
  category_id "1"
  category_name "스마트폰"
  brand "Apple"
  description "최신 아이폰 프로 모델"
  image_url "https://cdn.example.com/iphone15pro.jpg"
  status "ACTIVE"
  created_at "2025-08-18T10:00:00Z"
  updated_at "2025-08-18T10:00:00Z"

# 상품 통계 정보 Hash
HMSET product:stats:101
  total_orders "150"
  total_quantity "300"
  avg_rating "4.8"
  review_count "45"
  view_count "1250"
  like_count "89"
  
# 캐시 메타데이터 Hash
HMSET product:cache:101
  cached_at "2025-08-18T15:30:00Z"
  ttl "3600"
  version "v1.2"
  source "database"
```

### 🔄 캐시 로직 구현

#### Write-Through 패턴
```java
// 상품 정보 업데이트 시 캐시도 함께 업데이트
public Product updateProduct(Long productId, ProductUpdateRequest request) {
    // 1. DB 업데이트
    Product product = productRepository.save(request.toEntity());
    
    // 2. 캐시 업데이트 (Write-Through)
    updateProductCache(product);
    
    return product;
}

private void updateProductCache(Product product) {
    String cacheKey = "product:meta:" + product.getId();
    
    Map<String, String> hash = Map.of(
        "id", product.getId().toString(),
        "name", product.getName(),
        "price", product.getPrice().toString(),
        "stock", product.getStock().toString(),
        "updated_at", product.getUpdatedAt().toString()
    );
    
    redisTemplate.opsForHash().putAll(cacheKey, hash);
    redisTemplate.expire(cacheKey, Duration.ofHours(1));
}
```

#### Cache-Aside 패턴
```java
// 상품 조회 시 캐시 우선, 없으면 DB 조회 후 캐시 저장
public Product getProduct(Long productId) {
    String cacheKey = "product:meta:" + productId;
    
    // 1. 캐시에서 조회
    Map<Object, Object> cached = redisTemplate.opsForHash().entries(cacheKey);
    
    if (!cached.isEmpty()) {
        return convertHashToProduct(cached);
    }
    
    // 2. DB에서 조회
    Product product = productRepository.findById(productId)
        .orElseThrow(() -> new ProductNotFoundException(productId));
    
    // 3. 캐시에 저장
    cacheProduct(product);
    
    return product;
}
```

---

## 6. 성능 최적화 전략

### ⚡ 최적화 기법

#### 6.1 파이프라인 활용
```java
// 여러 상품 정보를 한 번에 조회
public List<Product> getProducts(List<Long> productIds) {
    List<Object> results = redisTemplate.executePipelined(
        (RedisCallback<Object>) connection -> {
            for (Long productId : productIds) {
                String key = "product:meta:" + productId;
                connection.hGetAll(key.getBytes());
            }
            return null;
        }
    );
    
    return results.stream()
        .map(this::convertHashToProduct)
        .collect(Collectors.toList());
}
```

#### 6.2 필드별 TTL 관리
```java
// 필드별로 다른 캐시 전략 적용
public void cacheProductWithFieldStrategy(Product product) {
    String key = "product:meta:" + product.getId();
    
    // 기본 정보: 1시간 캐시
    Map<String, String> basicInfo = Map.of(
        "name", product.getName(),
        "price", product.getPrice().toString(),
        "description", product.getDescription()
    );
    redisTemplate.opsForHash().putAll(key, basicInfo);
    redisTemplate.expire(key, Duration.ofHours(1));
    
    // 실시간 정보: 5분 캐시
    String stockKey = "product:realtime:" + product.getId();
    redisTemplate.opsForHash().put(stockKey, "stock", product.getStock().toString());
    redisTemplate.expire(stockKey, Duration.ofMinutes(5));
}
```

#### 6.3 메모리 최적화
```redis
# Hash 필드 개수가 적을 때 압축 저장 설정
CONFIG SET hash-max-ziplist-entries 512
CONFIG SET hash-max-ziplist-value 64

# 메모리 사용량 확인
MEMORY USAGE product:meta:101
```

### 📊 모니터링 및 디버깅

```java
// 캐시 히트율 모니터링
@Component
public class CacheMonitor {
    
    private final MeterRegistry meterRegistry;
    private final Counter cacheHit;
    private final Counter cacheMiss;
    
    public CacheMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.cacheHit = Counter.builder("cache.hit")
            .tag("type", "product")
            .register(meterRegistry);
        this.cacheMiss = Counter.builder("cache.miss")
            .tag("type", "product")
            .register(meterRegistry);
    }
    
    public void recordCacheHit() {
        cacheHit.increment();
    }
    
    public void recordCacheMiss() {
        cacheMiss.increment();
    }
}
```

---

## 7. 실제 코드 예시

### 🎯 완전한 상품 캐시 시스템

#### 7.1 상품 캐시 서비스
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductCacheService {
    
    private final RedissonClient redissonClient;
    private final ProductRepository productRepository;
    private final CacheMonitor cacheMonitor;
    
    private static final String PRODUCT_META_PREFIX = "product:meta:";
    private static final String PRODUCT_STATS_PREFIX = "product:stats:";
    private static final Duration CACHE_TTL = Duration.ofHours(1);
    
    /**
     * 상품 정보 조회 (캐시 우선)
     */
    public Optional<Product> getProduct(Long productId) {
        String cacheKey = PRODUCT_META_PREFIX + productId;
        RMap<String, String> productHash = redissonClient.getMap(cacheKey);
        
        // 캐시에서 조회 시도
        if (!productHash.isEmpty()) {
            cacheMonitor.recordCacheHit();
            return Optional.of(convertHashToProduct(productHash));
        }
        
        // 캐시 미스 - DB에서 조회
        cacheMonitor.recordCacheMiss();
        Optional<Product> product = productRepository.findById(productId);
        
        if (product.isPresent()) {
            cacheProduct(product.get());
        }
        
        return product;
    }
    
    /**
     * 상품 정보 캐시 저장
     */
    public void cacheProduct(Product product) {
        String cacheKey = PRODUCT_META_PREFIX + product.getId();
        RMap<String, String> productHash = redissonClient.getMap(cacheKey);
        
        Map<String, String> productData = Map.of(
            "id", product.getId().toString(),
            "name", product.getName(),
            "price", product.getPrice().toString(),
            "stock", product.getStock().toString(),
            "category", product.getCategory().getName(),
            "brand", product.getBrand(),
            "description", product.getDescription(),
            "image_url", product.getImageUrl(),
            "status", product.getStatus().toString(),
            "created_at", product.getCreatedAt().toString(),
            "updated_at", product.getUpdatedAt().toString()
        );
        
        productHash.putAll(productData);
        productHash.expire(CACHE_TTL);
        
        log.debug("상품 캐시 저장: productId={}", product.getId());
    }
    
    /**
     * 상품 재고 업데이트 (원자적 연산)
     */
    public void updateProductStock(Long productId, int stockChange) {
        String cacheKey = PRODUCT_META_PREFIX + productId;
        RMap<String, String> productHash = redissonClient.getMap(cacheKey);
        
        if (productHash.isExists()) {
            // 캐시에 있는 경우 원자적 업데이트
            String currentStock = productHash.get("stock");
            if (currentStock != null) {
                int newStock = Integer.parseInt(currentStock) + stockChange;
                productHash.put("stock", String.valueOf(newStock));
                productHash.put("updated_at", LocalDateTime.now().toString());
                
                log.debug("상품 재고 캐시 업데이트: productId={}, change={}, newStock={}", 
                         productId, stockChange, newStock);
            }
        }
    }
    
    /**
     * 상품 통계 정보 업데이트
     */
    public void updateProductStats(Long productId, ProductStatsUpdate update) {
        String cacheKey = PRODUCT_STATS_PREFIX + productId;
        RMap<String, String> statsHash = redissonClient.getMap(cacheKey);
        
        if (update.getOrderCount() != null) {
            statsHash.addAndGet("total_orders", update.getOrderCount().toString());
        }
        
        if (update.getQuantityChange() != null) {
            statsHash.addAndGet("total_quantity", update.getQuantityChange().toString());
        }
        
        if (update.getViewIncrement() != null) {
            statsHash.addAndGet("view_count", update.getViewIncrement().toString());
        }
        
        statsHash.expire(CACHE_TTL);
        
        log.debug("상품 통계 캐시 업데이트: productId={}", productId);
    }
    
    /**
     * 여러 상품 정보 일괄 조회
     */
    public List<Product> getProducts(List<Long> productIds) {
        List<Product> products = new ArrayList<>();
        List<Long> cacheMisses = new ArrayList<>();
        
        // 1. 캐시에서 일괄 조회
        for (Long productId : productIds) {
            String cacheKey = PRODUCT_META_PREFIX + productId;
            RMap<String, String> productHash = redissonClient.getMap(cacheKey);
            
            if (!productHash.isEmpty()) {
                products.add(convertHashToProduct(productHash));
                cacheMonitor.recordCacheHit();
            } else {
                cacheMisses.add(productId);
                cacheMonitor.recordCacheMiss();
            }
        }
        
        // 2. 캐시 미스 상품들을 DB에서 조회
        if (!cacheMisses.isEmpty()) {
            List<Product> dbProducts = productRepository.findAllById(cacheMisses);
            
            // 3. DB 조회 결과를 캐시에 저장
            dbProducts.forEach(this::cacheProduct);
            products.addAll(dbProducts);
        }
        
        return products;
    }
    
    /**
     * 상품 캐시 무효화
     */
    public void evictProduct(Long productId) {
        String metaCacheKey = PRODUCT_META_PREFIX + productId;
        String statsCacheKey = PRODUCT_STATS_PREFIX + productId;
        
        redissonClient.getMap(metaCacheKey).delete();
        redissonClient.getMap(statsCacheKey).delete();
        
        log.debug("상품 캐시 무효화: productId={}", productId);
    }
    
    /**
     * Hash를 Product 객체로 변환
     */
    private Product convertHashToProduct(RMap<String, String> productHash) {
        return Product.builder()
            .id(Long.parseLong(productHash.get("id")))
            .name(productHash.get("name"))
            .price(new BigDecimal(productHash.get("price")))
            .stock(Integer.parseInt(productHash.get("stock")))
            .brand(productHash.get("brand"))
            .description(productHash.get("description"))
            .imageUrl(productHash.get("image_url"))
            .status(ProductStatus.valueOf(productHash.get("status")))
            .createdAt(LocalDateTime.parse(productHash.get("created_at")))
            .updatedAt(LocalDateTime.parse(productHash.get("updated_at")))
            .build();
    }
}
```

#### 7.2 상품 서비스 통합
```java
@Service
@RequiredArgsConstructor
@Transactional
public class ProductService {
    
    private final ProductRepository productRepository;
    private final ProductCacheService productCacheService;
    
    /**
     * 상품 조회 (캐시 적용)
     */
    @Transactional(readOnly = true)
    public Product getProduct(Long productId) {
        return productCacheService.getProduct(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }
    
    /**
     * 상품 생성 (캐시 저장)
     */
    public Product createProduct(ProductCreateRequest request) {
        Product product = productRepository.save(request.toEntity());
        
        // 캐시에 즉시 저장 (Write-Through)
        productCacheService.cacheProduct(product);
        
        return product;
    }
    
    /**
     * 상품 수정 (캐시 업데이트)
     */
    public Product updateProduct(Long productId, ProductUpdateRequest request) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        product.update(request);
        Product savedProduct = productRepository.save(product);
        
        // 캐시 업데이트
        productCacheService.cacheProduct(savedProduct);
        
        return savedProduct;
    }
    
    /**
     * 상품 재고 차감 (캐시 동기화)
     */
    public void decreaseStock(Long productId, int quantity) {
        // DB 업데이트
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
            
        product.decreaseStock(quantity);
        productRepository.save(product);
        
        // 캐시 업데이트 (원자적 연산)
        productCacheService.updateProductStock(productId, -quantity);
    }
}
```

#### 7.3 이벤트 기반 캐시 관리
```java
@Component
@RequiredArgsConstructor
public class ProductCacheEventHandler {
    
    private final ProductCacheService productCacheService;
    
    /**
     * 주문 완료 시 상품 통계 업데이트
     */
    @EventListener
    @Async
    public void handleOrderCompleted(OrderCompletedEvent event) {
        for (OrderCompletedEvent.OrderItem item : event.getOrderItems()) {
            ProductStatsUpdate update = ProductStatsUpdate.builder()
                .orderCount(1)
                .quantityChange(item.getQuantity())
                .build();
                
            productCacheService.updateProductStats(item.getProductId(), update);
        }
    }
    
    /**
     * 상품 조회 시 조회수 증가
     */
    @EventListener
    @Async
    public void handleProductViewed(ProductViewedEvent event) {
        ProductStatsUpdate update = ProductStatsUpdate.builder()
            .viewIncrement(1)
            .build();
            
        productCacheService.updateProductStats(event.getProductId(), update);
    }
}
```

### 🧪 테스트 코드 예시

```java
@SpringBootTest
@Testcontainers
class ProductCacheServiceTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    
    @Autowired
    private ProductCacheService productCacheService;
    
    @Test
    void 상품_캐시_저장_및_조회_테스트() {
        // Given
        Product product = Product.builder()
            .id(1L)
            .name("테스트 상품")
            .price(new BigDecimal("10000"))
            .stock(100)
            .build();
        
        // When
        productCacheService.cacheProduct(product);
        Optional<Product> cached = productCacheService.getProduct(1L);
        
        // Then
        assertThat(cached).isPresent();
        assertThat(cached.get().getName()).isEqualTo("테스트 상품");
        assertThat(cached.get().getPrice()).isEqualTo(new BigDecimal("10000"));
    }
    
    @Test
    void 상품_재고_원자적_업데이트_테스트() {
        // Given
        Product product = Product.builder()
            .id(1L)
            .stock(100)
            .build();
        productCacheService.cacheProduct(product);
        
        // When
        productCacheService.updateProductStock(1L, -5);
        
        // Then
        Optional<Product> updated = productCacheService.getProduct(1L);
        assertThat(updated).isPresent();
        assertThat(updated.get().getStock()).isEqualTo(95);
    }
}
```

---

## 📚 정리 및 핵심 포인트

### ✅ Redis Hash를 사용해야 하는 이유
1. **구조적 저장**: 관련 데이터를 논리적으로 그룹화
2. **선택적 조회**: 필요한 필드만 효율적으로 조회
3. **원자적 연산**: 개별 필드 수정 시 원자성 보장
4. **메모리 효율**: 키 오버헤드 최소화로 메모리 절약

### 🎯 적용 시 주의사항
1. **필드 설계**: 자주 함께 사용되는 데이터를 같은 Hash에 저장
2. **TTL 관리**: 적절한 만료 시간으로 메모리 효율성 확보
3. **크기 제한**: 너무 큰 Hash는 성능 저하 가능성
4. **일관성**: 캐시와 DB 간 데이터 일관성 보장 전략 필요

### 🚀 이번 과제 적용 방안
1. **상품 메타데이터 캐싱**: 기본 정보와 통계 정보 분리 저장
2. **실시간 재고 관리**: 원자적 필드 업데이트로 정확성 보장
3. **성능 최적화**: 필요한 필드만 선택적 조회로 네트워크 비용 절약
4. **이벤트 기반 업데이트**: 비즈니스 이벤트 발생 시 실시간 캐시 갱신

이제 Redis Hash를 활용하여 효율적인 상품 메타데이터 캐싱 시스템을 구축할 수 있습니다!