# Spring Events 비동기 이벤트 처리

## 📋 목차
1. [Spring Events 기본 개념](#1-spring-events-기본-개념)
2. [동기 vs 비동기 이벤트 처리](#2-동기-vs-비동기-이벤트-처리)
3. [비동기 이벤트 설정과 최적화](#3-비동기-이벤트-설정과-최적화)
4. [트랜잭션과 이벤트 연동](#4-트랜잭션과-이벤트-연동)
5. [에러 처리와 복구 전략](#5-에러-처리와-복구-전략)
6. [실무 활용 패턴](#6-실무-활용-패턴)

---

## 1. Spring Events 기본 개념

### 🎯 Spring Events란?

**Spring Events**는 **애플리케이션 내에서 느슨하게 결합된 컴포넌트 간 통신을 위한 이벤트 기반 프로그래밍 모델**입니다.

### 🎭 실생활 비유: 연극 무대

연극 공연에서 벌어지는 상황을 생각해보세요:

```
무대 위 배우 (Publisher) → 이벤트 발생 → 무대 뒤 스태프들 (Listeners)
     ↓                      ↓                     ↓
"전쟁 장면 시작!"      CueEvent 발생         각자 역할 수행

┌──────────────┐     ┌─────────────────┐     ┌──────────────┐
│ 주인공 배우   │ →  │ "전쟁 장면" 신호  │ →  │ 조명 담당자   │
└──────────────┘     └─────────────────┘     │ 음향 담당자   │
                                              │ 분장 담당자   │
                                              │ 소품 담당자   │
                                              └──────────────┘
```

**연극과 Spring Events의 유사점:**
- **신호 발신**: 배우가 특정 신호를 보냄 (이벤트 발행)
- **각자 대응**: 스태프들이 독립적으로 반응 (이벤트 리스너)
- **느슨한 결합**: 배우는 스태프 세부사항을 모름
- **동시 진행**: 여러 스태프가 동시에 작업 (비동기 처리)

### 🔧 Spring Events 구조

```
┌─────────────────────────────────────────────────────┐
│                ApplicationContext                    │
├─────────────────────────────────────────────────────┤
│ Event Publisher → ApplicationEventMulticaster → Listeners │
│       ↓                      ↓                      ↓ │
│ OrderService    →    Event Bus    →      다양한 리스너  │
│ PaymentService  →      (동기)      →   @EventListener │
│ CouponService   →    (비동기)     →    @Async        │
└─────────────────────────────────────────────────────┘
```

### 💡 핵심 특징

1. **타입 세이프**: 컴파일 타임에 이벤트 타입 검증
2. **느슨한 결합**: Publisher와 Listener가 서로 몰라도 됨
3. **유연한 처리**: 동기/비동기 선택 가능
4. **트랜잭션 연동**: 트랜잭션 생명주기와 연동 가능
5. **조건부 처리**: SpEL 표현식으로 조건부 실행

---

## 2. 동기 vs 비동기 이벤트 처리

### 🔄 처리 방식 비교

#### 2.1 동기 이벤트 처리
```java
@EventListener
public void handleOrderCompleted(OrderCompletedEvent event) {
    // 메인 스레드에서 즉시 실행
    updateInventory(event.getOrderItems());     // 1초
    sendEmail(event.getUserId());               // 2초  
    updateAnalytics(event);                     // 1초
    // 총 4초 후 메서드 완료
}

// 호출하는 쪽
public void completeOrder(Long orderId) {
    Order order = processOrder(orderId);        // 1초
    eventPublisher.publishEvent(event);         // 4초 (위 리스너 실행)
    return order;                               // 총 5초 후 응답
}
```

#### 2.2 비동기 이벤트 처리
```java
@EventListener
@Async
public void handleOrderCompleted(OrderCompletedEvent event) {
    // 별도 스레드에서 실행
    updateInventory(event.getOrderItems());     // 백그라운드 실행
    sendEmail(event.getUserId());               // 백그라운드 실행
    updateAnalytics(event);                     // 백그라운드 실행
}

// 호출하는 쪽
public void completeOrder(Long orderId) {
    Order order = processOrder(orderId);        // 1초
    eventPublisher.publishEvent(event);         // 즉시 반환
    return order;                               // 1초 후 즉시 응답!
}
```

### 📊 성능 비교

| 구분 | 동기 처리 | 비동기 처리 |
|------|-----------|-------------|
| **응답 시간** | 5초 | 1초 |
| **사용자 경험** | 느림 | 빠름 |
| **에러 처리** | 즉시 감지 | 별도 처리 필요 |
| **트랜잭션** | 공유 | 별도 |
| **순서 보장** | 보장됨 | 보장 안됨 |

### 🎯 선택 기준

```java
// 동기 처리가 적절한 경우
@EventListener
public void validateOrderPayment(OrderCompletedEvent event) {
    // 결제 검증은 주문 완료와 동시에 이뤄져야 함
    if (!paymentValidator.isValid(event.getPaymentInfo())) {
        throw new PaymentValidationException("결제 정보 오류");
    }
}

// 비동기 처리가 적절한 경우  
@EventListener
@Async
public void sendWelcomeEmail(UserRegisteredEvent event) {
    // 이메일 발송은 회원가입과 독립적으로 처리 가능
    emailService.sendWelcomeEmail(event.getUser());
}
```

### 🔍 실제 응답 시간 측정

```java
@RestController
@RequiredArgsConstructor
public class OrderController {
    
    private final OrderService orderService;
    
    @PostMapping("/orders/{orderId}/complete")
    public ResponseEntity<OrderResponse> completeOrder(@PathVariable Long orderId) {
        long startTime = System.currentTimeMillis();
        
        Order order = orderService.completeOrder(orderId);
        
        long endTime = System.currentTimeMillis();
        log.info("주문 완료 처리 시간: {}ms", endTime - startTime);
        
        return ResponseEntity.ok(OrderResponse.from(order));
    }
}

// 동기 처리 결과: 4,892ms
// 비동기 처리 결과: 156ms (96% 성능 향상!)
```

---

## 3. 비동기 이벤트 설정과 최적화

### ⚙️ 비동기 설정

#### 3.1 기본 비동기 설정
```java
@Configuration
@EnableAsync
public class AsyncEventConfig {
    
    @Bean(name = "eventTaskExecutor")
    public TaskExecutor eventTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        
        // 기본 설정
        executor.setCorePoolSize(5);        // 기본 스레드 수
        executor.setMaxPoolSize(10);        // 최대 스레드 수  
        executor.setQueueCapacity(100);     // 큐 용량
        executor.setThreadNamePrefix("event-");
        
        // 스레드 관리
        executor.setKeepAliveSeconds(60);   // 유휴 스레드 생존 시간
        executor.setAllowCoreThreadTimeOut(true);
        
        // 거절 정책
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        
        executor.initialize();
        return executor;
    }
    
    @Bean
    public ApplicationEventMulticaster applicationEventMulticaster() {
        SimpleApplicationEventMulticaster eventMulticaster = 
            new SimpleApplicationEventMulticaster();
        eventMulticaster.setTaskExecutor(eventTaskExecutor());
        return eventMulticaster;
    }
}
```

#### 3.2 도메인별 스레드 풀 분리
```java
@Configuration
@EnableAsync
public class DomainSpecificAsyncConfig {
    
    /**
     * 주문 관련 이벤트 처리용 스레드 풀
     */
    @Bean(name = "orderEventExecutor")
    public TaskExecutor orderEventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(6);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("order-event-");
        executor.initialize();
        return executor;
    }
    
    /**
     * 알림 관련 이벤트 처리용 스레드 풀
     */
    @Bean(name = "notificationEventExecutor")  
    public TaskExecutor notificationEventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("notification-event-");
        executor.initialize();
        return executor;
    }
    
    /**
     * 분석 관련 이벤트 처리용 스레드 풀 (배치 처리)
     */
    @Bean(name = "analyticsEventExecutor")
    public TaskExecutor analyticsEventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(1);
        executor.setMaxPoolSize(2);
        executor.setQueueCapacity(1000);    // 큰 큐로 배치 처리
        executor.setThreadNamePrefix("analytics-event-");
        executor.initialize();
        return executor;
    }
}
```

#### 3.3 이벤트 리스너별 스레드 풀 지정
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventHandler {
    
    /**
     * 주문 관련 중요 로직 - 전용 스레드 풀
     */
    @EventListener
    @Async("orderEventExecutor")
    public void updateInventory(OrderCompletedEvent event) {
        try {
            inventoryService.decreaseStock(event.getOrderItems());
            log.info("재고 업데이트 완료: orderId={}", event.getOrderId());
        } catch (Exception e) {
            log.error("재고 업데이트 실패: orderId={}", event.getOrderId(), e);
        }
    }
    
    /**
     * 알림 발송 - 알림 전용 스레드 풀
     */
    @EventListener
    @Async("notificationEventExecutor")
    public void sendOrderConfirmation(OrderCompletedEvent event) {
        try {
            emailService.sendOrderConfirmation(event);
            smsService.sendOrderNotification(event);
            log.info("주문 알림 발송 완료: orderId={}", event.getOrderId());
        } catch (Exception e) {
            log.error("주문 알림 발송 실패: orderId={}", event.getOrderId(), e);
        }
    }
    
    /**
     * 분석 데이터 처리 - 분석 전용 스레드 풀
     */
    @EventListener
    @Async("analyticsEventExecutor")
    public void updateAnalytics(OrderCompletedEvent event) {
        try {
            analyticsService.recordOrderCompleted(event);
            rankingService.updateProductRanking(event);
            log.info("분석 데이터 업데이트 완료: orderId={}", event.getOrderId());
        } catch (Exception e) {
            log.error("분석 데이터 업데이트 실패: orderId={}", event.getOrderId(), e);
        }
    }
}
```

### 📊 성능 모니터링

#### 3.4 스레드 풀 모니터링
```java
@Component
@RequiredArgsConstructor
public class AsyncEventMetrics {
    
    private final MeterRegistry meterRegistry;
    
    @Autowired
    @Qualifier("orderEventExecutor")
    private ThreadPoolTaskExecutor orderEventExecutor;
    
    @PostConstruct
    public void initMetrics() {
        // 스레드 풀 상태 메트릭
        Gauge.builder("thread.pool.active")
            .tag("pool", "order-event")
            .register(meterRegistry, orderEventExecutor, 
                     executor -> executor.getActiveCount());
        
        Gauge.builder("thread.pool.queue.size")
            .tag("pool", "order-event")
            .register(meterRegistry, orderEventExecutor,
                     executor -> executor.getThreadPoolExecutor().getQueue().size());
    }
    
    /**
     * 이벤트 처리 시간 측정
     */
    @EventListener
    public void measureEventProcessingTime(OrderCompletedEvent event) {
        Timer.Sample sample = Timer.start(meterRegistry);
        // 실제 처리는 다른 리스너에서...
        sample.stop(Timer.builder("event.processing.duration")
                    .tag("event", "order.completed")
                    .register(meterRegistry));
    }
}
```

#### 3.5 동적 스레드 풀 조정
```java
@Component
@RequiredArgsConstructor
public class DynamicThreadPoolManager {
    
    @Autowired
    @Qualifier("orderEventExecutor")
    private ThreadPoolTaskExecutor orderEventExecutor;
    
    /**
     * 부하에 따른 동적 스레드 풀 조정
     */
    @Scheduled(fixedRate = 30000) // 30초마다
    public void adjustThreadPoolSize() {
        ThreadPoolExecutor executor = orderEventExecutor.getThreadPoolExecutor();
        
        int queueSize = executor.getQueue().size();
        int activeCount = executor.getActiveCount();
        int corePoolSize = executor.getCorePoolSize();
        
        // 큐가 가득 차면 스레드 수 증가
        if (queueSize > 40 && corePoolSize < 10) {
            executor.setCorePoolSize(corePoolSize + 1);
            log.info("스레드 풀 크기 증가: {} → {}", corePoolSize, corePoolSize + 1);
        }
        // 큐가 비어있고 스레드가 유휴상태면 감소
        else if (queueSize < 5 && activeCount < corePoolSize / 2 && corePoolSize > 3) {
            executor.setCorePoolSize(corePoolSize - 1);
            log.info("스레드 풀 크기 감소: {} → {}", corePoolSize, corePoolSize - 1);
        }
    }
}
```

---

## 4. 트랜잭션과 이벤트 연동

### 🔄 트랜잭션 이벤트 리스너

#### 4.1 기본 트랜잭션 연동
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class TransactionalEventHandler {
    
    /**
     * 트랜잭션 커밋 후 실행 (기본)
     */
    @TransactionalEventListener
    @Async
    public void handleOrderCompletedAfterCommit(OrderCompletedEvent event) {
        // 트랜잭션이 성공적으로 커밋된 후에만 실행
        emailService.sendOrderConfirmation(event);
        log.info("주문 완료 후 이메일 발송: orderId={}", event.getOrderId());
    }
    
    /**
     * 트랜잭션 커밋 전 실행
     */
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void validateBeforeCommit(OrderCompletedEvent event) {
        // 트랜잭션 커밋 전 최종 검증
        if (!orderValidator.isFinalValid(event.getOrderId())) {
            throw new OrderValidationException("최종 검증 실패");
        }
    }
    
    /**
     * 트랜잭션 롤백 시 실행
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleOrderRollback(OrderCompletedEvent event) {
        // 롤백 시 보상 처리
        compensationService.compensateOrder(event.getOrderId());
        log.warn("주문 롤백으로 인한 보상 처리: orderId={}", event.getOrderId());
    }
    
    /**
     * 트랜잭션 완료 시 실행 (커밋/롤백 무관)
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void handleOrderCompletion(OrderCompletedEvent event) {
        // 트랜잭션 결과와 무관하게 항상 실행
        auditService.logOrderProcessing(event.getOrderId());
    }
}
```

#### 4.2 트랜잭션 상태별 처리 패턴
```java
@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public void completeOrder(Long orderId) {
        try {
            // 1. 주문 처리 (트랜잭션 내)
            Order order = processOrder(orderId);
            
            // 2. 이벤트 발행 (트랜잭션 내)
            OrderCompletedEvent event = OrderCompletedEvent.from(order);
            eventPublisher.publishEvent(event);
            
            // 3. 추가 비즈니스 로직
            updateRelatedEntities(order);
            
        } catch (Exception e) {
            // 롤백 시 TransactionPhase.AFTER_ROLLBACK 리스너 실행
            log.error("주문 처리 실패: orderId={}", orderId, e);
            throw e;
        }
        // 성공 시 TransactionPhase.AFTER_COMMIT 리스너 실행
    }
}
```

#### 4.3 조건부 이벤트 처리
```java
@Component
public class ConditionalEventHandler {
    
    /**
     * 조건부 이벤트 처리 (SpEL 사용)
     */
    @EventListener(condition = "#event.totalAmount > 100000")
    @Async
    public void handleLargeOrder(OrderCompletedEvent event) {
        // 10만원 이상 주문에 대해서만 처리
        vipService.processLargeOrder(event);
    }
    
    @EventListener(condition = "#event.isFirstOrder")
    @Async
    public void handleFirstOrder(OrderCompletedEvent event) {
        // 첫 주문인 경우만 처리
        welcomeService.sendFirstOrderBonus(event.getUserId());
    }
    
    @EventListener(condition = "#event.orderItems.size() > 5")
    @Async  
    public void handleBulkOrder(OrderCompletedEvent event) {
        // 주문 상품이 5개 이상인 경우
        bulkOrderService.processBulkDiscount(event);
    }
}
```

### 🛡️ 트랜잭션 안전성 보장

#### 4.4 안전한 이벤트 발행 패턴
```java
@Service
@RequiredArgsConstructor
public class SafeEventPublishingService {
    
    private final ApplicationEventPublisher eventPublisher;
    private final TransactionTemplate transactionTemplate;
    
    /**
     * 트랜잭션 성공 후에만 이벤트 발행
     */
    public void safeProcessOrder(Long orderId) {
        Order order = transactionTemplate.execute(status -> {
            // 트랜잭션 내에서 주문 처리
            return processOrderInTransaction(orderId);
        });
        
        if (order != null) {
            // 트랜잭션 완료 후 이벤트 발행
            OrderCompletedEvent event = OrderCompletedEvent.from(order);
            eventPublisher.publishEvent(event);
        }
    }
    
    /**
     * 이벤트 발행 실패 시 보상 처리
     */
    public void processOrderWithCompensation(Long orderId) {
        Order order = null;
        
        try {
            order = transactionTemplate.execute(status -> 
                processOrderInTransaction(orderId));
            
            // 이벤트 발행
            OrderCompletedEvent event = OrderCompletedEvent.from(order);
            eventPublisher.publishEvent(event);
            
        } catch (Exception e) {
            if (order != null) {
                // 이벤트 발행 실패 시 주문 상태 보정
                compensationService.revertOrderStatus(order.getId());
            }
            throw e;
        }
    }
}
```

---

## 5. 에러 처리와 복구 전략

### 🚨 에러 처리 패턴

#### 5.1 기본 에러 처리
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class ResilientEventHandler {
    
    /**
     * 재시도 가능한 이벤트 처리
     */
    @EventListener
    @Async
    @Retryable(
        value = {TransientException.class, RedisConnectionException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public void handleOrderWithRetry(OrderCompletedEvent event) {
        try {
            // 외부 서비스 호출 (실패 가능성 있음)
            externalPaymentService.notifyPaymentCompleted(event.getOrderId());
            
            log.info("외부 결제 서비스 알림 성공: orderId={}", event.getOrderId());
            
        } catch (Exception e) {
            log.error("외부 결제 서비스 알림 실패 (재시도 예정): orderId={}, attempt={}", 
                     event.getOrderId(), getCurrentRetryCount(), e);
            throw e; // 재시도를 위해 예외 전파
        }
    }
    
    /**
     * 재시도 실패 시 복구 처리
     */
    @Recover
    public void recoverFromPaymentNotificationFailure(Exception ex, OrderCompletedEvent event) {
        log.error("외부 결제 서비스 알림 최종 실패: orderId={}", event.getOrderId(), ex);
        
        // Dead Letter Queue에 저장
        deadLetterQueueService.saveFailedEvent(event, ex);
        
        // 관리자 알림
        alertService.sendFailureAlert("결제 알림 실패", event.getOrderId(), ex);
    }
}
```

#### 5.2 Circuit Breaker 패턴
```java
@Component
@RequiredArgsConstructor  
public class CircuitBreakerEventHandler {
    
    /**
     * Circuit Breaker를 활용한 외부 서비스 호출
     */
    @EventListener
    @Async
    @CircuitBreaker(name = "external-service", fallbackMethod = "fallbackNotification")
    public void notifyExternalService(OrderCompletedEvent event) {
        try {
            externalService.notify(event);
            log.info("외부 서비스 알림 성공: orderId={}", event.getOrderId());
            
        } catch (Exception e) {
            log.error("외부 서비스 알림 실패: orderId={}", event.getOrderId(), e);
            throw e;
        }
    }
    
    /**
     * Circuit Breaker 열림 시 Fallback 처리
     */
    public void fallbackNotification(OrderCompletedEvent event, Exception ex) {
        log.warn("외부 서비스 Circuit Breaker 작동, Fallback 처리: orderId={}", 
                event.getOrderId());
        
        // 대안 처리: 내부 큐에 저장하여 나중에 재처리
        internalQueueService.enqueueForLaterProcessing(event);
        
        // 모니터링 알림
        monitoringService.recordCircuitBreakerActivation("external-service");
    }
}
```

#### 5.3 Dead Letter Queue 구현
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DeadLetterQueueService {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    
    /**
     * 실패한 이벤트를 Dead Letter Queue에 저장
     */
    public void saveFailedEvent(Object event, Exception error) {
        try {
            FailedEvent failedEvent = FailedEvent.builder()
                .eventType(event.getClass().getSimpleName())
                .eventData(objectMapper.writeValueAsString(event))
                .errorMessage(error.getMessage())
                .failedAt(LocalDateTime.now())
                .retryCount(0)
                .build();
            
            String dlqKey = "dlq:events:" + event.getClass().getSimpleName();
            redisTemplate.opsForList().leftPush(dlqKey, 
                objectMapper.writeValueAsString(failedEvent));
            
            log.info("Dead Letter Queue에 이벤트 저장: type={}", event.getClass().getSimpleName());
            
        } catch (Exception e) {
            log.error("Dead Letter Queue 저장 실패", e);
        }
    }
    
    /**
     * Dead Letter Queue에서 이벤트 재처리
     */
    @Scheduled(fixedDelay = 60000) // 1분마다
    public void reprocessFailedEvents() {
        Set<String> dlqKeys = redisTemplate.keys("dlq:events:*");
        
        for (String dlqKey : dlqKeys) {
            try {
                String failedEventJson = redisTemplate.opsForList().rightPop(dlqKey);
                if (failedEventJson != null) {
                    FailedEvent failedEvent = objectMapper.readValue(failedEventJson, FailedEvent.class);
                    
                    if (shouldRetry(failedEvent)) {
                        reprocessEvent(failedEvent);
                    } else {
                        moveToPermanentFailure(failedEvent);
                    }
                }
            } catch (Exception e) {
                log.error("Failed event 재처리 중 오류: key={}", dlqKey, e);
            }
        }
    }
    
    private boolean shouldRetry(FailedEvent failedEvent) {
        return failedEvent.getRetryCount() < 5 && 
               failedEvent.getFailedAt().isAfter(LocalDateTime.now().minusHours(24));
    }
}
```

### 🔄 이벤트 재처리 시스템

#### 5.4 지능형 재처리
```java
@Component
@RequiredArgsConstructor
public class IntelligentEventReprocessor {
    
    private final ApplicationEventPublisher eventPublisher;
    
    /**
     * 백오프 전략을 사용한 재처리
     */
    public void reprocessWithBackoff(FailedEvent failedEvent) {
        int retryCount = failedEvent.getRetryCount();
        
        // 지수 백오프: 1초, 2초, 4초, 8초, 16초
        long delayMs = (long) Math.pow(2, retryCount) * 1000;
        
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.schedule(() -> {
            try {
                Object originalEvent = deserializeEvent(failedEvent);
                eventPublisher.publishEvent(originalEvent);
                
                log.info("이벤트 재처리 성공: type={}, retryCount={}", 
                        failedEvent.getEventType(), retryCount);
                
            } catch (Exception e) {
                log.error("이벤트 재처리 실패: type={}, retryCount={}", 
                         failedEvent.getEventType(), retryCount, e);
                
                // 재시도 횟수 증가 후 다시 DLQ에 저장
                failedEvent.incrementRetryCount();
                deadLetterQueueService.saveFailedEvent(failedEvent, e);
            }
        }, delayMs, TimeUnit.MILLISECONDS);
    }
    
    /**
     * 배치 재처리 (성능 최적화)
     */
    @Scheduled(cron = "0 0 2 * * *") // 매일 새벽 2시
    public void batchReprocessFailedEvents() {
        List<FailedEvent> failedEvents = deadLetterQueueService.getAllFailedEvents();
        
        // 타입별로 그룹화하여 배치 처리
        Map<String, List<FailedEvent>> groupedEvents = failedEvents.stream()
            .collect(Collectors.groupingBy(FailedEvent::getEventType));
        
        for (Map.Entry<String, List<FailedEvent>> entry : groupedEvents.entrySet()) {
            String eventType = entry.getKey();
            List<FailedEvent> events = entry.getValue();
            
            log.info("배치 재처리 시작: type={}, count={}", eventType, events.size());
            
            for (FailedEvent event : events) {
                try {
                    reprocessEvent(event);
                    Thread.sleep(100); // 시스템 부하 방지
                } catch (Exception e) {
                    log.error("배치 재처리 실패: eventId={}", event.getId(), e);
                }
            }
        }
    }
}
```

---

## 6. 실무 활용 패턴

### 🎯 완전한 주문 이벤트 시스템

#### 6.1 이벤트 정의
```java
@Getter
@Builder
@ToString
public class OrderCompletedEvent {
    
    private final String eventId;
    private final Long orderId;
    private final Long userId;
    private final BigDecimal totalAmount;
    private final List<OrderItem> orderItems;
    private final LocalDateTime completedAt;
    private final boolean isFirstOrder;
    private final String paymentMethod;
    
    @Getter
    @Builder
    public static class OrderItem {
        private final Long productId;
        private final String productName;
        private final Integer quantity;
        private final BigDecimal price;
    }
    
    public static OrderCompletedEvent from(Order order) {
        return OrderCompletedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .orderId(order.getId())
            .userId(order.getUserId())
            .totalAmount(order.getTotalAmount())
            .orderItems(convertOrderItems(order.getOrderItems()))
            .completedAt(LocalDateTime.now())
            .isFirstOrder(order.isFirstOrder())
            .paymentMethod(order.getPaymentMethod())
            .build();
    }
}
```

#### 6.2 종합 이벤트 핸들러
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class ComprehensiveOrderEventHandler {
    
    private final ProductRankingService rankingService;
    private final NotificationService notificationService;
    private final InventoryService inventoryService;
    private final AnalyticsService analyticsService;
    private final LoyaltyService loyaltyService;
    
    /**
     * 1. 상품 랭킹 업데이트 (높은 우선순위)
     */
    @EventListener
    @Async("orderEventExecutor")
    @Order(1)
    @Retryable(maxAttempts = 3)
    public void updateProductRanking(OrderCompletedEvent event) {
        log.info("상품 랭킹 업데이트 시작: orderId={}", event.getOrderId());
        
        for (OrderCompletedEvent.OrderItem item : event.getOrderItems()) {
            rankingService.updateProductRanking(item.getProductId(), item.getQuantity());
        }
        
        log.info("상품 랭킹 업데이트 완료: orderId={}", event.getOrderId());
    }
    
    /**
     * 2. 재고 업데이트 (중요도 높음)
     */
    @EventListener
    @Async("orderEventExecutor")
    @Order(2)
    @Transactional
    public void updateInventory(OrderCompletedEvent event) {
        log.info("재고 업데이트 시작: orderId={}", event.getOrderId());
        
        try {
            inventoryService.decreaseStockForOrder(event.getOrderItems());
            log.info("재고 업데이트 완료: orderId={}", event.getOrderId());
            
        } catch (InsufficientStockException e) {
            log.error("재고 부족으로 업데이트 실패: orderId={}", event.getOrderId(), e);
            // 주문 취소 이벤트 발행
            eventPublisher.publishEvent(new OrderCancellationEvent(event.getOrderId(), 
                "재고 부족으로 인한 주문 취소"));
        }
    }
    
    /**
     * 3. 고객 알림 발송
     */
    @EventListener
    @Async("notificationEventExecutor")
    @Order(3)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendCustomerNotifications(OrderCompletedEvent event) {
        log.info("고객 알림 발송 시작: orderId={}", event.getOrderId());
        
        try {
            // 이메일 알림
            notificationService.sendOrderConfirmationEmail(event);
            
            // SMS 알림 (고액 주문의 경우)
            if (event.getTotalAmount().compareTo(BigDecimal.valueOf(100000)) >= 0) {
                notificationService.sendOrderConfirmationSMS(event);
            }
            
            // 푸시 알림
            notificationService.sendPushNotification(event);
            
            log.info("고객 알림 발송 완료: orderId={}", event.getOrderId());
            
        } catch (Exception e) {
            log.error("고객 알림 발송 실패: orderId={}", event.getOrderId(), e);
            // 알림 실패는 주문 처리에 영향을 주지 않음
        }
    }
    
    /**
     * 4. 로열티 포인트 적립
     */
    @EventListener(condition = "#event.totalAmount > 10000")
    @Async("orderEventExecutor")
    @Order(4)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void awardLoyaltyPoints(OrderCompletedEvent event) {
        log.info("로열티 포인트 적립 시작: orderId={}", event.getOrderId());
        
        try {
            // 주문 금액의 1% 적립
            BigDecimal pointAmount = event.getTotalAmount()
                .multiply(BigDecimal.valueOf(0.01))
                .setScale(0, RoundingMode.DOWN);
            
            if (pointAmount.compareTo(BigDecimal.ZERO) > 0) {
                loyaltyService.awardPoints(event.getUserId(), pointAmount.longValue(),
                    "주문 완료 포인트 적립 (주문번호: " + event.getOrderId() + ")");
                
                log.info("로열티 포인트 적립 완료: orderId={}, points={}", 
                        event.getOrderId(), pointAmount);
            }
            
        } catch (Exception e) {
            log.error("로열티 포인트 적립 실패: orderId={}", event.getOrderId(), e);
        }
    }
    
    /**
     * 5. 첫 주문 보너스 처리
     */
    @EventListener(condition = "#event.isFirstOrder")
    @Async("orderEventExecutor")
    @Order(5)
    public void handleFirstOrderBonus(OrderCompletedEvent event) {
        log.info("첫 주문 보너스 처리 시작: orderId={}, userId={}", 
                event.getOrderId(), event.getUserId());
        
        try {
            // 첫 주문 쿠폰 발급
            loyaltyService.issueFirstOrderCoupon(event.getUserId());
            
            // 환영 이메일 발송
            notificationService.sendWelcomeEmail(event.getUserId());
            
            log.info("첫 주문 보너스 처리 완료: orderId={}", event.getOrderId());
            
        } catch (Exception e) {
            log.error("첫 주문 보너스 처리 실패: orderId={}", event.getOrderId(), e);
        }
    }
    
    /**
     * 6. 분석 데이터 수집 (낮은 우선순위)
     */
    @EventListener
    @Async("analyticsEventExecutor")
    @Order(6)
    public void collectAnalyticsData(OrderCompletedEvent event) {
        log.debug("분석 데이터 수집 시작: orderId={}", event.getOrderId());
        
        try {
            // 주문 분석 데이터 기록
            AnalyticsData analyticsData = AnalyticsData.builder()
                .orderId(event.getOrderId())
                .userId(event.getUserId())
                .totalAmount(event.getTotalAmount())
                .itemCount(event.getOrderItems().size())
                .paymentMethod(event.getPaymentMethod())
                .isFirstOrder(event.isFirstOrder())
                .timestamp(event.getCompletedAt())
                .build();
            
            analyticsService.recordOrderAnalytics(analyticsData);
            
            log.debug("분석 데이터 수집 완료: orderId={}", event.getOrderId());
            
        } catch (Exception e) {
            log.error("분석 데이터 수집 실패: orderId={}", event.getOrderId(), e);
            // 분석 데이터 수집 실패는 시스템에 영향을 주지 않음
        }
    }
}
```

#### 6.3 이벤트 모니터링 대시보드
```java
@RestController
@RequestMapping("/api/admin/events")
@RequiredArgsConstructor
public class EventMonitoringController {
    
    private final EventMetricsService eventMetricsService;
    private final DeadLetterQueueService deadLetterQueueService;
    
    /**
     * 이벤트 처리 통계 조회
     */
    @GetMapping("/metrics")
    public ResponseEntity<EventMetrics> getEventMetrics() {
        EventMetrics metrics = eventMetricsService.getEventMetrics();
        return ResponseEntity.ok(metrics);
    }
    
    /**
     * 실패한 이벤트 목록 조회
     */
    @GetMapping("/failed")
    public ResponseEntity<List<FailedEventInfo>> getFailedEvents(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        List<FailedEventInfo> failedEvents = deadLetterQueueService
            .getFailedEvents(page, size);
        return ResponseEntity.ok(failedEvents);
    }
    
    /**
     * 실패한 이벤트 수동 재처리
     */
    @PostMapping("/failed/{eventId}/reprocess")
    public ResponseEntity<Void> reprocessFailedEvent(@PathVariable String eventId) {
        deadLetterQueueService.reprocessEvent(eventId);
        return ResponseEntity.ok().build();
    }
}

@Data
@Builder
public class EventMetrics {
    private long totalEventsPublished;
    private long totalEventsProcessed;
    private long totalEventsFailed;
    private double successRate;
    private Map<String, Long> eventTypeStatistics;
    private Map<String, Double> averageProcessingTime;
    private List<ThreadPoolStatus> threadPoolStatuses;
}
```

### 🧪 테스트 코드

#### 6.4 비동기 이벤트 테스트
```java
@SpringBootTest
@TestMethodOrder(OrderAnnotation.class)
class AsyncEventHandlerTest {
    
    @MockBean
    private ProductRankingService rankingService;
    
    @MockBean
    private NotificationService notificationService;
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Test
    @Order(1)
    void 비동기_이벤트_처리_테스트() throws InterruptedException {
        // Given
        OrderCompletedEvent event = OrderCompletedEvent.builder()
            .orderId(101L)
            .userId(123L)
            .totalAmount(BigDecimal.valueOf(50000))
            .orderItems(List.of(
                OrderCompletedEvent.OrderItem.builder()
                    .productId(1L)
                    .quantity(2)
                    .build()
            ))
            .build();
        
        // When
        eventPublisher.publishEvent(event);
        
        // Then - 비동기 처리 완료까지 대기
        await().atMost(Duration.ofSeconds(5))
            .untilAsserted(() -> {
                verify(rankingService, times(1))
                    .updateProductRanking(1L, 2);
                verify(notificationService, times(1))
                    .sendOrderConfirmationEmail(event);
            });
    }
    
    @Test
    @Order(2)
    void 이벤트_처리_실패_시_재시도_테스트() throws InterruptedException {
        // Given
        OrderCompletedEvent event = createTestEvent();
        
        doThrow(new RedisConnectionException("Connection failed"))
            .doThrow(new RedisConnectionException("Connection failed again"))
            .doNothing()
            .when(rankingService).updateProductRanking(any(), any());
        
        // When
        eventPublisher.publishEvent(event);
        
        // Then - 3번째 재시도에서 성공
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                verify(rankingService, times(3))
                    .updateProductRanking(any(), any());
            });
    }
}
```

---

## 📚 정리 및 핵심 포인트

### ✅ Spring Events 비동기 처리 장점
1. **성능 향상**: 응답 시간 대폭 단축 (96% 개선 사례)
2. **확장성**: 도메인별 스레드 풀로 리소스 최적화
3. **안정성**: 트랜잭션 연동과 에러 복구 메커니즘
4. **모니터링**: 실시간 메트릭과 장애 추적

### 🎯 적용 시 주의사항
1. **스레드 풀 설정**: 적절한 크기와 큐 용량 설정
2. **트랜잭션 범위**: 이벤트 리스너의 트랜잭션 독립성
3. **에러 처리**: 재시도, Circuit Breaker, DLQ 전략
4. **순서 보장**: 비동기 처리 시 순서 고려

### 🚀 이번 과제 적용 방안
1. **주문 완료 이벤트**: 랭킹, 알림, 분석 등 비동기 처리
2. **성능 최적화**: 도메인별 스레드 풀 분리
3. **장애 대응**: Dead Letter Queue와 재처리 시스템
4. **모니터링**: 실시간 이벤트 처리 상태 추적

이제 Spring Events를 활용하여 고성능의 비동기 이벤트 처리 시스템을 구축할 수 있습니다!

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"content": "\ud604\uc7ac \uad6c\ud604 \uc0c1\ud0dc \uc7ac\ubd84\uc11d (AS-IS)", "status": "completed", "id": "1"}, {"content": "STEP 13: Redis Sorted Set \uae30\ubc18 \uc0c1\ud488 \ub7ad\ud0b9 \uc2dc\uc2a4\ud15c \uc124\uacc4", "status": "completed", "id": "2"}, {"content": "Redis \ud575\uc2ec \uae30\uc220 \uc2a4\ud0dd \ud559\uc2b5\uc790\ub8cc \uc791\uc131", "status": "completed", "id": "3"}, {"content": "STEP 13: \uc0c1\ud488 \ub7ad\ud0b9 \uc2dc\uc2a4\ud15c \uad6c\ud604", "status": "pending", "id": "4"}, {"content": "STEP 14: Redis \uc6d0\uc790\uc801 \uc5f0\uc0b0 \uae30\ubc18 \uc120\ucc29\uc21c \ucfe0\ud3f0 \ubc1c\uae09 \uac1c\uc120", "status": "pending", "id": "5"}, {"content": "STEP 14: Lua Script\ub85c \uc120\ucc29\uc21c \ubc1c\uae09 \uc6d0\uc790\uc131 \ubcf4\uc7a5", "status": "pending", "id": "6"}, {"content": "\ud1b5\ud569 \ud14c\uc2a4\ud2b8 \ubc0f \uc131\ub2a5 \uac80\uc99d", "status": "pending", "id": "7"}, {"content": "\ubb38\uc11c\ud654 \ubc0f \ubcf4\uace0\uc11c \uc791\uc131", "status": "pending", "id": "8"}]