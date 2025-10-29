# Spring Batch 완벽 가이드

## 📋 목차
1. [Spring Batch 개념과 원리](#1-spring-batch-개념과-원리)
2. [왜 Spring Batch를 사용하는가?](#2-왜-spring-batch를-사용하는가)
3. [Batch vs 실시간 처리](#3-batch-vs-실시간-처리)
4. [핵심 아키텍처와 구성요소](#4-핵심-아키텍처와-구성요소)
5. [데이터 정합성 보장 배치 구현](#5-데이터-정합성-보장-배치-구현)
6. [성능 최적화 전략](#6-성능-최적화-전략)
7. [실제 코드 예시](#7-실제-코드-예시)

---

## 1. Spring Batch 개념과 원리

### 🎯 Spring Batch란?

**Spring Batch**는 **대용량 데이터를 안정적이고 효율적으로 처리하기 위한 배치 처리 프레임워크**입니다.

### 🏭 실생활 비유: 공장의 생산 라인

자동차 공장의 생산 과정을 생각해보세요:

```
원자재 창고 → [가공] → [조립] → [검품] → [포장] → 완제품 창고
    ↓           ↓        ↓        ↓        ↓         ↓
  Input     Process   Process  Validate  Process   Output
  
매일 밤 12시에 시작:
1. 오늘 생산 계획 확인 (Job 시작)
2. 원자재 100개씩 가져와서 (Chunk Size)
3. 각 단계별로 처리하고 (Step)
4. 문제 있으면 멈추고 기록 (Error Handling)
5. 완료되면 다음 단계로 (Step Transition)
```

### 🔧 배치 처리의 핵심 개념

```
Spring Batch 처리 흐름:
┌─────────────────────────────────────┐
│ Job: 일별 데이터 정합성 검증         │
├─────────────────────────────────────┤
│ Step 1: Redis 랭킹 데이터 읽기      │
│ Step 2: DB 실제 데이터와 비교       │
│ Step 3: 불일치 데이터 보정          │
│ Step 4: 결과 리포트 생성            │
└─────────────────────────────────────┘
```

**핵심 특징:**
- **청크 단위 처리**: 대용량 데이터를 작은 단위로 나누어 처리
- **재시작 가능**: 실패 지점부터 다시 시작 가능
- **트랜잭션 관리**: 안정적인 데이터 처리 보장
- **모니터링**: 실행 상태와 성능 추적 가능

---

## 2. 왜 Spring Batch를 사용하는가?

### 🚫 기존 방식의 문제점

#### 문제 상황: Redis 랭킹과 DB 데이터 정합성 검증

**일반적인 스케줄러 방식:**
```java
@Scheduled(cron = "0 0 2 * * *") // 매일 새벽 2시
public void validateRankingData() {
    // 모든 상품을 한번에 조회
    List<Product> allProducts = productRepository.findAll(); // 100만 개!
    
    for (Product product : allProducts) {
        // Redis 랭킹 점수 조회
        Double redisScore = getRedisScore(product.getId());
        
        // DB 실제 주문량 계산  
        Long actualOrders = calculateActualOrders(product.getId());
        
        // 불일치 시 보정
        if (!redisScore.equals(actualOrders.doubleValue())) {
            updateRedisRanking(product.getId(), actualOrders);
        }
    }
}
```

**문제점:**
1. **메모리 부족**: 100만 개 데이터를 한번에 메모리에 로드
2. **트랜잭션 문제**: 하나 실패하면 전체 롤백
3. **재시작 불가**: 중간에 실패하면 처음부터 다시 시작
4. **모니터링 부족**: 진행 상황과 실패 원인 추적 어려움
5. **성능 저하**: DB 커넥션 점유 시간 과다

### ✅ Spring Batch 방식의 해결책

```java
@Configuration
public class RankingValidationJobConfig {
    
    @Bean
    public Job rankingValidationJob() {
        return jobBuilderFactory.get("rankingValidationJob")
            .start(validateStep())
            .next(correctStep())
            .next(reportStep())
            .build();
    }
    
    @Bean
    public Step validateStep() {
        return stepBuilderFactory.get("validateStep")
            .<Product, RankingValidationResult>chunk(1000) // 1000개씩 처리
            .reader(productReader())
            .processor(rankingValidator())
            .writer(validationResultWriter())
            .faultTolerant()
            .skipLimit(100) // 100개까지 에러 허용
            .build();
    }
}
```

**장점:**
1. **메모리 효율**: 청크 단위로 나누어 처리 (1000개씩)
2. **안정성**: 청크별 트랜잭션으로 부분 실패 허용
3. **재시작 가능**: 실패 지점부터 이어서 처리
4. **모니터링**: 상세한 실행 통계와 로그 제공
5. **확장성**: 병렬 처리와 분산 처리 지원

### 📊 성능 비교

| 구분 | 일반 스케줄러 | Spring Batch |
|------|---------------|--------------|
| 메모리 사용량 | 100만 개 × 객체크기 | 1000개 × 객체크기 |
| 실패 복구 | 처음부터 재시작 | 실패 지점부터 재시작 |
| 트랜잭션 범위 | 전체 (All or Nothing) | 청크별 부분 성공 |
| 모니터링 | 제한적 | 상세한 메트릭 제공 |
| 확장성 | 단일 스레드 | 멀티스레드/분산 가능 |

---

## 3. Batch vs 실시간 처리

### 🔄 처리 방식 비교

#### 실시간 처리 (Online Processing)
```java
// 주문 완료 즉시 랭킹 업데이트
@EventListener
public void handleOrderCompleted(OrderCompletedEvent event) {
    for (OrderItem item : event.getOrderItems()) {
        updateRankingImmediately(item.getProductId(), item.getQuantity());
    }
}
```

#### 배치 처리 (Batch Processing)  
```java
// 매일 새벽에 전체 랭킹 재계산
@Bean
public Job dailyRankingRecalculationJob() {
    return jobBuilderFactory.get("dailyRankingRecalculation")
        .start(clearOldRankingStep())
        .next(recalculateRankingStep())
        .next(validateResultStep())
        .build();
}
```

### 📋 상황별 선택 가이드

| 요구사항 | 실시간 처리 | 배치 처리 | 추천 |
|----------|-------------|-----------|------|
| 즉시 반영 필요 | ✅ | ❌ | 실시간 |
| 대용량 데이터 | ❌ | ✅ | **배치** |
| 복잡한 계산 | ❌ | ✅ | **배치** |
| 정확성 최우선 | ❌ | ✅ | **배치** |
| 시스템 부하 최소화 | ❌ | ✅ | **배치** |
| 사용자 대기 불가 | ✅ | ❌ | 실시간 |

### 🎯 하이브리드 방식 (추천)

```java
// 실시간: 빠른 근사값 제공
@EventListener
public void updateRankingRealtime(OrderCompletedEvent event) {
    // Redis에 즉시 반영 (빠르지만 정확도 낮을 수 있음)
    updateRedisRanking(event);
}

// 배치: 정확한 정합성 보장
@Scheduled(cron = "0 0 2 * * *")
public void validateAndCorrectRanking() {
    // DB 기준으로 정확한 데이터 재계산 및 보정
    jobLauncher.run(rankingValidationJob, jobParameters);
}
```

---

## 4. 핵심 아키텍처와 구성요소

### 🏗 Spring Batch 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    Job Repository                        │
│               (실행 메타데이터 저장)                      │
└─────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────┐
│                    Job Launcher                         │
│                  (Job 실행 관리)                        │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                        Job                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Step 1    │→ │   Step 2    │→ │   Step 3    │      │
│  │             │  │             │  │             │      │
│  │ Reader      │  │ Reader      │  │ Reader      │      │
│  │ Processor   │  │ Processor   │  │ Processor   │      │
│  │ Writer      │  │ Writer      │  │ Writer      │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### 📦 핵심 구성요소

#### 4.1 Job (작업)
```java
@Bean
public Job productDataSyncJob() {
    return jobBuilderFactory.get("productDataSyncJob")
        .incrementer(new RunIdIncrementer()) // 재실행 허용
        .start(extractStep())
        .next(transformStep())
        .next(loadStep())
        .build();
}
```

#### 4.2 Step (단계)
```java
@Bean
public Step extractStep() {
    return stepBuilderFactory.get("extractStep")
        .<Product, ProductDto>chunk(1000)
        .reader(productItemReader())
        .processor(productItemProcessor())
        .writer(productItemWriter())
        .build();
}
```

#### 4.3 ItemReader (읽기)
```java
@Bean
public ItemReader<Product> productItemReader() {
    return new JpaPagingItemReaderBuilder<Product>()
        .name("productItemReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("SELECT p FROM Product p WHERE p.updatedAt >= :date")
        .parameterValues(Map.of("date", LocalDateTime.now().minusDays(1)))
        .pageSize(1000)
        .build();
}
```

#### 4.4 ItemProcessor (처리)
```java
@Component
public class ProductItemProcessor implements ItemProcessor<Product, ProductDto> {
    
    @Override
    public ProductDto process(Product product) throws Exception {
        // Redis에서 랭킹 점수 조회
        Double redisScore = redisRankingService.getScore(product.getId());
        
        // DB에서 실제 주문량 계산
        Long actualOrders = orderService.getProductOrderCount(product.getId());
        
        // 불일치 검사
        if (!Objects.equals(redisScore, actualOrders.doubleValue())) {
            return ProductDto.builder()
                .id(product.getId())
                .redisScore(redisScore)
                .actualOrders(actualOrders)
                .needsCorrection(true)
                .build();
        }
        
        return null; // 일치하는 경우 Writer로 전달하지 않음
    }
}
```

#### 4.5 ItemWriter (쓰기)
```java
@Component
public class ProductItemWriter implements ItemWriter<ProductDto> {
    
    @Override
    public void write(List<? extends ProductDto> items) throws Exception {
        for (ProductDto item : items) {
            if (item.isNeedsCorrection()) {
                // Redis 랭킹 점수 보정
                redisRankingService.setScore(item.getId(), item.getActualOrders());
                
                // 보정 로그 기록
                correctionLogService.log(item);
            }
        }
    }
}
```

---

## 5. 데이터 정합성 보장 배치 구현

### 🎯 랭킹 데이터 정합성 검증 시나리오

#### 5.1 문제 상황
```
실시간 랭킹 시스템의 잠재적 문제:
1. Redis 메모리 한계로 인한 데이터 손실
2. 네트워크 장애로 인한 업데이트 누락  
3. 동시성 이슈로 인한 카운트 오차
4. TTL 만료로 인한 데이터 소실

→ 정기적인 정합성 검증 및 보정 필요!
```

#### 5.2 배치 처리 설계

```java
@Configuration
@RequiredArgsConstructor
public class RankingValidationJobConfig {
    
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    
    /**
     * 랭킹 데이터 정합성 검증 Job
     */
    @Bean
    public Job rankingValidationJob() {
        return jobBuilderFactory.get("rankingValidationJob")
            .incrementer(new RunIdIncrementer())
            .listener(jobExecutionListener())
            .start(preparationStep())
            .next(validationStep())
            .next(correctionStep())
            .next(reportStep())
            .build();
    }
    
    /**
     * Step 1: 검증 준비 (임시 테이블 생성, 날짜 범위 설정)
     */
    @Bean
    public Step preparationStep() {
        return stepBuilderFactory.get("preparationStep")
            .tasklet(preparationTasklet())
            .build();
    }
    
    /**
     * Step 2: 데이터 검증 (Redis vs DB 비교)
     */
    @Bean
    public Step validationStep() {
        return stepBuilderFactory.get("validationStep")
            .<Product, RankingValidationResult>chunk(1000)
            .reader(productReader())
            .processor(rankingValidator())
            .writer(validationResultWriter())
            .faultTolerant()
            .skipLimit(100) // 에러 허용
            .skip(Exception.class)
            .listener(validationStepListener())
            .build();
    }
    
    /**
     * Step 3: 불일치 데이터 보정
     */
    @Bean
    public Step correctionStep() {
        return stepBuilderFactory.get("correctionStep")
            .<RankingValidationResult, RankingValidationResult>chunk(500)
            .reader(mismatchDataReader())
            .processor(rankingCorrector())
            .writer(correctionWriter())
            .build();
    }
    
    /**
     * Step 4: 결과 리포트 생성
     */
    @Bean
    public Step reportStep() {
        return stepBuilderFactory.get("reportStep")
            .tasklet(reportGeneratorTasklet())
            .build();
    }
}
```

### 🔍 검증 로직 구현

#### ItemReader - 검증 대상 상품 조회
```java
@Component
public class ProductReader implements ItemReader<Product> {
    
    private final ProductRepository productRepository;
    private final PagingQueryProvider queryProvider;
    private Iterator<Product> productIterator;
    private int pageSize = 1000;
    private int currentPage = 0;
    
    @Override
    public Product read() throws Exception {
        if (productIterator == null || !productIterator.hasNext()) {
            List<Product> products = loadNextPage();
            if (products.isEmpty()) {
                return null; // 더 이상 읽을 데이터 없음
            }
            productIterator = products.iterator();
        }
        
        return productIterator.hasNext() ? productIterator.next() : null;
    }
    
    private List<Product> loadNextPage() {
        Pageable pageable = PageRequest.of(currentPage++, pageSize);
        Page<Product> page = productRepository.findActiveProducts(pageable);
        return page.getContent();
    }
}
```

#### ItemProcessor - 랭킹 데이터 검증
```java
@Component
@RequiredArgsConstructor
public class RankingValidator implements ItemProcessor<Product, RankingValidationResult> {
    
    private final RedisRankingService redisRankingService;
    private final OrderService orderService;
    
    @Override
    public RankingValidationResult process(Product product) throws Exception {
        Long productId = product.getId();
        
        // 1. Redis에서 현재 랭킹 점수 조회
        Double redisScore = redisRankingService.getProductScore(productId, "daily");
        
        // 2. DB에서 실제 주문량 계산 (오늘 기준)
        LocalDate today = LocalDate.now();
        Long actualOrders = orderService.getProductOrderCountByDate(productId, today);
        
        // 3. 검증 결과 생성
        boolean isValid = Objects.equals(
            redisScore != null ? redisScore.longValue() : 0L, 
            actualOrders
        );
        
        RankingValidationResult result = RankingValidationResult.builder()
            .productId(productId)
            .productName(product.getName())
            .redisScore(redisScore != null ? redisScore.longValue() : 0L)
            .actualOrders(actualOrders)
            .isValid(isValid)
            .validatedAt(LocalDateTime.now())
            .build();
        
        // 4. 불일치하는 경우만 다음 단계로 전달
        return isValid ? null : result;
    }
}
```

#### ItemWriter - 검증 결과 저장
```java
@Component
@RequiredArgsConstructor
public class ValidationResultWriter implements ItemWriter<RankingValidationResult> {
    
    private final ValidationResultRepository validationResultRepository;
    private final SlackNotificationService slackService;
    
    @Override
    public void write(List<? extends RankingValidationResult> items) throws Exception {
        if (items.isEmpty()) {
            return;
        }
        
        // 1. 검증 결과 DB 저장
        List<ValidationResult> entities = items.stream()
            .map(this::convertToEntity)
            .collect(Collectors.toList());
        
        validationResultRepository.saveAll(entities);
        
        // 2. 심각한 불일치 건수가 많으면 알림
        long criticalMismatches = items.stream()
            .filter(item -> Math.abs(item.getRedisScore() - item.getActualOrders()) > 10)
            .count();
        
        if (criticalMismatches > 10) {
            slackService.sendAlert(
                "랭킹 데이터 심각한 불일치 발견: " + criticalMismatches + "건"
            );
        }
        
        log.info("검증 결과 저장 완료: {}건 (불일치: {}건)", 
                items.size(), items.size());
    }
}
```

### 🔧 데이터 보정 로직

```java
@Component
@RequiredArgsConstructor
public class RankingCorrector implements ItemProcessor<RankingValidationResult, RankingValidationResult> {
    
    private final RedisRankingService redisRankingService;
    
    @Override
    public RankingValidationResult process(RankingValidationResult item) throws Exception {
        try {
            // Redis 랭킹 점수를 실제 주문량으로 보정
            redisRankingService.setProductScore(
                item.getProductId(), 
                item.getActualOrders(), 
                "daily"
            );
            
            item.setCorrected(true);
            item.setCorrectedAt(LocalDateTime.now());
            
            log.debug("랭킹 보정 완료: productId={}, {} → {}", 
                     item.getProductId(), item.getRedisScore(), item.getActualOrders());
            
            return item;
            
        } catch (Exception e) {
            log.error("랭킹 보정 실패: productId={}", item.getProductId(), e);
            item.setCorrected(false);
            item.setErrorMessage(e.getMessage());
            return item;
        }
    }
}
```

---

## 6. 성능 최적화 전략

### ⚡ 최적화 기법

#### 6.1 청크 크기 조정
```java
@Bean
public Step optimizedValidationStep() {
    return stepBuilderFactory.get("validationStep")
        .<Product, RankingValidationResult>chunk(getOptimalChunkSize())
        .reader(productReader())
        .processor(rankingValidator())
        .writer(validationResultWriter())
        .build();
}

private int getOptimalChunkSize() {
    // 시스템 리소스에 따른 동적 청크 크기 계산
    long availableMemory = Runtime.getRuntime().freeMemory();
    int cores = Runtime.getRuntime().availableProcessors();
    
    // 메모리가 충분하고 CPU 코어가 많으면 더 큰 청크 사용
    if (availableMemory > 1024 * 1024 * 1024 && cores >= 8) { // 1GB 이상, 8코어 이상
        return 2000;
    } else if (availableMemory > 512 * 1024 * 1024 && cores >= 4) { // 512MB 이상, 4코어 이상
        return 1000;
    } else {
        return 500;
    }
}
```

#### 6.2 병렬 처리
```java
@Bean
public Step parallelValidationStep() {
    return stepBuilderFactory.get("parallelValidationStep")
        .<Product, RankingValidationResult>chunk(1000)
        .reader(productReader())
        .processor(rankingValidator())
        .writer(validationResultWriter())
        .taskExecutor(taskExecutor()) // 병렬 처리
        .throttleLimit(4) // 동시 실행 스레드 수 제한
        .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("batch-");
    executor.initialize();
    return executor;
}
```

#### 6.3 파티셔닝
```java
@Bean
public Step managerStep() {
    return stepBuilderFactory.get("managerStep")
        .partitioner("workerStep", productPartitioner())
        .step(workerStep())
        .gridSize(4) // 4개 파티션으로 분할
        .taskExecutor(taskExecutor())
        .build();
}

@Bean
public Partitioner productPartitioner() {
    return new ProductPartitioner(productRepository);
}

// 상품 ID 범위별로 파티션 분할
public class ProductPartitioner implements Partitioner {
    
    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Map<String, ExecutionContext> partitions = new HashMap<>();
        
        // 전체 상품 수 조회
        long totalProducts = productRepository.count();
        long partitionSize = totalProducts / gridSize;
        
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext context = new ExecutionContext();
            context.putLong("minId", i * partitionSize);
            context.putLong("maxId", (i + 1) * partitionSize - 1);
            partitions.put("partition" + i, context);
        }
        
        return partitions;
    }
}
```

#### 6.4 데이터베이스 최적화
```java
@Bean
public ItemReader<Product> optimizedProductReader() {
    return new JpaPagingItemReaderBuilder<Product>()
        .name("productReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("""
            SELECT p FROM Product p 
            WHERE p.id BETWEEN :minId AND :maxId 
            AND p.status = 'ACTIVE'
            ORDER BY p.id
            """)
        .parameterValues(getPartitionParameters())
        .pageSize(1000)
        .saveState(false) // 상태 저장 비활성화로 성능 향상
        .build();
}
```

### 📊 모니터링 및 메트릭

```java
@Component
@RequiredArgsConstructor
public class BatchMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void handleStepExecution(StepExecutionEvent event) {
        StepExecution stepExecution = event.getStepExecution();
        
        // 처리 속도 메트릭
        Timer.Sample sample = Timer.start(meterRegistry);
        sample.stop(Timer.builder("batch.step.duration")
            .tag("job", stepExecution.getJobExecution().getJobInstance().getJobName())
            .tag("step", stepExecution.getStepName())
            .register(meterRegistry));
        
        // 처리량 메트릭
        Gauge.builder("batch.step.read.count")
            .tag("step", stepExecution.getStepName())
            .register(meterRegistry, stepExecution, StepExecution::getReadCount);
        
        Gauge.builder("batch.step.write.count")
            .tag("step", stepExecution.getStepName())
            .register(meterRegistry, stepExecution, StepExecution::getWriteCount);
    }
}
```

---

## 7. 실제 코드 예시

### 🎯 완전한 랭킹 정합성 검증 배치

#### 7.1 Job 구성
```java
@Configuration
@RequiredArgsConstructor
@Slf4j
public class RankingValidationJobConfiguration {
    
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final EntityManagerFactory entityManagerFactory;
    
    @Bean
    public Job rankingValidationJob() {
        return jobBuilderFactory.get("rankingValidationJob")
            .incrementer(new RunIdIncrementer())
            .listener(jobExecutionListener())
            .start(preparationStep())
            .next(validationStep())
            .next(correctionStep())
            .next(cleanupStep())
            .build();
    }
    
    @Bean
    public Step preparationStep() {
        return stepBuilderFactory.get("preparationStep")
            .tasklet((contribution, chunkContext) -> {
                log.info("=== 랭킹 검증 배치 시작 ===");
                
                // 검증 대상 날짜 설정
                String targetDate = LocalDate.now().toString();
                chunkContext.getStepContext()
                    .getStepExecution()
                    .getJobExecution()
                    .getExecutionContext()
                    .putString("targetDate", targetDate);
                
                log.info("검증 대상 날짜: {}", targetDate);
                return RepeatStatus.FINISHED;
            })
            .build();
    }
    
    @Bean
    public Step validationStep() {
        return stepBuilderFactory.get("validationStep")
            .<Product, ValidationResult>chunk(1000)
            .reader(productReader())
            .processor(validationProcessor())
            .writer(validationWriter())
            .faultTolerant()
            .skipLimit(100)
            .skip(Exception.class)
            .listener(stepExecutionListener())
            .build();
    }
    
    @Bean
    public Step correctionStep() {
        return stepBuilderFactory.get("correctionStep")
            .<ValidationResult, CorrectionResult>chunk(500)
            .reader(mismatchReader())
            .processor(correctionProcessor())
            .writer(correctionWriter())
            .build();
    }
    
    @Bean
    public Step cleanupStep() {
        return stepBuilderFactory.get("cleanupStep")
            .tasklet((contribution, chunkContext) -> {
                // 임시 데이터 정리
                log.info("=== 랭킹 검증 배치 완료 ===");
                return RepeatStatus.FINISHED;
            })
            .build();
    }
}
```

#### 7.2 데이터 모델
```java
@Entity
@Table(name = "validation_results")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ValidationResult {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "product_id", nullable = false)
    private Long productId;
    
    @Column(name = "product_name")
    private String productName;
    
    @Column(name = "redis_score")
    private Long redisScore;
    
    @Column(name = "actual_orders")
    private Long actualOrders;
    
    @Column(name = "is_valid")
    private Boolean isValid;
    
    @Column(name = "difference")
    private Long difference;
    
    @Column(name = "validated_at")
    private LocalDateTime validatedAt;
    
    @Column(name = "batch_date")
    private LocalDate batchDate;
    
    @PrePersist
    private void prePersist() {
        this.difference = Math.abs(redisScore - actualOrders);
        this.validatedAt = LocalDateTime.now();
        this.batchDate = LocalDate.now();
    }
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CorrectionResult {
    private Long productId;
    private Long oldScore;
    private Long newScore;
    private boolean corrected;
    private String errorMessage;
    private LocalDateTime correctedAt;
}
```

#### 7.3 핵심 처리 로직
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class RankingValidationProcessor implements ItemProcessor<Product, ValidationResult> {
    
    private final RedisRankingService redisRankingService;
    private final OrderStatisticsService orderStatisticsService;
    
    @Override
    public ValidationResult process(Product product) throws Exception {
        Long productId = product.getId();
        
        try {
            // 1. Redis 랭킹 점수 조회
            Double redisScore = redisRankingService.getProductScore(productId, "daily");
            Long redisValue = redisScore != null ? redisScore.longValue() : 0L;
            
            // 2. DB 실제 주문량 조회 (오늘 기준)
            LocalDate today = LocalDate.now();
            Long actualOrders = orderStatisticsService.getProductOrderCount(productId, today);
            
            // 3. 검증 결과 생성
            boolean isValid = Objects.equals(redisValue, actualOrders);
            
            ValidationResult result = ValidationResult.builder()
                .productId(productId)
                .productName(product.getName())
                .redisScore(redisValue)
                .actualOrders(actualOrders)
                .isValid(isValid)
                .build();
            
            // 4. 유효한 데이터는 필터링 (불일치 데이터만 처리)
            if (isValid) {
                log.debug("검증 통과: productId={}, score={}", productId, redisValue);
                return null;
            }
            
            log.warn("검증 실패: productId={}, redis={}, actual={}", 
                    productId, redisValue, actualOrders);
            
            return result;
            
        } catch (Exception e) {
            log.error("검증 처리 오류: productId={}", productId, e);
            throw e;
        }
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
public class RankingCorrectionProcessor implements ItemProcessor<ValidationResult, CorrectionResult> {
    
    private final RedisRankingService redisRankingService;
    private final NotificationService notificationService;
    
    @Override
    public CorrectionResult process(ValidationResult validation) throws Exception {
        Long productId = validation.getProductId();
        Long oldScore = validation.getRedisScore();
        Long newScore = validation.getActualOrders();
        
        try {
            // Redis 랭킹 점수 보정
            redisRankingService.setProductScore(productId, newScore, "daily");
            
            CorrectionResult result = CorrectionResult.builder()
                .productId(productId)
                .oldScore(oldScore)
                .newScore(newScore)
                .corrected(true)
                .correctedAt(LocalDateTime.now())
                .build();
            
            log.info("랭킹 보정 완료: productId={}, {} → {}", productId, oldScore, newScore);
            
            // 큰 차이가 나는 경우 알림
            if (Math.abs(oldScore - newScore) > 50) {
                notificationService.sendRankingCorrectionAlert(productId, oldScore, newScore);
            }
            
            return result;
            
        } catch (Exception e) {
            log.error("랭킹 보정 실패: productId={}", productId, e);
            
            return CorrectionResult.builder()
                .productId(productId)
                .oldScore(oldScore)
                .newScore(newScore)
                .corrected(false)
                .errorMessage(e.getMessage())
                .correctedAt(LocalDateTime.now())
                .build();
        }
    }
}
```

#### 7.4 배치 실행 및 모니터링
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class RankingValidationScheduler {
    
    private final JobLauncher jobLauncher;
    private final Job rankingValidationJob;
    
    @Scheduled(cron = "0 0 2 * * *") // 매일 새벽 2시
    public void runRankingValidation() {
        try {
            JobParameters jobParameters = new JobParametersBuilder()
                .addString("targetDate", LocalDate.now().toString())
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();
            
            JobExecution jobExecution = jobLauncher.run(rankingValidationJob, jobParameters);
            
            log.info("랭킹 검증 배치 실행: jobId={}, status={}", 
                    jobExecution.getId(), jobExecution.getStatus());
            
        } catch (Exception e) {
            log.error("랭킹 검증 배치 실행 실패", e);
        }
    }
}

@Component
@Slf4j
public class BatchJobListener implements JobExecutionListener {
    
    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("=== Job 시작: {} ===", jobExecution.getJobInstance().getJobName());
    }
    
    @Override
    public void afterJob(JobExecution jobExecution) {
        BatchStatus status = jobExecution.getStatus();
        String jobName = jobExecution.getJobInstance().getJobName();
        Duration duration = Duration.between(
            jobExecution.getStartTime().toInstant(), 
            jobExecution.getEndTime().toInstant()
        );
        
        log.info("=== Job 완료: {} === 상태: {}, 소요시간: {}초", 
                jobName, status, duration.getSeconds());
        
        // 실행 통계 로그
        jobExecution.getStepExecutions().forEach(stepExecution -> {
            log.info("Step: {}, 읽기: {}, 쓰기: {}, 건너뛰기: {}", 
                    stepExecution.getStepName(),
                    stepExecution.getReadCount(),
                    stepExecution.getWriteCount(),
                    stepExecution.getSkipCount());
        });
    }
}
```

### 🧪 테스트 코드
```java
@SpringBatchTest
@SpringBootTest
@Testcontainers
class RankingValidationJobTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    
    @Autowired
    private TestJobLauncher jobLauncher;
    
    @Autowired
    private Job rankingValidationJob;
    
    @Test
    void 랭킹_검증_배치_테스트() throws Exception {
        // Given
        JobParameters jobParameters = new JobParametersBuilder()
            .addString("targetDate", LocalDate.now().toString())
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters();
        
        // When
        JobExecution jobExecution = jobLauncher.run(rankingValidationJob, jobParameters);
        
        // Then
        assertThat(jobExecution.getStatus()).isEqualTo(BatchStatus.COMPLETED);
        assertThat(jobExecution.getStepExecutions()).hasSize(4);
    }
}
```

---

## 📚 정리 및 핵심 포인트

### ✅ Spring Batch를 사용해야 하는 이유
1. **안정성**: 청크 단위 트랜잭션으로 부분 실패 허용
2. **재시작**: 실패 지점부터 이어서 처리 가능
3. **모니터링**: 상세한 실행 통계와 로그 제공
4. **확장성**: 병렬 처리와 파티셔닝 지원

### 🎯 적용 시 주의사항
1. **청크 크기**: 메모리와 성능을 고려한 적절한 크기 설정
2. **트랜잭션 범위**: 청크별 트랜잭션으로 안정성 확보
3. **에러 처리**: skip, retry 정책으로 장애 대응
4. **모니터링**: 실행 상태와 성능 지표 추적

### 🚀 이번 과제 적용 방안
1. **정합성 검증**: Redis 랭킹과 DB 데이터 일치성 확인
2. **데이터 보정**: 불일치 데이터 자동 수정
3. **정기 실행**: 스케줄러로 주기적 배치 실행
4. **알림 시스템**: 심각한 불일치 발견 시 즉시 알림

이제 Spring Batch를 활용하여 안정적이고 확장 가능한 데이터 정합성 보장 시스템을 구축할 수 있습니다!