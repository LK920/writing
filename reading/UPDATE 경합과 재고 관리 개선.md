
## 1. UPDATE 경합이란?
UPDATE 경합은 데이터베이스에서 **여러 트랜잭션이 동시에 동일한 레코드를 수정하려고 경쟁**하면서 발생하는 문제입니다. 특히 다수의 사용자가 특정 상품의 재고(`stock`, `reserved_stock`)를 동시에 업데이트하려고 할 때, 데이터베이스가 해당 레코드를 락(Lock)하거나 충돌이 발생하여 성능 저하, 데이터 무결성 문제, 또는 데드락(Deadlock)이 생길 수 있습니다.

### UPDATE 경합이 주로 발생하는 이유

- **로우 락(Row Lock)**: UPDATE 연산은 특정 레코드의 데이터를 변경하므로, 데이터베이스는 해당 레코드에 **로우 락(Row Lock)**을 걸게 됩니다. 이는 다른 트랜잭션이 동일한 레코드를 동시에 수정하는 것을 방지하여 데이터 무결성을 보장하기 위함입니다.
    
- **동시 수정의 빈번함**: 재고 감소, 잔액 차감 등과 같이 여러 사용자가 동시에 동일한 데이터를 변경하려는 시나리오에서 UPDATE 요청이 빈번하게 발생합니다.
    
- **대기 시간 발생**: 하나의 트랜잭션이 락을 획득하면, 동일한 레코드를 수정하려는 다른 트랜잭션들은 해당 락이 해제될 때까지 **대기(waiting)**하게 됩니다. 이 대기 시간이 길어지면 시스템의 **성능 저하**로 이어집니다.
    
- **데이터 무결성 위험**: 동시 업데이트 시 연산 순서에 따라 데이터가 잘못 계산될 위험이 있습니다. 예를 들어, `stock = stock - 2`와 `stock = stock - 3`이 동시에 실행될 때 최종 재고 값이 예상과 다르게 나올 수 있습니다.
    
- **데드락 위험**: 복잡한 트랜잭션에서 여러 락이 얽히는 경우, **데드락(Deadlock)**이 발생할 위험도 증가합니다.
    
### 조회(Read)는 경합에 큰 영향을 주지 않는 이유

- **공유 락(Shared Lock)**: 대부분의 데이터베이스 시스템에서 SELECT(조회) 연산은 **공유 락(Shared Lock)**을 사용합니다. 공유 락은 다른 트랜잭션이 동시에 해당 데이터를 읽는 것을 허용합니다. 즉, 여러 사용자가 동시에 같은 데이터를 조회하더라도 서로를 기다리게 하지 않습니다.
    
- **읽기 일관성(Read Consistency)**: 데이터베이스는 읽기 일관성을 보장하기 위해 특정 시점의 스냅샷을 제공하는 등의 메커니즘을 사용합니다. 이는 조회 작업이 업데이트 작업에 의해 차단되지 않도록 합니다.
    

### 쓰기(Insert)는 경합에 상대적으로 덜 영향을 주는 이유

- **락 범위 최소화**: INSERT 연산은 주로 새로운 레코드를 데이터베이스에 추가하는 것이므로, 기존 레코드에 대한 락이 필요 없거나, 필요하더라도 그 범위가 매우 제한적입니다.
    
- **쓰기 전용 테이블의 이점**: `stock_histories`나 `balance_transactions`와 같이 이력을 기록하기 위한 **쓰기 전용(Insert Only)** 테이블의 경우, 새로운 레코드가 계속 추가되더라도 레코드 간에 충돌이 발생할 가능성이 매우 낮아 경합이 최소화됩니다. 이는 새로운 레코드를 삽입하는 작업은 이미 존재하는 다른 레코드를 수정하는 것이 아니기 때문입니다.
    

### 결론

따라서, 데이터베이스 트랜잭션에서 동시성 문제와 성능 저하의 주된 원인은 **UPDATE 경합**이며, 이를 해결하기 위해 테이블 분리, 낙관적 락(`version` 컬럼), 그리고 쓰기 전용 이력 테이블(`stock_histories`, `balance_transactions`) 도입과 같은 전략들이 제안되는 것입니다.


### 1.1. 현재 문제: `Product` 테이블
현재 `Product` 테이블은 상품 정보와 재고 정보를 함께 관리합니다:
```sql
CREATE TABLE product (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10, 2),
    stock INT NOT NULL DEFAULT 0,         -- 전체 재고
    reserved_stock INT NOT NULL DEFAULT 0, -- 예약된 재고
    created_at TIMESTAMP
);
```
- **문제점**:
  - **전체 레코드 락**: `stock` 또는 `reserved_stock`을 업데이트할 때, `product` 테이블의 전체 레코드(예: `name`, `price` 포함)를 락해야 함.
    - 예: `UPDATE product SET stock = stock - 2 WHERE id = 1` → 불필요한 필드까지 락.
  - **경합 발생**: 여러 사용자가 동시에 같은 상품(예: `id = 1`)의 재고를 수정하려고 하면, 로우 락(Row Lock)으로 인해 트랜잭션이 대기 → **성능 저하**.
  - **데이터 무결성 위험**: 동시 업데이트 시 `stock`이나 `reserved_stock` 값이 잘못 계산될 가능성.
    - 예: `stock = 100`에서 사용자 A(2개 감소), B(3개 감소)가 동시에 요청 → `stock`이 95가 되어야 하지만, 경합으로 98이나 97로 잘못 저장 가능.
  - **이력 부족**: 재고 변경 원인(주문, 취소, 재입고 등)을 추적할 수 없음.

### 1.2. 경합의 구체적 예
- **상황**: 사용자 A와 B가 동시에 상품(`id = 1`, `stock = 100`)을 구매.
  - 사용자 A: `UPDATE product SET stock = stock - 2, reserved_stock = reserved_stock + 2 WHERE id = 1`.
  - 사용자 B: `UPDATE product SET stock = stock - 3, reserved_stock = reserved_stock + 3 WHERE id = 1`.
- **문제**:
  - DB는 `id = 1` 레코드를 락 → 사용자 B는 A의 트랜잭션이 끝날 때까지 대기.
  - 동시 요청이 많아질수록 대기 시간 증가 → 성능 병목.
  - 비관적 락(Pessimistic Locking) 사용 시 데드락 위험.

## 2. 개선안: 테이블 분리
재고 관리(`stock`, `reserved_stock`)를 `product` 테이블에서 분리하여 `product_stocks`와 `stock_histories` 테이블로 관리합니다.

### 2.1. `product_stocks` 테이블
```sql
CREATE TABLE product_stocks (
    product_id BIGINT PRIMARY KEY,
    available_stock INT NOT NULL DEFAULT 0,
    reserved_stock INT NOT NULL DEFAULT 0,
    version BIGINT NOT NULL DEFAULT 0,  -- 낙관적 락
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
- **목적**:
  - 상품 정보(`name`, `price`)와 재고 정보(`available_stock`, `reserved_stock`)를 분리 → **관심사 분리**.
  - 재고 업데이트 시 `product_stocks`만 락 → 락 범위 최소화.
- **낙관적 락 (`version`)**:
  - 동시 업데이트 충돌 감지: 트랜잭션이 레코드를 읽을 때 `version` 확인, 업데이트 시 `version`이 동일해야 성공.
  - 예:
    ```sql
    -- 트랜잭션 1: 읽기
    SELECT available_stock, version FROM product_stocks WHERE product_id = 1; -- version = 1
    -- 트랜잭션 2가 먼저 업데이트: version = 2로 증가
    -- 트랜잭션 1 업데이트 시도
    UPDATE product_stocks SET available_stock = available_stock - 2, version = version + 1
    WHERE product_id = 1 AND version = 1; -- 실패 (version 불일치)
    ```
  - **효과**: 비관적 락보다 락 충돌 감소, 동시성 향상.

### 2.2. `stock_histories` 테이블
```sql
CREATE TABLE stock_histories (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT,
    change_type VARCHAR(20), -- RESERVE, CONFIRM, CANCEL, RESTOCK
    quantity INT,
    before_stock INT,
    after_stock INT,
    order_id BIGINT,
    created_at TIMESTAMP
);
```
- **목적**:
  - 재고 변경 이력(예: 예약, 확정, 취소, 재입고)을 기록 → 감사 로그(Audit Log).
  - 예: 주문 `order_id=123`으로 2개 예약:
    ```sql
    INSERT INTO stock_histories (product_id, change_type, quantity, before_stock, after_stock, order_id, created_at)
    VALUES (1, 'RESERVE', 2, 100, 98, 123, NOW());
    ```
- **효과**:
  - **경합 없음**: `stock_histories`는 쓰기 전용(Insert Only) → 업데이트 경합 발생 안 함.
  - **추적 가능**: 재고 변동 원인(주문, 취소 등) 추적 및 디버깅 용이.

## 3. 개선안의 이점
| 항목                  | 현재 (`product` 테이블)                          | 개선안 (`product_stocks` + `stock_histories`) |
|-----------------------|-----------------------------------------------|---------------------------------------------|
| **재고 관리**         | `stock`, `reserved_stock` 포함                  | `product_stocks`로 분리                     |
| **락 범위**          | 전체 레코드 락 (`name`, `price` 포함)           | `product_stocks`만 락, 범위 최소화          |
| **경합 문제**        | 로우 락으로 성능 저하                          | 낙관적 락으로 동시성 향상                  |
| **이력 관리**        | 변경 이력 없음, 원인 추적 어려움               | `stock_histories`로 상세 이력 기록          |
| **성능**            | 동시 요청 시 병목                              | 락 범위 축소, 동시 처리 효율 증가           |

## 4. 구현 예시
### 4.1. 현재 코드 (문제 발생)
```java
@Service
public class ProductService {
    @Autowired
    private ProductRepository productRepository;

    @Transactional
    public void reserveStock(Long productId, int quantity, Long orderId) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new RuntimeException("Product not found"));
        // 전체 레코드 락
        if (product.getStock() < quantity) {
            throw new InsufficientStockException("재고 부족");
        }
        product.setStock(product.getStock() - quantity);
        product.setReservedStock(product.getReservedStock() + quantity);
        productRepository.save(product);
        // 이력 기록 없음
    }
}
```
- **문제**: 동시 요청 시 `product(id=1)` 레코드에 로우 락 → 경합, 성능 저하.

### 4.2. 개선 코드
```java
@Service
public class ProductService {
    @Autowired
    private ProductStockRepository productStockRepository;
    @Autowired
    private StockHistoryRepository stockHistoryRepository;

    @Transactional
    public void reserveStock(Long productId, int quantity, Long orderId) {
        // product_stocks만 락
        ProductStock stock = productStockRepository.findByProductId(productId)
            .orElseThrow(() -> new RuntimeException("Stock not found"));
        
        if (stock.getAvailableStock() < quantity) {
            throw new InsufficientStockException("재고 부족");
        }

        // 낙관적 락 적용
        int updated = productStockRepository.updateStock(
            productId,
            stock.getAvailableStock() - quantity,
            stock.getReservedStock() + quantity,
            stock.getVersion()
        );
        if (updated == 0) {
            throw new OptimisticLockException("동시 업데이트 충돌");
        }

        // 이력 기록
        StockHistory history = new StockHistory();
        history.setProductId(productId);
        history.setChangeType("RESERVE");
        history.setQuantity(quantity);
        history.setBeforeStock(stock.getAvailableStock());
        history.setAfterStock(stock.getAvailableStock() - quantity);
        history.setOrderId(orderId);
        stockHistoryRepository.save(history);
    }
}

@Repository
public interface ProductStockRepository extends JpaRepository<ProductStock, Long> {
    @Query("UPDATE ProductStock ps SET ps.availableStock = :newStock, ps.reservedStock = :newReserved, ps.version = ps.version + 1 WHERE ps.productId = :productId AND ps.version = :version")
    @Modifying
    int updateStock(@Param("productId") Long productId, @Param("newStock") int newStock, @Param("newReserved") int newReserved, @Param("version") long version);
}
```
- **설명**:
  - `product_stocks`만 업데이트 → 락 범위 최소화.
  - `version`으로 낙관적 락 적용 → 동시성 향상.
  - `stock_histories`에 이력 기록 → 감사 로그 제공.

## 5. 데이터 마이그레이션
기존 `product` 테이블에서 `product_stocks`로 데이터 이동:
```sql
INSERT INTO product_stocks (product_id, available_stock, reserved_stock, version)
SELECT id, stock, reserved_stock, 0 FROM product;
ALTER TABLE product DROP COLUMN stock, DROP COLUMN reserved_stock;
```

## 6. 주의사항
- **낙관적 락 관리**: 동시 업데이트 충돌 시 재시도 로직 추가(예: 재시도 3회).
- **이력 기록 일관성**: `stock_histories`에 모든 재고 변경 기록(예: `RESERVE`, `CANCEL`) 필수.
- **성능 테스트**: 고부하 환경에서 낙관적 락 성능 점검.
- **스키마 설계**: `product_stocks`의 `product_id`는 `product` 테이블과 외래 키(Foreign Key)로 연결 권장.

## 7. 결론
UPDATE 경합은 여러 트랜잭션이 동시에 `product` 테이블의 같은 레코드를 수정하려고 경쟁하면서 발생합니다. `product_stocks` 테이블로 재고를 분리하면 락 범위가 줄어들고, 낙관적 락(`version`)으로 동시성을 향상시킵니다. `stock_histories`는 재고 변경 이력을 기록해 디버깅과 분석을 돕습니다. 이 개선안은 동시 요청 처리 성능을 높이고, 데이터 무결성과 추적 가능성을 강화합니다.