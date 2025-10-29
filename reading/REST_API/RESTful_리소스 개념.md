# RESTful API 복수형 표현 및 리소스 계층 구조 학습 가이드

이 문서는 RESTful API 설계에서 복수형 표현(예: `/users`, `/orders`)과 리소스 계층 구조의 원칙을 상세히 설명한다. 사용자가 단수형(예: `/user`, `/coupon`)과 `/list` 접미사(예: `/user/list`)를 사용한 API 구조의 문제점을 지적하며, 왜 `/list`가 동작 중심이고 RESTful 원칙에 어긋나는지, 단수형의 한계는 무엇인지 명확히 다룬다. 이커머스 플랫폼의 맥락에서 초보자도 이해할 수 있도록 비유, 예시, 설계 원칙, 개선 가이드를 포함하며, 사용자의 기존 API를 RESTful 방식으로 개선하는 방법을 제시한다.

## 목차

- [1. RESTful API란 무엇인가?](#1-restful-api란-무엇인가)
- [2. 복수형 표현과 /list 접미사의 문제](#2-복수형-표현과-list-접미사의-문제)
- [3. 비유로 이해하는 RESTful API](#3-비유로-이해하는-restful-api)
- [4. 복수형 표현의 원칙과 판단 근거](#4-복수형-표현의-원칙과-판단-근거)
- [5. 리소스 계층 구조 설계](#5-리소스-계층-구조-설계)
- [6. 이커머스 플랫폼에서의 RESTful API 예시](#6-이커머스-플랫폼에서의-restful-api-예시)
- [7. RESTful API 설계 원칙](#7-restful-api-설계-원칙)
- [8. 현재 API 설계 검토 및 개선](#8-현재-api-설계-검토-및-개선)
- [9. 참고 자료](#9-참고-자료)

## 1. RESTful API란 무엇인가?

**정의**: RESTful API는 Representational State Transfer(REST) 아키텍처 스타일을 따르는 웹 API로, 리소스를 중심으로 설계되며, HTTP 메서드와 URI를 통해 리소스를 조작한다.

**핵심 특징**:
- **리소스 중심**: 데이터(예: 사용자, 주문)를 리소스로 표현하고, 고유한 URI로 식별.
- **표준 HTTP 메서드**: GET(조회), POST(생성), PUT/PATCH(수정), DELETE(삭제).
- **무상태성**: 각 요청은 독립적이며, 서버는 클라이언트 상태를 저장하지 않음.
- **계층 구조**: 리소스 간 관계를 URI로 표현(예: `/users/{id}/orders`).
- **일관성**: 직관적이고 예측 가능한 URI와 응답 구조.

**왜 중요한가?**:
- RESTful 설계는 API를 직관적이고 확장 가능하게 만들며, 클라이언트와 서버 간 인터페이스를 명확히 정의한다.
- 복수형 표현과 계층 구조는 리소스의 의도와 관계를 명확히 전달해 개발자와 클라이언트의 이해를 돕는다.

**사용자의 질문**: "왜 `/list`가 동작이야? 단수형에 `/list` 붙이는 게 이상한 거야?"
- **답변**: `/list`는 "목록 조회"라는 동작을 URI에 포함시켜 REST의 리소스 중심 원칙을 위반한다. REST에서는 동작(예: list, get)을 HTTP 메서드(GET, POST 등)로 표현하고, URI는 리소스 자체(예: `/users`, `/orders`)를 나타낸다. 단수형(예: `/user`)은 리소스가 단일 객체인지 컬렉션인지 모호하게 만들어 클라이언트에게 혼란을 줄 수 있다.

## 2. 복수형 표현과 /list 접미사의 문제

### 2.1. 복수형 표현
- **정의**: RESTful API에서 리소스는 보통 복수형 명사(예: `/users`, `/orders`)로 표현한다. 이는 리소스가 컬렉션(집합)을 나타낸다는 점을 강조한다.
- **예시**:
  - `/users`: 모든 사용자의 컬렉션.
  - `/users/{id}`: 특정 사용자.
  - `/users/{id}/orders`: 특정 사용자의 주문 컬렉션.

### 2.2. /list 접미사의 문제
- **문제점**: `/user/list`처럼 `/list`를 붙이는 것은 동작(목록 조회)을 URI에 포함시키는 방식으로, REST의 리소스 중심 원칙에 어긋난다.
- **왜 동작 중심인가?**:
  - REST에서는 URI가 리소스 자체를 나타내야 한다(예: `/users`는 사용자 컬렉션).
  - "list"는 "조회"라는 동작을 의미하며, 이는 HTTP 메서드 GET으로 표현해야 한다.
  - 예: `/users`에 GET 요청을 보내면 사용자 목록을 반환한다. `/list`는 불필요한 중복 표현이다.
- **단수형의 문제**:
  - 단수형(예: `/user`)은 리소스가 단일 객체인지 컬렉션인지 모호하다.
  - 예: `/user`가 모든 사용자를 나타내는지, 특정 사용자를 나타내는지 불분명.
  - 복수형(`/users`)은 컬렉션임을 명확히 하고, `/users/{id}`로 단일 리소스를 식별한다.
- **사용자의 질문에 대한 답변**:
  - `/list`는 동작 중심 표현으로, RESTful 원칙에서 피해야 한다. 예: `/user/list` 대신 GET `/users`를 사용.
  - 단수형(`/user`)은 직관성이 떨어지고, 복수형(`/users`)이 업계 표준이다. 단수형으로 통일하면 클라이언트가 리소스의 범위를 잘못 이해할 가능성이 크다.
  - RESTful API는 복수형과 HTTP 메서드를 조합해 명확하고 예측 가능한 구조를 제공한다.

## 3. 비유로 이해하는 RESTful API

RESTful API를 **도서관 시스템**으로 비유하면 다음과 같다:
- **상황**: 도서관(서버)에서 책(리소스)을 관리하며, 사서(클라이언트)가 책을 요청한다.
- **복수형 표현**:
  - 책은 여러 권(컬렉션)으로 관리되며, 도서관은 이를 "books"로 나타낸다.
  - 특정 책은 `/books/123`으로 식별한다.
  - 단수형(`/book`)은 한 권의 책을 의미할 수 있어 모호하다. 예: `/book`이 전체 책 컬렉션인지, 단일 책인지 불분명.
  - `/book/list`는 "책 목록을 가져오세요"라는 동작 지시로, 도서관 직원이 책 선반(`/books`)에 가서 목록을 가져오는 대신 불필요한 명령을 받는 셈이다.
- **리소스 계층 구조**:
  - 회원(members)이 빌린 책(loans)은 `/members/123/loans`로 표현한다. 이는 회원과 대출의 관계를 명확히 한다.
  - `/member/123/loanList`는 관계를 모호하게 하고, "loanList"는 동작 중심 용어로 혼란을 초래한다.
- **사용자의 문제 비유**:
  - `/user/list`는 사서가 "사용자 목록을 가져오세요"라고 요청하는 것과 같다. 하지만 도서관은 단순히 `/users` 선반에서 책을 꺼내면 된다(GET `/users`).
  - 단수형 `/user`는 "한 명의 사용자"를 의미할 수 있어, 사서가 전체 사용자인지 특정 사용자인지 헷갈린다.
- **비유의 교훈**:
  - 복수형(`/users`)은 리소스가 컬렉션임을 명확히 하여 혼란을 줄인다.
  - `/list` 같은 동작 중심 용어는 불필요하며, HTTP 메서드(GET)로 충분히 표현된다.

## 4. 복수형 표현의 원칙과 판단 근거

### 4.1. 복수형 사용 원칙
- **원칙**: 리소스는 복수형 명사(예: `/users`, `/orders`)로 표현한다. 단일 리소스는 ID를 통해 식별(예: `/users/{id}`).
- **판단 근거**:
  - **직관성**: 복수형은 리소스가 여러 개의 인스턴스를 포함할 수 있음을 나타낸다. 예: `/users`는 사용자 컬렉션, `/users/123`은 특정 사용자.
  - **일관성**: 모든 리소스에 복수형을 적용하면 URI가 예측 가능해진다.
  - **업계 표준**: GitHub, Twitter, Stripe 같은 주요 API는 복수형을 사용한다.
  - **단수형의 단점**: `/user`는 단일 리소스를 의미할 수 있어 모호하다. 클라이언트가 `/user`를 호출할 때 전체 목록인지, 단일 객체인지 혼란 가능.
  - **/list의 단점**: `/user/list`는 동작(목록 조회)을 URI에 포함시켜 REST의 리소스 중심 원칙을 위반한다. GET `/users`로 충분히 표현 가능.
- **사용자의 선호 반박**:
  - "복수형이 병신같다"는 느낌은 이해하지만, 복수형은 전 세계 개발자들이 따르는 표준이다. 단수형은 개인 프로젝트에서는 편할 수 있지만, 팀 협업이나 외부 클라이언트(예: 프론트엔드 개발자)가 사용할 때는 혼란을 초래한다.
  - `/list`는 직관적일 것 같지만, RESTful API에서는 동작을 URI에 넣지 않는 것이 원칙이다. 예: `/users`에 GET 요청이 목록 조회를 의미하므로 `/list`는 중복이고 비효율적이다.

### 4.2. 단수형 vs 복수형 vs /list 비교
| 표현 방식       | 예시                | 문제점                                      | RESTful 대안         |
|-----------------|---------------------|---------------------------------------------|----------------------|
| 단수형          | `/user`            | 컬렉션인지 단일 리소스인지 모호            | `/users`            |
| 목록 접미사     | `/user/list`       | 동작 중심, RESTful 원칙 위반                | `/users` (GET)      |
| 단일 리소스     | `/user/123`        | 단수형이지만 ID로 식별 가능, 허용 가능      | `/users/123` (권장) |
| 하위 리소스     | `/user/123/order`  | 상위 리소스가 단수형, 계층 구조 혼란 가능  | `/users/123/orders` |

**사용자의 문제 분석**:
- **단수형(`/user`, `/coupon`)**: 리소스가 컬렉션인지 단일 객체인지 불분명하여 클라이언트(예: 프론트엔드 개발자)가 혼란스러워할 수 있다.
- **/list 접미사(`/user/list`)**: "목록 조회"라는 동작을 URI에 포함시켜 RESTful 원칙(리소스 중심, 동작은 HTTP 메서드로 표현)을 위반한다.
- **개선 방향**: 복수형(`/users`, `/coupons`)으로 통일하고, 목록 조회는 GET 메서드로 처리(예: GET `/users`).

## 5. 리소스 계층 구조 설계

### 5.1. 계층 구조 원칙
- **원칙**: 리소스 간 관계를 URI의 계층 구조로 표현한다. 상위 리소스와 하위 리소스를 슬래시(/)로 연결하여 직관성을 높인다.
- **예시**:
  - `/users`: 모든 사용자 컬렉션.
  - `/users/{id}`: 특정 사용자.
  - `/users/{id}/orders`: 특정 사용자의 주문 컬렉션.
  - `/users/{id}/orders/{orderId}`: 특정 사용자의 특정 주문.
- **판단 근거**:
  - **관계 명확화**: 계층 구조는 리소스 간 소유 관계(예: 사용자 → 주문)를 명확히 표현한다.
  - **확장성**: 새로운 하위 리소스(예: `/users/{id}/coupons`)를 쉽게 추가 가능.
  - **직관성**: 클라이언트 개발자가 URI만 보고 리소스 관계를 이해할 수 있다.
  - **문제 사례**: `/user/123/order`는 단수형과 동작 중심 용어로 관계를 모호하게 만든다.

### 5.2. 계층 구조 설계 가이드
1. **상위 리소스 식별**: 주요 리소스(예: 사용자, 주문, 상품)를 복수형으로 정의.
2. **하위 리소스 연결**: 소유 관계를 슬래시(/)로 표현(예: `/users/{id}/orders`).
3. **동작은 HTTP 메서드 사용**: 조회(list), 생성(create) 등은 GET, POST로 표현.
4. **필터링은 쿼리 파라미터**: 특정 조건의 리소스는 쿼리 파라미터로 처리(예: `/orders?status=pending`).

**사용자의 문제 분석**:
- `/user/123/order`는 계층 구조를 표현하지만, 단수형(`/user`, `/order`)으로 모호하다.
- `/user/list`는 계층 구조가 아니라 동작 중심 표현이다.
- **개선 방향**: `/users/123/orders`로 복수형과 계층 구조를 명확히 한다.

## 6. 이커머스 플랫폼에서의 RESTful API 예시

이커머스 플랫폼의 주문 관리 API를 통해 복수형 표현과 리소스 계층 구조를 설명한다.

### 6.1. 시나리오: 주문 관리
- **요구사항**:
  - 모든 주문 조회.
  - 특정 사용자의 주문 조회.
  - 주문 생성.
  - 특정 주문 조회 및 수정.
- **현재 문제**:
  - 단수형: `/order`, `/user`.
  - 목록 접미사: `/order/list`, `/user/list`.
  - 비RESTful 계층: `/user/123/order`.

### 6.2. RESTful API 설계
```plaintext
# 사용자 관리
GET    /users            # 모든 사용자 조회
GET    /users/{id}       # 특정 사용자 조회
POST   /users            # 사용자 생성
PUT    /users/{id}       # 사용자 수정
DELETE /users/{id}       # 사용자 삭제

# 주문 관리
GET    /orders           # 모든 주문 조회
POST   /orders           # 주문 생성
GET    /orders/{id}      # 특정 주문 조회
PUT    /orders/{id}      # 주문 수정
DELETE /orders/{id}      # 주문 삭제

# 사용자별 주문
GET    /users/{id}/orders       # 특정 사용자의 주문 조회
POST   /users/{id}/orders       # 특정 사용자의 주문 생성
GET    /users/{id}/orders/{orderId}  # 특정 사용자의 특정 주문 조회

# 쿠폰 관리
GET    /coupons          # 모든 쿠폰 조회
POST   /coupons          # 쿠폰 생성
GET    /users/{id}/coupons  # 특정 사용자의 쿠폰 조회
POST   /users/{id}/coupons  # 특정 사용자에게 쿠폰 발급
```

### 6.3. DTO 설계 (주문 생성 예시)
```java
// Request DTO
public record CreateOrderRequest(
    @NotNull(message = "User ID is required") Long userId,
    @NotEmpty(message = "Product IDs cannot be empty") @Size(max = 10) List<Long> productIds
) {}

// Response DTO
public record OrderResponse(
    Long orderId,
    String status,
    BigDecimal totalAmount,
    List<ProductResponse> products
) {
    public record ProductResponse(Long productId, String name, BigDecimal price) {}
}
```

### 6.4. 컨트롤러 구현
```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderUseCase orderUseCase;

    // 모든 주문 조회
    @GetMapping
    public ResponseEntity<List<OrderResponse>> getAllOrders() {
        List<Order> orders = orderUseCase.findAll();
        return ResponseEntity.ok(orders.stream().map(this::toResponse).toList());
    }

    // 주문 생성
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        Order order = orderUseCase.createOrder(request.userId(), request.productIds());
        return ResponseEntity.status(HttpStatus.CREATED).body(toResponse(order));
    }

    // 특정 주문 조회
    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {
        Order order = orderUseCase.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
        return ResponseEntity.ok(toResponse(order));
    }

    // 사용자별 주문 조회
    @GetMapping("/users/{userId}/orders")
    public ResponseEntity<List<OrderResponse>> getUserOrders(@PathVariable Long userId) {
        List<Order> orders = orderUseCase.findByUserId(userId);
        return ResponseEntity.ok(orders.stream().map(this::toResponse).toList());
    }

    private OrderResponse toResponse(Order order) {
        List<OrderResponse.ProductResponse> products = order.getProducts().stream()
            .map(p -> new OrderResponse.ProductResponse(p.getId(), p.getName(), p.getPrice()))
            .toList();
        return new OrderResponse(order.getId(), order.getStatus().name(), order.getTotalAmount(), products);
    }
}
```

### 6.5. 분석
- **복수형 표현**:
  - `/orders`, `/users`, `/coupons`로 컬렉션을 명확히 표현.
  - `/list` 접미사 제거, GET 메서드로 목록 조회 표현.
- **리소스 계층**:
  - `/users/{id}/orders`로 사용자와 주문의 소유 관계를 명확히.
  - `/users/{id}/orders/{orderId}`로 특정 주문을 식별.
- **RESTful 원칙 준수**:
  - HTTP 메서드(GET, POST, PUT, DELETE)로 동작 표현.
  - 쿼리 파라미터로 필터링(예: `/orders?status=pending`).
- **사용자의 문제 해결**:
  - 단수형(`/order`, `/user`) → 복수형(`/orders`, `/users`).
  - `/order/list` → GET `/orders`.
  - `/user/123/order` → `/users/123/orders`.
- **왜 /list는 동작 중심인가?**:
  - `/list`는 "목록을 가져오세요"라는 동작을 URI에 명시한다. REST에서는 동작을 URI가 아닌 HTTP 메서드(GET)로 표현한다.
  - 예: GET `/orders`는 "주문 컬렉션에서 목록을 조회"를 의미하며, `/list`는 불필요한 중복이다.

## 7. RESTful API 설계 원칙

1. **복수형 명사 사용**:
   - 리소스는 복수형으로 표현(예: `/users`, `/orders`).
   - 단일 리소스는 ID로 식별(예: `/users/{id}`).
2. **리소스 중심**:
   - URI는 리소스를 나타내며, 동작(list, get)은 HTTP 메서드로 표현.
   - 예: `/orders` (GET)로 목록 조회, `/orders` (POST)로 생성.
3. **계층 구조**:
   - 리소스 간 관계를 슬래시(/)로 표현(예: `/users/{id}/orders`).
4. **HTTP 메서드 활용**:
   - GET: 조회, POST: 생성, PUT/PATCH: 수정, DELETE: 삭제.
5. **쿼리 파라미터로 필터링**:
   - 조건 기반 조회는 쿼리 파라미터 사용(예: `/orders?status=pending`).
6. **상태 코드 사용**:
   - 200(OK), 201(Created), 404(Not Found) 등 적절한 HTTP 상태 코드 반환.
7. **일관성 유지**:
   - 모든 API에서 복수형, 계층 구조, 명명 규칙을 일관되게 적용.

**사용자의 선호에 대한 답변**:
- 단수형과 `/list`를 선호하는 이유는 개인적으로 직관적이라고 느낄 수 있지만, RESTful API는 전 세계 개발자와 클라이언트가 이해하기 쉬운 표준을 따르기 위해 복수형과 HTTP 메서드를 사용한다.
- 단수형은 모호하고, `/list`는 동작 중심으로 REST 원칙을 어긴다. 팀 협업이나 외부 API 소비자(프론트엔드, 서드파티)가 사용할 때는 복수형(`/users`)과 GET 메서드가 더 명확하고 표준적이다.

## 8. 현재 API 설계 검토 및 개선

### 8.1. 검토 기준
1. **복수형 표현**:
   - 리소스가 단수형(예: `/user`, `/coupon`)인지 확인.
   - **문제**: 단수형은 컬렉션인지 단일 리소스인지 모호.
   - **해결**: 복수형(`/users`, `/coupons`)으로 변경.
2. **동작 중심 URI**:
   - `/list`, `/get`, `/create` 같은 동작 중심 접미사 사용 여부 확인.
   - **문제**: `/user/list`는 동작 중심으로 RESTful 원칙 위반.
   - **해결**: HTTP 메서드(GET `/users`)로 동작 표현.
3. **계층 구조**:
   - 리소스 간 관계가 계층적으로 표현되지 않은 경우(예: `/user/123/order`) 확인.
   - **문제**: 단수형과 비계층 구조로 관계 모호.
   - **해결**: `/users/{id}/orders`로 계층 표현.
4. **일관성**:
   - API 전체에서 명명 규칙과 구조가 일관되지 않은 경우 확인.
   - **해결**: 모든 URI에 복수형과 계층 구조 적용.
5. **HTTP 메서드 사용**:
   - 동작이 URI에 포함(예: `/order/create`)인지 확인.
   - **문제**: `/order/create`는 동작 중심.
   - **해결**: POST `/orders`로 변경.

### 8.2. 개선 예시
**현재 API (문제)**:
```plaintext
GET    /user/list          # 모든 사용자 조회
GET    /user/123           # 특정 사용자 조회
POST   /user/create        # 사용자 생성
GET    /user/123/order     # 특정 사용자의 주문 조회
GET    /order/list         # 모든 주문 조회
POST   /order/create       # 주문 생성
```

**개선된 API (RESTful)**:
```plaintext
GET    /users              # 모든 사용자 조회
GET    /users/{id}         # 특정 사용자 조회
POST   /users              # 사용자 생성
GET    /users/{id}/orders  # 특정 사용자의 주문 조회
GET    /orders             # 모든 주문 조회
POST   /orders             # 주문 생성
```

**개선 결과**:
- **단수형 제거**: `/user`, `/order` → `/users`, `/orders`.
- **동작 접미사 제거**: `/user/list`, `/order/create` → GET `/users`, POST `/orders`.
- **계층 구조 명확화**: `/user/123/order` → `/users/123/orders`.
- **이점**:
  - 클라이언트가 URI만 보고 리소스와 관계를 이해 가능.
  - 업계 표준 준수로 협업 및 유지보수 용이.
  - 동작 중심 용어 제거로 RESTful 원칙 준수.

### 8.3. 개선 프로세스
1. **코드 리뷰**:
   - 기존 API 엔드포인트를 점검하여 단수형, 동작 중심 URI, 비계층 구조 식별.
   - 예: `/user/list`, `/user/123/order` 확인.
2. **리팩토링**:
   - 단수형을 복수형으로 변경(예: `/user` → `/users`).
   - 동작 접미사 제거(예: `/order/list` → GET `/orders`).
   - 계층 구조 개선(예: `/user/123/order` → `/users/123/orders`).
3. **테스트**:
   - 수정된 API 엔드포인트 테스트(예: Postman으로 GET `/users` 확인).
   - 기존 클라이언트 코드(예: 프론트엔드)와의 호환성 점검.
4. **문서화**:
   - 개선된 API를 Swagger 또는 문서에 반영.
   - 예: `/users/{id}/orders`의 엔드포인트와 HTTP 메서드 기록.

## 9. 참고 자료

- **도서**:
  - *RESTful Web APIs* (Leonard Richardson): RESTful 설계 원칙.
  - *Clean Architecture* (Robert C. Martin): 리소스 중심 설계.
- **온라인 자료**:
  - [REST API Design Guidelines](https://restfulapi.net): 복수형과 계층 구조.
  - [Microsoft REST API Guidelines](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design): RESTful URI 설계.
  - [Baeldung: REST API Naming](https://www.baeldung.com/rest-api-naming-best-practices): 복수형과 계층 구조 예시.
- **코드 예시**:
  - [Spring Boot REST API](https://github.com/spring-projects/spring-boot): RESTful API 구현 참고.
- **팀 자료**:
  - `api/` 디렉토리: 기존 API 코드.
  - 팀 Wiki: 프로젝트 아키텍처 문서.

---

## 결론

RESTful API에서 복수형 표현(`/users`, `/orders`)과 리소스 계층 구조(`/users/{id}/orders`)는 직관성과 일관성을 높인다. 사용자의 현재 API는 단수형(`/user`, `/order`)과 동작 중심 접미사(`/list`, `/create`)를 사용해 RESTful 원칙을 위반하며, 모호하고 비직관적이다. `/list`는 동작(목록 조회)을 URI에 포함시켜 REST의 리소스 중심 원칙을 어기고, 단수형은 리소스가 컬렉션인지 단일 객체인지 불분명하다. 복수형 URI와 HTTP 메서드를 활용한 계층 구조로 개선하면 클라이언트와 개발자의 이해가 쉬워지고, 유지보수성이 향상된다. 위 원칙과 예시를 참고하여 API를 리팩토링하면 RESTful 설계의 이점을 극대화할 수 있다.