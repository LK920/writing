# 주문과 결제 분리: 재고 관리 관점 분석

## 1. 배경

e-커머스 시스템에서 주문(`Order`)과 결제(`Payment`)를 별도의 엔티티로 분리한 설계(`Ecommerce_Simplified_ERD.md`)를 바탕으로, 재고 관리 관점에서 분리의 장단점을 분석합니다. 설계서(`e-커머스 서비스 설계서.md`, `API_시퀀스_다이어그램_통합.html`)에 따르면, 재고 관리는 `Product` 엔티티의 `stock`(실제 재고)과 `reservedStock`(임시 예약 재고)을 통해 이루어지며, 동시성 제어(`LockingPort`)와 멱등성 처리(`IdempotencyPort`)를 사용합니다. 사용자의 우려("재고가 맛탱이 간다")를 고려해, 분리와 통합의 재고 관리 영향을 명확히 비교합니다.

## 2. 주문과 결제 분리: 재고 관리 흐름

### 2.1 현재 설계 (분리)
- **주문 생성 (`POST /orders`)**:
  - **역할**: 사용자가 상품을 선택해 주문 생성, 재고를 임시 예약.
  - **재고 처리**:
    - `Product.reservedStock` 증가: 주문한 수량만큼 재고 예약.
    - `LockingPort`로 상품별 동시성 제어.
    - 예: 상품 A 10개 주문 → `reservedStock += 10`.
  - **시퀀스 다이어그램**:
    - 클라이언트 → `CreateOrderUseCase` → `StoragePort`: `Order`, `OrderItem` 저장, `reservedStock` 증가.
    - `EventLog` 저장, `OrderCreatedEvent` 발행(`MessagingPort`).
  - **멱등성**: `IdempotencyKey`로 중복 주문 방지.

- **결제 처리 (`POST /orders/{orderId}/pay`)**:
  - **역할**: 주문에 대한 결제 처리, 재고 확정.
  - **재고 처리**:
    - `Product.reservedStock` 감소, `Product.stock` 감소: 결제 성공 시 예약된 재고를 실제 재고에서 차감.
    - 예: 상품 A 10개 결제 → `reservedStock -= 10`, `stock -= 10`.
    - `LockingPort`로 주문 및 사용자별 동시성 제어.
  - **시퀀스 다이어그램**:
    - 클라이언트 → `PayOrderUseCase` → `StoragePort`: `Payment` 저장, `reservedStock`, `stock` 업데이트.
    - `EventLog` 저장, `PaymentCompletedEvent` 발행.
  - **멱등성**: `IdempotencyKey`로 중복 결제 방지.

- **ERD 반영**:
  - `Order`: 주문 정보(`userId`, `totalAmount`).
  - `Payment`: 결제 정보(`orderId`, `status`, `amount`, `couponId`).
  - `Product`: `stock`, `reservedStock`으로 재고 관리.

### 2.2 재고 관리 흐름
1. **주문 생성**:
   - 사용자가 상품 A 10개 주문 → `reservedStock += 10`.
   - 실제 재고(`stock`)는 그대로, 예약 재고만 증가.
   - 동시성 제어로 중복 예약 방지.
2. **결제 처리**:
   - 결제 성공 → `reservedStock -= 10`, `stock -= 10`.
   - 결제 실패/취소 → `reservedStock -= 10` (재고 복구).
   - 동시성 제어로 재고 무결성 보장.
3. **타임아웃 처리** (설계서 미명시, 추정):
   - 주문 후 결제 안 하면 `reservedStock` 복구(예: 30분 후).
   - 스케줄링으로 처리 가능.

## 3. 주문과 결제 분리 vs 통합: 재고 관리 비교

### 3.1 분리의 장점 (재고 관리 관점)
- **재고 무결성 보장**:
  - **임시 예약(`reservedStock`)**: 주문 시 재고를 예약만 하고, 결제 시 확정. 중간에 결제 실패 시 `reservedStock`을 복구해 재고 손실/과다 판매 방지.
  - 예: 사용자 A가 상품 10개 주문(예약) 후 결제 실패 → `reservedStock -= 10`, 다른 사용자 구매 가능.
- **동시성 제어 분리**:
  - 주문: `LockingPort`로 상품별 락(`productId`).
  - 결제: `LockingPort`로 주문 및 사용자별 락(`orderId`, `userId`).
  - 각각 다른 락으로 충돌 최소화, 재고 업데이트 안전.
- **비즈니스 유연성**:
  - 주문 후 결제 지연(예: 은행 송금) 허용.
  - 결제 재시도 가능(`Payment` 엔티티로 여러 시도 기록).
- **사용자 경험**:
  - 재고 예약으로 주문 시점에 재고 확보, 결제 실패 시 복구.
  - "재고 맛탱이" 위험 감소: 동시 주문/결제 시 재고 초과 판매 방지.

### 3.2 분리의 단점
- **복잡성 증가**:
  - `reservedStock` 관리, 타임아웃 로직 추가(예: 주문 후 30분 내 결제 안 하면 복구).
  - `Order`와 `Payment` 간 트랜잭션 관리 필요.
- **동기화 문제**:
  - `reservedStock`과 `stock` 동기화 실패 시 데이터 불일치 가능(드물지만, 트랜잭션 오류 처리 필요).
- **개발 비용**:
  - 별도 `Payment` 엔티티와 로직 구현, 테스트 추가.

### 3.3 통합의 장점
- **단순성**:
  - `Order`에 결제 정보(`status`, `couponId`) 포함, 단일 트랜잭션으로 처리.
  - 예: `POST /orders`에서 재고 감소(`stock -= quantity`)와 결제 동시 처리.
  - `reservedStock` 관리 불필요.
- **적은 코드**:
  - `Payment` 엔티티와 별도 API(`POST /orders/{orderId}/pay`) 불필요.
  - 단일 `OrderController`로 처리.

### 3.4 통합의 단점 (재고 관리 관점)
- **재고 초과 판매 위험**:
  - 주문과 결제를 한 번에 처리 → 결제 실패 시 재고 복구 로직 복잡.
  - 예: 사용자 A가 상품 10개 주문 중 결제 실패 → `stock` 이미 감소했으면 복구 안 되면 다른 사용자 구매 불가.
  - 동시 요청 시 트랜잭션 충돌로 재고 불일치 가능성("재고 맛탱이" 상황).
- **유연성 감소**:
  - 결제 지연(예: 은행 송금) 불가능, 주문 즉시 결제 강제.
  - 결제 재시도 관리 어려움(단일 `Order` 상태로 처리).
- **동시성 부하**:
  - 주문+결제 단일 트랜잭션은 더 많은 리소스(DB 락, 잔액/재고 동시 처리) 필요.
  - 동시 요청 시 `409 Conflict` 빈도 증가.

### 3.5 "재고 맛탱이" 문제 분석
사용자의 우려처럼, 통합 시 재고가 잘못 관리될 가능성이 높습니다:
- **통합 시나리오**:
  - 사용자 A, B가 동시에 상품 A(재고 10개) 주문, 각각 10개 요청.
  - `POST /orders`에서 재고 감소(`stock -= 10`)와 결제 처리.
  - 동시성 제어 실패 시 둘 다 성공 → 재고 -10(음수), 초과 판매.
- **분리 시나리오**:
  - 주문 시 `reservedStock += 10` (A, B 각각).
  - 결제 시 `reservedStock -= 10`, `stock -= 10`.
  - 동시성 제어(`LockingPort`)로 한 명만 성공, 다른 한 명은 `409 Conflict`.
  - 결제 실패 시 `reservedStock` 복구 → 재고 무결성 보장.

**결론**: 분리된 설계가 재고 초과 판매 방지, 동시성 관리에 유리. "재고 맛탱이" 위험 감소.

## 4. 재고 관리 최적화 (분리 기준)
- **임시 예약(`reservedStock`)**:
  - 주문 시 재고를 예약해 다른 사용자의 주문 충돌 방지.
  - 예: `Product` 테이블에 `stock=100`, `reservedStock=10` → 실제 가용 재고는 90.
- **동시성 제어**:
  - `LockingPort`로 상품별 락(`productId`) 사용.
  - Redis 분산 락 또는 DB 락으로 구현.
- **타임아웃 처리**:
  - 주문 후 결제 안 하면 `reservedStock` 복구.
  - 예: 스케줄링(`@Scheduled`)으로 30분 후 복구.
  ```java
  @Scheduled(fixedDelay = 60000)
  public void cleanupExpiredOrders() {
      List<Order> expiredOrders = storagePort.findExpiredOrders(Duration.ofMinutes(30));
      for (Order order : expiredOrders) {
          storagePort.releaseReservedStock(order.getItems());
          storagePort.updateOrderStatus(order.getId(), "CANCELLED");
      }
  }
  ```
- **멱등성**:
  - `IdempotencyKey`로 중복 주문/결제 방지.
  - Redis(`CachePort`)에 저장, TTL 24시간.

## 5. 구현 예시
### 주문 생성 (OrderController)
```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final CreateOrderUseCase createOrderUseCase;
    private final IdempotencyPort idempotencyPort;

    public OrderController(CreateOrderUseCase createOrderUseCase, IdempotencyPort idempotencyPort) {
        this.createOrderUseCase = createOrderUseCase;
        this.idempotencyPort = idempotencyPort;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody OrderRequest request) {
        Optional<String> cached = idempotencyPort.checkIdempotencyKey(idempotencyKey);
        if (cached.isPresent()) {
            return ResponseEntity.ok(new OrderResponse(cached.get()));
        }

        Order order = createOrderUseCase.createOrder(request.getUserId(), request.getItems());
        idempotencyPort.saveIdempotencyKey(idempotencyKey, order.toJson(), 24 * 3600);
        return ResponseEntity.status(201).body(new OrderResponse(order));
    }
}
```

### 결제 처리 (PaymentController)
```java
@RestController
@RequestMapping("/orders")
public class PaymentController {
    private final PayOrderUseCase payOrderUseCase;
    private final IdempotencyPort idempotencyPort;

    public PaymentController(PayOrderUseCase payOrderUseCase, IdempotencyPort idempotencyPort) {
        this.payOrderUseCase = payOrderUseCase;
        this.idempotencyPort = idempotencyPort;
    }

    @PostMapping("/{orderId}/pay")
    public ResponseEntity<PaymentResponse> payOrder(
            @PathVariable String orderId,
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody PaymentRequest request) {
        Optional<String> cached = idempotencyPort.checkIdempotencyKey(idempotencyKey);
        if (cached.isPresent()) {
            return ResponseEntity.ok(new PaymentResponse(cached.get()));
        }

        Payment payment = payOrderUseCase.payOrder(orderId, request.getUserId(), request.getCouponId());
        idempotencyPort.saveIdempotencyKey(idempotencyKey, payment.toJson(), 24 * 3600);
        return ResponseEntity.ok(new PaymentResponse(payment));
    }
}
```

## 6. 비즈니스 가치
- **재고 무결성**: 분리로 주문 시 재고 예약, 결제 시 확정 → 초과 판매 방지.
- **사용자 경험**: 결제 실패/지연 허용, 재고 확보로 안정적 구매.
- **확장성**: 결제 재시도, 타임아웃 처리 용이.
- **오버 엔지니어링 방지**: `reservedStock`과 동시성 제어로 최소한의 복잡성 유지.

## 7. 결론
주문과 결제를 분리하는 것은 재고 관리 관점에서 **훨씬 나은 선택**입니다. `reservedStock`으로 재고를 예약하고, 결제 시 확정하며, 동시성 제어(`LockingPort`)와 멱등성(`IdempotencyPort`)으로 무결성을 보장합니다. 통합 시 재고 초과 판매("재고 맛탱이") 위험이 크고, 동시성 충돌 빈도 증가, 유연성 감소합니다. 분리는 약간의 복잡성을 추가하지만, 재고 무결성과 사용자 경험을 크게 개선합니다.

**다음 단계**:
- `reservedStock` 복구 스케줄링 구현.
- 동시 주문/결제 테스트(`Testcontainers`로 Redis, DB).
- `IdempotencyPort`로 멱등성 테스트.