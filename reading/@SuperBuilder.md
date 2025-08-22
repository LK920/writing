
`@SuperBuilder`는 Lombok 라이브러리에서 제공하는 어노테이션으로, 기본 `@Builder` 기능을 확장하여 **상속 구조**에서 Builder 패턴을 더 쉽게 구현할 수 있도록 돕습니다. 특히, 클래스 계층 구조(부모 클래스와 자식 클래스)에서 각 클래스의 필드를 모두 포함하는 Builder를 생성할 때 유용합니다.

## `@SuperBuilder`란?
- **기본 `@Builder`의 한계**: `@Builder`는 단일 클래스에 대해서만 Builder를 생성하며, 상속된 필드(부모 클래스의 필드)를 처리하기 어렵습니다.
- **`@SuperBuilder`의 역할**: 부모 클래스와 자식 클래스의 필드를 모두 포함하는 Builder를 자동 생성하여, 상속 구조에서도 체이닝 방식으로 객체를 쉽게 생성할 수 있습니다.

## 주요 특징
1. **상속 지원**: 부모 클래스와 자식 클래스의 필드를 모두 Builder에 포함.
2. **체이닝 방식**: 메서드 체이닝으로 가독성 높은 객체 생성.
3. **복잡한 계층 구조 처리**: 다중 상속(계층 구조)에서도 일관된 Builder 제공.
4. **Lombok 자동화**: 수동으로 복잡한 Builder 코드를 작성할 필요 없음.

## 설정
`@SuperBuilder`를 사용하려면 Lombok 의존성이 필요하며, 부모 클래스와 자식 클래스 모두에 `@SuperBuilder`를 적용해야 합니다.

**`pom.xml` 예시**:
```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.34</version>
    <scope>provided</scope>
</dependency>
```

## 예제 코드
다음은 `@SuperBuilder`를 사용한 부모-자식 클래스 예제입니다.

### 부모 클래스
```java
import lombok.Getter;
import lombok.experimental.SuperBuilder;

@Getter
@SuperBuilder
public abstract class Product {
    private final Long id;
    private final String name;
    private final BigDecimal price;
}
```

### 자식 클래스
```java
import lombok.Getter;
import lombok.experimental.SuperBuilder;

@Getter
@SuperBuilder
public class ElectronicProduct extends Product {
    private final int stock;
    private final String warrantyPeriod;
}
```

### 사용 예시
```java
ElectronicProduct product = ElectronicProduct.builder()
    .id(1L)
    .name("노트북")
    .price(new BigDecimal("1200000"))
    .stock(50)
    .warrantyPeriod("2년")
    .build();
```

### 동작 원리
- **부모 클래스의 필드**(`id`, `name`, `price`)와 **자식 클래스의 필드**(`stock`, `warrantyPeriod`)를 모두 Builder에 포함.
- Lombok이 컴파일 시점에 두 클래스의 필드를 처리하는 Builder 코드를 자동 생성.
- `builder()` 메서드는 체이닝 방식으로 모든 필드를 설정할 수 있게 해줌.

## `@SuperBuilder`와 `@Builder`의 차이
| 기능 | `@Builder` | `@SuperBuilder` |
|------|-----------|-----------------|
| **단일 클래스** | 지원 | 지원 |
| **상속 구조** | 미지원 (부모 필드 접근 불가) | 지원 (부모/자식 필드 모두 처리) |
| **코드 복잡도** | 간단 | 상속 처리로 약간 복잡 |
| **사용 편의성** | 단일 클래스에 적합 | 계층 구조에 적합 |

## `@SuperBuilder`의 주요 옵션
- **`toBuilder = true`**: 기존 객체를 기반으로 새로운 Builder를 생성 가능.
  ```java
  ElectronicProduct newProduct = product.toBuilder()
      .stock(100) // 기존 객체 수정
      .build();
  ```
- **`@SuperBuilder.Default`**: 필드의 기본값 설정 가능.
  ```java
  @SuperBuilder
  public class ElectronicProduct extends Product {
      @SuperBuilder.Default
      private final int stock = 0;
  }
  ```

## Spring Boot에서의 활용
Spring Boot에서 `@SuperBuilder`는 DTO나 엔티티의 상속 구조에서 자주 사용됩니다. 예를 들어, 공통 필드를 가진 `Product` 클래스를 상속받아 다양한 제품 타입을 정의할 때 유용합니다.

### 컨트롤러 예제
```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.math.BigDecimal;

@RestController
@RequestMapping("/api/product")
public class ProductController {
    
    @GetMapping("/electronic")
    public ElectronicProduct getElectronicProduct() {
        return ElectronicProduct.builder()
            .id(1L)
            .name("노트북")
            .price(new BigDecimal("1200000"))
            .stock(50)
            .warrantyPeriod("2년")
            .build();
    }
}
```

### 응답 예시
```json
{
  "id": 1,
  "name": "노트북",
  "price": "1200000",
  "stock": 50,
  "warrantyPeriod": "2년"
}
```

## 장단점
### 장점
- **상속 지원**: 복잡한 클래스 계층 구조에서 유연한 객체 생성.
- **가독성**: 체이닝 방식으로 직관적인 코드.
- **자동화**: Lombok이 Builder 코드를 자동 생성.
- **불변성**: `final` 필드로 불변 객체 생성 가능.

### 단점
- **Lombok 의존성**: Lombok에 의존해야 하며, IDE 설정 필요.
- **컴파일러 의존**: 컴파일 시점에 생성된 코드를 이해해야 디버깅 가능.
- **복잡성**: 간단한 클래스에는 `@Builder`가 더 적합할 수 있음.

## 학습 포인트
1. **상속과 Builder 패턴**:
   - 상속 구조에서 Builder 패턴의 필요성 이해.
   - `@SuperBuilder`로 부모/자식 필드 통합 처리.
2. **Lombok 활용**:
   - `@SuperBuilder`와 `@Builder`의 차이 비교.
   - `toBuilder`, `@SuperBuilder.Default` 사용법.
3. **Spring Boot 통합**:
   - DTO/엔티티 상속 구조에서의 활용.
   - REST API 응답과의 결합.

## 추천 실습
1. **상속 구조 구현**:
   - `Product`와 `ElectronicProduct` 클래스를 `@SuperBuilder`로 작성.
   - Builder로 객체 생성 테스트.
2. **toBuilder 활용**:
   - 기존 객체를 수정하는 `toBuilder` 메서드 사용.
3. **Spring Boot 통합**:
   - `@SuperBuilder`로 DTO 생성 후 컨트롤러에서 반환.
4. **CommonResponse 결합**:
   - 이전 가이드의 `CommonResponse`와 함께 사용하여 표준화된 응답 생성.

## 추가 학습 자료
- **Lombok 공식 문서**: https://projectlombok.org/features/SuperBuilder
- **Spring Boot REST API**: Spring 공식 문서 (https://spring.io)
- **디자인 패턴**: GoF의 Builder 패턴 챕터

이 가이드를 통해 `@SuperBuilder`의 개념과 사용법을 익히고, Spring Boot 프로젝트에서 상속 구조를 다룰 때 활용해 보세요!