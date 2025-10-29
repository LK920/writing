# 데드락 방지: Safe Sequence 학습 자료 (전자상거래 재고 경합 예시)

이 자료는 데드락(Deadlock)의 원인과 방지 전략, 특히 Safe Sequence(안전한 순서)를 중심으로 전자상거래(e-commerce) 재고 경합 시나리오에 적용하는 방법을 다룹니다. Safe Sequence는 락 획득 순서를 표준화하여 데드락의 순환 대기 조건을 제거하는 강력한 방법입니다. 실습 과제와 추가 리소스를 포함하여 실무 적용 가능성을 높였습니다.

---

## 1. 데드락 이해
- **정의**: 여러 트랜잭션이 서로가 점유한 자원을 기다리며 무한 대기 상태에 빠지는 상황.
- **데드락 발생 조건**:
  - **상호 배제**: 자원은 한 번에 한 트랜잭션만 점유 가능.
  - **점유 대기**: 자원을 점유한 상태에서 추가 자원 요청.
  - **비선점**: 자원을 강제로 뺏을 수 없음.
  - **순환 대기**: 트랜잭션들이 서로의 자원을 기다리며 순환 형성.
- **전자상거래 예시**:
  ```sql
  -- 트랜잭션 1
  START TRANSACTION;
  SELECT * FROM product WHERE id=100 FOR UPDATE; -- product 락
  SELECT * FROM orders WHERE order_id=200 FOR UPDATE; -- order 락
  COMMIT;
  -- 트랜잭션 2
  START TRANSACTION;
  SELECT * FROM orders WHERE order_id=200 FOR UPDATE; -- order �락
  SELECT * FROM product WHERE id=100 FOR UPDATE; -- product 락
  COMMIT;
  ```
  - **결과**: 트랜잭션 1이 `product` 락을, 트랜잭션 2가 `order` 락을 점유하며 서로 대기 → 데드락.

---

## 2. Safe Sequence란?
- **정의**: 자원(락)을 요청하는 순서를 모든 트랜잭션에서 일관되게 정의하여 순환 대기 조건을 제거하는 전략.
- **원리**:
  - 모든 트랜잭션이 동일한 순서로 자원을 요청하도록 강제.
  - 예: `product` → `order` 순서로 락 획득.
  - 순환 대기가 불가능해 데드락 방지.
- **전자상거래 적용**:
  ```sql
  -- 모든 트랜잭션에서 동일 순서
  START TRANSACTION;
  SELECT * FROM product WHERE id=100 FOR UPDATE;
  SELECT * FROM orders WHERE order_id=200 FOR UPDATE;
  UPDATE product SET stock=stock-1 WHERE id=100;
  INSERT INTO orders(order_id, product_id, quantity) VALUES(200, 100, 1);
  COMMIT;
  ```
- **장점**:
  - 데드락 원천 방지.
  - 구현 간단, 코드 리뷰로 준수 확인 가능.
- **단점**:
  - 락 순서 표준화로 인한 설계 제약.
  - 복잡한 시스템에서 최적 순서 정의 어려움.

---

## 3. Safe Sequence 구현 가이드
- **단계**:
  1. **자원 식별**: 트랜잭션에서 사용하는 테이블(`product`, `order`) 및 락 대상 파악.
  2. **순서 정의**: 자원에 고유한 순서 부여 (예: 테이블 이름 알파벳순, `product` → `order`).
  3. **코드 적용**: 모든 트랜잭션이 정의된 순서 준수.
  4. **검증**: 코드 리뷰 및 테스트로 순서 준수 확인.
- **예제** (Java/Spring):
  ```java
  @Transactional
  public void processOrder(Long productId, Long orderId) {
      // 항상 product → order 순서로 락 획득
      entityManager.createQuery("SELECT p FROM Product p WHERE p.id = :id FOR UPDATE")
          .setParameter("id", productId)
          .getSingleResult();
      entityManager.createQuery("SELECT o FROM Order o WHERE o.id = :id FOR UPDATE")
          .setParameter("id", orderId)
          .getSingleResult();
      // 비즈니스 로직: 재고 차감 및 주문 생성
      entityManager.createQuery("UPDATE Product p SET p.stock = p.stock - 1 WHERE p.id = :id")
          .setParameter("id", productId)
          .executeUpdate();
      entityManager.persist(new Order(orderId, productId, 1));
  }
  ```
- **실무 팁**:
  - 테이블 이름을 알파벳순으로 락 획득 (예: `order` → `product`).
  - 자원 의존성을 UML 다이어그램으로 시각화해 순서 정의.
  - 코드 주석으로 락 순서 명시:
    ```java
    // 락 순서: product → order
    ```

---

## 4. Safe Sequence와 다른 데드락 방지 전략 비교
- **락 타임아웃**:
  - **특징**: 락 대기 시간 제한 후 실패 처리 (예: MySQL `innodb_lock_wait_timeout`).
  - **예제**:
    ```sql
    SET SESSION innodb_lock_wait_timeout = 5;
    ```
  - **장점**: 빠른 실패로 서비스 다운타임 감소.
  - **단점**: 재시도 로직 필요, 고경쟁 환경에서 빈번한 타임아웃.
  - **Safe Sequence와 비교**: Safe Sequence는 데드락을 예방, 타임아웃은 발생 후 처리.
- **낙관적 락**:
  - **특징**: 락 없이 버전 확인으로 충돌 감지.
  - **예제**:
    ```java
    @Entity
    public class Product {
        @Id private Long id;
        @Version private Long version;
        private int stock;
    }
    @Transactional
    public void decreaseStock(Long productId) {
        Product product = entityManager.find(Product.class, productId);
        if (product.getStock() <= 0) {
            throw new IllegalStateException("재고 부족");
        }
        product.setStock(product.getStock() - 1);
    }
    ```
  - **장점**: 락 기반 데드락 없음.
  - **단점**: 충돌 빈발 시 재시도 오버헤드.
  - **Safe Sequence와 비교**: 낙관적 락은 락 자체를 피하지만, Safe Sequence는 락 사용 시 안전성 보장.
- **트랜잭션 최소화**:
  - **특징**: 트랜잭션 범위 축소로 락 유지 시간 단축.
  - **예제**:
    ```sql
    START TRANSACTION;
    UPDATE product SET stock=stock-1 WHERE id=100 AND stock>0 FOR UPDATE;
    COMMIT;
    ```
  - **장점**: 락 경합 감소.
  - **단점**: 정합성 유지를 위한 추가 설계 필요.
  - **Safe Sequence와 비교**: 트랜잭션 최소화는 간접적 방지, Safe Sequence는 직접적 예방.
- **리소스 파티셔닝**:
  - **특징**: 데이터를 분할해 동시 접근 감소.
  - **예제**:
    ```sql
    CREATE TABLE product (
        id BIGINT,
        category_id BIGINT,
        stock INT
    ) PARTITION BY HASH(category_id) PARTITIONS 10;
    ```
  - **장점**: 락 경합 분산.
  - **단점**: 스키마 복잡성 증가.
  - **Safe Sequence와 비교**: 파티셔닝은 경합 자체를 줄이고, Safe Sequence는 경합 발생 시 데드락 방지.

---

## 5. 실습 과제
### 과제 1: Safe Sequence 구현
- **목표**: 전자상거래 시스템에서 Safe Sequence를 적용해 재고 경합 시 데드락 방지.
- **활동**:
  1. 두 테이블(`product`, `order`)에 대한 트랜잭션 정의.
  2. 락 순서를 `product` → `order`로 표준화.
  3. 아래 코드 구현:
     ```sql
     START TRANSACTION;
     SELECT * FROM product WHERE id=100 FOR UPDATE;
     SELECT * FROM orders WHERE order_id=200 FOR UPDATE;
     UPDATE product SET stock=stock-1 WHERE id=100;
     INSERT INTO orders(order_id, product_id, quantity) VALUES(200, 100, 1);
     COMMIT;
     ```
  4. JMeter로 100개 동시 요청 테스트, 데드락 발생 여부 확인.
- **결과 보고서**:
  ```markdown
  # Safe Sequence 테스트 보고서
  ## 배경
  전자상거래 시스템에서 동시 재고 차감 시 데드락 발생.
  ## 해결
  락 순서 `product` → `order` 표준화.
  ## 실험
  JMeter로 100개 요청 → 데드락 0건.
  ## 결론
  Safe Sequence로 데드락 완전 방지.
  ```

### 과제 2: 데드락 재현 및 Safe Sequence 적용
- **목표**: 데드락 발생 시나리오 재현 후 Safe Sequence로 해결.
- **활동**:
  1. 두 트랜잭션에서 락 순서를 반대로 설정해 데드락 재현:
     ```sql
     -- 트랜잭션 1
     START TRANSACTION;
     SELECT * FROM product WHERE id=100 FOR UPDATE;
     -- 대기
     SELECT * FROM orders WHERE order_id=200 FOR UPDATE;
     COMMIT;
     -- 트랜잭션 2
     START TRANSACTION;
     SELECT * FROM orders WHERE order_id=200 FOR UPDATE;
     -- 대기
     SELECT * FROM product WHERE id=100 FOR UPDATE;
     COMMIT;
     ```
  2. 락 순서를 `product` → `order`로 통일:
     ```sql
     START TRANSACTION;
     SELECT * FROM product WHERE id=100 FOR UPDATE;
     SELECT * FROM orders WHERE order_id=200 FOR UPDATE;
     COMMIT;
     ```
  3. MySQL 로그로 데드락 감지 확인:
     ```sql
     SHOW ENGINE INNODB STATUS;
     ```
  4. 결과 비교 및 보고서 작성.

---

## 6. 추가 학습 리소스
- **문서**:
  - MySQL 공식 문서: [InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
  - PostgreSQL 공식 문서: [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
  - OS Banker’s Algorithm: [https://www.geeksforgeeks.org/bankers-algorithm-in-operating-system/](https://www.geeksforgeeks.org/bankers-algorithm-in-operating-system/)
- **도구**:
  - JMeter: 동시 요청 테스트.
  - Testcontainers: DB 통합 테스트 환경.
- **블로그**:
  - [데드락과 Safe Sequence](https://www.baeldung.com/cs/deadlock-prevention)
  - [DB 락 관리](https://vladmihalcea.com/database-locking/)

---

## 7. 평가 기준
- **Safe Sequence 구현**:
  - 락 순서 표준화 정확성.
  - 데드락 방지 효과 입증 (테스트 결과).
- **데드락 재현 및 해결**:
  - 데드락 재현 성공.
  - Safe Sequence 적용 후 데드락 제거 확인.
- **보고서**:
  - 문제 분석, 해결 과정, 실험 결과의 명확성.
  - 코드와 테스트 결과 포함 여부.