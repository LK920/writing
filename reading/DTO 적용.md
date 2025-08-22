# DTO(Data Transfer Object) 학습 가이드

이 문서는 DTO의 정의, 역할, 레이어별 사용 여부와 판단 근거, 클린 아키텍처와 헥사고날 아키텍처에서의 사용법을 상세히 설명한다. 특히, "계층 간 혼용"의 의미와 컨트롤러 외 계층에서의 DTO/엔티티 사용 기준을 명확히 다룬다. 초보자가 DTO와 엔티티의 사용 원칙을 이해하고, 이커머스 플랫폼에서 적절히 적용할 수 있도록 비유, 예시, 설계 원칙, 코드 개선 가이드를 포함한다.

## 목차

- [1. DTO란 무엇인가?](#1-dto란-무엇인가)
- [2. DTO의 역할](#2-dto의-역할)
- [3. 비유로 이해하는 DTO](#3-비유로-이해하는-dto)
- [4. 레이어별 DTO와 엔티티 사용](#4-레이어별-dto와-엔티티-사용)
- [5. 클린 아키텍처와 헥사고날 아키텍처에서의 DTO 사용](#5-클린-아키텍처와-헥사고날-아키텍처에서의-dto-사용)
- [6. 이커머스 플랫폼에서의 DTO 예시](#6-이커머스-플랫폼에서의-dto-예시)
- [7. DTO 설계 원칙](#7-dto-설계-원칙)
- [8. DTO 사용 검토 및 개선](#8-dto-사용-검토-및-개선)
- [9. 참고 자료](#9-참고-자료)

## 1. DTO란 무엇인가?

**정의**: DTO(Data Transfer Object)는 계층 간 데이터를 전달하기 위해 설계된 객체로, 비즈니스 로직을 포함하지 않고 데이터를 캡슐화한다.

**핵심 특징**:
- **데이터 홀더**: 필드, getter, setter만 포함하며, 비즈니스 로직은 없다.
- **계층 간 전달**: 컨트롤러, 유스케이스, 레포지토리 어댑터 간 데이터를 주고받는다.
- **독립성**: 도메인 객체(엔티티)와 분리되어 특정 계층의 요구사항에 맞게 설계된다.

**DTO vs 엔티티**:
- **DTO**: 계층 간 데이터 전달용. 외부 인터페이스(예: HTTP 요청/응답)와 형식을 맞춘다.
- **엔티티**: 비즈니스 로직과 상태를 포함한 도메인 객체. 내부 로직(예: 상태 전이, 계산)을 캡슐화한다.

## 2. DTO의 역할

DTO는 소프트웨어 계층 구조에서 데이터를 효율적이고 안전하게 전달한다. 주요 역할은 다음과 같다:

1. **데이터 캡슐화**: 필요한 데이터만 전달하여 불필요한 데이터 노출 방지.
2. **형식 변환**: 외부 요청(예: JSON)과 도메인 객체 간 데이터 형식 변환.
3. **보안 강화**: 민감 데이터(예: 비밀번호)를 제외하거나 가공.
4. **유효성 검증**: 입력 데이터의 형식을 검증하여 오류 감지.
5. **의존성 격리**: 계층 간 독립성을 유지하여 변경 영향 최소화.

## 3. 비유로 이해하는 DTO

DTO를 **택배 서비스**로 비유하면 다음과 같다:
- **상황**: 고객(클라이언트)이 물건(데이터)을 주문하면, 택배 회사(컨트롤러)가 창고(유스케이스/레포지토리)에 물건을 요청하고 배달한다.
- **DTO의 역할**:
  - **주문서(Request DTO)**: 고객이 주문 시 작성하는 서류. 물건 종류와 수량을 명확히 기재하고, 잘못된 주문(예: 빈 주문서)을 걸러낸다.
  - **배달 상자(Response DTO)**: 창고에서 꺼낸 물건을 고객이 받기 편한 상자에 담는다. 상자는 고객이 필요로 하는 물건만 포함하며, 창고 내부 정보는 노출하지 않는다.
  - **내부 전달서(DTO)**: 택배 회사와 창고 간 물건을 주고받을 때 사용하는 표준 서류. 창고는 내부 물건(엔티티)을 서류에 맞춰 준비한다.
- **엔티티의 역할**: 창고에 보관된 실제 물건. 물건의 상태(예: 재고 수량)를 관리하며, 내부 규칙(예: 재고 감소 로직)을 수행한다.
- **계층 간 혼용 비유**: 동일한 주문서(DTO)를 고객과 창고 모두에 사용하면, 고객은 창고 내부 정보를 알게 되고, 창고는 고객의 요구사항에 맞춰 불필요한 데이터를 처리해야 한다. 이는 혼란과 비효율을 초래한다.

**비유의 교훈**: DTO는 특정 계층의 목적에 맞게 설계되어야 하며, 계층 간 혼용은 피해야 한다.

## 4. 레이어별 DTO와 엔티티 사용

헥사고날 아키텍처를 기준으로 각 계층에서 DTO와 엔티티의 사용 여부와 판단 근거를 설명한다. 사용자의 질문("컨트롤러의 요청/응답만 DTO, 나머지 계층은 엔티티")에 대한 답변도 포함한다.

### 4.1. 클라이언트 ↔ 컨트롤러 (API Layer)
- **사용 객체**: Request DTO, Response DTO
- **역할**:
  - **Request DTO**: 클라이언트의 HTTP 요청 데이터를 받아 유효성 검증(예: `@NotNull`, `@Size`) 수행.
  - **Response DTO**: 도메인 객체를 클라이언트가 이해하기 쉬운 JSON 형식으로 변환.
- **판단 근거**:
  - **DTO 사용 이유**: 클라이언트는 JSON 형식의 데이터를 주고받으며, 엔티티의 내부 구조(예: JPA 어노테이션, 관계)를 알 필요 없다. DTO는 필요한 데이터만 노출하고, 유효성 검증으로 잘못된 요청을 걸러낸다.
  - **엔티티 사용 금지**: 엔티티를 직접 노출하면 민감 데이터(예: 비밀번호) 유출, JPA Lazy Loading 문제, 또는 클라이언트와의 강한 결합 발생.
- **질문에 대한 답변**: 컨트롤러에서 Request/Response DTO를 사용하는 것은 정확하다. 클라이언트와의 인터페이스는 DTO로 처리해야 한다.
- **예시**:
  ```java
  public record CreateOrderRequest(@NotNull Long userId, @NotEmpty List<Long> productIds) {}
  public record OrderResponse(Long orderId, String status, BigDecimal totalAmount) {}
  ```

### 4.2. 컨트롤러 ↔ 유스케이스 (API Layer ↔ Domain Layer)
- **사용 객체**: 엔티티 (선호), DTO (제한적)
- **역할**:
  - 컨트롤러는 Request DTO의 데이터를 엔티티로 변환하거나, 간단한 입력 데이터(예: ID 리스트)를 직접 유스케이스로 전달.
  - 유스케이스는 엔티티를 반환하며, 컨트롤러가 이를 Response DTO로 변환.
- **판단 근거**:
  - **엔티티 선호 이유**: 유스케이스는 비즈니스 로직을 처리하므로, 엔티티(비즈니스 로직과 상태 포함)를 사용하는 것이 적합하다. 예: `Order` 엔티티로 주문 생성 로직 수행.
  - **DTO 사용 제한**: 컨트롤러-유스케이스 간 별도의 "도메인 DTO"는 일반적으로 불필요하다. DTO를 사용하면 데이터 변환 오버헤드가 발생하고, 비즈니스 로직이 DTO에 분산될 위험이 있다.
  - **도메인 DTO 사용 사례**: 복잡한 입력(예: 여러 엔티티의 조합)이 필요하거나, 유스케이스에 전달할 데이터가 엔티티와 구조가 크게 다를 때 제한적으로 사용. 예: 주문 생성 시 여러 데이터를 묶는 `OrderInputDTO`.
  - **계층 간 혼용 문제**: 동일한 DTO가 컨트롤러와 유스케이스에서 사용되면, 클라이언트의 요구사항(예: JSON 구조)과 유스케이스의 요구사항(예: 비즈니스 로직 입력)이 섞여 코드가 복잡해진다.
  - **질문에 대한 답변**: "컨트롤러-유스케이스 간에 도메인 DTO"는 일반적이지 않다. 엔티티를 직접 사용하는 것이 표준이며, 도메인 DTO는 복잡한 입력 구조가 필요할 때 예외적으로 사용된다.
- **예시**:
  ```java
  // 컨트롤러에서 유스케이스로 엔티티 전달
  Order order = orderUseCase.createOrder(request.userId(), request.productIds());
  ```

### 4.3. 유스케이스 ↔ 레포지토리 어댑터 (Domain Layer ↔ Adapter Layer)
- **사용 객체**: 엔티티 (선호), DTO (제한적)
- **역할**:
  - 유스케이스는 엔티티를 포트(레포지토리 인터페이스)에 전달.
  - 레포지토리 어댑터는 엔티티를 외부 시스템(예: DB, 캐시) 형식으로 변환하거나, 외부 데이터를 엔티티로 변환.
- **판단 근거**:
  - **엔티티 사용 이유**: 유스케이스와 레포지토리는 도메인 레이어의 일부로, 비즈니스 로직을 처리하는 엔티티를 직접 사용한다. 포트는 엔티티를 기준으로 설계되어 데이터 일관성을 유지한다.
  - **DTO 사용 제한**: 외부 시스템의 데이터 형식이 엔티티와 크게 다르거나, 복잡한 쿼리 결과(예: 조인)를 처리할 때 DTO를 사용. 예: DB에서 조회한 데이터를 `OrderQueryDTO`로 받아 엔티티로 변환.
  - **계층 간 혼용 문제**: 컨트롤러용 DTO(예: `OrderResponse`)가 레포지토리 어댑터에서 사용되면, 클라이언트 요구사항과 저장소 요구사항이 혼합되어 어댑터 코드가 복잡해진다.
  - **질문에 대한 답변**: "유스케이스-레포지토리 어댑터 간에 엔티티"는 정확하다. 포트는 주로 엔티티를 사용하며, DTO는 외부 시스템과의 형식 차이를 해결할 때 제한적으로 사용된다.
- **예시**:
  ```java
  public interface OrderPort {
      Order save(Order order);
      Optional<Order> findById(Long id);
  }
  ```

### 4.4. 판단 근거 요약
| 계층 간 연결                  | DTO 사용 여부 | 엔티티 사용 여부 | 근거                                                                 |
|-------------------------------|--------------|------------------|----------------------------------------------------------------------|
| **클라이언트 ↔ 컨트롤러**     | O (Request/Response DTO) | X                | 클라이언트 인터페이스에 맞춘 데이터 전달, 유효성 검증, 민감 데이터 제외 |
| **컨트롤러 ↔ 유스케이스**     | △ (도메인 DTO 제한적) | O (선호)         | 비즈니스 로직은 엔티티로 처리, 복잡한 입력 시 도메인 DTO 제한적 사용   |
| **유스케이스 ↔ 레포지토리 어댑터** | △ (제한적)   | O (선호)         | 도메인 로직은 엔티티 중심, 외부 시스템 형식 변환 시 DTO 제한적 사용    |

**계층 간 혼용 방지 원칙**:
- DTO는 특정 계층의 목적(예: 클라이언트 요청 처리)에 맞게 설계한다.
- 동일 DTO를 여러 계층에서 사용하면, 계층별 요구사항 충돌로 코드 복잡도가 증가한다.
- 예: `OrderDTO`를 컨트롤러와 유스케이스에서 공유하면, 클라이언트의 JSON 구조 변경이 유스케이스 로직에 영향을 미칠 수 있다.

**질문에 대한 최종 답변**:
- **컨트롤러의 요청/응답만 DTO**: 정확하다. 클라이언트와의 인터페이스는 Request/Response DTO로 처리한다.
- **나머지 계층에서 엔티티**: 대체로 정확하다. 컨트롤러-유스케이스와 유스케이스-레포지토리 어댑터 간에는 엔티티를 주로 사용하며, DTO는 복잡한 입력이나 외부 시스템 연동 시 제한적으로 사용한다.
- **계층 간 혼용**: DTO가 특정 계층의 전용 목적을 벗어나 다른 계층에서 재사용되는 것을 피해야 한다. 이는 계층 간 책임 분리를 약화시킨다.

## 5. 클린 아키텍처와 헥사고날 아키텍처에서의 DTO 사용

### 5.1. 클린 아키텍처에서의 DTO
**구조**:
- 클린 아키텍처는 **엔터프라이즈 비즈니스 규칙(엔티티)**, **애플리케이션 비즈니스 규칙(유스케이스)**, **인터페이스 어댑터(컨트롤러, 프레젠터)**, **프레임워크와 드라이버**로 구성된다.
- DTO는 인터페이스 어댑터에서 외부 시스템(클라이언트, DB)과의 데이터 전달을 담당한다.

**사용법**:
- **컨트롤러**: 클라이언트 요청을 `Request DTO`로 받아 유효성 검증 후 유스케이스로 전달.
  - 예: `CreateOrderRequest`로 주문 데이터를 받아 유스케이스 호출.
- **프레젠터**: 유스케이스의 결과를 `Response DTO`로 변환해 클라이언트에 반환.
  - 예: `OrderResponse`로 주문 데이터를 전달.
- **유스케이스**: 엔티티를 사용하며, DTO는 입력/출력 경계에서만 변환.
  - 예: `OrderUseCase`는 `Order` 엔티티를 처리.
- **계층 간 혼용 방지**: 컨트롤러용 DTO를 유스케이스에서 사용하면, 클라이언트 요구사항이 도메인 로직에 영향을 미칠 수 있다.

**근거**:
- 도메인(엔티티, 유스케이스)을 외부 기술(HTTP, DB)로부터 격리.
- DTO는 인터페이스 어댑터에서 외부와의 결합을 관리, 도메인 레이어는 순수성 유지.

### 5.2. 헥사고날 아키텍처에서의 DTO
**구조**:
- 헥사고날 아키텍처는 도메인 레이어를 중심으로 **포트(인터페이스)**와 **어댑터(Primary/Secondary)**로 구성된다.
- DTO는 **Primary Adapter**(컨트롤러)와 **Secondary Adapter**(레포지토리, 캐시)에서 사용된다.

**사용법**:
- **Primary Adapter (컨트롤러)**:
  - 클라이언트 요청을 `Request DTO`로 받아 유효성 검증 후 포트로 전달.
  - 포트의 결과를 `Response DTO`로 변환.
  - 예: `CreateOrderRequest` → `OrderPort.createOrder` → `OrderResponse`.
- **Secondary Adapter (레포지토리, 캐시)**:
  - 외부 시스템의 데이터를 DTO로 받아 포트에 전달하거나, 포트의 엔티티를 외부 형식으로 변환.
  - 예: DB 쿼리 결과를 `OrderQueryDTO`로 받아 `Order` 엔티티로 변환.
- **도메인 레이어**:
  - 유스케이스와 엔티티는 엔티티 중심.
  - DTO는 포트 인터페이스의 입력/출력으로 제한적 사용.
- **포트**:
  - 엔티티를 기준으로 메서드 정의(예: `save(Order order)`).
  - 복잡한 데이터 구조 시 DTO 사용 가능.
- **계층 간 혼용 방지**: 컨트롤러용 DTO가 레포지토리 어댑터에서 사용되면, 외부 시스템의 데이터 형식이 도메인 로직에 영향을 미칠 수 있다.

**근거**:
- 도메인을 외부 시스템과 격리하기 위해 포트를 사용.
- DTO는 외부 시스템과의 데이터 형식 차이를 해결, 도메인은 엔티티로 비즈니스 로직 처리.

**차이점 요약**:
| 아키텍처        | DTO 사용 계층                     | 엔티티 사용 계층               | 주요 특징                                                                 |
|------------------|------------------------------------|--------------------------------|---------------------------------------------------------------------------|
| 클린 아키텍처   | 인터페이스 어댑터                 | 엔티티, 유스케이스             | DTO는 외부 인터페이스에 한정, 도메인 레이어는 엔티티 중심                    |
| 헥사고날 아키텍처 | Primary/Secondary Adapter          | 도메인 레이어(유스케이스, 엔티티) | DTO는 어댑터에서 외부 시스템과 연결, 포트는 엔티티 또는 DTO 제한적 사용 |

## 6. 이커머스 플랫폼에서의 DTO 예시

이커머스 플랫폼의 주문 생성 시나리오를 통해 DTO와 엔티티 사용을 설명한다.

### 6.1. 시나리오: 주문 생성
- **요청**: 사용자가 상품 2개를 주문.
- **흐름**: 클라이언트 → 컨트롤러 → 유스케이스 → 포트 → 어댑터.

### 6.2. DTO 및 엔티티 설계
```java
// Request DTO (클라이언트 → 컨트롤러)
public record CreateOrderRequest(
    @NotNull(message = "User ID is required") Long userId,
    @NotEmpty(message = "Product IDs cannot be empty") @Size(max = 10) List<Long> productIds
) {}

// Response DTO (컨트롤러 → 클라이언트)
public record OrderResponse(
    Long orderId,
    String status,
    BigDecimal totalAmount,
    List<ProductResponse> products
) {
    public record ProductResponse(Long productId, String name, BigDecimal price) {}
}

// 도메인 엔티티 (도메인 레이어)
public class Order {
    private Long id;
    private Long userId;
    private List<Product> products;
    private OrderStatus status;
    private BigDecimal totalAmount;

    public void create() {
        this.status = OrderStatus.CREATED;
        this.totalAmount = products.stream()
            .map(Product::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// 포트 (도메인 ↔ 어댑터)
public interface OrderPort {
    Order save(Order order);
    Optional<Order> findById(Long id);
}
```

### 6.3. 계층 간 흐름
```java
@RestController
@RequestMapping("/api/order")
public class OrderController {
    private final OrderUseCase orderUseCase;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        Order order = orderUseCase.createOrder(request.userId(), request.productIds());
        return ResponseEntity.ok(toResponse(order));
    }

    private OrderResponse toResponse(Order order) {
        List<OrderResponse.ProductResponse> products = order.getProducts().stream()
            .map(p -> new OrderResponse.ProductResponse(p.getId(), p.getName(), p.getPrice()))
            .toList();
        return new OrderResponse(order.getId(), order.getStatus().name(), order.getTotalAmount(), products);
    }
}

@Service
public class OrderUseCase {
    private final OrderPort orderPort;
    private final ProductPort productPort;

    public Order createOrder(Long userId, List<Long> productIds) {
        List<Product> products = productPort.findByIds(productIds);
        Order order = new Order(userId, products);
        order.create();
        return orderPort.save(order);
    }
}

public class InMemoryOrderAdapter implements OrderPort {
    private final Map<Long, Order> store = new ConcurrentHashMap<>();

    @Override
    public Order save(Order order) {
        store.put(order.getId(), order);
        return order;
    }

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }
}
```

### 6.4. 분석
- **클라이언트 ↔ 컨트롤러**:
  - `CreateOrderRequest`: 클라이언트 입력을 받아 유효성 검증.
  - `OrderResponse`: 클라이언트에 필요한 주문 정보만 반환.
- **컨트롤러 ↔ 유스케이스**:
  - `Order` 엔티티 사용. 입력 데이터가 간단하므로 도메인 DTO 불필요.
  - 도메인 DTO 예시(복잡한 경우):
    ```java
    public record OrderInputDTO(Long userId, List<Long> productIds, String couponCode) {}
    ```
- **유스케이스 ↔ 레포지토리 어댑터**:
  - `Order` 엔티티를 `OrderPort`로 전달.
  - DTO는 외부 시스템(DB)과 형식이 다를 경우 제한적으로 사용(예: 복잡한 쿼리 결과).
- **계층 간 혼용 방지**: `OrderResponse`를 유스케이스나 어댑터에서 사용하지 않도록 설계.
- **적용 이유**: DTO는 클라이언트 인터페이스와 외부 시스템 연동에 집중, 엔티티는 도메인 로직을 처리.

## 7. DTO 설계 원칙

1. **단일 책임**:
   - DTO는 데이터 전달만 담당. 비즈니스 로직은 엔티티에.
   - 예: `OrderResponse`는 표시 데이터만 포함.
2. **최소 데이터 노출**:
   - 민감 데이터(예: 비밀번호) 제외.
   - 예: `UserResponse`에서 비밀번호 제거.
3. **명확한 네이밍**:
   - 역할 반영(예: `CreateOrderRequest`, `OrderResponse`).
4. **불변성**:
   - Java Record로 수정 방지.
   - 예: `public record OrderResponse(...)`.
5. **유효성 검증**:
   - Request DTO에 Bean Validation 적용.
   - 예: `@NotNull`로 필수 입력 강제.
6. **계층별 분리**:
   - 컨트롤러와 유스케이스용 DTO 분리.
   - 예: `CreateOrderRequest`(컨트롤러)와 `Order`(유스케이스).
7. **재사용 제한**:
   - DTO는 특정 계층/작업에 맞게 설계.
   - 예: 주문 생성과 조회 DTO 분리.

## 8. DTO 사용 검토 및 개선

### 8.1. 검토 기준
1. **데이터 노출**:
   - 민감 데이터(예: 비밀번호, 내부 ID)가 DTO에 포함되었는지 확인.
   - 해결: 불필요한 필드 제거.
2. **유효성 검증**:
   - Request DTO에 검증 어노테이션(`@NotNull`, `@Size`) 적용 여부 확인.
   - 해결: Bean Validation 추가.
3. **네이밍**:
   - DTO 이름이 역할(예: Request, Response)을 명확히 반영하는지 확인.
   - 해결: `OrderDTO` → `CreateOrderRequest`, `OrderResponse`.
4. **불변성**:
   - DTO가 수정 가능한지 확인.
   - 해결: Java Record 사용.
5. **계층 간 혼용**:
   - 동일 DTO가 여러 계층(컨트롤러, 유스케이스, 어댑터)에서 사용되는지 확인.
   - **문제 예시**: `OrderDTO`가 컨트롤러에서 클라이언트 응답용으로 설계되었으나, 유스케이스나 어댑터에서 재사용되면 클라이언트의 JSON 구조 변경이 도메인 로직에 영향을 미칠 수 있다.
   - **해결**: 계층별 전용 DTO 설계(예: `CreateOrderRequest`는 컨트롤러용, `Order` 엔티티는 유스케이스용).

### 8.2. 개선 예시
**문제**: `OrderDTO`가 컨트롤러와 유스케이스에서 혼용, 민감 데이터 포함.
```java
public class OrderDTO {
    private Long orderId;
    private Long userId;
    private String userPassword; // 민감 데이터
    private List<Long> productIds;
    private String status;
}
```

**개선**:
```java
// Request DTO (컨트롤러용)
public record CreateOrderRequest(@NotNull Long userId, @NotEmpty List<Long> productIds) {}

// Response DTO (컨트롤러용)
public record OrderResponse(Long orderId, String status, BigDecimal totalAmount) {}

// 엔티티 (유스케이스 및 어댑터용)
public class Order {
    private Long id;
    private Long userId;
    private List<Product> products;
    private OrderStatus status;
    private BigDecimal totalAmount;
}
```

**개선 결과**:
- 민감 데이터(`userPassword`) 제거.
- 유효성 검증 추가.
- 역할별 DTO 분리(컨트롤러용 DTO와 유스케이스용 엔티티).
- 계층 간 혼용 방지: `OrderResponse`는 컨트롤러에서만 사용.

### 8.3. 개선 프로세스
1. **코드 리뷰**: DTO 사용 현황 점검, 계층 간 혼용 여부 확인.
2. **리팩토링**: 역할별 DTO 분리, Record 적용, 민감 데이터 제거.
3. **테스트**: 유효성 검증 및 변환 로직 테스트 추가(예: JUnit으로 `@NotNull` 검증).
4. **문서화**: 개선된 DTO 구조를 프로젝트 문서에 기록.

## 9. 참고 자료

- **도서**:
  - *Domain-Driven Design* (Eric Evans): 엔티티와 DTO의 역할 분리.
  - *Clean Architecture* (Robert C. Martin): 클린 아키텍처에서의 DTO.
- **온라인 자료**:
  - [Spring Boot 문서](https://spring.io/projects/spring-boot): REST API와 DTO.
  - [Baeldung: DTO Pattern](https://www.baeldung.com/java-dto-pattern): DTO 사용 사례.
- **코드 예시**:
  - [Spring Boot REST API](https://github.com/spring-projects/spring-boot): DTO 구현 참고.
- **팀 자료**:
  - `api/dto/` 디렉토리: 기존 DTO 코드.
  - 팀 Wiki: 헥사고날 아키텍처 문서.

---

## 결론

DTO는 계층 간 데이터를 안전하고 효율적으로 전달하며, 엔티티는 비즈니스 로직과 상태를 관리한다. 클라이언트-컨트롤러 간에는 Request/Response DTO를 사용하고, 컨트롤러-유스케이스와 유스케이스-레포지토리 어댑터 간에는 주로 엔티티를 사용한다. 도메인 DTO는 복잡한 입력 구조가 필요할 때 제한적으로 사용된다. 계층 간 혼용은 동일 DTO가 여러 계층에서 재사용되어 책임 분리가 흐려지는 것을 의미하며, 이를 피하기 위해 DTO를 계층별로 분리 설계한다. 클린/헥사고날 아키텍처에서는 DTO를 외부 인터페이스와 어댑터에 집중시켜 도메인 레이어의 순수성을 유지한다. 위 원칙과 예시를 통해 DTO를 검토하고 개선하면 코드 구조와 유지보수성이 향상된다.