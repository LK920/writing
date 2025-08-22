# 낙관적 락과 @Version 어노테이션 학습 자료

## 1. 낙관적 락(Optimistic Locking)이란?

낙관적 락은 데이터베이스 트랜잭션에서 동시성 문제를 해결하기 위한 방법 중 하나입니다. 동시성 문제란 여러 사용자가 동시에 같은 데이터를 수정하려고 할 때 발생할 수 있는 데이터 불일치 문제를 말합니다. 낙관적 락은 "대부분의 경우 충돌이 발생하지 않을 것"이라는 낙관적인 가정 하에 동작하며, 데이터 충돌이 실제로 발생할 때만 이를 처리합니다.

### 특징
- **충돌이 드물다고 가정**: 트랜잭션이 데이터를 수정할 때, 다른 트랜잭션이 동시에 같은 데이터를 수정할 가능성이 낮다고 가정합니다.
- **락을 걸지 않음**: 데이터에 직접적인 락(예: 비관적 락)을 걸지 않고, 버전 정보를 통해 충돌을 감지합니다.
- **버전 관리**: 데이터의 버전 정보를 기록하여, 트랜잭션이 데이터를 수정하기 전에 버전을 확인합니다. 만약 버전이 일치하지 않으면 충돌로 간주하고 예외를 발생시킵니다.
- **성능 이점**: 락을 걸지 않으므로 비관적 락(Pessimistic Locking)에 비해 성능이 좋고, 데이터베이스 자원을 덜 소모합니다.
- **적합한 상황**: 데이터 충돌이 드문 환경(예: 읽기 작업이 많고 쓰기 작업이 적은 경우)에 적합합니다.

### 낙관적 락의 동작 원리
1. 엔티티를 읽을 때, 해당 엔티티의 버전 정보를 함께 가져옵니다.
2. 엔티티를 수정하고 저장하려고 할 때, 저장 전 버전 정보를 확인합니다.
3. 버전이 일치하면 데이터를 저장하고 버전을 증가시킵니다.
4. 버전이 일치하지 않으면(다른 트랜잭션이 먼저 수정한 경우) `OptimisticLockException` 같은 예외가 발생하고, 트랜잭션은 롤백됩니다.

---

## 2. JPA에서 낙관적 락 구현: `@Version` 어노테이션

Java Persistence API(JPA)에서는 낙관적 락을 쉽게 구현할 수 있도록 `@Version` 어노테이션을 제공합니다. 이 어노테이션은 엔티티 클래스에 버전 관리용 필드를 지정하는 데 사용됩니다.

### `@Version` 어노테이션의 역할
- 엔티티에 버전 관리 필드를 추가하여, 엔티티가 수정될 때마다 버전 값을 증가시킵니다.
- JPA가 자동으로 버전 값을 관리하며, 트랜잭션 커밋 시 버전 충돌 여부를 확인합니다.
- 버전 필드는 일반적으로 숫자(`int`, `Long`) 또는 타임스탬프(`Timestamp`)로 정의됩니다.

### `@Version` 사용 예제
다음은 `@Version`을 사용한 간단한 JPA 엔티티 예제입니다.

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Version;

@Entity
public class Product {
    @Id
    private Long id;
    
    private String name;
    private double price;
    
    @Version
    private int version; // 버전 관리 필드
    
    // 기본 생성자
    public Product() {}
    
    // Getter와 Setter
    public Long getId() {
        return id;
    }
    
    public void setId(Long id) {
        this.id = id;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public double getPrice() {
        return price;
    }
    
    public void setPrice(double price) {
        this.price = price;
    }
    
    public int getVersion() {
        return version;
    }
}
```

#### 코드 설명
- `@Version`이 붙은 `version` 필드는 JPA가 자동으로 관리합니다.
- 엔티티가 처음 저장될 때는 `version` 값이 0으로 설정됩니다.
- 엔티티가 수정될 때마다 `version` 값이 1씩 증가합니다.
- 다른 트랜잭션이 동일한 엔티티를 수정하여 `version` 값이 변경되었다면, JPA는 이를 감지하고 `OptimisticLockException`을 발생시킵니다.

---

## 3. 낙관적 락의 동작 시나리오

아래는 두 트랜잭션이 같은 `Product` 엔티티를 수정하려고 할 때의 시나리오입니다.

1. **초기 상태**:
   - `Product` 엔티티 (ID: 1, name: "Laptop", price: 1000, version: 0)

2. **트랜잭션 A**:
   - `Product` 엔티티를 읽음 (version: 0).
   - `price`를 1200으로 변경.
   - 커밋 시 JPA가 `version`이 0인지 확인하고, 맞다면 데이터를 업데이트하고 `version`을 1로 증가.

3. **트랜잭션 B** (동시에 실행):
   - `Product` 엔티티를 읽음 (version: 0).
   - `price`를 1500으로 변경.
   - 커밋 시 JPA가 `version`을 확인. 하지만 트랜잭션 A가 이미 `version`을 1로 업데이트했으므로, 트랜잭션 B는 `OptimisticLockException`을 발생시키고 롤백됨.

---

## 4. 낙관적 락의 장점과 단점

### 장점
- **성능 최적화**: 락을 걸지 않으므로 데이터베이스 자원을 효율적으로 사용.
- **간단한 구현**: `@Version` 어노테이션으로 쉽게 적용 가능.
- **확장성**: 다수의 읽기 작업이 많은 환경에서 확장성이 좋음.

### 단점
- **충돌 처리 필요**: 충돌이 발생하면 `OptimisticLockException`을 처리해야 함.
- **복잡한 비즈니스 로직**: 충돌이 자주 발생하는 환경에서는 비즈니스 로직이 복잡해질 수 있음.
- **재시도 로직 필요**: 충돌 발생 시 트랜잭션을 재시도하는 로직을 추가로 구현해야 할 수 있음.

---

## 5. 예외 처리와 재시도 로직

`OptimisticLockException`이 발생하면, 애플리케이션은 이를 적절히 처리해야 합니다. 아래는 예외 처리와 재시도를 포함한 서비스 레이어 코드 예제입니다.

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.OptimisticLockException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProductService {
    private final EntityManager entityManager;
    
    public ProductService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }
    
    @Transactional
    public void updateProductPrice(Long productId, double newPrice) {
        int maxRetries = 3;
        int retryCount = 0;
        
        while (retryCount < maxRetries) {
            try {
                Product product = entityManager.find(Product.class, productId);
                product.setPrice(newPrice);
                entityManager.merge(product);
                return; // 성공 시 종료
            } catch (OptimisticLockException e) {
                retryCount++;
                if (retryCount >= maxRetries) {
                    throw new RuntimeException("최대 재시도 횟수 초과", e);
                }
                // 잠시 대기 후 재시도
                try {
                    Thread.sleep(100);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }
}
```

#### 코드 설명
- `OptimisticLockException`이 발생하면 최대 3번까지 재시도합니다.
- 재시도 전 약간의 대기 시간을 두어 충돌 가능성을 줄입니다.
- 최대 재시도 횟수를 초과하면 예외를 던져 호출자에게 알립니다.

---

## 6. 낙관적 락 설정 방법

JPA에서 낙관적 락을 설정하는 방법은 두 가지가 있습니다.

### 1) `@Version` 어노테이션 사용
앞서 설명한 대로, 엔티티에 `@Version`을 붙여 자동으로 낙관적 락을 활성화합니다.

### 2) 명시적 락 설정
JPA의 `EntityManager`나 `Query` 객체를 통해 낙관적 락을 명시적으로 설정할 수 있습니다.

```java
Product product = entityManager.find(Product.class, productId, LockModeType.OPTIMISTIC);
```

- `LockModeType.OPTIMISTIC`: 트랜잭션 종료 시 버전 체크를 수행.
- `LockModeType.OPTIMISTIC_FORCE_INCREMENT`: 버전 체크와 함께 강제로 버전을 증가시킴.

---

## 7. 실제 사용 시 고려사항

1. **버전 필드의 타입 선택**:
   - `int` 또는 `Long`: 단순하고 증가 방식이 명확.
   - `Timestamp`: 시간 기반 충돌 감지에 유용하지만, 시간 동기화 문제가 발생할 수 있음.

2. **충돌 빈도 분석**:
   - 데이터 충돌이 자주 발생한다면 비관적 락을 고려해야 할 수 있음.

3. **비즈니스 로직과의 조화**:
   - 충돌 발생 시 사용자에게 적절한 피드백(예: "다른 사용자가 수정했습니다. 다시 시도하세요")을 제공해야 함.

4. **테스트**:
   - 동시성 테스트를 통해 낙관적 락이 올바르게 동작하는지 확인해야 함.

---

## 8. 낙관적 락 vs 비관적 락

| **구분**            | **낙관적 락**                              | **비관적 락**                              |
|---------------------|--------------------------------------------|--------------------------------------------|
| **락 방식**         | 버전 정보를 통해 충돌 감지                 | 데이터에 직접 락을 걸어 접근 제한          |
| **성능**           | 락을 걸지 않아 성능이 좋음                 | 락으로 인해 성능 저하 가능                 |
| **충돌 빈도**       | 충돌이 드문 환경에 적합                   | 충돌이 잦은 환경에 적합                   |
| **복잡도**         | 예외 처리와 재시도 로직 필요               | 락 관리로 인한 복잡도 증가                |
| **JPA 지원**       | `@Version`, `LockModeType.OPTIMISTIC`      | `LockModeType.PESSIMISTIC_READ/WRITE`      |

---

## 9. 결론

낙관적 락은 동시성 제어가 필요한 환경에서 성능과 확장성을 높이는 좋은 방법입니다. JPA의 `@Version` 어노테이션을 사용하면 이를 쉽게 구현할 수 있으며, 충돌이 드문 환경에서 특히 효과적입니다. 하지만 충돌이 발생할 경우를 대비해 적절한 예외 처리와 재시도 로직을 설계하는 것이 중요합니다.

이 학습 자료를 통해 낙관적 락과 `@Version`의 개념, 구현 방법, 그리고 실제 사용 시 고려사항을 이해할 수 있기를 바랍니다.