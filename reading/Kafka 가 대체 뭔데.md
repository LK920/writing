

# Apache Kafka: 초보자를 위한 설명과 활용 사례

## 1. Kafka가 뭔가요?

Apache Kafka는 데이터를 빠르고 안정적으로 주고받을 수 있게 해주는 **소프트웨어 시스템**입니다. 쉽게 말해, **데이터를 한 곳에서 다른 곳으로 실시간으로 보내고 저장하는 도구**입니다. 예를 들어, 온라인 쇼핑몰에서 누군가가 주문을 하면 그 정보를 즉시 분석 시스템이나 알림 시스템으로 보내야 할 때 Kafka를 사용합니다.

- **어떤 서비스인가?**
    - Kafka는 **메시지 브로커**와 **이벤트 스트리밍 플랫폼** 역할을 합니다.
    - 메시지 브로커: 앱 간 데이터를 전달하는 중개자(예: 우체국).
    - 이벤트 스트리밍: 앱에서 일어나는 사건(예: 주문, 결제)을 실시간으로 처리하고 전달.
    - 데이터를 **대규모로**, **빠르게**, **안정적으로** 처리하는 데 최적화되어 있습니다.
    
- **어떤 데이터를 다루나?**    
    - **이벤트**: 시스템에서 일어나는 중요한 일. 예: 사용자가 쿠폰을 발급받음, 결제를 완료함.
    - 예: e-커머스에서 사용자가 쿠폰을 발급받으면 `CouponAppliedEvent`라는 데이터가 생성되고, Kafka가 이를 분석 시스템이나 알림 시스템으로 보냅니다.
    
- **어디에 쓰이나?**
    - 대규모 웹사이트(예: 아마존, 넷플릭스), 은행, 앱 등에서 사용자 행동, 로그, 거래 데이터를 처리.
    - e-커머스 과제: 주문 생성, 결제, 쿠폰 발급 이벤트를 데이터 플랫폼에 보내거나 사용자에게 알림을 보냄.

## 2. Kafka의 기본 개념

Kafka를 이해하려면 몇 가지 핵심 용어를 알아야 합니다. 초보자를 위해 간단히 설명합니다.

1. **토픽 (Topic)**:
    - 데이터를 저장하는 카테고리 또는 폴더.
    - 예: `order.created`는 주문 생성 데이터를 저장하는 토픽, `coupon.applied`는 쿠폰 발급 데이터를 저장.
    - 각 토픽은 데이터를 순서대로 기록(로그처럼).
    
2. **브로커 (Broker)**:
    - Kafka 서버. 데이터를 저장하고 전달하는 역할을 합니다.
    - 여러 브로커가 모여 **클러스터**를 이루며, 데이터를 안전하게 보관.
    
3. **파티션 (Partition)**:
    - 토픽을 여러 조각으로 나눈 것. 각 조각은 독립적으로 처리되어 속도가 빠름.
    - 예: `order.created` 토픽이 4개 파티션으로 나뉘면, 4개의 서버가 동시에 처리 가능.
    
4. **생산자 (Producer)**:
    - 데이터를 만들어 토픽에 보내는 프로그램.
    - 예: 주문 서비스가 `OrderCreatedEvent`를 `order.created` 토픽에 보냄.
    
5. **소비자 (Consumer)**:
    - 토픽에서 데이터를 읽어 처리하는 프로그램.
    - 예: 데이터 분석 시스템이 `order.created` 토픽을 읽어 주문 통계 계산.
    
6. **소비자 그룹 (Consumer Group)**:
    - 여러 소비자가 팀을 이루어 데이터를 나눠 처리.
    - 예: 알림 시스템과 분석 시스템이 같은 `order.created` 데이터를 각각 처리.
    
7. **오프셋 (Offset)**:
    - 각 메시지의 위치 번호. 소비자가 어디까지 읽었는지 기록.
    - 예: 오프셋 100까지 읽었다면, 다음은 101부터 읽음.

## 3. Kafka가 이벤트, 메시징, 대규모 처리에 적합한 이유

Kafka가 왜 이런 용도로 쓰이는지, 초보자도 이해할 수 있게 설명합니다.

### 3.1 이벤트 처리
- **이벤트란?**: 시스템에서 일어나는 중요한 일. 예: 사용자가 주문을 하거나, 쿠폰을 발급받음.
- **Kafka의 역할**:
    - 이벤트를 **실시간으로** 처리하고 전달.
    - 예: e-커머스에서 사용자가 쿠폰을 발급받으면(`POST /coupons/issue`), Kafka가 `CouponAppliedEvent`를 `coupon.applied` 토픽으로 보내고, 알림 시스템이 이를 읽어 사용자에게 "쿠폰 발급 완료" 메시지를 보냄.
- **왜 적합?**:
    - **빠른 처리**: 이벤트를 2~5ms 안에 전달.
    - **다중 소비자**: 같은 이벤트를 여러 시스템(예: 알림, 분석)이 동시에 처리 가능.
    - **데이터 보존**: 이벤트를 디스크에 저장해 나중에 다시 읽을 수 있음.

### 3.2 메시징
- **메시징이란?**: 앱 간 데이터를 주고받는 것. 예: 주문 서비스가 결제 정보를 분석 시스템에 보내는 것.
- **Kafka의 역할**:
    - 메시지를 토픽에 저장하고, 필요한 앱이 읽어 처리.
    - 예: 결제 성공 시 `OrderPaidEvent`를 `order.paid` 토픽으로 보내, 데이터 플랫폼이 이를 읽어 분석.
- **왜 적합?**:
    - **안정성**: 데이터를 여러 브로커에 복제해 서버 장애에도 손실 없음.
    - **확장성**: 메시지량이 많아지면 파티션과 브로커를 추가해 처리.
    - **비동기 처리**: 메시지를 보내고 기다리지 않아도 됨, 앱 성능 향상.

### 3.3 대규모 데이터 처리
- **대규모 데이터란?**: 하루에 수백만~수십억 개의 이벤트(예: 주문, 클릭, 조회).
- **Kafka의 역할**:
    - 많은 데이터를 빠르고 안정적으로 처리.
    - 예: e-커머스에서 하루 수백만 주문 데이터를 Kafka로 보내, 데이터 플랫폼이 실시간으로 분석.
- **왜 적합?**:
    - **높은 처리량**: 초당 수백만 메시지 처리 가능.
    - **파티션**: 데이터를 여러 파티션으로 나눠 병렬 처리.
    - **장기 저장**: 데이터를 며칠~몇 주간 저장, 필요 시 과거 데이터 재처리.

## 4. Kafka의 동작 원리 (쉽게)

1. **데이터 보내기**:
    - 주문 서비스가 `OrderCreatedEvent`를 `order.created` 토픽에 보냄.
    - Kafka는 이벤트를 특정 파티션에 저장(예: 사용자 ID 기준).
    
2. **데이터 저장**:
    - 브로커가 이벤트를 디스크에 저장.
    - 복제본을 다른 브로커에 저장해 장애 대비.
    
3. **데이터 읽기**:
    - 분석 시스템, 알림 시스템 등이 `order.created` 토픽에서 데이터를 읽음.
    - 각 시스템은 자신의 속도로, 원하는 만큼 읽음.
    
4. **위치 관리**:
    - 소비자는 읽은 데이터의 오프셋을 저장, 중간에 멈춰도 이어서 처리 가능.

## 5. e-커머스 과제에서의 Kafka 사용

e-커머스 과제에서 Kafka는 `MessagingPort`로 구현되어 이벤트를 비동기적으로 전송합니다. 주요 사용 사례는 다음과 같습니다:

- **주문 생성** (`POST /orders`):
    - `OrderCreatedEvent`를 `order.created` 토픽으로 보내, 데이터 플랫폼이 주문 정보를 기록.
- **결제 성공** (`POST /orders/{orderId}/pay`):
    - `OrderPaidEvent`를 `order.paid` 토픽으로 보내, 실시간 분석 및 알림.
- **잔액 충전** (`POST /balance/charge`):
    - `BalanceUpdatedEvent`를 `balance.updated` 토픽으로 보내, 사용자 활동 추적.
- **쿠폰 발급** (`POST /coupons/issue`):
    - `CouponAppliedEvent`를 `coupon.applied` 토픽으로 보내, 사용자 알림(예: "쿠폰 발급 완료" 푸시) 및 데이터 분석.

**구현 예시 (Spring Kafka)**:

```java
@Component
public class KafkaAdapter implements MessagingPort {
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public KafkaAdapter(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    @Override
    public void publish(String topic, Object event) {
        kafkaTemplate.send(topic, event)
            .addCallback(
                result -> logger.info("Published to topic {}: {}", topic, event),
                ex -> logger.error("Failed to publish to topic {}: {}", topic, ex.getMessage())
            );
    }
}
```

**설정 (application.yml)**:

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

## 6. Kafka의 장점

- **빠름**: 이벤트를 밀리초 단위로 처리.
- **확장 가능**: 서버(브로커)와 파티션을 늘려 대규모 데이터 처리.
- **안정적**: 데이터를 복제해 서버 장애에도 손실 없음.
- **유연함**: 같은 데이터를 여러 앱이 각기 다른 목적으로 사용 가능.
- **데이터 보존**: 이벤트를 설정 기간(예: 7일) 동안 저장, 나중에 재처리 가능.

## 7. Kafka의 단점과 주의점

- **복잡함**: 설정과 관리(브로커, 파티션, 소비자 그룹)가 복잡, 초보자에게 어려움.
- **리소스**: 서버 자원(CPU, 디스크) 사용량 많음.
- **소규모 프로젝트**: 단순한 메시지 전달에는 RabbitMQ가 더 간단할 수 있음.
- **e-커머스 과제**:
    - 소규모라면 RabbitMQ나 Redis로 충분.
    - 대규모 주문/쿠폰 이벤트 처리 시 Kafka 적합.

## 8. Kafka가 쓰이는 실제 사례

- **넷플릭스**: 사용자가 본 영상, 검색 기록을 Kafka로 수집, 추천 시스템에 활용.
- **아마존**: 주문, 결제, 배송 이벤트를 Kafka로 처리, 실시간 분석.
- **우버**: 예약, 운전자 위치 데이터를 Kafka로 처리, 실시간 매칭.
- **e-커머스 과제**:
    - 선착순 쿠폰 발급 시 `CouponAppliedEvent`를 Kafka로 보내, 사용자 알림과 데이터 분석.
    - 인기 상품 분석(`GET /products/popular`)을 위해 주문 데이터를 Kafka로 처리.

## 9. e-커머스 과제에서의 Kafka 설계

- **토픽**:
    - `order.created`: 주문 생성 이벤트.
    - `order.paid`: 결제 성공 이벤트.
    - `balance.updated`: 잔액 충전 이벤트.
    - `coupon.applied`: 쿠폰 발급 이벤트.
- **확장성**: 주문량 증가 시 파티션과 브로커 추가.
- **장애 내성**: 복제로 데이터 손실 방지.
- **테스트**: `Testcontainers`로 Kafka 통합 테스트, `MockMessagingAdapter`로 단위 테스트.

## 10. 결론

Apache Kafka는 **데이터를 실시간으로 주고받고, 대규모로 처리하는 시스템**입니다. 이벤트를 빠르고 안정적으로 전달하며, 여러 앱이 동시에 데이터를 사용할 수 있게 합니다. e-커머스 과제에서는 주문, 결제, 쿠폰 이벤트를 데이터 플랫폼과 알림 시스템에 보내는 데 적합합니다. `MessagingPort`로 추상화하여 기술 강결합을 피하고, 비즈니스 가치를 극대화합니다.

**다음 단계**:

- 로컬에서 Kafka 실행해 보기.
- 간단한 생산자/소비자 코드 작성.
- e-커머스 과제에서 `coupon.applied` 토픽 테스트.



---

### 비유로 이해하기
Kafka를 **고속도로**로 생각해 보세요:
- **차량(메시지)**: 주문, 결제, 쿠폰 발급 같은 이벤트 데이터.
- **차선(파티션)**: 데이터를 나눠서 빠르게 처리할 수 있는 구간.
- **톨게이트(브로커)**: 데이터를 받아 저장하고 전달하는 서버.
- **운전자(소비자)**: 데이터를 받아 처리하는 애플리케이션.

이 고속도로는 차량이 아무리 많아도 막히지 않고, 사고(서버 장애)가 나도 데이터를 잃지 않으며, 차선을 늘려(확장) 더 많은 차량을 처리할 수 있습니다.

## 2. Kafka가 왜 이벤트, 메시징, 대규모 처리에 적합할까?

Kafka는 **이벤트 스트리밍**, **메시지 브로커**, **대규모 데이터 처리**에 적합한 이유가 그 설계와 구조에 있습니다. 아래에서 각각의 이유를 쉽게 설명합니다.

### 2.1 이벤트 스트리밍이란?
이벤트는 시스템에서 일어나는 중요한 행동(예: 사용자가 주문, 결제, 쿠폰 사용)입니다. Kafka는 이런 이벤트를 실시간으로 스트리밍(흐르게) 처리합니다. 예를 들어:
- e-커머스에서 사용자가 상품을 장바구니에 추가하면 `CartAddedEvent`가 발생.
- Kafka는 이 이벤트를 즉시 데이터 플랫폼에 보내거나, 사용자에게 알림을 보내는 시스템으로 전달.

**왜 적합한가?**
- **실시간 처리**: 이벤트를 2ms 이내에 처리(낮은 지연 시간).[](https://kafka.apache.org/)
- **내구성**: 이벤트를 디스크에 저장해 데이터 손실 방지.
- **다중 소비자 지원**: 같은 이벤트를 여러 시스템(예: 분석 시스템, 알림 시스템)이 동시에 처리 가능.

### 2.2 메시지 브로커로서의 Kafka
메시지 브로커는 애플리케이션 간 데이터를 전달하는 중개자입니다. Kafka는 전통적인 메시지 브로커(예: RabbitMQ)보다 강력한 기능을 제공합니다:
- **높은 처리량**: 초당 수백만 메시지 처리 가능.[](https://docs.confluent.io/kafka/design/delivery-semantics.html)
- **파티셔닝**: 데이터를 파티션으로 나눠 병렬 처리, 확장성 제공.
- **복제**: 데이터를 여러 서버에 복사해 장애 발생 시에도 데이터 보존.
- **e-커머스 예**: 결제 성공 시 `OrderPaidEvent`를 Kafka로 보내 데이터 플랫폼과 알림 시스템에 동시에 전달.

**Kafka vs RabbitMQ**:
- RabbitMQ: 소규모, 안정적 메시지 전달에 적합. 메시지가 소비되면 삭제.
- Kafka: 대규모 데이터 스트리밍, 메시지 장기 저장, 다중 소비자 지원.[](https://aws.amazon.com/what-is/apache-kafka/)

### 2.3 대규모 데이터 처리
Kafka는 대규모 데이터를 처리하는 데 최적화되어 있습니다:
- **확장성**: 브로커(서버)와 파티션을 추가해 클러스터 확장. 최대 수천 브로커, 하루 수조 메시지 처리 가능.[](https://kafka.apache.org/)
- **파티션 기반 병렬 처리**: 데이터를 여러 파티션으로 나눠 병렬로 처리, 처리 속도 향상.
- **장기 저장**: 메시지를 디스크에 저장해 필요 시 과거 데이터 재처리 가능(전통적 큐는 메시지 소비 후 삭제).
- **e-커머스 예**: 하루 수백만 주문 데이터를 데이터 플랫폼으로 전송하거나, 인기 상품 분석을 위해 과거 데이터 재처리.

## 3. Kafka의 주요 구성 요소

Kafka의 동작을 이해하려면 핵심 구성 요소를 알아야 합니다. 아래는 초보자를 위한 간단한 설명입니다.

1. **Topic (토픽)**:
   - 데이터를 정리하는 폴더. 예: `order.created`, `coupon.applied`.
   - 각 토픽은 이벤트를 저장하는 로그(기록)로, 메시지가 순서대로 추가됨.
   - e-커머스 예: 주문 생성 이벤트는 `order.created` 토픽에 저장.

2. **Partition (파티션)**:
   - 토픽을 여러 조각으로 나눈 것. 각 파티션은 독립적으로 처리되어 병렬 처리 가능.
   - 예: `order.created` 토픽이 4개 파티션으로 나뉘면, 4개의 소비자가 동시에 처리 가능.

3. **Broker (브로커)**:
   - Kafka 서버. 토픽과 파티션을 저장하고, 생산자와 소비자 간 메시지 전달.
   - 클러스터는 여러 브로커로 구성, 데이터 복제(replication)로 장애 내성 제공.

4. **Producer (생산자)**:
   - 메시지를 생성해 토픽에 보내는 애플리케이션.
   - 예: 주문 서비스가 `OrderCreatedEvent`를 `order.created` 토픽에 전송.

5. **Consumer (소비자)**:
   - 토픽에서 메시지를 읽어 처리하는 애플리케이션.
   - **소비자 그룹**: 여러 소비자가 협력해 파티션의 메시지를 나눠 처리.
   - 예: 데이터 플랫폼이 `order.created`를 읽어 분석, 알림 시스템이 읽어 사용자 알림 전송.

6. **Offset (오프셋)**:
   - 각 메시지의 위치를 나타내는 번호. 소비자가 어디까지 읽었는지 추적.
   - 예: 소비자가 오프셋 100까지 읽었다면, 다음은 101부터 읽음.

7. **Kafka Connect**:
   - 외부 시스템(예: DB, S3)과 Kafka를 연결. 예: MySQL의 주문 데이터를 Kafka로 가져오기.

8. **Kafka Streams**:
   - 실시간 데이터 처리 라이브러리. 예: 주문 데이터를 필터링하거나 집계해 새로운 토픽 생성.[](https://kafka.apache.org/uses)

**비유**: 토픽은 편지함, 파티션은 편지함의 칸막이, 브로커는 우체국, 생산자는 편지 보내는 사람, 소비자는 편지 읽는 사람입니다.

## 4. Kafka의 동작 원리 (간단히)

1. **생산자가 메시지 전송**:
   - 주문 서비스가 `OrderCreatedEvent`를 `order.created` 토픽에 보냄.
   - 메시지는 키(예: `userId`)를 기반으로 특정 파티션에 저장.

2. **브로커가 메시지 저장**:
   - 브로커는 메시지를 디스크에 저장하고, 복제본을 다른 브로커에 유지.
   - 메시지는 삭제되지 않고 설정된 보존 기간(예: 7일)까지 유지.[](https://stackoverflow.blog/2022/07/21/event-driven-topic-design-using-kafka/)

3. **소비자가 메시지 읽기**:
   - 소비자 그룹이 토픽의 파티션에서 메시지를 읽음.
   - 각 소비자는 자신에게 할당된 파티션만 처리, 병렬 처리로 효율성 증가.[](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch04.html)

4. **오프셋 관리**:
   - 소비자는 읽은 메시지의 오프셋을 저장, 장애 발생 시 중복/누락 없이 재처리 가능.

**e-커머스 예**:
- 사용자가 쿠폰을 발급하면(`POST /coupons/issue`):
  1. 쿠폰 서비스가 `CouponAppliedEvent`를 `coupon.applied` 토픽에 전송.
  2. 브로커가 이벤트를 저장, 복제.
  3. 데이터 플랫폼이 이벤트를 읽어 분석, 알림 시스템이 읽어 푸시 알림 전송.

## 5. e-커머스 과제에서의 Kafka 활용

e-커머스 과제에서 Kafka는 `MessagingPort`로 구현되어 이벤트를 비동기적으로 전송합니다. 주요 사용 사례는 다음과 같습니다:

- **주문 생성**: `OrderCreatedEvent`를 `order.created` 토픽으로 전송, 데이터 플랫폼에 기록.
- **결제 성공**: `OrderPaidEvent`를 `order.paid` 토픽으로 전송, 실시간 분석 및 알림.
- **잔액 충전**: `BalanceUpdatedEvent`를 `balance.updated` 토픽으로 전송, 사용자 활동 추적.
- **쿠폰 발급**: `CouponAppliedEvent`를 `coupon.applied` 토픽으로 전송, 선착순 쿠폰 알림 및 데이터 플랫폼 연동.

**구현 예시 (Spring Kafka)**:
```java
@Component
public class KafkaAdapter implements MessagingPort {
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public KafkaAdapter(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    @Override
    public void publish(String topic, Object event) {
        kafkaTemplate.send(topic, event)
            .addCallback(
                result -> logger.info("Published to topic {}: {}", topic, event),
                ex -> logger.error("Failed to publish to topic {}: {}", topic, ex.getMessage())
            );
    }
}
```

**설정 (application.yml)**:
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

## 6. Kafka의 주요 장점

1. **높은 처리량**: 초당 수백만 메시지 처리, 대규모 이벤트 처리에 적합.[](https://docs.confluent.io/kafka/design/delivery-semantics.html)
2. **낮은 지연 시간**: 2ms 이내 메시지 전달, 실시간 처리 가능.[](https://kafka.apache.org/)
3. **확장성**: 브로커와 파티션 추가로 클러스터 확장, 수조 메시지/일 처리.[](https://kafka.apache.org/)
4. **내구성**: 디스크에 메시지 저장, 복제로 데이터 손실 방지.
5. **장애 내성**: 브로커 장애 시 복제본에서 데이터 제공.
6. **다중 소비자**: 동일 토픽을 여러 소비자 그룹이 독립적으로 처리.[](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch04.html)
7. **이벤트 소싱**: 상태 변화를 이벤트 로그로 저장, 과거 데이터 재처리 가능.[](https://www.tinybird.co/blog-posts/event-sourcing-with-kafka)

## 7. Kafka의 활용 사례

Kafka는 다양한 산업에서 사용됩니다. e-커머스 외의 예시를 포함해 정리합니다:

1. **사용자 활동 추적**:
   - LinkedIn: 페이지 조회, 검색, 클릭 이벤트를 실시간으로 Kafka 토픽에 저장, 분석.[](https://www.upsolver.com/blog/apache-kafka-use-cases-when-to-use-not)
   - e-커머스: 상품 조회, 장바구니 추가 이벤트를 `user.activity` 토픽으로 전송.

2. **로그 수집**:
   - 서버 로그를 Kafka로 수집, 중앙화된 분석 시스템으로 전송.
   - Kafka는 파일 세부사항을 추상화해 낮은 지연 시간으로 처리.[](https://kafka.apache.org/08/documentation.html)

3. **실시간 분석**:
   - 은행: 거래 이벤트를 Kafka로 처리, 실시간으로 의심스러운 거래 감지.[](https://www.upsolver.com/blog/apache-kafka-use-cases-when-to-use-not)
   - e-커머스: 인기 상품 집계(`GET /products/popular`)를 Kafka Streams로 처리.

4. **마이크로서비스 통신**:
   - 라이드헤일링 앱: 예약 서비스가 `ride.booked` 이벤트를 Kafka로 전송, 드라이버 매칭 서비스가 소비.[](https://www.upsolver.com/blog/apache-kafka-use-cases-when-to-use-not)
   - e-커머스: 주문 서비스와 결제 서비스 간 이벤트 교환.

5. **이벤트 소싱**:
   - 상태 변화를 이벤트 로그로 저장, 필요 시 재구성. 예: 은행 계좌 잔액을 거래 이벤트로 계산.[](https://www.tinybird.co/blog-posts/event-sourcing-with-kafka)

## 8. Kafka의 한계와 고려사항

- **설정 복잡성**: 클러스터 설정, 파티션 관리, 소비자 그룹 조정에 전문 지식 필요.[](https://www.prodyna.com/insights/event-driven-architecture-and-kafka)
- **대용량 메시지 처리**: Kafka는 대용량 메시지(예: 20MB 이상)에 비효율적, 파일 저장소와 결합 권장.[](https://www.baeldung.com/java-kafka-send-large-message)
- **소규모 시스템**: 소규모 프로젝트에서는 RabbitMQ가 더 간단할 수 있음.
- **e-커머스 고려사항**:
  - 선착순 쿠폰 발급 시 높은 처리량 필요, Kafka 적합.
  - 테스트 환경에서는 `MockMessagingAdapter`로 대체 가능.

## 9. e-커머스 과제에서의 Kafka 설계

- **구현체**: `MessagingPort`의 `KafkaAdapter`로 구현, `KafkaTemplate` 사용.
- **토픽 설계**:
  - `order.created`: 주문 생성 이벤트.
  - `order.paid`: 결제 성공 이벤트.
  - `balance.updated`: 잔액 충전 이벤트.
  - `coupon.applied`: 쿠폰 발급 이벤트.
- **확장성**: 주문/쿠폰 이벤트 증가 시 파티션과 브로커 추가.
- **장애 내성**: 복제를 통해 데이터 손실 방지.
- **테스트**: `Testcontainers`로 Kafka 통합 테스트, `MockMessagingAdapter`로 단위 테스트.

## 10. 결론

Apache Kafka는 **실시간 이벤트 스트리밍**, **메시지 브로커**, **대규모 데이터 처리**에 최적화된 플랫폼입니다. 높은 처리량, 낮은 지연 시간, 확장성, 내구성으로 e-커머스 과제에서 주문, 결제, 쿠폰 이벤트를 실시간으로 처리하는 데 적합합니다. `MessagingPort`로 추상화하여 기술 강결합을 피하고, 비즈니스 가치를 극대화합니다. Kafka는 복잡하지만, 대규모 시스템에서는 강력한 솔루션입니다.

**다음 단계**:
- Kafka 기본 설정 실습: 로컬에서 단일 브로커 실행.
- Spring Kafka로 간단한 생산자/소비자 구현.
- e-커머스 과제에서 `coupon.applied` 토픽 테스트.

---

### 추가 설명 (Markdown 외부)
Kafka가 이벤트, 메시징, 대규모 처리에 적합한 이유를 요약하면:
- **이벤트**: 이벤트를 실시간으로 스트리밍, 다중 소비자 지원.
- **메시징**: 높은 처리량과 복제로 안정적 메시지 전달.
- **대규모 처리**: 파티션과 클러스터로 확장성 제공.

e-커머스 과제에서는 `CouponAppliedEvent` 같은 이벤트를 Kafka로 전송해 데이터 플랫폼과 알림 시스템에 실시간으로 전달, 비즈니스 요구사항(실시간 데이터 전송, 선착순 쿠폰 알림)을 충족합니다. 추가로 궁금한 점(예: Kafka 설정 방법, 특정 코드, RabbitMQ와 비교)이 있으면 알려주세요!