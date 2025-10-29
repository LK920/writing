---
aliases:
  - "# @Builder vs @SuperBuilder: 언제 무엇을 사용할까?"
---

이 문서는 Lombok의 `@Builder`와 `@SuperBuilder` 어노테이션의 차이점을 비교하고, `@SuperBuilder`만 사용해도 되는지에 대한 질문에 답하며, Spring Boot 환경에서의 사용 가이드와 학습 포인트를 정리합니다. 초보자도 이해할 수 있도록 간결하고 명확하게 작성했습니다.

---

## 1. 질문: @SuperBuilder만 써도 되나?

`@SuperBuilder`는 상속 구조를 지원하는 강력한 기능이 있지만, **모든 경우에 `@SuperBuilder`를 사용하는 것은 비효율적**일 수 있습니다. `@Builder`는 단일 클래스에서 간단한 Builder 패턴을 구현할 때 적합하며, 코드가 더 간단하고 컴파일 시 생성되는 코드도 가볍습니다. 반면, `@SuperBuilder`는 상속 계층 구조에서 필수적이지만, 단일 클래스에서는 불필요한 오버헤드를 초래할 수 있습니다.

**결론**: 상속이 필요 없는 단순한 클래스에서는 `@Builder`를, 상속 구조가 있는 경우에는 `@SuperBuilder`를 사용하는 것이 적절합니다.

---

## 2. @Builder와 @SuperBuilder 비교

|항목|`@Builder`|`@SuperBuilder`|
|---|---|---|
|**지원 범위**|단일 클래스|상속 구조 (부모/자식 클래스)|
|**코드 복잡도**|간단 (단일 클래스 Builder 생성)|복잡 (부모/자식 클래스 통합 Builder 생성)|
|**사용 용도**|DTO, 엔티티 등 단순 객체|상속 계층 구조 (예: 추상 클래스와 하위 클래스)|
|**컴파일 오버헤드**|낮음|높음 (상속 처리로 추가 코드 생성)|
|**기능**|기본 Builder, `@Builder.Default`, `@Singular`|`@Builder`의 모든 기능 + 상속 지원, `toBuilder`|
|**적합한 상황**|단일 클래스 객체 생성|복잡한 계층 구조의 객체 생성|

**핵심 차이**:

- `@Builder`: 단일 클래스의 필드만 처리하며, 상속된 필드는 다룰 수 없음.
- `@SuperBuilder`: 부모 클래스와 자식 클래스의 필드를 모두 포함하는 Builder를 생성, 상속 구조에 최적화.

---

## 3. 언제 @Builder를 사용할까?

`@Builder`는 다음과 같은 경우에 적합합니다:

- **단일 클래스**: 상속이 없는 DTO, 엔티티, 또는 VO(Value Object).
- **간단한 객체 생성**: 필드가 적고 복잡한 계층 구조가 없는 경우.
- **성능 최적화**: 컴파일 시 생성되는 코드가 간단하여 오버헤드가 적음.
- **가독성 우선**: 간단한 Builder 패턴으로 가독성을 높이고 싶을 때.

### 예제: 단순 DTO

```java
import lombok.Builder;
import lombok.Getter;
import java.math.BigDecimal;

@Getter
@Builder
public class ProductResponse {
    private final Long id;
    private final String name;
    private final BigDecimal price;
    private final int stock;
}
```

**사용**:

```java
ProductResponse product = ProductResponse.builder()
    .id(1L)
    .name("노트북")
    .price(new BigDecimal("1200000"))
    .stock(50)
    .build();
```

**왜 @Builder?**

- 상속이 필요 없으므로 `@SuperBuilder`의 복잡한 기능이 불필요.
- 코드가 간단하고 컴파일 오버헤드가 적음.

---

## 4. 언제 @SuperBuilder를 사용할까?

`@SuperBuilder`는 다음과 같은 경우에 적합합니다:

- **상속 구조**: 부모 클래스와 자식 클래스의 필드를 모두 포함해야 할 때.
- **복잡한 객체 생성**: 계층 구조에서 모든 필드를 체이닝 방식으로 설정해야 할 때.
- **객체 복사/수정**: `toBuilder`를 사용해 기존 객체를 기반으로 수정된 객체를 생성할 때.
- **유연한 설계**: 다양한 하위 클래스를 가진 추상 클래스와 함께 사용할 때.

### 예제: 상속 구조

```java
import lombok.Getter;
import lombok.experimental.SuperBuilder;
import java.math.BigDecimal;

@Getter
@SuperBuilder
public abstract class Product {
    private final Long id;
    private final String name;
    private final BigDecimal price;
}

@Getter
@SuperBuilder
public class ElectronicProduct extends Product {
    private final int stock;
    private final String warrantyPeriod;
}
```

**사용**:

```java
ElectronicProduct product = ElectronicProduct.builder()
    .id(1L)
    .name("노트북")
    .price(new BigDecimal("1200000"))
    .stock(50)
    .warrantyPeriod("2년")
    .build();
```

**왜 @SuperBuilder?**

- 부모 클래스(`Product`)의 필드(`id`, `name`, `price`)와 자식 클래스(`ElectronicProduct`)의 필드(`stock`, `warrantyPeriod`)를 모두 처리.
- 상속 구조에서 일관된 Builder 제공.

---

## 5. Spring Boot에서의 활용 예제

Spring Boot에서 `@Builder`와 `@SuperBuilder`는 주로 DTO나 엔티티 생성에 사용됩니다. 아래는 두 가지를 상황에 따라 사용하는 컨트롤러 예제입니다.

### @Builder 활용: 단순 DTO

```java
import lombok.Builder;
import lombok.Getter;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.math.BigDecimal;

@Getter
@Builder
class SimpleProductResponse {
    private final Long id;
    private final String name;
}

@RestController
@RequestMapping("/api/product")
public class ProductController {
    
    @GetMapping("/simple")
    public SimpleProductResponse getSimpleProduct() {
        return SimpleProductResponse.builder()
            .id(1L)
            .name("노트북")
            .build();
    }
}
```

**응답**:

```json
{
  "id": 1,
  "name": "노트북"
}
```

### @SuperBuilder 활용: 상속 구조

```java
import lombok.Getter;
import lombok.experimental.SuperBuilder;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.math.BigDecimal;

@Getter
@SuperBuilder
abstract class Product {
    private final Long id;
    private final String name;
}

@Getter
@SuperBuilder
class ElectronicProduct extends Product {
    private final int stock;
}

@RestController
@RequestMapping("/api/product")
public class ProductController {
    
    @GetMapping("/electronic")
    public ElectronicProduct getElectronicProduct() {
        return ElectronicProduct.builder()
            .id(1L)
            .name("노트북")
            .stock(50)
            .build();
    }
}
```

**응답**:

```json
{
  "id": 1,
  "name": "노트북",
  "stock": 50
}
```

**CommonResponse와 결합** (이전 가이드 참조):

```json
{
  "success": true,
  "message": "성공",
  "data": {
    "id": 1,
    "name": "노트북",
    "stock": 50
  },
  "timestamp": "2025-07-18T16:47:00"
}
```

---

## 6. 장단점과 고려사항

### @Builder의 장단점

**장점**:

- 코드가 간단하고 컴파일 오버헤드 적음.
- 단일 클래스에 적합, 빠른 구현 가능.
- DTO, 간단한 엔티티에 최적화.

**단점**:

- 상속 구조 미지원.
- 부모 클래스의 필드를 처리하려면 별도 코드 필요.

### @SuperBuilder의 장단점

**장점**:

- 상속 구조에서 부모/자식 필드 모두 처리.
- 복잡한 계층 구조에서도 일관된 Builder 제공.
- `toBuilder`로 객체 수정 용이.

**단점**:

- 컴파일 시 생성되는 코드가 복잡, 오버헤드 증가.
- 상속이 필요 없는 단순 클래스에서는 불필요한 복잡성 추가.
- 부모/자식 클래스 모두 `@SuperBuilder` 적용 필요.

### 고려사항

- **프로젝트 규모**: 소규모 프로젝트에서는 `@Builder`로 충분. 대규모 프로젝트에서 상속 구조가 많다면 `@SuperBuilder` 고려.
- **성능**: `@SuperBuilder`는 더 많은 코드를 생성하므로, 단순 클래스에서는 `@Builder`가 효율적.
- **유지보수**: 상속 구조가 변경될 가능성이 높다면 `@SuperBuilder`로 유연성 확보.
- **팀 익숙도**: 팀이 Lombok과 `@SuperBuilder`에 익숙하지 않다면 학습 비용 고려.

---

## 7. 학습 포인트와 실습

### 학습 포인트

1. **@Builder vs @SuperBuilder**:
    
    - `@Builder`는 단일 클래스, `@SuperBuilder`는 상속 구조에 적합.
    - 상황에 따른 선택 기준 이해.
2. **Lombok 활용**:
    
    - `@Builder.Default`, `@Singular`, `toBuilder` 사용법.
    - Lombok 컴파일 시점 코드 생성 원리.
3. **Spring Boot 통합**:
    
    - DTO/엔티티에서 `@Builder`와 `@SuperBuilder` 적용.
    - `CommonResponse`와의 결합 (이전 가이드 참조).
4. **설계 판단**
    
    - 상속 구조가 필요한지, 단순 클래스로 충분한지 판단.
    - 성능과 유지보수성의 균형 고려.

### 추천 실습

1. **@Builder 구현**:
    - 간단한 DTO(`ProductResponse`)를 `@Builder`로 작성.
    - 컨트롤러에서 반환 테스트.
2. **@SuperBuilder 구현**:
    - 부모 클래스(`Product`)와 자식 클래스(`ElectronicProduct`)를 `@SuperBuilder`로 작성.
    - 상속된 필드와 자식 클래스 필드를 모두 설정해 객체 생성.
3. **toBuilder 활용**:
    - `@SuperBuilder(toBuilder = true)`로 기존 객체 수정 테스트.
4. **Spring Boot 통합**:
    - `@Builder`와 `@SuperBuilder`를 각각 컨트롤러에서 사용.
    - `CommonResponse`와 결합해 표준화된 응답 생성.
5. **성능 비교**:
    - 간단한 클래스에 `@Builder`와 `@SuperBuilder`를 적용해 컴파일 시간 비교.
    - 상속 구조가 없는 경우 `@SuperBuilder`의 오버헤드 확인.

### 추가 학습 자료

- **Lombok 공식 문서**: [https://projectlombok.org/features/Builder](https://projectlombok.org/features/Builder)
- **@SuperBuilder 문서**: [https://projectlombok.org/features/SuperBuilder](https://projectlombok.org/features/SuperBuilder)
- **Spring Boot REST API**: [https://spring.io](https://spring.io/)
- **디자인 패턴**: GoF의 Builder 패턴 챕터

---

## 결론

`@SuperBuilder`는 상속 구조에서 강력하지만, 상속이 필요 없는 단순 클래스에서는 `@Builder`가 더 간단하고 효율적입니다. 프로젝트 요구사항, 클래스 구조, 성능, 팀의 익숙도를 고려해 적절히 선택하세요. Spring Boot 프로젝트에서 `@Builder`와 `@SuperBuilder`를 상황에 맞게 사용하면 가독성과 유지보수성을 높일 수 있습니다.