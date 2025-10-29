---
tags:
  - test
  - study
  - TDD
---


# JUnit 5 ParameterizedTest 완벽 가이드

JUnit 5의 `@ParameterizedTest`는 테스트 코드의 중복을 줄이고 가독성을 높여주는 강력한 기능입니다. 동일한 테스트 로직을 여러 다른 입력 값으로 실행하고 싶을 때 사용하며, 각각의 입력을 개별 테스트 실행으로 간주하여 결과를 명확하게 보여줍니다.

이 가이드에서는 `@ParameterizedTest`의 기본 개념부터 다양한 실용적인 예제까지 다룹니다.

---

### 1. 왜 ParameterizedTest를 사용해야 하는가?

하나의 테스트 메서드 안에서 `for` 반복문을 사용해 여러 케이스를 검증하는 것과 비교했을 때, `@ParameterizedTest`는 다음과 같은 명확한 장점이 있습니다.

-   **결과의 명확성**: `for`문 안에서 특정 값 때문에 테스트가 실패하면, 전체 테스트 메서드가 실패로 표시되어 어떤 값에서 문제가 발생했는지 파악하기 어렵습니다. 반면, `@ParameterizedTest`는 각 파라미터별로 독립된 테스트 결과를 보여주므로, 어떤 입력값이 실패를 유발했는지 즉시 알 수 있습니다.
-   **가독성 및 유지보수**: 테스트 로직과 테스트 데이터가 분리되어 코드가 더 깔끔해지고, 새로운 테스트 케이스를 추가하거나 수정하기 용이합니다.

---

### 2. 기본 사용법: `@ValueSource`

가장 간단하게 파라미터를 제공하는 방법입니다. `longs`, `strings` 등 기본 타입의 값 목록을 직접 지정할 수 있습니다.

**중요**: 자바는 정적 타입 언어이므로, 메서드에 정의된 타입과 다른 타입의 값이 들어오는 경우는 컴파일 시점에 차단됩니다. 따라서 `long` 타입 파라미터에 `String` 값이 들어오는 것과 같은 "타입" 자체를 검증하는 테스트는 불필요합니다.

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

class ValueSourceTest {

    @ParameterizedTest
    @ValueSource(longs = {0L, -1L, 100L}) // long 타입 배열 제공
    @DisplayName("다양한 long 값 테스트")
    void testWithLongs(long number) {
        // number 파라미터에 0, -1, 100이 차례로 주입됨
        assertTrue(number >= -1);
    }

    @ParameterizedTest
    @ValueSource(strings = {"hello", "world", "junit"}) // String 타입 배열 제공
    @DisplayName("다양한 String 값 테스트")
    void testWithStrings(String text) {
        assertNotNull(text);
    }
}
```

---

### 3. Null, 빈 값, 공백 값 테스트

`null`이나 빈 문자열(`""`) 같은 경계 값을 테스트하는 것은 매우 흔한 시나리오입니다. JUnit 5는 이를 위한 편리한 어노테이션을 제공합니다.

-   `@NullSource`: `null` 값을 파라미터로 제공합니다.
-   `@EmptySource`: 빈 값(`""`, `[]`, `{}`)을 파라미터로 제공합니다.
-   `@NullAndEmptySource`: `@NullSource`와 `@EmptySource`를 합친 편리한 기능입니다.

```java
@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {" ", "   ", "\t", "\n"}) // 공백 문자열들 추가
@DisplayName("null, 비어있거나, 공백만 있는 문자열 테스트")
void testNullEmptyAndBlankStrings(String text) {
    // text 파라미터에 null, "", " ", "   ", ... 등이 주입됨
    assertTrue(text == null || text.trim().isEmpty());
}
```

---

### 4. Enum 상수 테스트: `@EnumSource`

`Enum` 클래스의 모든 상수를 순서대로 테스트에 사용할 수 있습니다.

```java
import io.hhplus.tdd.common.constants.TransactionType;

@ParameterizedTest
@EnumSource(TransactionType.class) // CHARGE, USE 상수가 차례로 주입됨
@DisplayName("모든 트랜잭션 타입 테스트")
void testAllTransactionTypes(TransactionType type) {
    assertNotNull(type);
}

// 특정 Enum 값만 테스트하거나 제외할 수도 있습니다.
@ParameterizedTest
@EnumSource(
    value = TransactionType.class,
    names = {"CHARGE"}, // CHARGE만 테스트
    mode = EnumSource.Mode.INCLUDE
)
void testOnlyChargeType(TransactionType type) {
    assertEquals(TransactionType.CHARGE, type);
}
```

---

### 5. 여러 인자 값 조합 테스트: `@CsvSource`

`CSV(Comma-Separated Values)` 형식으로 여러 개의 인자를 한 번에 제공할 수 있습니다. 각 문자열이 하나의 테스트 케이스가 됩니다.

```java
@ParameterizedTest
@CsvSource({
    "1, 100, CHARGE",  // 1번 케이스
    "2, -50, USE",     // 2번 케이스
    "3, 0, CHARGE"     // 3번 케이스
})
@DisplayName("사용자 ID, 금액, 타입 조합 테스트")
void testWithCsvSource(long userId, long amount, TransactionType type) {
    System.out.printf("userId=%d, amount=%d, type=%s\n", userId, amount, type);
    assertNotNull(type);
}
```
> JUnit은 `String`을 `long`, `boolean`, `enum` 등 다양한 타입으로 자동 변환해주는 똑똑한 기능이 내장되어 있습니다.

---

### 6. 외부 파일로 데이터 분리: `@CsvFileSource`

테스트 케이스가 많아지면 코드 안에 데이터를 두는 것이 부담스러울 수 있습니다. 이때 `CSV` 파일을 외부 리소스로 분리하여 사용할 수 있습니다.

`src/test/resources/test-data.csv`
```csv
1, 100, CHARGE
2, -50, USE
```

```java
@ParameterizedTest
@CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1) // 첫 줄(헤더)은 건너뛰기
@DisplayName("CSV 파일로부터 데이터 읽어서 테스트")
void testWithCsvFileSource(long userId, long amount, TransactionType type) {
    // ...
}
```

---

### 7. 복잡한 객체 생성 및 동적 데이터: `@MethodSource`

`@ValueSource`나 `@CsvSource`로 만들기 힘든 복잡한 객체를 파라미터로 제공해야 할 때 사용합니다. `Stream`을 반환하는 **정적(static) 팩토리 메서드**를 지정합니다.

```java
import java.util.stream.Stream;
import org.junit.jupiter.params.provider.Arguments;

class MethodSourceTest {

    // 파라미터를 제공할 정적 메서드
    private static Stream<Arguments> provideUserPointAndAmount() {
        return Stream.of(
            Arguments.of(new UserPoint(1L, 1000L), 500L),
            Arguments.of(new UserPoint(2L, 0L), 200L)
        );
    }

    @ParameterizedTest
    @MethodSource("provideUserPointAndAmount") // 메서드 이름을 문자열로 지정
    @DisplayName("메서드 소스를 이용한 복잡한 객체 테스트")
    void testWithMethodSource(UserPoint userPoint, long amount) {
        // ...
    }
}
```

---

### 8. 테스트 이름 커스터마이징

테스트 실행 결과에 표시되는 이름을 커스텀하여 가독성을 높일 수 있습니다.

-   `{index}`: 테스트 실행 순서 (1부터 시작)
-   `{arguments}`: 전체 인자 목록
-   `{0}`, `{1}`, ... : 인덱스별 파라미터 값

```java
@ParameterizedTest(name = "[{index}] 사용자 {0}의 포인트 {1}원 충전 테스트")
@CsvSource({"1, 100", "2, 200"})
void testWithCustomName(long userId, long amount) {
    // ...
}
// 실행 결과:
// [1] 사용자 1의 포인트 100원 충전 테스트
// [2] 사용자 2의 포인트 200원 충전 테스트
``` 


---

### JUnit `@MethodSource`와 `Stream<Arguments>` 심층 분석

`@MethodSource`는 `@ParameterizedTest`에 동적이고 복잡한 파라미터를 제공하기 위한 강력한 기능입니다. 이 가이드에서는 `Stream`, `Arguments`, 그리고 이들이 함께 동작하는 원리를 "선물 상자와 컨베이어 벨트" 비유를 통해 쉽게 설명합니다.

---

#### 1. 각 구성 요소의 역할

| 요소                | 역할                                          | 비유                                           |
| ------------------- | --------------------------------------------- | ---------------------------------------------- |
| **`Arguments`**     | 테스트 1회분에 필요한 **파라미터 묶음**       | 테스트용 선물 상자                             |
| **`Arguments.of()`**| 여러 파라미터를 `Arguments` 객체로 **포장**   | 선물 상자에 내용물(파라미터)을 담는 행위       |
| **`Stream`**        | 데이터의 **흐름** 또는 파이프라인             | 컨베이어 벨트                                  |
| **`Stream.of()`**   | 여러 데이터를 `Stream` 위로 **올려놓는** 행위 | 컨베이어 벨트에 선물 상자들을 차례로 올려놓는 것 |

---

#### 2. 동작 원리 상세히 보기

`@MethodSource`는 특정 정적(static) 메서드를 호출하여 테스트에 필요한 파라미터 목록을 받아옵니다. 이 과정은 다음과 같이 진행됩니다.

##### **Step 1: 파라미터 묶음 만들기 (`Arguments.of`)**

테스트 메서드는 여러 개의 파라미터를 받을 수 있습니다. `@MethodSource`는 이 파라미터들을 하나의 단위로 묶어서 관리해야 합니다. 이때 사용되는 것이 바로 `Arguments`입니다.

`Arguments.of(1L, 100L, TransactionType.CHARGE)`
- 이 코드는 `1L`, `100L`, `TransactionType.CHARGE` 라는 3개의 값을 `Arguments`라는 하나의 "선물 상자"에 담아 포장하는 것과 같습니다.
- 이 "선물 상자" 하나가 테스트를 **1회 실행**시키는 데 필요한 모든 파라미터를 담고 있습니다.

##### **Step 2: 파라미터 묶음의 흐름 만들기 (`Stream.of`)**

여러 개의 테스트 케이스가 있다면, 여러 개의 "선물 상자"가 필요합니다. `Stream`은 이 상자들이 흘러가는 "컨베이어 벨트" 역할을 합니다.

`Stream.of(Arguments.of(...), Arguments.of(...))`
- 이 코드는 여러 개의 `Arguments` 선물 상자를 `Stream`이라는 컨베이어 벨트 위에 순서대로 올려놓습니다.

##### **Step 3: 전체 프로세스 조합**

아래 코드는 이 모든 과정을 보여줍니다.

```java
import java.util.stream.Stream;
import org.junit.jupiter.params.provider.Arguments;

// ...

// 1. JUnit이 이 정적 메서드를 호출하여 파라미터 소스를 요청합니다.
private static Stream<Arguments> providePointHistoryArguments() {
    // 2. Stream이라는 "컨베이어 벨트"를 만듭니다.
    return Stream.of(
            // 3. 첫 번째 "선물 상자"를 만들어 벨트에 올립니다.
            Arguments.of(1L, 100L, TransactionType.CHARGE),

            // 4. 두 번째 "선물 상자"를 만들어 벨트에 올립니다.
            Arguments.of(2L, 200L, TransactionType.USE)
    );
}

@ParameterizedTest
@MethodSource("providePointHistoryArguments") // 5. 위 메서드를 파라미터 소스로 지정
void save(long userId, long amount, TransactionType transactionType) {
    // 6. JUnit이 컨베이어 벨트에서 상자를 하나씩 꺼내 내용물을 파라미터로 주입하며 테스트를 반복 실행합니다.
}
```

---

#### 3. 왜 그냥 `List`를 사용하지 않는가?

`@MethodSource`는 `Stream` 외에 `List<Arguments>` 같은 `Iterable` 타입도 반환할 수 있습니다. 그럼에도 `Stream` 사용이 권장되는 이유는 다음과 같습니다.

1.  **JUnit의 표준 방식**: JUnit 5 공식 문서와 대부분의 예제는 `Stream`을 표준으로 사용하여 자바 8+의 함수형 패러다임을 따릅니다.
2.  **메모리 효율성 (지연 로딩)**: `Stream`은 최종 연산 시점까지 데이터 처리를 지연시킵니다(Lazy Loading). 테스트 데이터가 매우 크거나 생성 비용이 비쌀 경우, 모든 데이터를 한 번에 메모리에 올리는 `List`보다 필요할 때마다 데이터를 생성하여 전달하는 `Stream`이 메모리 측면에서 더 효율적입니다.
3.  **유연성과 확장성**: `Stream` API는 `filter()`, `map()`, `limit()` 등 데이터를 동적으로 가공하고 조작할 수 있는 강력한 파이프라인 연산을 제공합니다. 이는 복잡한 테스트 데이터를 생성하거나 특정 조건에 맞는 데이터만 필터링할 때 `List`보다 훨씬 간결하고 유연한 코드를 작성할 수 있게 해줍니다.

**결론:** 간단한 케이스에서는 `List`도 충분히 사용 가능하지만, `Stream`을 사용하는 것이 JUnit 5의 표준에 부합하며, 더 높은 유연성과 성능 잠재력을 제공하는 현대적인 접근 방식입니다.