# Event-Driven Architecture 완벽 가이드

## 📋 목차
1. [Event-Driven 개념과 원리](#1-event-driven-개념과-원리)
2. [왜 Event-Driven을 사용하는가?](#2-왜-event-driven을-사용하는가)
3. [Event vs 직접 호출](#3-event-vs-직접-호출)
4. [Spring Events 아키텍처](#4-spring-events-아키텍처)
5. [주문 완료 이벤트 기반 업데이트 구현](#5-주문-완료-이벤트-기반-업데이트-구현)
6. [비동기 처리와 성능 최적화](#6-비동기-처리와-성능-최적화)
7. [실제 코드 예시](#7-실제-코드-예시)

---

## 1. Event-Driven 개념과 원리

### 🎯 Event-Driven Architecture란?

**Event-Driven Architecture(EDA)**는 **이벤트의 생성, 감지, 소비 및 반응을 중심으로 설계된 아키텍처 패턴**입니다.

### 🏪 실생활 비유: 백화점의 업무 프로세스

고객이 백화점에서 상품을 구매하는 과정을 생각해보세요:

```
고객 결제 완료 → [결제 완료 이벤트 발생]
                     ↓
    ┌─────────────────┼─────────────────┐
    ↓                 ↓                 ↓
재고 팀         마케팅 팀         회계 팀
(재고 차감)    (포인트 적립)    (매출 기록)
    ↓                 ↓                 ↓
배송 팀         고객 서비스팀     분석 팀
(배송 준비)    (감사 메일 발송)  (판매 통계)
```

**기존 방식 (직접 호출):**
```java
public void processPayment(Order order) {
    paymentService.process(order);           // 결제 처리
    inventoryService.decreaseStock(order);   // 재고 차감
    pointService.addPoints(order);           // 포인트 적립
    emailService.sendConfirmation(order);    // 확인 메일
    analyticsService.recordSale(order);      // 판매 통계
    // ... 더 많은 후속 처리들
}
```

**Event-Driven 방식:**
```java
public void processPayment(Order order) {
    paymentService.process(order);           // 결제 처리
    
    // 이벤트 발행 - 나머지는 각 팀이 알아서 처리
    eventPublisher.publishEvent(new PaymentCompletedEvent(order));
}
```

### 🔧 핵심 개념

```
Event-Driven 구조:
┌─────────────────────────────────────┐
│ Event Producer (이벤트 생성자)       │ 
│ - OrderService                      │
│ - PaymentService                    │
└─────────────────┬───────────────────┘
                  │ publishes
                  ↓
┌─────────────────────────────────────┐
│ Event (이벤트)                      │
│ - OrderCompletedEvent              │
│ - PaymentCompletedEvent            │
└─────────────────┬───────────────────┘
                  │ delivers
                  ↓
┌─────────────────────────────────────┐
│ Event Consumer (이벤트 소비자)       │
│ - InventoryService                 │
│ - RankingService                   │
│ - NotificationService              │
└─────────────────────────────────────┘
```

**핵심 특징:**
- **느슨한 결합**: Producer와 Consumer가 서로를 직접 알지 못함
- **확장성**: 새로운 Consumer 추가가 쉬움
- **비동기 처리**: 이벤트 처리가 독립적으로 실행
- **장애 격리**: 한 Consumer 실패가 다른 처리에 영향 없음

---

## 2. 왜 Event-Driven을 사용하는가?

### 🚫 기존 방식의 문제점

#### 문제 상황: 주문 완료 시 후속 처리

**직접 호출 방식:**
```java
@Service
@Transactional
public class OrderService {
    
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final PointService pointService;
    private final EmailService emailService;
    private final RankingService rankingService;
    private final AnalyticsService analyticsService;
    private final DeliveryService deliveryService;
    // ... 더 많은 의존성들
    
    public void completeOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        
        // 결제 처리
        Payment payment = paymentService.processPayment(order);
        
        // 재고 차감
        inventoryService.decreaseStock(order.getItems());
        
        // 포인트 적립
        pointService.addPoints(order.getUserId(), order.getAmount());
        
        // 확인 메일 발송
        emailService.sendOrderConfirmation(order);
        
        // 상품 랭킹 업데이트
        rankingService.updateProductRanking(order.getItems());
        
        // 판매 통계 업데이트
        analyticsService.recordSale(order);
        
        // 배송 시작
        deliveryService.startDelivery(order);
    }
}
```

**문제점:**
1. **강한 결합**: OrderService가 모든 후속 서비스를 직접 의존
2. **단일 장애점**: 하나의 서비스 실패 시 전체 트랜잭션 롤백
3. **성능 저하**: 모든 후속 처리가 동기적으로 실행되어 응답 지연
4. **확장성 부족**: 새로운 후속 처리 추가 시 OrderService 수정 필요
5. **테스트 복잡성**: 모든 의존 서비스를 모킹해야 함

### ✅ Event-Driven 방식의 해결책

```java
@Service
@Transactional
public class OrderService {
    
    private final PaymentService paymentService;
    private final ApplicationEventPublisher eventPublisher;
    
    public void completeOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        
        // 핵심 비즈니스 로직만 실행
        Payment payment = paymentService.processPayment(order);
        order.complete();
        
        // 이벤트 발행 - 나머지는 각 서비스가 알아서 처리
        eventPublisher.publishEvent(new OrderCompletedEvent(order));
    }
}

// 각 서비스는 독립적으로 이벤트 처리
@EventListener
@Async
public void handleOrderCompleted(OrderCompletedEvent event) {
    inventoryService.decreaseStock(event.getOrderItems());
}

@EventListener
@Async  
public void updateRanking(OrderCompletedEvent event) {
    rankingService.updateProductRanking(event.getOrderItems());
}
```

**장점:**
1. **느슨한 결합**: OrderService는 이벤트만 발행, 구체적인 후속 처리 모름
2. **장애 격리**: 개별 이벤트 처리 실패가 전체에 영향 없음
3. **성능 향상**: 비동기 처리로 응답 시간 단축
4. **확장성**: 새로운 이벤트 리스너 추가가 쉬움
5. **테스트 용이성**: 이벤트 발행 여부만 검증하면 됨

### 📊 성능 비교

| 구분 | 직접 호출 방식 | Event-Driven 방식 |
|------|----------------|-------------------|
| 응답 시간 | 2~5초 | 100~300ms |
| 결합도 | 강함 | 느슨함 |
| 확장성 | 낮음 | 높음 |
| 장애 전파 | 전체 실패 | 격리됨 |
| 트랜잭션 범위 | 전체 | 핵심 로직만 |

---

## 3. Event vs 직접 호출

### 🔄 상황별 비교

#### 3.1 핵심 비즈니스 로직 (직접 호출 권장)
```java
// 주문 생성 - 필수적이고 즉시 처리되어야 함
public Order createOrder(CreateOrderRequest request) {
    // 재고 확인 - 실패하면 주문 자체가 불가능
    validateStock(request.getItems());
    
    // 주문 생성 - 핵심 로직
    Order order = new Order(request);
    return orderRepository.save(order);
}
```

#### 3.2 부가적인 비즈니스 로직 (이벤트 권장)
```java
// 주문 완료 후 후속 처리 - 주문 성공과 무관한 부가 처리
@EventListener
@Async
public void handleOrderCompleted(OrderCompletedEvent event) {
    // 고객에게 이메일 발송 - 실패해도 주문은 유효
    emailService.sendOrderConfirmation(event.getOrder());
    
    // 상품 랭킹 업데이트 - 즉시 반영되지 않아도 됨
    rankingService.updateProductRanking(event.getOrderItems());
}
```

### 📋 선택 기준

| 특성 | 직접 호출 | 이벤트 |
|------|-----------|--------|
| **비즈니스 중요도** | 필수 | 부가적 |
| **실패 영향도** | 전체 실패 | 부분 실패 허용 |
| **응답 속도** | 즉시 필요 | 지연 가능 |
| **데이터 일관성** | 강한 일관성 | 최종 일관성 |
| **결합도** | 강결합 허용 | 느슨한 결합 선호 |

### 🎯 실제 적용 예시

```java
@Service
@Transactional
public class OrderService {
    
    public OrderResult processOrder(CreateOrderRequest request) {
        // 1. 핵심 로직 - 직접 호출 (동기)
        validateOrderRequest(request);                    // 필수 검증
        Product product = productService.getProduct();    // 상품 정보 조회
        validateStock(product, request.getQuantity());    // 재고 검증
        
        Order order = createOrder(request);               // 주문 생성
        Payment payment = processPayment(order);          // 결제 처리
        
        // 2. 부가 로직 - 이벤트 발행 (비동기)
        eventPublisher.publishEvent(new OrderCompletedEvent(order));
        
        return OrderResult.success(order, payment);
    }
}
```

---

## 4. Spring Events 아키텍처

### 🏗 Spring Events 구조

```
┌─────────────────────────────────────────────────────────┐
│                Application Context                       │
├─────────────────────────────────────────────────────────┤
│  EventPublisher  →  ApplicationEventMulticaster  →  Listeners  │
│       ↓                       ↓                       ↓  │
│  OrderService    →    SimpleApplicationEventMulticaster  │
│  PaymentService  →         ThreadPoolExecutor          │
│  CouponService   →           @EventListener            │
└─────────────────────────────────────────────────────────┘
```

### 📦 핵심 구성요소

#### 4.1 이벤트 클래스
```java
// 기본 이벤트
public class OrderCompletedEvent {
    private final Order order;
    private final LocalDateTime timestamp;
    
    public OrderCompletedEvent(Order order) {
        this.order = order;
        this.timestamp = LocalDateTime.now();
    }
    
    // getters...
}

// ApplicationEvent 상속 (Spring 4.2 이전 스타일)
public class PaymentCompletedEvent extends ApplicationEvent {
    
    private final Payment payment;
    
    public PaymentCompletedEvent(Object source, Payment payment) {
        super(source);
        this.payment = payment;
    }
}
```

#### 4.2 이벤트 발행
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public void completeOrder(Long orderId) {
        Order order = findOrder(orderId);
        order.complete();
        
        // 이벤트 발행
        eventPublisher.publishEvent(new OrderCompletedEvent(order));
        
        // 또는 빌더 패턴 사용
        eventPublisher.publishEvent(
            OrderCompletedEvent.builder()
                .orderId(orderId)
                .userId(order.getUserId())
                .items(order.getItems())
                .completedAt(LocalDateTime.now())
                .build()
        );
    }
}
```

#### 4.3 이벤트 리스너
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventHandler {
    
    private final RankingService rankingService;
    private final NotificationService notificationService;
    
    // 기본 동기 처리
    @EventListener
    public void handleOrderCompleted(OrderCompletedEvent event) {
        log.info("주문 완료 이벤트 수신: orderId={}", event.getOrderId());
        // 동기 처리 로직
    }
    
    // 비동기 처리
    @EventListener
    @Async
    public void updateProductRanking(OrderCompletedEvent event) {
        for (OrderItem item : event.getOrderItems()) {
            rankingService.updateProductRanking(
                item.getProductId(), 
                item.getQuantity()
            );
        }
    }
    
    // 조건부 처리
    @EventListener(condition = "#event.orderAmount > 100000")
    public void handleLargeOrder(OrderCompletedEvent event) {
        // 10만원 이상 주문에 대해서만 처리
        notificationService.sendVipNotification(event);
    }
    
    // 트랜잭션 이벤트 (커밋 후 실행)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendOrderConfirmation(OrderCompletedEvent event) {
        emailService.sendOrderConfirmation(event.getOrder());
    }
}
```

### ⚙️ 비동기 설정

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "eventTaskExecutor")
    public TaskExecutor eventTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-");
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

---

## 5. 주문 완료 이벤트 기반 업데이트 구현

### 🎯 시스템 설계

#### 5.1 이벤트 설계
```java
@Getter
@Builder
@AllArgsConstructor
public class OrderCompletedEvent {
    
    private final Long orderId;
    private final Long userId;
    private final BigDecimal totalAmount;
    private final List<OrderItem> orderItems;
    private final LocalDateTime completedAt;
    
    @Getter
    @Builder
    @AllArgsConstructor
    public static class OrderItem {
        private final Long productId;
        private final String productName;
        private final Integer quantity;
        private final BigDecimal price;
    }
    
    // 팩토리 메서드
    public static OrderCompletedEvent from(Order order) {
        List<OrderItem> items = order.getOrderItems().stream()
            .map(item -> OrderItem.builder()
                .productId(item.getProductId())
                .productName(item.getProductName())
                .quantity(item.getQuantity())
                .price(item.getPrice())
                .build())
            .collect(Collectors.toList());
        
        return OrderCompletedEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .totalAmount(order.getTotalAmount())
            .orderItems(items)
            .completedAt(order.getCompletedAt())
            .build();
    }
}
```

#### 5.2 이벤트 발행 로직
```java
@Service
@RequiredArgsConstructional
@Transactional
public class OrderService {
    
    private final ApplicationEventPublisher eventPublisher;
    private final OrderRepository orderRepository;
    
    public Payment completeOrder(Long orderId, PaymentRequest request) {
        // 1. 핵심 비즈니스 로직 (동기)
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        Payment payment = processPayment(order, request);
        order.complete();
        orderRepository.save(order);
        
        // 2. 이벤트 발행 (비동기 후속 처리 트리거)
        OrderCompletedEvent event = OrderCompletedEvent.from(order);
        eventPublisher.publishEvent(event);
        
        log.info("주문 완료 처리: orderId={}, amount={}", orderId, order.getTotalAmount());
        
        return payment;
    }
}
```

### 🔄 이벤트 처리 구현

#### 5.3 상품 랭킹 업데이트
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class ProductRankingEventHandler {
    
    private final ProductRankingService rankingService;
    
    @EventListener
    @Async("eventTaskExecutor")
    @Retryable(value = {Exception.class}, maxAttempts = 3)
    public void updateProductRanking(OrderCompletedEvent event) {
        log.debug("상품 랭킹 업데이트 시작: orderId={}", event.getOrderId());
        
        for (OrderCompletedEvent.OrderItem item : event.getOrderItems()) {
            try {
                rankingService.updateProductRanking(
                    item.getProductId(), 
                    item.getQuantity()
                );
                
                log.debug("상품 랭킹 업데이트 완료: productId={}, quantity={}", 
                         item.getProductId(), item.getQuantity());
                         
            } catch (Exception e) {
                log.error("상품 랭킹 업데이트 실패: productId={}", 
                         item.getProductId(), e);
                throw e; // Retry 트리거
            }
        }
        
        log.info("주문 완료 랭킹 업데이트 완료: orderId={}, itemCount={}", 
                event.getOrderId(), event.getOrderItems().size());
    }
    
    @Recover
    public void recoverRankingUpdate(Exception ex, OrderCompletedEvent event) {
        log.error("상품 랭킹 업데이트 최종 실패: orderId={}", event.getOrderId(), ex);
        // 실패한 업데이트를 별도 큐에 저장하여 나중에 재처리
        deadLetterQueueService.sendRankingUpdateFailure(event, ex);
    }
}
```

#### 5.4 재고 차감
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class InventoryEventHandler {
    
    private final InventoryService inventoryService;
    
    @EventListener
    @Async("eventTaskExecutor")
    @Transactional
    public void decreaseInventory(OrderCompletedEvent event) {
        log.debug("재고 차감 시작: orderId={}", event.getOrderId());
        
        List<InventoryDeductionRequest> deductions = event.getOrderItems()
            .stream()
            .map(item -> InventoryDeductionRequest.builder()
                .productId(item.getProductId())
                .quantity(item.getQuantity())
                .orderId(event.getOrderId())
                .build())
            .collect(Collectors.toList());
        
        try {
            inventoryService.bulkDecrease(deductions);
            log.info("재고 차감 완료: orderId={}, itemCount={}", 
                    event.getOrderId(), deductions.size());
                    
        } catch (InsufficientStockException e) {
            log.error("재고 부족으로 차감 실패: orderId={}", event.getOrderId(), e);
            // 주문 취소 이벤트 발행
            eventPublisher.publishEvent(new OrderCancellationEvent(event.getOrderId(), 
                "재고 부족으로 인한 주문 취소"));
        }
    }
}
```

#### 5.5 포인트 적립
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class PointEventHandler {
    
    private final PointService pointService;
    
    @EventListener
    @Async("eventTaskExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void addPoints(OrderCompletedEvent event) {
        try {
            // 주문 금액의 1% 포인트 적립
            BigDecimal pointAmount = event.getTotalAmount()
                .multiply(BigDecimal.valueOf(0.01))
                .setScale(0, RoundingMode.DOWN);
            
            if (pointAmount.compareTo(BigDecimal.ZERO) > 0) {
                pointService.addPoints(
                    event.getUserId(), 
                    pointAmount.longValue(),
                    "주문 완료 적립 (주문번호: " + event.getOrderId() + ")"
                );
                
                log.info("포인트 적립 완료: userId={}, orderId={}, points={}", 
                        event.getUserId(), event.getOrderId(), pointAmount);
            }
            
        } catch (Exception e) {
            log.error("포인트 적립 실패: userId={}, orderId={}", 
                     event.getUserId(), event.getOrderId(), e);
        }
    }
}
```

#### 5.6 알림 발송
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class NotificationEventHandler {
    
    private final EmailService emailService;
    private final SmsService smsService;
    private final PushNotificationService pushService;
    
    @EventListener
    @Async("eventTaskExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendOrderConfirmation(OrderCompletedEvent event) {
        try {
            // 이메일 발송
            OrderConfirmationEmail email = OrderConfirmationEmail.builder()
                .orderId(event.getOrderId())
                .userId(event.getUserId())
                .orderItems(event.getOrderItems())
                .totalAmount(event.getTotalAmount())
                .build();
            
            emailService.sendOrderConfirmation(email);
            
            // 푸시 알림 발송
            PushNotification push = PushNotification.builder()
                .userId(event.getUserId())
                .title("주문이 완료되었습니다")
                .message("주문번호 " + event.getOrderId() + "의 결제가 완료되었습니다.")
                .build();
            
            pushService.send(push);
            
            log.info("주문 완료 알림 발송 완료: orderId={}, userId={}", 
                    event.getOrderId(), event.getUserId());
            
        } catch (Exception e) {
            log.error("주문 완료 알림 발송 실패: orderId={}", event.getOrderId(), e);
        }
    }
    
    @EventListener(condition = "#event.totalAmount.compareTo(new java.math.BigDecimal('500000')) >= 0")
    @Async("eventTaskExecutor")
    public void sendVipOrderNotification(OrderCompletedEvent event) {
        // 50만원 이상 주문에 대한 VIP 고객 관리팀 알림
        try {
            VipOrderNotification notification = VipOrderNotification.builder()
                .orderId(event.getOrderId())
                .userId(event.getUserId())
                .orderAmount(event.getTotalAmount())
                .build();
            
            slackService.sendVipOrderAlert(notification);
            
        } catch (Exception e) {
            log.error("VIP 주문 알림 실패: orderId={}", event.getOrderId(), e);
        }
    }
}
```

---

## 6. 비동기 처리와 성능 최적화

### ⚡ 성능 최적화 전략

#### 6.1 스레드 풀 설정
```java
@Configuration
@EnableAsync
public class EventAsyncConfig {
    
    @Bean(name = "eventTaskExecutor")
    public TaskExecutor eventTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        
        // 시스템 리소스에 따른 동적 설정
        int cores = Runtime.getRuntime().availableProcessors();
        executor.setCorePoolSize(cores);
        executor.setMaxPoolSize(cores * 2);
        executor.setQueueCapacity(1000);
        
        // 스레드 네이밍
        executor.setThreadNamePrefix("event-handler-");
        
        // 거절 정책: 큐가 가득 찬 경우 호출 스레드에서 직접 실행
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        
        // 스레드 Keep-Alive 시간
        executor.setKeepAliveSeconds(60);
        executor.setAllowCoreThreadTimeOut(true);
        
        executor.initialize();
        return executor;
    }
    
    @Bean(name = "rankingUpdateExecutor")
    public TaskExecutor rankingUpdateExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        
        // 랭킹 업데이트는 빠른 처리가 중요하므로 별도 풀
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(4);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("ranking-update-");
        
        executor.initialize();
        return executor;
    }
}
```

#### 6.2 배치 처리 최적화
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class BatchedRankingEventHandler {
    
    private final ProductRankingService rankingService;
    private final Map<Long, AtomicInteger> pendingUpdates = new ConcurrentHashMap<>();
    
    @EventListener
    @Async("rankingUpdateExecutor")
    public void updateProductRanking(OrderCompletedEvent event) {
        // 1. 개별 이벤트를 배치로 모으기
        for (OrderCompletedEvent.OrderItem item : event.getOrderItems()) {
            pendingUpdates.computeIfAbsent(item.getProductId(), k -> new AtomicInteger(0))
                         .addAndGet(item.getQuantity());
        }
    }
    
    @Scheduled(fixedDelay = 5000) // 5초마다 배치 처리
    public void processBatchedUpdates() {
        if (pendingUpdates.isEmpty()) {
            return;
        }
        
        // 2. 모인 업데이트를 배치로 처리
        Map<Long, AtomicInteger> currentBatch = new HashMap<>(pendingUpdates);
        pendingUpdates.clear();
        
        try {
            List<RankingUpdate> updates = currentBatch.entrySet().stream()
                .map(entry -> new RankingUpdate(entry.getKey(), entry.getValue().get()))
                .collect(Collectors.toList());
            
            rankingService.batchUpdateRanking(updates);
            
            log.info("배치 랭킹 업데이트 완료: {}건", updates.size());
            
        } catch (Exception e) {
            log.error("배치 랭킹 업데이트 실패", e);
            // 실패한 업데이트를 다시 큐에 추가
            currentBatch.forEach(pendingUpdates::put);
        }
    }
}
```

#### 6.3 에러 처리 및 복구
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class ResilientEventHandler {
    
    @EventListener
    @Async("eventTaskExecutor")
    @Retryable(
        value = {RedisConnectionException.class, DataAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public void updateProductRanking(OrderCompletedEvent event) {
        try {
            rankingService.updateProductRanking(event);
            
        } catch (Exception e) {
            log.error("랭킹 업데이트 실패: orderId={}, attempt={}", 
                     event.getOrderId(), getCurrentRetryCount(), e);
            throw e;
        }
    }
    
    @Recover
    public void recoverRankingUpdate(Exception ex, OrderCompletedEvent event) {
        log.error("랭킹 업데이트 최종 실패: orderId={}", event.getOrderId(), ex);
        
        // Dead Letter Queue에 저장
        deadLetterQueueService.saveFailedEvent(event, ex);
        
        // 모니터링 알림
        alertService.sendEventProcessingFailureAlert(event, ex);
    }
    
    // Circuit Breaker 패턴 적용
    @CircuitBreaker(name = "ranking-service", fallbackMethod = "fallbackRankingUpdate")
    public void updateProductRankingWithCircuitBreaker(OrderCompletedEvent event) {
        rankingService.updateProductRanking(event);
    }
    
    public void fallbackRankingUpdate(OrderCompletedEvent event, Exception ex) {
        log.warn("랭킹 서비스 Circuit Breaker 작동: orderId={}", event.getOrderId());
        // 폴백 로직: 이벤트를 별도 큐에 저장하여 나중에 처리
        fallbackQueueService.enqueueRankingUpdate(event);
    }
}
```

### 📊 모니터링 및 메트릭

```java
@Component
@RequiredArgsConstructor
public class EventMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final Counter eventPublishedCounter;
    private final Counter eventProcessedCounter;
    private final Timer eventProcessingTimer;
    
    @PostConstruct
    public void initMetrics() {
        eventPublishedCounter = Counter.builder("events.published")
            .tag("type", "order_completed")
            .register(meterRegistry);
            
        eventProcessedCounter = Counter.builder("events.processed")
            .tag("type", "order_completed")
            .register(meterRegistry);
            
        eventProcessingTimer = Timer.builder("events.processing.duration")
            .register(meterRegistry);
    }
    
    @EventListener
    public void recordEventPublished(OrderCompletedEvent event) {
        eventPublishedCounter.increment();
    }
    
    @EventListener
    public void recordEventProcessed(OrderCompletedEvent event) {
        Timer.Sample sample = Timer.start(meterRegistry);
        eventProcessedCounter.increment();
        sample.stop(eventProcessingTimer);
    }
}
```

---

## 7. 실제 코드 예시

### 🎯 완전한 주문 완료 이벤트 시스템

#### 7.1 이벤트 정의
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
    private final String eventVersion;
    
    @Getter
    @Builder
    @ToString
    public static class OrderItem {
        private final Long productId;
        private final String productName;
        private final Integer quantity;
        private final BigDecimal price;
        private final String category;
    }
    
    // 팩토리 메서드
    public static OrderCompletedEvent from(Order order) {
        return OrderCompletedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .orderId(order.getId())
            .userId(order.getUserId())
            .totalAmount(order.getTotalAmount())
            .orderItems(convertOrderItems(order.getOrderItems()))
            .completedAt(LocalDateTime.now())
            .eventVersion("1.0")
            .build();
    }
    
    private static List<OrderItem> convertOrderItems(List<kr.hhplus.be.server.domain.entity.OrderItem> items) {
        return items.stream()
            .map(item -> OrderItem.builder()
                .productId(item.getProductId())
                .productName(item.getProductName())
                .quantity(item.getQuantity())
                .price(item.getPrice())
                .category(item.getProduct().getCategory().getName())
                .build())
            .collect(Collectors.toList());
    }
}
```

#### 7.2 이벤트 발행 서비스
```java
@Service
@RequiredArgsConstructor
@Transactional
@Slf4j
public class OrderCompletionService {
    
    private final ApplicationEventPublisher eventPublisher;
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    
    public Payment completeOrder(Long orderId, PaymentRequest request) {
        log.info("주문 완료 처리 시작: orderId={}", orderId);
        
        try {
            // 1. 주문 조회 및 검증
            Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));
            
            validateOrderForCompletion(order);
            
            // 2. 결제 처리 (핵심 비즈니스 로직)
            Payment payment = paymentService.processPayment(order, request);
            
            // 3. 주문 상태 변경
            order.complete();
            orderRepository.save(order);
            
            // 4. 이벤트 발행 (후속 처리 트리거)
            OrderCompletedEvent event = OrderCompletedEvent.from(order);
            eventPublisher.publishEvent(event);
            
            log.info("주문 완료 처리 성공: orderId={}, paymentId={}, eventId={}", 
                    orderId, payment.getId(), event.getEventId());
            
            return payment;
            
        } catch (Exception e) {
            log.error("주문 완료 처리 실패: orderId={}", orderId, e);
            throw e;
        }
    }
    
    private void validateOrderForCompletion(Order order) {
        if (order.isCompleted()) {
            throw new OrderAlreadyCompletedException(order.getId());
        }
        
        if (order.isCancelled()) {
            throw new OrderCancelledException(order.getId());
        }
    }
}
```

#### 7.3 통합 이벤트 핸들러
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderCompletedEventHandler {
    
    private final ProductRankingService rankingService;
    private final InventoryService inventoryService;
    private final PointService pointService;
    private final NotificationService notificationService;
    private final AnalyticsService analyticsService;
    
    /**
     * 상품 랭킹 업데이트
     */
    @EventListener
    @Async("rankingUpdateExecutor")
    @Order(1) // 우선순위 지정
    public void updateProductRanking(OrderCompletedEvent event) {
        log.debug("상품 랭킹 업데이트 시작: eventId={}", event.getEventId());
        
        try {
            for (OrderCompletedEvent.OrderItem item : event.getOrderItems()) {
                rankingService.updateProductRanking(
                    item.getProductId(), 
                    item.getQuantity()
                );
            }
            
            log.info("상품 랭킹 업데이트 완료: eventId={}, itemCount={}", 
                    event.getEventId(), event.getOrderItems().size());
            
        } catch (Exception e) {
            log.error("상품 랭킹 업데이트 실패: eventId={}", event.getEventId(), e);
            throw e; // 재시도를 위해 예외 전파
        }
    }
    
    /**
     * 재고 차감
     */
    @EventListener
    @Async("eventTaskExecutor")
    @Order(2)
    @Transactional
    public void updateInventory(OrderCompletedEvent event) {
        log.debug("재고 업데이트 시작: eventId={}", event.getEventId());
        
        try {
            List<InventoryUpdateRequest> updates = event.getOrderItems().stream()
                .map(item -> InventoryUpdateRequest.builder()
                    .productId(item.getProductId())
                    .quantityChange(-item.getQuantity()) // 차감
                    .reason("주문 완료 (주문번호: " + event.getOrderId() + ")")
                    .build())
                .collect(Collectors.toList());
            
            inventoryService.bulkUpdate(updates);
            
            log.info("재고 업데이트 완료: eventId={}, updateCount={}", 
                    event.getEventId(), updates.size());
            
        } catch (Exception e) {
            log.error("재고 업데이트 실패: eventId={}", event.getEventId(), e);
        }
    }
    
    /**
     * 포인트 적립
     */
    @EventListener
    @Async("eventTaskExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void addRewardPoints(OrderCompletedEvent event) {
        log.debug("포인트 적립 시작: eventId={}", event.getEventId());
        
        try {
            // 주문 금액의 1% 적립
            BigDecimal pointAmount = event.getTotalAmount()
                .multiply(BigDecimal.valueOf(0.01))
                .setScale(0, RoundingMode.DOWN);
            
            if (pointAmount.compareTo(BigDecimal.ZERO) > 0) {
                pointService.addPoints(
                    event.getUserId(),
                    pointAmount.longValue(),
                    PointEarnType.ORDER_COMPLETION,
                    "주문 완료 적립 (주문번호: " + event.getOrderId() + ")"
                );
                
                log.info("포인트 적립 완료: eventId={}, userId={}, points={}", 
                        event.getEventId(), event.getUserId(), pointAmount);
            }
            
        } catch (Exception e) {
            log.error("포인트 적립 실패: eventId={}", event.getEventId(), e);
        }
    }
    
    /**
     * 주문 완료 알림 발송
     */
    @EventListener
    @Async("eventTaskExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendNotifications(OrderCompletedEvent event) {
        log.debug("알림 발송 시작: eventId={}", event.getEventId());
        
        try {
            // 이메일 알림
            OrderCompletionNotification notification = OrderCompletionNotification.builder()
                .orderId(event.getOrderId())
                .userId(event.getUserId())
                .orderItems(event.getOrderItems())
                .totalAmount(event.getTotalAmount())
                .completedAt(event.getCompletedAt())
                .build();
            
            notificationService.sendOrderCompletionNotification(notification);
            
            log.info("알림 발송 완료: eventId={}, userId={}", 
                    event.getEventId(), event.getUserId());
            
        } catch (Exception e) {
            log.error("알림 발송 실패: eventId={}", event.getEventId(), e);
        }
    }
    
    /**
     * 판매 통계 업데이트
     */
    @EventListener
    @Async("eventTaskExecutor")
    public void updateSalesAnalytics(OrderCompletedEvent event) {
        log.debug("판매 통계 업데이트 시작: eventId={}", event.getEventId());
        
        try {
            SalesDataPoint salesData = SalesDataPoint.builder()
                .orderId(event.getOrderId())
                .userId(event.getUserId())
                .totalAmount(event.getTotalAmount())
                .itemCount(event.getOrderItems().size())
                .completedAt(event.getCompletedAt())
                .productCategories(extractCategories(event.getOrderItems()))
                .build();
            
            analyticsService.recordSalesData(salesData);
            
            log.info("판매 통계 업데이트 완료: eventId={}", event.getEventId());
            
        } catch (Exception e) {
            log.error("판매 통계 업데이트 실패: eventId={}", event.getEventId(), e);
        }
    }
    
    private Set<String> extractCategories(List<OrderCompletedEvent.OrderItem> items) {
        return items.stream()
            .map(OrderCompletedEvent.OrderItem::getCategory)
            .collect(Collectors.toSet());
    }
}
```

#### 7.4 이벤트 모니터링
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventMonitor {
    
    private final MeterRegistry meterRegistry;
    
    @EventListener
    public void monitorEventPublishing(OrderCompletedEvent event) {
        // 이벤트 발행 메트릭
        Counter.builder("order.events.published")
            .tag("type", "order_completed")
            .register(meterRegistry)
            .increment();
        
        // 주문 금액별 분포
        Timer.builder("order.amount.distribution")
            .tag("amount_range", getAmountRange(event.getTotalAmount()))
            .register(meterRegistry)
            .record(Duration.ZERO);
        
        log.info("주문 완료 이벤트 모니터링: eventId={}, orderId={}, amount={}", 
                event.getEventId(), event.getOrderId(), event.getTotalAmount());
    }
    
    private String getAmountRange(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.valueOf(50000)) < 0) {
            return "under_50k";
        } else if (amount.compareTo(BigDecimal.valueOf(100000)) < 0) {
            return "50k_to_100k";
        } else if (amount.compareTo(BigDecimal.valueOf(500000)) < 0) {
            return "100k_to_500k";
        } else {
            return "over_500k";
        }
    }
}
```

### 🧪 테스트 코드
```java
@SpringBootTest
@TestMethodOrder(OrderAnnotation.class)
class OrderEventIntegrationTest {
    
    @MockBean
    private ProductRankingService rankingService;
    
    @MockBean
    private NotificationService notificationService;
    
    @Autowired
    private OrderCompletionService orderCompletionService;
    
    @Autowired
    private TestEventPublisher testEventPublisher;
    
    @Test
    @Order(1)
    void 주문_완료_이벤트_발행_테스트() {
        // Given
        Order order = createTestOrder();
        PaymentRequest request = createPaymentRequest();
        
        // When
        Payment payment = orderCompletionService.completeOrder(order.getId(), request);
        
        // Then
        assertThat(payment).isNotNull();
        
        // 이벤트 발행 검증
        await().atMost(Duration.ofSeconds(5))
            .untilAsserted(() -> {
                verify(rankingService, times(1)).updateProductRanking(any(), any());
                verify(notificationService, times(1)).sendOrderCompletionNotification(any());
            });
    }
    
    @Test
    @Order(2)
    void 이벤트_처리_실패_시_재시도_테스트() {
        // Given
        Order order = createTestOrder();
        doThrow(new RedisConnectionException("Connection failed"))
            .doNothing()
            .when(rankingService).updateProductRanking(any(), any());
        
        // When
        OrderCompletedEvent event = OrderCompletedEvent.from(order);
        testEventPublisher.publishEvent(event);
        
        // Then - 재시도 후 성공 검증
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                verify(rankingService, times(2)).updateProductRanking(any(), any());
            });
    }
}
```

---

## 📚 정리 및 핵심 포인트

### ✅ Event-Driven을 사용해야 하는 이유
1. **느슨한 결합**: 서비스 간 의존성 최소화
2. **확장성**: 새로운 이벤트 처리 로직 추가 용이
3. **성능**: 비동기 처리로 응답 시간 단축
4. **장애 격리**: 개별 처리 실패가 전체에 영향 없음

### 🎯 적용 시 주의사항
1. **이벤트 설계**: 명확하고 불변적인 이벤트 구조
2. **순서 보장**: 필요한 경우 이벤트 순서 처리
3. **에러 처리**: 재시도, Circuit Breaker 등 장애 대응
4. **모니터링**: 이벤트 발행/처리 상태 추적

### 🚀 이번 과제 적용 방안
1. **주문 완료 이벤트**: 랭킹 업데이트, 알림 등 후속 처리
2. **비동기 처리**: 응답 성능 향상
3. **장애 대응**: 재시도 및 폴백 메커니즘
4. **확장성**: 새로운 이벤트 처리 로직 추가 대비

이제 Event-Driven Architecture를 활용하여 확장 가능하고 유연한 시스템을 구축할 수 있습니다!

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"content": "\ud604\uc7ac \uad6c\ud604 \uc0c1\ud0dc \uc7ac\ubd84\uc11d (AS-IS)", "status": "completed", "id": "1"}, {"content": "STEP 13: Redis Sorted Set \uae30\ubc18 \uc0c1\ud488 \ub7ad\ud0b9 \uc2dc\uc2a4\ud15c \uc124\uacc4", "status": "completed", "id": "2"}, {"content": "Redis \ud575\uc2ec \uae30\uc220 \uc2a4\ud0dd \ud559\uc2b5\uc790\ub8cc \uc791\uc131", "status": "completed", "id": "3"}, {"content": "STEP 13: \uc0c1\ud488 \ub7ad\ud0b9 \uc2dc\uc2a4\ud15c \uad6c\ud604", "status": "pending", "id": "4"}, {"content": "STEP 14: Redis \uc6d0\uc790\uc801 \uc5f0\uc0b0 \uae30\ubc18 \uc120\ucc29\uc21c \ucfe0\ud3f0 \ubc1c\uae09 \uac1c\uc120", "status": "pending", "id": "5"}, {"content": "STEP 14: Lua Script\ub85c \uc120\ucc29\uc21c \ubc1c\uae09 \uc6d0\uc790\uc131 \ubcf4\uc7a5", "status": "pending", "id": "6"}, {"content": "\ud1b5\ud569 \ud14c\uc2a4\ud2b8 \ubc0f \uc131\ub2a5 \uac80\uc99d", "status": "pending", "id": "7"}, {"content": "\ubb38\uc11c\ud654 \ubc0f \ubcf4\uace0\uc11c \uc791\uc131", "status": "pending", "id": "8"}]