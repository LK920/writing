# Redis Stream 이벤트 스트리밍

## 📋 목차
1. [Redis Stream 개념과 구조](#1-redis-stream-개념과-구조)
2. [기존 메시징 시스템과의 차이점](#2-기존-메시징-시스템과의-차이점)
3. [Stream 핵심 명령어와 패턴](#3-stream-핵심-명령어와-패턴)
4. [이벤트 스트리밍 시스템 구현](#4-이벤트-스트리밍-시스템-구현)
5. [Consumer Group과 분산 처리](#5-consumer-group과-분산-처리)
6. [실전 활용 사례](#6-실전-활용-사례)

---

## 1. Redis Stream 개념과 구조

### 🎯 Redis Stream이란?

**Redis Stream**은 **시간 순서가 보장되는 불변 로그 기반의 메시징 시스템**입니다.

### 🏞 실생활 비유: 강물의 흐름

강물이 흐르는 모습을 생각해보세요:

```
상류 (Producer) → 강물 (Stream) → 하류 (Consumer)
    ↓               ↓              ↓
 이벤트 생성    시간순 저장      순차 소비

🏔️ 상류: 주문 완료 이벤트 발생
   ↓
💧 강물: order-events Stream
   ├─ 1692345600-0: {orderId: 101, userId: 123}
   ├─ 1692345601-0: {orderId: 102, userId: 456}  
   ├─ 1692345602-0: {orderId: 103, userId: 789}
   └─ 1692345603-0: {orderId: 104, userId: 321}
   ↓
🏘️ 하류: 랭킹 서비스, 알림 서비스, 분석 서비스
```

**강물의 특징과 Stream의 유사점:**
- **한 방향 흐름**: 이벤트는 시간 순으로만 흐름
- **지속성**: 물(이벤트)이 지나간 흔적이 남아있음
- **여러 취수구**: 다양한 Consumer가 동시에 소비 가능
- **위치 추적**: 어디까지 마셨는지(처리했는지) 기억

### 🔧 Stream 데이터 구조

```
Redis Stream 구조:
┌─────────────────────────────────────────────┐
│ Stream Key: "order-events"                  │
├─────────────────────────────────────────────┤
│ Entry ID    │ Field-Value Pairs             │
├─────────────────────────────────────────────┤
│ 1692345600-0│ orderId: 101, userId: 123     │
│ 1692345601-0│ orderId: 102, userId: 456     │  
│ 1692345602-0│ orderId: 103, userId: 789     │
│ 1692345603-0│ orderId: 104, userId: 321     │
└─────────────────────────────────────────────┘

Entry ID 구조: timestamp-sequence
- timestamp: Unix 밀리초 타임스탬프
- sequence: 같은 밀리초 내 순서 번호
```

### 💡 핵심 특징

1. **순서 보장**: 시간 순서대로 엄격하게 정렬
2. **영속성**: 데이터가 삭제되지 않고 계속 저장
3. **Fan-out**: 하나의 이벤트를 여러 Consumer가 소비
4. **Backpressure**: Consumer 속도에 맞춰 처리 가능
5. **At-least-once**: 최소 한 번은 전달 보장

---

## 2. 기존 메시징 시스템과의 차이점

### 🔄 메시징 시스템 비교

#### 2.1 Redis Pub/Sub vs Stream
```redis
# Pub/Sub (휘발성)
PUBLISH order-channel '{"orderId": 101}'
# 구독자가 없으면 메시지 소실!

# Stream (영속성)  
XADD order-events * orderId 101 userId 123
# 메시지가 Stream에 영구 저장
```

| 특성 | Pub/Sub | Stream |
|------|---------|--------|
| **메시지 영속성** | ❌ 휘발성 | ✅ 영구 저장 |
| **순서 보장** | ❌ 보장 안됨 | ✅ 엄격한 순서 |
| **Consumer 장애 복구** | ❌ 불가능 | ✅ 가능 |
| **메시지 재처리** | ❌ 불가능 | ✅ 가능 |
| **부하 분산** | ❌ 제한적 | ✅ Consumer Group |

#### 2.2 Kafka vs Redis Stream
```
Kafka:
- 대용량 스트리밍에 특화
- 복잡한 설정과 운영
- 높은 처리량과 내구성

Redis Stream:
- 중소규모 이벤트 처리에 적합
- 간단한 설정과 운영
- Redis 인프라와 통합
```

### 🎯 사용 사례별 선택 가이드

| 요구사항 | 권장 솔루션 | 이유 |
|----------|-------------|------|
| 실시간 알림 | Pub/Sub | 즉시성, 단순함 |
| 주문 이벤트 처리 | **Stream** | 순서, 재처리 필요 |
| 대용량 로그 수집 | Kafka | 처리량, 영속성 |
| 채팅 메시지 | Stream | 순서, 이력 보관 |
| 시스템 모니터링 | Pub/Sub | 실시간성 |

### 💻 실제 시나리오 비교

```java
// Pub/Sub 방식 - 실시간 알림
@EventListener
public void handleOrderCompleted(OrderCompletedEvent event) {
    // 즉시 푸시 알림 발송 (놓치면 재전송 불가)
    pushNotificationService.send(event.getUserId(), "주문 완료");
}

// Stream 방식 - 이벤트 처리
@EventListener  
public void handleOrderCompleted(OrderCompletedEvent event) {
    // Stream에 이벤트 저장 (안전하게 보관)
    streamTemplate.add("order-events", 
        Map.of("orderId", event.getOrderId(),
               "userId", event.getUserId(),
               "eventType", "ORDER_COMPLETED"));
}
```

---

## 3. Stream 핵심 명령어와 패턴

### 📝 기본 명령어

#### 3.1 메시지 추가 (Producer)
```redis
# 기본 추가 (자동 ID 생성)
XADD order-events * orderId 101 userId 123 amount 50000

# 응답: "1692345600123-0"

# 수동 ID 지정
XADD order-events 1692345600123-0 orderId 102 userId 456

# 최대 길이 제한 (오래된 메시지 자동 삭제)
XADD order-events MAXLEN 1000 * orderId 103 userId 789

# 근사 최대 길이 (성능 최적화)
XADD order-events MAXLEN ~ 1000 * orderId 104 userId 321
```

#### 3.2 메시지 조회 (Consumer)
```redis
# 범위 조회
XRANGE order-events - +           # 전체 조회
XRANGE order-events 1692345600000 1692345700000  # 시간 범위
XRANGE order-events - + COUNT 10  # 최대 10개

# 역순 조회
XREVRANGE order-events + - COUNT 5  # 최신 5개

# 실시간 조회 (Blocking)
XREAD BLOCK 0 STREAMS order-events $  # 새 메시지 대기

# 특정 ID부터 조회
XREAD STREAMS order-events 1692345600123-0
```

#### 3.3 Consumer Group
```redis
# Consumer Group 생성
XGROUP CREATE order-events processing-group $ MKSTREAM

# Consumer로 메시지 읽기
XREADGROUP GROUP processing-group consumer1 COUNT 1 STREAMS order-events >

# 메시지 처리 완료 확인
XACK order-events processing-group 1692345600123-0

# 처리 중인 메시지 확인
XPENDING order-events processing-group

# 처리 실패한 메시지 재할당
XCLAIM order-events processing-group consumer2 3600000 1692345600123-0
```

### 🎯 실제 사용 패턴

#### 3.4 Producer 패턴
```java
@Service
@RequiredArgsConstructor
public class EventStreamProducer {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    /**
     * 주문 완료 이벤트 발행
     */
    public String publishOrderCompletedEvent(OrderCompletedEvent event) {
        Map<String, String> eventData = Map.of(
            "eventType", "ORDER_COMPLETED",
            "orderId", event.getOrderId().toString(),
            "userId", event.getUserId().toString(),
            "amount", event.getTotalAmount().toString(),
            "timestamp", event.getCompletedAt().toString()
        );
        
        // Stream에 이벤트 추가
        RecordId recordId = redisTemplate.opsForStream()
            .add("order-events", eventData);
        
        log.info("주문 완료 이벤트 발행: orderId={}, recordId={}", 
                event.getOrderId(), recordId.getValue());
        
        return recordId.getValue();
    }
    
    /**
     * 배치 이벤트 발행
     */
    public List<RecordId> publishBatchEvents(List<OrderCompletedEvent> events) {
        List<RecordId> recordIds = new ArrayList<>();
        
        // Pipeline 사용으로 성능 최적화
        List<Object> results = redisTemplate.executePipelined(
            (RedisCallback<Object>) connection -> {
                for (OrderCompletedEvent event : events) {
                    Map<String, String> eventData = convertToMap(event);
                    connection.xAdd("order-events".getBytes(), 
                                  eventData, XAddOptions.maxlen(10000));
                }
                return null;
            }
        );
        
        return recordIds;
    }
}
```

#### 3.5 Consumer 패턴
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class EventStreamConsumer {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final ProductRankingService rankingService;
    
    /**
     * 실시간 이벤트 소비
     */
    @Scheduled(fixedDelay = 1000)
    public void consumeOrderEvents() {
        try {
            // Consumer Group에서 메시지 읽기
            List<MapRecord<String, Object, Object>> records = 
                redisTemplate.opsForStream()
                    .read(Consumer.from("processing-group", "consumer-1"),
                          StreamReadOptions.empty().count(10).block(Duration.ofSeconds(1)),
                          StreamOffset.create("order-events", ReadOffset.lastConsumed()));
            
            for (MapRecord<String, Object, Object> record : records) {
                processOrderEvent(record);
                
                // 처리 완료 확인
                redisTemplate.opsForStream()
                    .acknowledge("order-events", "processing-group", record.getId());
            }
            
        } catch (Exception e) {
            log.error("이벤트 소비 중 오류", e);
        }
    }
    
    private void processOrderEvent(MapRecord<String, Object, Object> record) {
        try {
            Map<Object, Object> eventData = record.getValue();
            String eventType = (String) eventData.get("eventType");
            
            if ("ORDER_COMPLETED".equals(eventType)) {
                Long orderId = Long.parseLong((String) eventData.get("orderId"));
                Long productId = Long.parseLong((String) eventData.get("productId"));
                Integer quantity = Integer.parseInt((String) eventData.get("quantity"));
                
                // 상품 랭킹 업데이트
                rankingService.updateProductRanking(productId, quantity);
                
                log.info("주문 이벤트 처리 완료: recordId={}, orderId={}", 
                        record.getId().getValue(), orderId);
            }
            
        } catch (Exception e) {
            log.error("이벤트 처리 실패: recordId={}", record.getId().getValue(), e);
            throw e; // 재처리를 위해 예외 전파
        }
    }
}
```

---

## 4. 이벤트 스트리밍 시스템 구현

### 🏗 시스템 아키텍처

```
Order Service → Redis Stream → Multiple Consumers
     ↓              ↓              ↓
  이벤트 발행    영구 저장      분산 처리
     
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│OrderService │ → │ order-events │ → │RankingService│
└─────────────┘    │              │    └─────────────┘
                   │              │    ┌─────────────┐
                   │              │ → │EmailService │
                   │              │    └─────────────┘
                   │              │    ┌─────────────┐
                   │              │ → │AnalyticsService│
                   └──────────────┘    └─────────────┘
```

### 💻 핵심 구현

#### 4.1 이벤트 스트림 구성
```java
@Configuration
@EnableConfigurationProperties(StreamProperties.class)
public class StreamConfig {
    
    @Bean
    public RedisTemplate<String, String> streamRedisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setDefaultSerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new StringRedisSerializer());
        return template;
    }
    
    @Bean
    @ConditionalOnProperty(name = "stream.auto-create-groups", havingValue = "true")
    public CommandLineRunner createConsumerGroups(
            RedisTemplate<String, String> redisTemplate,
            StreamProperties properties) {
        
        return args -> {
            for (StreamProperties.StreamConfig stream : properties.getStreams()) {
                try {
                    // Consumer Group 생성
                    redisTemplate.opsForStream()
                        .createGroup(stream.getName(), stream.getGroupName());
                    
                    log.info("Consumer Group 생성: stream={}, group={}", 
                            stream.getName(), stream.getGroupName());
                            
                } catch (RedisSystemException e) {
                    if (e.getMessage().contains("BUSYGROUP")) {
                        log.info("Consumer Group 이미 존재: stream={}, group={}", 
                                stream.getName(), stream.getGroupName());
                    } else {
                        throw e;
                    }
                }
            }
        };
    }
}

@ConfigurationProperties(prefix = "stream")
@Data
public class StreamProperties {
    
    private boolean autoCreateGroups = true;
    private List<StreamConfig> streams = new ArrayList<>();
    
    @Data
    public static class StreamConfig {
        private String name;
        private String groupName;
        private int maxLen = 10000;
        private Duration blockTimeout = Duration.ofSeconds(5);
    }
}
```

#### 4.2 범용 이벤트 발행자
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class StreamEventPublisher {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    
    /**
     * 이벤트 발행 (JSON 직렬화)
     */
    public <T> RecordId publishEvent(String streamName, String eventType, T eventData) {
        try {
            Map<String, String> streamData = new HashMap<>();
            streamData.put("eventType", eventType);
            streamData.put("eventId", UUID.randomUUID().toString());
            streamData.put("timestamp", Instant.now().toString());
            streamData.put("data", objectMapper.writeValueAsString(eventData));
            
            RecordId recordId = redisTemplate.opsForStream()
                .add(streamName, streamData);
            
            log.debug("이벤트 발행 완료: stream={}, type={}, recordId={}", 
                     streamName, eventType, recordId.getValue());
            
            return recordId;
            
        } catch (Exception e) {
            log.error("이벤트 발행 실패: stream={}, type={}", streamName, eventType, e);
            throw new EventPublishException("이벤트 발행 실패", e);
        }
    }
    
    /**
     * 조건부 이벤트 발행 (Lua Script)
     */
    public RecordId publishEventIfCondition(String streamName, String eventType, 
                                          Object eventData, String condition) {
        String script = """
            local stream_name = KEYS[1]
            local condition_key = KEYS[2]
            local event_data = ARGV[1]
            
            -- 조건 확인
            local condition_value = redis.call('GET', condition_key)
            if condition_value == 'true' then
                return redis.call('XADD', stream_name, '*', 
                    'eventType', ARGV[2], 
                    'data', event_data,
                    'timestamp', ARGV[3])
            else
                return nil
            end
            """;
        
        List<String> keys = Arrays.asList(streamName, "condition:" + condition);
        List<String> args = Arrays.asList(
            serializeToJson(eventData),
            eventType,
            Instant.now().toString()
        );
        
        String recordId = redisTemplate.execute(
            RedisScript.of(script, String.class),
            keys,
            args.toArray()
        );
        
        return recordId != null ? RecordId.of(recordId) : null;
    }
}
```

#### 4.3 이벤트 소비자 기반 클래스
```java
@Component
@RequiredArgsConstructor
@Slf4j
public abstract class AbstractStreamConsumer {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    
    @Value("${spring.application.name}")
    private String applicationName;
    
    /**
     * 이벤트 소비 스케줄러
     */
    @Scheduled(fixedDelay = 1000)
    public void consumeEvents() {
        String streamName = getStreamName();
        String groupName = getGroupName();
        String consumerName = applicationName + "-" + InetAddress.getLocalHost().getHostName();
        
        try {
            List<MapRecord<String, Object, Object>> records = 
                redisTemplate.opsForStream()
                    .read(Consumer.from(groupName, consumerName),
                          StreamReadOptions.empty()
                              .count(getBatchSize())
                              .block(getBlockTimeout()),
                          StreamOffset.create(streamName, ReadOffset.lastConsumed()));
            
            for (MapRecord<String, Object, Object> record : records) {
                processRecord(record);
            }
            
        } catch (Exception e) {
            log.error("이벤트 소비 중 오류: stream={}, group={}", streamName, groupName, e);
        }
    }
    
    private void processRecord(MapRecord<String, Object, Object> record) {
        String recordId = record.getId().getValue();
        
        try {
            Map<Object, Object> data = record.getValue();
            String eventType = (String) data.get("eventType");
            String eventData = (String) data.get("data");
            String timestamp = (String) data.get("timestamp");
            
            // 하위 클래스에서 구현
            handleEvent(eventType, eventData, timestamp);
            
            // 처리 완료 확인
            redisTemplate.opsForStream()
                .acknowledge(getStreamName(), getGroupName(), record.getId());
            
            log.debug("이벤트 처리 완료: recordId={}, type={}", recordId, eventType);
            
        } catch (Exception e) {
            log.error("이벤트 처리 실패: recordId={}", recordId, e);
            
            // 실패한 메시지를 Dead Letter Stream으로 이동
            moveToDeadLetterStream(record, e);
        }
    }
    
    /**
     * 하위 클래스에서 구현해야 하는 메서드들
     */
    protected abstract String getStreamName();
    protected abstract String getGroupName();
    protected abstract void handleEvent(String eventType, String eventData, String timestamp);
    
    protected int getBatchSize() { return 10; }
    protected Duration getBlockTimeout() { return Duration.ofSeconds(5); }
    
    private void moveToDeadLetterStream(MapRecord<String, Object, Object> record, Exception error) {
        try {
            Map<String, String> deadLetterData = new HashMap<>(record.getValue());
            deadLetterData.put("originalRecordId", record.getId().getValue());
            deadLetterData.put("errorMessage", error.getMessage());
            deadLetterData.put("failedAt", Instant.now().toString());
            
            redisTemplate.opsForStream()
                .add(getStreamName() + ":dead-letter", deadLetterData);
            
            // 원본 메시지 ACK 처리
            redisTemplate.opsForStream()
                .acknowledge(getStreamName(), getGroupName(), record.getId());
                
        } catch (Exception e) {
            log.error("Dead Letter Stream 이동 실패", e);
        }
    }
}
```

---

## 5. Consumer Group과 분산 처리

### 👥 Consumer Group 작동 원리

```
Consumer Group: "order-processing"
┌─────────────────────────────────────────────┐
│ Stream: order-events                        │
│ ├─ 1001-0: {orderId: 101} → Consumer-1     │
│ ├─ 1002-0: {orderId: 102} → Consumer-2     │
│ ├─ 1003-0: {orderId: 103} → Consumer-1     │
│ └─ 1004-0: {orderId: 104} → Consumer-3     │
└─────────────────────────────────────────────┘

Consumer-1 (Instance A)  Consumer-2 (Instance B)  Consumer-3 (Instance C)
     ↓                        ↓                       ↓
  처리 완료                 처리 중                장애 발생
     ↓                        ↓                       ↓
  XACK 전송                대기 중              XCLAIM으로 재할당
```

### 🔧 분산 처리 구현

#### 5.1 확장 가능한 Consumer
```java
@Component
@ConditionalOnProperty(name = "stream.consumers.order-processing.enabled", havingValue = "true")
public class OrderEventStreamConsumer extends AbstractStreamConsumer {
    
    private final ProductRankingService rankingService;
    private final NotificationService notificationService;
    private final AnalyticsService analyticsService;
    
    @Override
    protected String getStreamName() {
        return "order-events";
    }
    
    @Override
    protected String getGroupName() {
        return "order-processing";
    }
    
    @Override
    protected void handleEvent(String eventType, String eventData, String timestamp) {
        switch (eventType) {
            case "ORDER_COMPLETED":
                handleOrderCompleted(eventData);
                break;
            case "ORDER_CANCELLED":
                handleOrderCancelled(eventData);
                break;
            case "PAYMENT_COMPLETED":
                handlePaymentCompleted(eventData);
                break;
            default:
                log.warn("알 수 없는 이벤트 타입: {}", eventType);
        }
    }
    
    private void handleOrderCompleted(String eventData) {
        try {
            OrderCompletedEvent event = objectMapper.readValue(eventData, OrderCompletedEvent.class);
            
            // 병렬 처리
            CompletableFuture.allOf(
                CompletableFuture.runAsync(() -> 
                    rankingService.updateProductRanking(event)),
                CompletableFuture.runAsync(() -> 
                    notificationService.sendOrderConfirmation(event)),
                CompletableFuture.runAsync(() -> 
                    analyticsService.recordOrderCompleted(event))
            ).join();
            
        } catch (Exception e) {
            log.error("주문 완료 이벤트 처리 실패", e);
            throw new EventProcessingException("주문 완료 이벤트 처리 실패", e);
        }
    }
}
```

#### 5.2 Consumer 스케일링
```yaml
# application.yml
stream:
  consumers:
    order-processing:
      enabled: true
      instances: 3
      batch-size: 5
      block-timeout: 2s
    ranking-update:
      enabled: true
      instances: 2
      batch-size: 10
      block-timeout: 1s
```

```java
@Configuration
public class StreamConsumerConfig {
    
    @Bean
    @ConditionalOnProperty(name = "stream.consumers.ranking-update.enabled")
    public List<RankingUpdateConsumer> rankingUpdateConsumers(
            @Value("${stream.consumers.ranking-update.instances:1}") int instances) {
        
        List<RankingUpdateConsumer> consumers = new ArrayList<>();
        
        for (int i = 0; i < instances; i++) {
            RankingUpdateConsumer consumer = new RankingUpdateConsumer();
            consumer.setConsumerInstance(i);
            consumers.add(consumer);
        }
        
        return consumers;
    }
}
```

#### 5.3 장애 복구 메커니즘
```java
@Component
@RequiredArgsConstructor
public class StreamHealthChecker {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    /**
     * 처리되지 않은 메시지 모니터링
     */
    @Scheduled(fixedRate = 30000) // 30초마다
    public void checkPendingMessages() {
        String streamName = "order-events";
        String groupName = "order-processing";
        
        try {
            // 처리 중인 메시지 목록 조회
            PendingMessagesSummary summary = redisTemplate.opsForStream()
                .pending(streamName, groupName);
            
            if (summary.getTotalPendingMessages() > 0) {
                log.warn("처리 대기 중인 메시지: {}건", summary.getTotalPendingMessages());
                
                // 오래된 메시지 재할당
                reclaimOldMessages(streamName, groupName);
            }
            
        } catch (Exception e) {
            log.error("Pending 메시지 확인 실패", e);
        }
    }
    
    private void reclaimOldMessages(String streamName, String groupName) {
        try {
            // 1시간 이상 처리되지 않은 메시지 조회
            long oneHourAgo = Instant.now().minus(1, ChronoUnit.HOURS).toEpochMilli();
            
            PendingMessages pendingMessages = redisTemplate.opsForStream()
                .pending(streamName, groupName, Range.unbounded(), 100L);
            
            for (PendingMessage message : pendingMessages) {
                if (message.getElapsedTimeSinceLastDelivery().toMillis() > oneHourAgo) {
                    // 메시지를 새로운 Consumer에게 재할당
                    String newConsumer = "recovery-consumer-" + System.currentTimeMillis();
                    
                    List<MapRecord<String, Object, Object>> claimed = 
                        redisTemplate.opsForStream()
                            .claim(streamName, groupName, newConsumer,
                                   Duration.ofHours(1), message.getId());
                    
                    if (!claimed.isEmpty()) {
                        log.info("메시지 재할당 완료: recordId={}, newConsumer={}", 
                                message.getId().getValue(), newConsumer);
                        
                        // 재처리 로직 실행
                        processClaimedMessage(claimed.get(0));
                    }
                }
            }
            
        } catch (Exception e) {
            log.error("메시지 재할당 실패", e);
        }
    }
}
```

---

## 6. 실전 활용 사례

### 🎯 주문 이벤트 스트리밍 시스템

#### 6.1 주문 서비스 통합
```java
@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {
    
    private final StreamEventPublisher eventPublisher;
    private final OrderRepository orderRepository;
    
    public void completeOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        // 1. 주문 완료 처리
        order.complete();
        orderRepository.save(order);
        
        // 2. 이벤트 발행
        OrderCompletedEvent event = OrderCompletedEvent.from(order);
        eventPublisher.publishEvent("order-events", "ORDER_COMPLETED", event);
        
        log.info("주문 완료 및 이벤트 발행: orderId={}", orderId);
    }
    
    /**
     * 트랜잭션 커밋 후 이벤트 발행
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void publishOrderCompletedEvent(OrderCompletedEvent event) {
        eventPublisher.publishEvent("order-events", "ORDER_COMPLETED", event);
    }
}
```

#### 6.2 실시간 대시보드 시스템
```java
@Component
public class RealTimeDashboardConsumer extends AbstractStreamConsumer {
    
    private final SimpMessagingTemplate messagingTemplate;
    private final DashboardMetricsService metricsService;
    
    @Override
    protected String getStreamName() {
        return "order-events";
    }
    
    @Override  
    protected String getGroupName() {
        return "dashboard-updates";
    }
    
    @Override
    protected void handleEvent(String eventType, String eventData, String timestamp) {
        try {
            switch (eventType) {
                case "ORDER_COMPLETED":
                    updateOrderMetrics(eventData);
                    break;
                case "PAYMENT_COMPLETED":
                    updatePaymentMetrics(eventData);
                    break;
            }
            
        } catch (Exception e) {
            log.error("대시보드 업데이트 실패: eventType={}", eventType, e);
        }
    }
    
    private void updateOrderMetrics(String eventData) {
        OrderCompletedEvent event = parseEvent(eventData, OrderCompletedEvent.class);
        
        // 실시간 메트릭 업데이트
        DashboardMetrics metrics = metricsService.updateOrderMetrics(event);
        
        // WebSocket으로 실시간 전송
        messagingTemplate.convertAndSend("/topic/dashboard/orders", metrics);
        
        log.debug("대시보드 주문 메트릭 업데이트: orderId={}", event.getOrderId());
    }
}
```

#### 6.3 이벤트 재처리 시스템
```java
@Service
@RequiredArgsConstructor
public class EventReprocessingService {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final StreamEventPublisher eventPublisher;
    
    /**
     * 특정 시간 범위의 이벤트 재처리
     */
    public void reprocessEvents(String streamName, LocalDateTime from, LocalDateTime to) {
        String fromId = from.toInstant(ZoneOffset.UTC).toEpochMilli() + "-0";
        String toId = to.toInstant(ZoneOffset.UTC).toEpochMilli() + "-0";
        
        try {
            List<MapRecord<String, Object, Object>> records = 
                redisTemplate.opsForStream()
                    .range(streamName, Range.closed(fromId, toId));
            
            for (MapRecord<String, Object, Object> record : records) {
                // 재처리용 스트림에 복사
                Map<Object, Object> data = new HashMap<>(record.getValue());
                data.put("reprocessed", "true");
                data.put("originalRecordId", record.getId().getValue());
                
                eventPublisher.publishEvent(streamName + ":reprocess", 
                                          (String) data.get("eventType"), data);
            }
            
            log.info("이벤트 재처리 완료: stream={}, count={}", streamName, records.size());
            
        } catch (Exception e) {
            log.error("이벤트 재처리 실패: stream={}", streamName, e);
            throw new EventReprocessingException("이벤트 재처리 실패", e);
        }
    }
    
    /**
     * 실패한 이벤트 Dead Letter Queue에서 재처리
     */
    public void reprocessDeadLetterEvents(String deadLetterStream) {
        try {
            List<MapRecord<String, Object, Object>> deadLetterRecords = 
                redisTemplate.opsForStream()
                    .range(deadLetterStream, Range.unbounded());
            
            for (MapRecord<String, Object, Object> record : deadLetterRecords) {
                Map<Object, Object> data = record.getValue();
                String originalEventType = (String) data.get("eventType");
                String originalEventData = (String) data.get("data");
                
                // 원본 스트림에 재발행
                String originalStream = deadLetterStream.replace(":dead-letter", "");
                eventPublisher.publishEvent(originalStream, originalEventType, originalEventData);
                
                // Dead Letter에서 제거
                redisTemplate.opsForStream().delete(deadLetterStream, record.getId());
            }
            
        } catch (Exception e) {
            log.error("Dead Letter 이벤트 재처리 실패", e);
        }
    }
}
```

### 🧪 테스트 코드

#### 6.4 Stream 통합 테스트
```java
@SpringBootTest
@Testcontainers
class OrderEventStreamTest {
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);
    
    @Autowired
    private StreamEventPublisher eventPublisher;
    
    @Autowired  
    private RedisTemplate<String, String> redisTemplate;
    
    @Test
    void 이벤트_발행_및_소비_테스트() throws InterruptedException {
        // Given
        String streamName = "test-events";
        String groupName = "test-group";
        
        // Consumer Group 생성
        redisTemplate.opsForStream().createGroup(streamName, groupName);
        
        OrderCompletedEvent event = OrderCompletedEvent.builder()
            .orderId(101L)
            .userId(123L)
            .totalAmount(BigDecimal.valueOf(50000))
            .build();
        
        // When
        RecordId recordId = eventPublisher.publishEvent(streamName, "ORDER_COMPLETED", event);
        
        // Then
        assertThat(recordId).isNotNull();
        
        // Consumer로 읽기 테스트
        List<MapRecord<String, Object, Object>> records = 
            redisTemplate.opsForStream()
                .read(Consumer.from(groupName, "test-consumer"),
                      StreamReadOptions.empty().count(1),
                      StreamOffset.create(streamName, ReadOffset.lastConsumed()));
        
        assertThat(records).hasSize(1);
        
        MapRecord<String, Object, Object> record = records.get(0);
        assertThat(record.getValue().get("eventType")).isEqualTo("ORDER_COMPLETED");
        
        // ACK 확인
        Long ackCount = redisTemplate.opsForStream()
            .acknowledge(streamName, groupName, record.getId());
        assertThat(ackCount).isEqualTo(1L);
    }
}
```

---

## 📚 정리 및 핵심 포인트

### ✅ Redis Stream 사용해야 하는 이유
1. **순서 보장**: 엄격한 시간 순서로 이벤트 처리
2. **영속성**: 메시지 손실 방지와 재처리 가능
3. **확장성**: Consumer Group으로 수평 확장
4. **내결함성**: 장애 복구와 메시지 재할당

### 🎯 적용 시 주의사항
1. **메모리 관리**: MAXLEN으로 Stream 크기 제한
2. **Consumer 관리**: 장애 발생 시 메시지 재할당 필요
3. **순서 처리**: Consumer 간 메시지 순서 고려
4. **모니터링**: Pending 메시지와 처리율 추적

### 🚀 이번 과제 적용 방안
1. **주문 이벤트**: 완료, 취소, 결제 이벤트 스트리밍
2. **실시간 랭킹**: 주문 이벤트 기반 실시간 랭킹 업데이트
3. **분산 처리**: 여러 Consumer로 이벤트 병렬 처리
4. **장애 복구**: Dead Letter Queue와 메시지 재처리

이제 Redis Stream을 활용하여 안정적이고 확장 가능한 이벤트 스트리밍 시스템을 구축할 수 있습니다!