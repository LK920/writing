
---

## 1. 데드락 예방을 위한 락 순서 일관성 유지

**질문**: 콘서트 공연 정보와 다른 테이블에 대해 락을 걸 때, 락 순서를 일치시켜야 한다고 하셨는데, 서비스 로직이 복잡할 경우 락 순서를 확인하고 유지하기 어렵지 않나요? 데드락 발생 후 처리 방안은 무엇인가요?

**코치 답변**:
- 락 순서를 일치시키는 것은 데드락 예방의 핵심입니다. 트랜잭션이 동일한 자원에 접근할 때 락을 거는 순서를 표준화하면 데드락 가능성을 줄일 수 있습니다. 예를 들어, 트랜잭션 A와 B가 각각 좌석과 공연 테이블에 락을 걸 때, 항상 공연 테이블 → 좌석 테이블 순으로 락을 걸도록 설계하면 데드락을 방지할 수 있습니다.
- 데드락이 발생하면 MySQL은 이를 감지하고 한 트랜잭션을 롤백합니다. 이 경우 애플리케이션은 SQL 에러(예: MySQL의 40001 에러 코드)를 캐치하여 로깅하거나 재시도 로직을 구현해야 합니다.
- 실무에서는 데드락 모니터링과 알림 설정이 중요합니다. DBA가 모니터링을 수행하더라도, 애플리케이션 개발자는 예외 처리와 로깅을 철저히 해야 합니다.

**추가 팁**:
- **락 순서 표준화**: 락 순서를 문서화하고, 코드 리뷰 시 이를 점검하는 프로세스를 도입하세요. 예를 들어, 테이블 접근 순서를 코드 주석이나 설계 문서에 명시하면 실수를 줄일 수 있습니다.
- **타임아웃 설정**: 락 대기 시간을 제한하여 데드락 상황에서 무한 대기를 방지하세요. MySQL의 `innodb_lock_wait_timeout` 설정(기본값 50초)을 조정하거나, 애플리케이션 레벨에서 타임아웃을 설정할 수 있습니다.
- **모니터링 도구**: New Relic, Prometheus, 또는 MySQL의 Performance Schema를 활용하여 데드락 발생을 실시간으로 감지하고, 발생 원인을 분석하세요. 예를 들어, `information_schema.innodb_trx` 테이블을 쿼리하여 현재 락 상태를 확인할 수 있습니다.
- **예제 코드** (JPA에서 비관적 락 사용):
  ```java
  @Transactional
  public void reserveSeat(Long seatId) {
      Seat seat = seatRepository.findByIdWithLock(seatId); // SELECT ... FOR UPDATE
      if (seat.isAvailable()) {
          seat.reserve();
          seatRepository.save(seat);
      } else {
          throw new IllegalStateException("좌석이 이미 예약되었습니다.");
      }
  }
  ```

**보완 설명**: 데드락은 복잡한 트랜잭션에서 자주 발생하므로, 트랜잭션 크기를 최소화하고 락 범위를 좁히는 것이 중요합니다. 예를 들어, 불필요한 테이블까지 락을 걸지 않도록 쿼리를 최적화하고, 트랜잭션 내에서 외부 API 호출을 피해야 합니다.

---

## 2. 외부 API 연동 시 트랜잭션 처리

**질문**: 결제 기능에서 외부 API(예: PG사, 메일 발송)와 연동할 때 트랜잭션 처리를 어떻게 조정해야 하나요? 메일 발송 실패 시 전체 트랜잭션을 롤백해야 하나요?

**코치 답변**:
- **메일 발송**: 메일 발송은 트랜잭션에서 분리하는 것이 적절합니다. 메일 발송은 벤더사마다 응답 속도가 다르고, 성공/실패 여부를 즉시 확인하기 어려운 경우가 많습니다. 따라서 메일 발송을 비동기 작업(예: 작업 큐, 스케줄링)으로 처리하거나 별도 트랜잭션으로 분리하여 주문 처리와 독립적으로 동작하도록 설계하세요.
- **결제(PG사)**: PG사가 동기적으로 성공/실패 응답을 제공한다면, 결제 요청을 트랜잭션에 포함시켜 실패 시 롤백할 수 있습니다. 하지만 트래픽이 많거나 응답이 느린 경우, 결제 요청을 비동기적으로 처리하고, 주문 상태를 "결제 요청 중"으로 저장한 뒤 나중에 콜백이나 폴링으로 상태를 업데이트하는 방식이 적합합니다.
- **트랜잭션 범위**: 트랜잭션에는 반드시 함께 성공해야 하는 작업만 포함시키세요. 예를 들어, 재고 차감과 주문 저장은 트랜잭션에 포함되지만, 메일 발송은 제외하는 것이 일반적입니다.

**추가 팁**:
- **비동기 처리 구현**: Spring의 `@Async` 어노테이션이나 메시지 큐(RabbitMQ, Kafka)를 활용하여 메일 발송을 비동기적으로 처리하세요. 예를 들어:
  ```java
  @Async
  public void sendOrderConfirmationEmail(Order order) {
      emailService.sendEmail(order.getCustomerEmail(), "주문 확인", "주문이 완료되었습니다.");
  }
  ```
- **결제 상태 관리**: 결제 상태를 관리하기 위해 `Order` 테이블에 `payment_status` 컬럼을 추가하고, 상태 전이(예: PENDING → SUCCESS/FAILED)를 기록하세요. 실패 시 수동 롤백 로직을 구현할 수 있습니다.
  ```java
  public void processPayment(Long orderId) {
      Order order = orderRepository.findById(orderId);
      order.setPaymentStatus(PaymentStatus.PENDING);
      orderRepository.save(order);
      // PG사에 비동기 결제 요청 전송
      paymentGateway.requestPayment(order);
  }
  ```
- **콜백 처리**: PG사의 콜백 API를 수신하여 결제 상태를 업데이트하세요. Redis나 메시지 큐를 사용하여 콜백 데이터를 비동기적으로 처리하면 안정성을 높일 수 있습니다.
- **재시도 메커니즘**: 외부 API 호출 실패 시 재시도 로직을 구현하세요. Spring Retry나 Resilience4j를 사용하면 간단히 설정할 수 있습니다.
  ```java
  @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000))
  public void sendEmail(String to, String subject, String body) {
      emailService.send(to, subject, body);
  }
  ```

**보완 설명**: 외부 API 연동은 네트워크 지연, 타임아웃, 장애 가능성 때문에 트랜잭션에서 분리하는 것이 일반적입니다. 특히 결제와 같은 중요한 작업은 상태 기계를 활용하여 상태 전이를 명확히 관리하고, 비동기 처리로 시스템 부하를 줄이는 설계가 중요합니다.

---

## 3. 동시성 테스트 전략

**질문**: 동시성 테스트를 위해 스레드를 여러 개 생성하여 요청을 보내고 카운트를 확인하는 방식 외에, 실무에서 더 나은 테스트 방식은 무엇인가요?

**코치 답변**:
- 스레드 풀(ExecutorService)을 사용한 동시성 테스트는 유효하지만, 더 정교한 방법으로 트랜잭션 쿼리를 순차적으로 실행하여 데드락이나 동시성 문제를 재현할 수 있습니다.
- 통합 테스트에서 JPA Repository나 ORM 호출을 모의 실행하여 쿼리 순서를 확인하고, 데드락 발생 여부를 검증하세요. 예를 들어, 두 세션에서 쿼리를 수동으로 실행하여 데드락 상황을 재현한 뒤 테스트 케이스를 작성합니다.
- 동시성 테스트는 성능에 영향을 받으므로, Gradle 태스크를 분리하여 단위 테스트와 통합 테스트를 별도로 실행하세요. 동시성 테스트만 별도로 실행하도록 설정하면 효율적입니다.

**추가 팁**:
- **테스트 컨테이너 활용**: Testcontainers를 사용하여 실제 MySQL 환경을 시뮬레이션하고, 멀티스레드 테스트를 실행하세요. 예:
  ```java
  @Testcontainers
  public class StockConcurrencyTest {
      @Container
      private static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

      @Test
      public void testConcurrentStockDeduction() throws InterruptedException {
          ExecutorService executor = Executors.newFixedThreadPool(10);
          CountDownLatch latch = new CountDownLatch(10);
          for (int i = 0; i < 10; i++) {
              executor.submit(() -> {
                  stockService.deductStock(777L, 1);
                  latch.countDown();
              });
          }
          latch.await();
          assertEquals(0, stockRepository.findById(777L).getStock());
      }
  }
  ```
- **쿼리 로그 분석**: MySQL의 General Query Log나 Slow Query Log를 활성화하여 테스트 중 실행된 쿼리를 분석하세요. 이를 통해 락 충돌이나 데드락 원인을 파악할 수 있습니다.
- **JMeter/ Gatling**: 부하 테스트 도구를 사용하여 실제 트래픽 패턴을 시뮬레이션하고, 동시성 문제를 재현하세요. 예를 들어, JMeter로 100개의 동시 요청을 보내 재고 차감 로직의 정합성을 검증할 수 있습니다.
- **데드락 재현**: 두 트랜잭션이 서로 다른 순서로 락을 걸도록 테스트 케이스를 작성하여 데드락을 의도적으로 유발하고, 이를 감지하는 로직을 검증하세요.

**보완 설명**: 동시성 테스트는 실제 프로덕션 환경과 유사한 조건에서 실행해야 정확도가 높습니다. 테스트 컨테이너와 부하 테스트 도구를 조합하여 다양한 시나리오(예: 1000명 동시 구매)를 테스트하면 실무에서의 문제를 사전에 발견할 수 있습니다.

---

## 4. 낙관적 락에서 버전 컬럼 외 대안

**질문**: 낙관적 락을 사용할 때 버전 컬럼 대신 재고 값(stock)을 WHERE 절에 넣어 처리해도 괜찮은가요? 예: `UPDATE product SET stock = stock - 1 WHERE id = 777 AND stock = 40`

**코치 답변**:
- 재고 값(`stock`)을 WHERE 절에 포함시켜 낙관적 �락을 구현하는 것은 가능합니다. 예를 들어, 재고가 40일 때 차감하도록 쿼리를 작성하면 충돌을 방지할 수 있습니다.
- 하지만 재고 값만으로 조건을 설정하면, 결제 취소 등으로 재고가 복원되는 경우 문제가 발생할 수 있습니다. 예: 트랜잭션 A가 `stock=40`으로 조회한 후, 다른 트랜잭션 B가 재고를 30으로 차감 후 결제 취소로 40으로 복원하면, 트랜잭션 A가 잘못된 데이터를 업데이트할 수 있습니다.
- 버전 컬럼은 단일 조건으로 데이터의 변경 여부를 명확히 추적하므로, 여러 컬럼을 업데이트하거나 복잡한 조건이 필요한 경우 더 안정적이고 최적화된 방법입니다.

**추가 팁**:
- **버전 컬럼 사용 예제** (JPA):
  ```java
  @Entity
  public class Product {
      @Id
      private Long id;
      private int stock;
      @Version
      private int version;

      public void deductStock(int quantity) {
          if (stock >= quantity) {
              stock -= quantity;
          } else {
              throw new IllegalStateException("재고 부족");
          }
      }
  }
  ```
  ```sql
  UPDATE product SET stock = stock - 1, version = version + 1 WHERE id = 777 AND version = 1;
  ```
- **인덱스 최적화**: WHERE 절에 사용하는 컬럼(버전 또는 재고)에 인덱스를 설정하여 쿼리 성능을 최적화하세요. 예: `CREATE INDEX idx_product_version ON product(version);`
- **대안으로 타임스탬프**: 버전 컬럼 대신 `updated_at` 타임스탬프를 사용할 수 있지만, 타임스탬프 충돌 가능성(동일 밀리초 내 업데이트)이 있으므로 버전 컬럼이 더 안전합니다.
- **복잡한 조건 회피**: 여러 컬럼을 조건으로 사용하면 인덱스 누락으로 Slow Query가 발생할 가능성이 높아지므로, 단일 버전 컬럼을 사용하는 것이 권장됩니다.

**보완 설명**: 버전 컬럼은 낙관적 락의 표준 방식으로, JPA의 `@Version` 어노테이션을 통해 간단히 구현할 수 있습니다. 재고 값 기반 접근은 간단한 시나리오에서 유효하지만, 복잡한 비즈니스 로직에서는 유지보수성과 안정성 면에서 한계가 있습니다.

---

## 5. 동시성 문제: 좌석 예약 시나리오

**질문**: 사용자 A가 유효한 점유 상태에서 예약 API를 호출했지만, 예약 완료 전에 점유가 만료되어 사용자 B가 동일 좌석을 점유하는 경우, 동시성 문제를 예방하려면 어떤 기법을 사용해야 하나요?

**코치 답변**:
- 이 시나리오는 데드락보다는 동시성 문제로, 예약 완료 전에 점유가 만료되면 예약 API는 실패해야 합니다.
- 해결 방안:
  - 좌석 정보 조회 시 비관적 락(`SELECT ... FOR UPDATE`)을 사용하여 다른 트랜잭션의 접근을 차단합니다.
  - 점유 상태를 검증한 후, 유효하지 않으면 트랜잭션을 롤백하거나 예약 요청을 실패 처리합니다.
  - 예: 사용자 A의 예약 요청은 실패, 사용자 B의 요청은 성공하도록 설계.
- 트랜잭션 내에서 점유 상태 검증과 업데이트를 원자적으로 처리해야 합니다.

**추가 팁**:
- **비관적 락 구현**:
  ```java
  @Transactional
  public void reserveSeat(Long seatId, Long userId) {
      Seat seat = seatRepository.findByIdWithLock(seatId); // SELECT ... FOR UPDATE
      if (!seat.isValidOccupancy()) {
          throw new IllegalStateException("점유 만료로 예약 불가");
      }
      seat.reserve(userId);
      seatRepository.save(seat);
  }
  ```
- **Redis로 점유 관리**: Redis의 `SETNX`를 사용하여 분산 환경에서 점유 상태를 관리할 수 있습니다. 예:
  ```java
  public boolean occupySeat(String seatId, String userId, int ttlSeconds) {
      return redisTemplate.opsForValue().setIfAbsent("seat:" + seatId, userId, ttlSeconds, TimeUnit.SECONDS);
  }
  ```
- **타임스탬프 검증**: 점유 상태에 타임스탬프를 추가하여 만료 여부를 확인하세요. 예: `occupancy_expires_at < NOW()` 조건을 추가.
- **모니터링**: 점유 만료로 인한 예약 실패를 로깅하고, 사용자에게 명확한 에러 메시지를 제공하여 UX를 개선하세요.

**보완 설명**: 좌석 예약은 초 단위로 충돌이 빈번하므로, 비관적 락이나 분산 락(Redis)을 사용하여 정합성을 보장해야 합니다. 점유 만료 시간을 명확히 관리하고, 트랜잭션 내에서 검증 로직을 강화하면 동시성 문제를 효과적으로 예방할 수 있습니다.

---

## 6. 트랜잭션과 락의 차이

**질문**: 트랜잭션과 락은 비슷한 개념으로 느껴지는데, DB 차원에서 락을 트랜잭션으로 이해해도 되나요? 트랜잭션 격리 수준과 락의 차이는 무엇인가요?

**코치 답변**:
- 트랜잭션과 락은 다른 개념입니다. 트랜잭션은 ACID 특성을 보장하는 논리적 작업 단위이고, 락은 데이터 접근을 제어하는 메커니즘입니다.
- 트랜잭션 격리 수준(예: Repeatable Read)은 읽기 동시성 문제를 해결하지만, 쓰기 동시성 문제(예: 재고 차감)는 해결하지 못합니다. 예를 들어, Repeatable Read는 Non-Repeatable Read를 방지하지만, 두 트랜잭션이 동시에 재고를 차감하면 Lost Update 문제가 발생할 수 있습니다.
- 락은 조회된 데이터에 대해 읽기/쓰기 제약을 걸어 동시성 문제를 해결합니다. 예: 비관적 락(`SELECT ... FOR UPDATE`)은 다른 트랜잭션의 쓰기를 차단합니다.
- Uncommitted Read에 락을 걸어도 롤백된 데이터에 대한 락은 의미가 없으므로, 트랜잭션 격리 수준과 락은 해결하는 문제가 다릅니다.

**추가 팁**:
- **격리 수준별 동시성 문제**:
  - **Uncommitted Read**: Dirty Read 발생 가능.
  - **Committed Read**: Non-Repeatable Read 발생 가능.
  - **Repeatable Read**: Phantom Read 발생 가능, MySQL 기본 격리 수준.
  - **Serializable**: 모든 동시성 문제 해결 but 성능 저하 및 데드락 위험.
- **락 종류**:
  - **공유 락(Shared Lock)**: `SELECT ... FOR SHARE`, 읽기 허용, 쓰기 차단.
  - **배타 락(Exclusive Lock)**: `SELECT ... FOR UPDATE`, 읽기/쓰기 모두 차단.
- **격리 수준과 락 조합**: Repeatable Read에서 비관적 락을 사용하면 대부분의 동시성 문제를 해결할 수 있습니다. 예:
  ```sql
  START TRANSACTION;
  SELECT stock FROM product WHERE id = 777 FOR UPDATE;
  UPDATE product SET stock = stock - 1 WHERE id = 777;
  COMMIT;
  ```
- **JPA 락 설정**:
  ```java
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  @Query("SELECT p FROM Product p WHERE p.id = :id")
  Product findByIdWithLock(@Param("id") Long id);
  ```

**보완 설명**: 트랜잭션 격리 수준은 읽기 일관성을 보장하지만, 쓰기 충돌은 락으로 해결해야 합니다. 특히 전자상거래 시스템에서는 재고 차감, 좌석 예약 등 쓰기 작업이 빈번하므로, 비관적 락이나 낙관적 락을 적절히 조합하여 사용하세요.

---

## 7. 애플리케이션 레벨 락 vs. DB 커넥션 풀 관리

**질문**: Slow Query 최적화, 트랜잭션 분리, 커넥션 누수 제거 후에도 커넥션 풀이 부족하다면, 애플리케이션 레벨에서 락을 걸어야 하나요?

**코치 답변**:
- 락은 필요할 때 사용해야 하지만, 커넥션 풀이 부족하다면 먼저 커넥션 풀 크기를 늘리는 것이 우선입니다.
- 성능 테스트를 통해 애플리케이션 서버와 DB의 리소스 사용량을 확인하세요. 예: 커넥션 풀이 가득 차면 애플리케이션 스레드가 대기 상태에 빠지고, DB도 유휴 상태가 될 수 있습니다.
- 최적화 순서:
  1. Slow Query 튜닝.
  2. 트랜잭션 최소화.
  3. 커넥션 누수 제거.
  4. 커넥션 풀 크기 조정.
- 커넥션 풀이 계속 차는 경우, CPU/DB 사용량이 60~70%를 초과하지 않도록 조정하고, 필요하면 스케일 아웃이나 비동기 API 설계를 고려하세요.

**추가 팁**:
- **커넥션 풀 설정** (HikariCP 예시):
```properties
  spring.datasource.hikari.maximum-pool-size=20
  spring.datasource.hikari.minimum-idle=5
  spring.datasource.hikari.connection-timeout=30000
```
- **모니터링**: HikariCP의 메트릭을 Prometheus로 수집하거나, Spring Actuator를 사용하여 커넥션 풀 상태를 모니터링하세요.
  ```java
  @Bean
  public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
      return registry -> registry.config().commonTags("application", "ecommerce");
  }
  ```
- **비동기 처리**: 트래픽이 많을 경우, 메시지 큐를 활용한 비동기 처리가 커넥션 풀 부하를 줄이는 데 
  효과적입니다. 예: Kafka로 주문 이벤트를 발행하고, 별도 소비자가 처리.
- **애플리케이션 레벨 락**: 동시성 충돌이 낮은 경우, 낙관적 락을 우선 고려하세요. 예:
  ```java
  @Transactional
  public void deductStock(Long productId, int quantity) {
      Product product = productRepository.findById(productId);
      if (product.getStock() >= quantity) {
          product.deductStock(quantity);
          productRepository.save(product);
      } else {
          throw new IllegalStateException("재고 부족");
      }
  }
  ```

**보완 설명**: 애플리케이션 레벨 락은 DB 락보다 유연하지만, 분산 환경에서는 Redis와 같은 외부 저장소를 사용한 분산 락이 필요합니다. 커넥션 풀 문제는 쿼리 최적화와 비동기 처리로 먼저 해결하고, 락은 최후의 수단으로 사용하세요.

---

## 8. 테이블 파티셔닝과 샤딩

**질문**: 결제 내역과 같은 테이블을 일자별로 나누어 관리하는 것은 어떤가요?

**코치 답변**:
- **파티셔닝**: 결제 내역이나 히스토리성 데이터는 월 단위로 파티셔닝하는 것이 일반적입니다. 데이터가 많아지고, 개인정보 삭제 요건(예: 1년 후 삭제) 때문에 파티셔닝이 유용합니다.
- **샤딩**: 사용자 ID(예: 1~10000번, 10001~20000번)로 테이블을 나누는 방식입니다. 하지만 샤딩은 집계 쿼리가 여러 샤드를 조회해야 하므로 복잡도가 높아집니다.
- **제약 조건**: 파티셔닝 키는 프라이머리 키에 포함되어야 하며, 샤딩은 키 선정의 자유도가 높지만 집계 작업이 어려울 수 있습니다.

**추가 팁**:
- **파티셔닝 예제** (MySQL):
  ```sql
  CREATE TABLE payment_history (
      id BIGINT PRIMARY KEY,
      order_id BIGINT,
      created_at DATE
  ) PARTITION BY RANGE (UNIX_TIMESTAMP(created_at)) (
      PARTITION p202508 VALUES LESS THAN (UNIX_TIMESTAMP('2025-09-01')),
      PARTITION p202509 VALUES LESS THAN (UNIX_TIMESTAMP('2025-10-01'))
  );
  ```
- **샤딩 예제**: 사용자 ID 기반 샤딩은 애플리케이션 레벨에서 샤드 키를 계산하여 라우팅합니다.
  ```java
  public String getShardTableName(Long userId) {
      return "payment_history_shard_" + (userId % 10);
  }
  ```
- **삭제 용이성**: 파티셔닝은 `DROP PARTITION`으로 오래된 데이터를 쉽게 삭제할 수 있어, 대량 삭제로 인한 장애를 방지합니다.
- **성능 고려**: 파티셔닝은 쿼리 성능을 개선하지만, 잘못된 키 선정은 오히려 성능을 저하시킬 수 있으므로, 데이터 접근 패턴을 분석하여 키를 선택하세요.

**보완 설명**: 파티셔닝은 히스토리성 데이터 관리에 적합하며, MySQL의 RANGE 파티셔닝이 가장 일반적입니다. 샤딩은 대규모 시스템에서 사용되지만, 구현 복잡도와 유지보수 비용이 높으므로 신중히 설계해야 합니다.

---

## 9. 트랜잭션 단위 분리와 데이터 정합성

**질문**: 데이터 정합성을 유지하면서 트랜잭션을 작게 분리하려면 어떻게 해야 하나요?

**코치 답변**:
- 트랜잭션을 작게 분리하면 성능이 향상되고, 실패 시 롤백 비용이 줄어듭니다. 예: 결제, 재고 차감, 주문 상태 업데이트를 별도 트랜잭션으로 처리하고, 각 작업의 성공/실패 상태를 기록하세요.
- 상태 기반 설계: 주문 상태(`ORDER_SUCCESS`, `PAYMENT_SUCCESS`)를 컬럼으로 관리하여 각 트랜잭션이 독립적으로 동작하도록 설계합니다.
- 비동기 처리: 이메일 발송, 재고 처리 등을 비동기 작업으로 분리하여 트랜잭션 크기를 줄입니다.
- 실패 처리: 트랜잭션 실패 시 수동 롤백(예: 재고 복원)을 구현하여 정합성을 유지합니다.

**추가 팁**:
- **상태 기계 설계**:
  ```java
  @Entity
  public class Order {
      @Id
      private Long id;
      private String status; // PENDING, PAYMENT_SUCCESS, PAYMENT_FAILED
      private int stockDeducted;

      public void updatePaymentStatus(String newStatus) {
          this.status = newStatus;
      }

      public void rollbackStock(int quantity) {
          this.stockDeducted -= quantity;
      }
  }
  ```
- **사가 패턴**: 마이크로서비스 환경에서는 사가 패턴을 사용하여 트랜잭션을 분리하고, 보상 트랜잭션으로 실패를 처리합니다.
  ```java
  public class OrderSaga {
      @Transactional
      public void handlePaymentFailure(Long orderId) {
          Order order = orderRepository.findById(orderId);
          stockService.restoreStock(order.getProductId(), order.getStockDeducted());
          order.updatePaymentStatus("PAYMENT_FAILED");
          orderRepository.save(order);
      }
  }
  ```
- **이벤트 소싱**: 각 트랜잭션의 이벤트를 기록하고, 이벤트 재생을 통해 정합성을 복원할 수 있습니다. 예: Kafka로 `OrderCreatedEvent`, `PaymentFailedEvent`를 발행.
- **모니터링**: 트랜잭션 실패율과 롤백 빈도를 모니터링하여 설계의 안정성을 검증하세요.

**보완 설명**: 트랜잭션 분리는 정합성과 성능 간 균형을 맞추는 핵심 전략입니다. 사가 패턴이나 이벤트 소싱은 분산 시스템에서 특히 유용하며, 상태 기반 설계는 단일 DB 환경에서도 효과적입니다.

---

## 결론

Q&A 세션에서 다룬 질문들은 전자상거래 서비스의 동시성 문제와 테스트 전략의 핵심적인 측면을 다루었습니다. 데드락 예방, 외부 API 연동, 동시성 테스트, 낙관적 락, 테이블 관리, 트랜잭션 분리 등 실무에서 자주 접하는 문제에 대한 해결 방안을 학습했습니다. 추가 팁과 예제 코드를 통해 이론을 실무에 적용하는 방법을 구체화하였으며, 모니터링과 최적화 전략을 강조하여 안정적인 시스템 설계를 지원합니다.

**코치 팁**: 동시성 문제는 실무에서 자주 발생하므로, 다양한 시나리오를 테스트하고 문서화하세요. 이는 포트폴리오와 업무 성과를 정리하는 데도 큰 도움이 됩니다.

**보완 설명**: 실무에서는 동시성 문제를 사전에 예방하고, 발생 시 신속히 대응할 수 있는 모니터링과 로깅 체계가 필수입니다. 특히, 분산 환경에서는 Redis, Kafka 같은 도구를 활용한 설계가 점점 더 중요해지고 있습니다.
