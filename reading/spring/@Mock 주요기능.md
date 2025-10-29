---
tags:
  - study
  - test
  - TDD
---
---

# Mockito `@Mock` 주요 기능 정리

`@Mock` 어노테이션은 단위 테스트에서 실제 의존 객체 대신 가짜(mock) 객체를 생성하여, 테스트하려는 코드(SUT, System Under Test)를 외부 의존성으로부터 격리시키는 역할을 합니다.

Mockito 프레임워크를 사용하며, 주로 `@ExtendWith(MockitoExtension.class)`와 함께 사용됩니다.

## 1. Stubbing: Mock 객체의 행동 정의하기

Stubbing은 Mock 객체가 특정 메소드를 호출받았을 때 어떻게 행동할지를 미리 지정하는 과정입니다.

### `when(...).thenReturn(...)`

가장 기본적이고 많이 사용되는 Stubbing 메소드입니다. 특정 메소드가 호출되면 정해진 값을 반환하도록 설정합니다.

```java
// userPointRepository.findById(1L)이 호출되면,
// new UserPoint(1L, 1000L, ...) 객체를 반환하도록 설정
when(userPointRepository.findById(1L))
    .thenReturn(new UserPoint(1L, 1000L, System.currentTimeMillis()));
```

### `when(...).thenThrow(...)`

메소드 호출 시 특정 예외를 발생시키도록 설정합니다. 예외 상황을 테스트할 때 유용합니다.

```java
// 존재하지 않는 유저 ID로 조회 시 RuntimeException을 발생시키도록 설정
when(userPointRepository.findById(999L))
    .thenThrow(new RuntimeException("User not found"));
```

### `when(...).thenAnswer(...)`

메소드 호출 시 입력된 인자에 따라 동적으로 다른 결과를 반환해야 할 때 사용합니다.

```java
// id가 짝수이면 1000포인트, 홀수이면 2000포인트를 반환하도록 설정
when(userPointRepository.save(anyLong(), anyLong()))
    .thenAnswer(invocation -> {
        long id = invocation.getArgument(0);
        long amount = invocation.getArgument(1);
        if (id % 2 == 0) {
            return new UserPoint(id, amount + 1000, System.currentTimeMillis());
        } else {
            return new UserPoint(id, amount + 2000, System.currentTimeMillis());
        }
    });
```

## 2. Verification: Mock 객체의 행위 검증하기

Verification은 테스트가 실행된 후, Mock 객체의 특정 메소드가 예상대로 호출되었는지를 검증하는 과정입니다.

### `verify(...)`

메소드가 최소 1번 호출되었는지 검증합니다. `times(1)`과 동일합니다.

```java
// pointService.charge가 실행된 후, userPointRepository.save가 최소 1번 호출되었는지 검증
verify(userPointRepository).save(anyLong(), anyLong());
```

### `verify(..., times(n))`

메소드가 정확히 `n`번 호출되었는지 검증합니다.

```java
// userPointRepository.save가 정확히 1번 호출되었는지 검증
verify(userPointRepository, times(1)).save(1L, 1000L);
```

### `verify(..., never())`

메소드가 한 번도 호출되지 않았는지 검증합니다.

```java
// 특정 조건에서 save 메소드가 호출되면 안 되는 것을 검증
verify(userPointRepository, never()).save(anyLong(), anyLong());
```

### 기타 검증 메소드

*   `atLeast(n)`: 최소 `n`번 호출되었는지 검증
*   `atMost(n)`: 최대 `n`번 호출되었는지 검증

## 3. Argument Matchers: 인자 유연하게 매칭하기

Stubbing이나 Verification을 할 때, 정확한 값 대신 유연한 조건을 사용하여 인자를 매칭할 수 있습니다.

### `any()` 계열

어떤 값이든 매칭합니다. 타입별로 `anyString()`, `anyLong()`, `any(List.class)` 등을 사용할 수 있습니다.

```java
// userId가 무엇이든, amount가 1000L이면 stubbing이 동작함
when(userPointRepository.save(anyLong(), eq(1000L)))
    .thenReturn(someUserPoint);

// 어떤 long 타입 인자가 2개 들어왔을 때 save 메소드 호출을 검증
verify(userPointRepository).save(anyLong(), anyLong());
```

### `eq(value)`

`any()`와 함께 사용할 때, 특정 인자만 고정된 값으로 지정하고 싶을 때 사용합니다. Mockito는 한 인자라도 matcher를 사용하면 모든 인자를 matcher로 감싸야 합니다.

### `argThat(matcher)`

커스텀 조건을 만들어 인자를 검증하고 싶을 때 사용합니다.

```java
// 1000보다 큰 포인트 충전이 있었는지 검증
verify(userPointRepository).save(anyLong(), argThat(amount -> amount > 1000));
```

## 4. ArgumentCaptor: 메소드 인자 캡처하기

메소드에 전달된 실제 인자 값을 캡처하여, 더 복잡한 검증 로직을 수행할 수 있습니다.

```java
// ArgumentCaptor 생성
ArgumentCaptor<UserPoint> userPointCaptor = ArgumentCaptor.forClass(UserPoint.class);

// ... 테스트 로직 실행 ...

// save 메소드가 호출될 때의 UserPoint 인자를 캡처
verify(userPointRepository).save(userPointCaptor.capture());

// 캡처된 객체를 가져와서 상세 검증
UserPoint capturedUserPoint = userPointCaptor.getValue();
assertThat(capturedUserPoint.point()).isEqualTo(500L);
```

## `@Spy`와의 차이점

*   **`@Mock`**: 완전히 비어있는 껍데기 객체를 만듭니다. Stubbing하지 않은 메소드는 `null`, `0`, `false` 등 기본값을 반환합니다.
*   **`@Spy`**: 실제 객체를 감싸는(wrapping) 프록시 객체를 만듭니다. Stubbing하지 않은 메소드는 **실제 객체의 로직을 그대로 수행**합니다. 실제 객체의 일부 동작만 변경하고 싶을 때 사용됩니다.

---

요약하자면, `@Mock`은 메소드 호출 여부, 횟수, 인자 등 **객체 간의 상호작용(interaction)**을 정밀하게 검증하는 데 강력한 기능을 제공합니다. 하지만 이는 테스트가 객체의 내부 구현에 강하게 결합될 수 있다는 단점도 있습니다.