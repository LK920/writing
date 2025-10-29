---
tags:
  - study
  - lang/java
---
좋은 질문입니다. 여러 `enum`을 하나의 파일로 묶어서 관리하고 싶은 생각은 특히 상수가 많아질 때 자연스럽게 들 수 있습니다.

결론부터 말씀드리면, **현재 방식처럼 각자의 의미를 가진 `enum`으로 파일을 분리해서 관리하는 것이 훨씬 더 좋은, 현대 자바 개발에서 권장되는 방식**입니다. `CommonCode.java` 같은 파일로 묶는 것은 여러 단점을 가진 안티패턴(Anti-pattern)으로 여겨집니다.

왜 그런지 두 방식을 비교하며 설명해 드릴게요.

### 만약 `CommonCode.java`로 묶는다면?

아마 이런 형태가 될 것입니다. (보통은 `enum`이 아닌 인터페이스나 클래스에 상수를 정의하는 방식을 생각하게 됩니다.)

```java
// ❌ 좋지 않은 예시: CommonCode.java
public class CommonCode {
    // 거래 종류
    public static final String TRANSACTION_CHARGE = "CHARGE";
    public static final String TRANSACTION_USE = "USE";

    // 생존 여부
    public static final String ALIVE_ALIVE = "ALIVE";
    public static final String ALIVE_DEAD = "DEAD";
}
```

또는 하나의 거대한 `enum` 안에 모든 것을 포함시킬 수도 있습니다.

```java
// ❌ 좋지 않은 예시 2: 거대한 Enum
public enum CommonCode {
    CHARGE, USE, // Transaction Types
    ALIVE, DEAD  // Alive Types
}
```

### `CommonCode.java` 방식의 단점

1.  **타입 안정성(Type Safety)이 깨진다.**
    이것이 가장 치명적인 단점입니다. 예를 들어, 거래 종류만 받아야 하는 메서드가 있다고 가정해 봅시다.

    ```java
    // String 상수를 사용할 경우
    public void processTransaction(String transactionType) { ... }

    // "ALIVE"는 거래 종류가 아니지만, 컴파일러는 이 실수를 잡아주지 못합니다.
    processTransaction(CommonCode.ALIVE_ALIVE); // 문법적으로는 완벽한 코드
    ```

    `String`은 그냥 `String`일 뿐, "거래 종류"라는 문맥을 강제할 수 없습니다. 이는 런타임에 예기치 않은 버그를 유발합니다.

2.  **단일 책임 원칙(Single Responsibility Principle) 위반**
    `CommonCode` 클래스는 서로 아무 관련이 없는 "거래 종류"와 "생존 여부"라는 두 가지 책임을 동시에 갖게 됩니다. 나중에 "결제 수단" 같은 새로운 코드가 추가되면 이 파일은 점점 더 비대해지고 관련 없는 코드들의 잡동사니 창고가 되어 유지보수가 어려워집니다.

3.  **가독성과 명확성 저하**
    `TransactionType.CHARGE`는 코드를 읽는 사람에게 "이것은 거래 종류 중 충전이다"라는 의미를 명확하게 전달합니다. 반면 `CommonCode.TRANSACTION_CHARGE`는 한 단계를 더 거쳐야 하고, `CommonCode`가 정확히 무엇을 하는 클래스인지 파악하기 전까지는 의미가 모호합니다.

4.  **확장성 부족**
    `enum`은 단순한 상수가 아니라 완전한 클래스입니다. 내부에 메서드나 필드를 추가하여 특정 `enum` 상수와 관련된 행위를 정의할 수 있습니다. 예를 들어,

    ```java
    public enum TransactionType {
        CHARGE, USE;

        public boolean isCharge() {
            return this == CHARGE;
        }
    }
    ```
    `CommonCode` 방식으로는 이런 확장이 불가능합니다.

---

### 현재 방식(`enum` 분리)의 장점

| 장점 | 설명 | 코드 예시 |
| :--- | :--- | :--- |
| **타입 안정성** | 컴파일러가 타입 오류를 즉시 잡아줍니다. 잘못된 타입의 값을 전달하는 것이 원천적으로 불가능합니다. | `process(TransactionType.CHARGE);` **(O)**<br>`process(AliveType.ALIVE);` **(X, 컴파일 에러!)** |
| **명확한 책임** | `TransactionType.java`는 오직 '거래 종류'에 대한 것만 책임집니다. 코드가 응집력 있고 이해하기 쉽습니다. | - |
| **가독성** | `TransactionType.CHARGE` 자체만으로 의미가 명확합니다. | `if (type == TransactionType.CHARGE)` |
| **확장성** | `enum`에 관련 로직을 추가하여 더욱 풍부한 객체로 만들 수 있습니다. | `type.isCharge()` |

### 결론

여러 `enum` 파일을 만드는 것이 번거롭게 느껴질 수 있지만, 이는 **소프트웨어의 안정성과 유지보수성을 크게 향상시키는 아주 좋은 설계 습관**입니다.

`TransactionType`과 `AliveType`은 서로 다른 도메인 개념이므로, 현재처럼 별도의 파일로 명확하게 분리하여 관리하는 것이 최선의 방법입니다. `CommonCode`로 묶는 것은 장점보다 단점이 훨씬 많은 방식이므로 피하는 것이 좋습니다.

---
## 그거 물어본거 아닌데 라고 했더니

아, 내부 클래스(Inner Class)나 정적 중첩 클래스(Static Nested Class)로 `enum`을 묶는 방식을 말씀하시는 거군요. 이해했습니다.

예를 들어 이런 구조를 생각하고 계신 것 같습니다.

```java
// 제안하신 방식: CommonCode.java
public final class CommonCode {
    // 생성자를 private으로 막아 인스턴스화 방지
    private CommonCode() {} 

    public enum AliveType {
        ALIVE, DEAD
    }

    public enum TransactionType {
        CHARGE, USE
    }
}
```

이렇게 하면 `CommonCode.AliveType.ALIVE` 나 `CommonCode.TransactionType.CHARGE` 와 같이 접근할 수 있죠.

이 방식이 앞서 설명드린 `String` 상수를 사용하는 것보다는 **훨씬 좋습니다.** 왜냐하면 `enum`의 핵심 장점인 **타입 안정성**은 그대로 유지되기 때문입니다.

`process(CommonCode.TransactionType type)` 메서드는 여전히 `CommonCode.AliveType`을 받을 수 없으므로, 가장 치명적인 문제는 해결됩니다.

### 그럼에도 불구하고, 왜 여전히 개별 파일이 더 나은가?

이 접근 방식도 현대 자바 개발에서는 잘 사용되지 않는데, 몇 가지 미묘하지만 중요한 단점들이 있기 때문입니다.

1.  **불필요한 네임스페이스 추가로 인한 가독성 저하**
    코드를 사용할 때 `CommonCode.` 라는 접두사가 항상 붙게 됩니다.
    -   `import static io.hhplus.tdd.common.constants.CommonCode.TransactionType;` 와 같이 static import를 사용하면 `TransactionType.CHARGE`로 쓸 수는 있지만, 이럴 거면 애초에 분리된 파일로 두는 것과 차이가 없습니다. 오히려 import 구문만 더 복잡해집니다.
    -   `TransactionType.CHARGE` vs `CommonCode.TransactionType.CHARGE`
        -   전자는 "거래 타입은 충전이다"라고 바로 읽히지만, 후자는 "공통 코드 안에 있는 거래 타입은 충전이다"라고 읽힙니다. `CommonCode`라는 불필요한 문맥이 추가되어 코드를 이해하는 데 아주 약간의 인지 부하를 더합니다.

2.  **클래스의 역할이 모호해짐 (낮은 응집도)**
    `CommonCode` 라는 클래스는 그 자체로 어떤 역할을 하는지 명확하지 않습니다. 그저 관련 없는 `enum`들을 담아두는 **'컨테이너' 또는 '폴더' 역할**을 할 뿐입니다. 객체지향 설계에서 클래스는 특정 역할과 책임을 가져야 하는데, 이 방식은 그 원칙에 약간 어긋납니다. `TransactionType`과 `AliveType`은 논리적으로 아무런 연관이 없습니다.

3.  **파일 탐색의 어려움**
    IDE에서 `TransactionType`을 찾고 싶을 때, 개발자는 `TransactionType.java`라는 파일을 바로 검색하는 것이 직관적입니다. 하지만 이 방식에서는 `CommonCode.java` 파일을 먼저 열고 그 안에서 해당 `enum`을 찾아야 합니다. 코드가 많아질수록 이 차이는 더 크게 느껴집니다. "어떤 상수가 어느 그룹 파일에 있었지?" 하고 고민하게 될 수 있습니다.

4.  **Java 패키지 시스템의 본질적 목적**
    자바의 `package`는 이미 관련 있는 클래스들을 묶어주는 훌륭한 **'폴더'이자 '네임스페이스'**입니다. `io.hhplus.tdd.common.constants` 라는 패키지 안에 `AliveType.java`와 `TransactionType.java`가 있는 것만으로도 "이것들은 공통 상수를 모아둔 것"이라는 의미가 충분히 전달됩니다. 굳이 `CommonCode`라는 클래스로 한 번 더 묶을 필요가 없는 것이죠.

### 결론: Top-Level Class가 가장 좋다

자바에서는 특별한 이유가 없는 한, **모든 클래스(와 `enum`, 인터페이스)는 자신만의 독립적인 파일(`Top-Level Class`)로 만드는 것을 표준으로 삼고 권장**합니다.

지금 제안하신 방식이 동작은 하지만, 다음과 같은 이유로 **현재의 개별 파일 방식이 더 우수**합니다.

-   **명료함**: `TransactionType.java`는 그 파일의 목적을 명확히 보여줍니다.
-   **단순함**: 불필요한 계층(wrapping class)이 없습니다.
-   **표준**: 자바 커뮤니티의 일반적인 코딩 스타일에 부합합니다.

따라서 현재 구조를 그대로 유지하시는 것이 장기적으로 더 이점이 많은 결정입니다.