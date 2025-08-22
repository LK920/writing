---
tags:
  - study
  - test
  - TDD
---

# 효과적인 Mock 객체 관리 및 테스트 작성 팁

TDD(테스트 주도 개발)와 단위 테스트에서 Mock(가짜) 객체를 잘 활용하는 것은 매우 중요합니다. Mock을 통해 외부 의존성을 제어하고, 순수하게 테스트 대상 코드의 로직에만 집중할 수 있기 때문입니다. 이 문서에서는 Mockito를 기준으로, Mock 객체를 효과적으로 관리하고 더 나은 테스트 코드를 작성하는 몇 가지 팁을 공유합니다.

---

### 1. 기본 원칙: 테스트의 독립성과 명확성을 유지하라

가장 중요한 원칙은 **각 테스트는 하나의 시나리오만 검증하며, 서로에게 영향을 주지 않아야 한다**는 것입니다.

-   **`@BeforeEach`에서의 Mock 설정은 신중하게**: 모든 테스트에서 100% 동일하게 사용되는 Mock 설정이 아니라면, `@BeforeEach`에서 `given()`으로 특정 행동을 미리 정의하는 것은 피하는 것이 좋습니다. 각 테스트가 어떤 조건(Given) 하에 실행되는지 파악하기 어려워지고, 한 테스트의 변경이 다른 테스트에 영향을 줄 수 있기 때문입니다.

-   **Mock 설정은 각 테스트 메서드 안에서**: 테스트에 필요한 Mock 객체의 행동 정의(Stubbing)는 해당 테스트 메서드 내에 두는 것이 가장 명확합니다.

    ```java
    @Test
    @DisplayName("ID로 사용자 조회를 성공한다")
    void findUserById_success() {
        // Given: 이 테스트에만 필요한 Mock 행동을 명확히 정의
        long userId = 1L;
        User expectedUser = new User(userId, "tester", UserStatus.ACTIVE);
        given(userRepository.findById(userId)).willReturn(expectedUser);

        // When
        Optional<User> foundUser = userService.findUserById(userId);

        // Then
        assertThat(foundUser).isPresent();
        assertThat(foundUser.get()).isEqualTo(expectedUser);
    }
    ```

---

### 2. 강력한 팁: 헬퍼(Helper) 메서드로 반복을 줄이고 가독성을 높여라

여러 테스트에서 동일한 상황(Context)을 설정해야 할 때, 반복적인 `given-willReturn` 코드가 나타납니다. 이때 **상황을 설명하는 이름의 private 헬퍼 메서드**를 만들어 사용하면 테스트의 의도가 훨씬 명확해지고 중복을 제거할 수 있습니다.

**수정 전: 반복적인 Mock 설정**
```java
@Test
void A테스트() {
    // given
    User user = new User(1L, "tester", UserStatus.ACTIVE);
    given(userRepository.findById(1L)).willReturn(user); // <-- 중복 코드
    ...
}

@Test
void B테스트() {
    // given
    User user = new User(1L, "tester", UserStatus.ACTIVE);
    given(userRepository.findById(1L)).willReturn(user); // <-- 중복 코드
    ...
}
```

**수정 후: 헬퍼 메서드 활용**
```java
// 테스트 클래스 내부에 헬퍼 메서드 추가
private User givenUserExists(long userId) {
    User user = new User(userId, "tester", UserStatus.ACTIVE);
    given(userRepository.findById(userId)).willReturn(user);
    return user;
}

@Test
void A테스트() {
    // given
    User user = givenUserExists(1L); // "1번 유저가 존재하는 상황"을 한 줄로 표현
    ...
}

@Test
void B테스트() {
    // given
    User user = givenUserExists(1L);
    ...
}
```
이 방식은 "각 테스트마다 mock을 지정한다"는 원칙을 지키면서도, 코드를 훨씬 깔끔하고 유지보수하기 쉽게 만들어 줍니다.

---

### 3. BDD 스타일 Mockito로 가독성을 극대화하라

Mockito는 테스트 코드의 가독성을 높이기 위해 BDD(행위 주도 개발) 스타일의 API를 제공합니다.

-   `when(mock.method()).thenReturn(value)` 대신 `given(mock.method()).willReturn(value)`를 사용하세요.
-   **Given-When-Then** 테스트 구조와 완벽하게 맞아떨어져 코드를 읽기가 훨씬 자연스러워집니다.

```java
import static org.mockito.BDDMockito.*;

// Good: BDD 스타일
given(userRepository.findById(1L)).willReturn(someUser);

// Bad: 일반적인 Mockito 스타일
when(userRepository.findById(1L)).thenReturn(someUser);
```

---

### 4. 상태 검증을 넘어 행위 검증까지 (`verify`)

테스트는 단순히 결과 값이 맞는지만 확인하는 것(상태 검증)에서 그치지 않습니다. 때로는 **"특정 메서드가 정확한 인자와 함께 호출되었는가?"**를 검증(행위 검증)해야 합니다.

-   `verify()`를 사용하여 Mock 객체의 메서드 호출 여부와 횟수를 검증할 수 있습니다.
-   `argThat()`을 사용하면 더 복잡한 인자 조건을 검증할 수 있습니다.

```java
@Test
@DisplayName("이름 변경 시 findById와 update가 각각 한 번씩 호출된다")
void updateName_calls_repository_methods() {
    // given
    long userId = 1L;
    String newName = "new-tester";
    User user = givenUserExists(userId); // 헬퍼 메서드 사용
    given(userRepository.update(any(User.class))).willReturn(new User(userId, newName, UserStatus.ACTIVE));

    // when
    userService.updateName(userId, newName);

    // then
    // 행위 검증: findById가 userId 인자와 함께 1번 호출되었는지 확인
    verify(userRepository, times(1)).findById(userId);

    // 행위 검증: update 메서드가 특정 조건을 만족하는 User 객체와 함께 호출되었는지 확인
    verify(userRepository).update(argThat(u -> u.id() == userId && u.name().equals(newName)));
}
```

---

### 5. `@Mock`과 `@InjectMocks`로 초기화 코드를 간소화하라

`@ExtendWith(MockitoExtension.class)`와 함께 `@Mock`, `@InjectMocks` 어노테이션을 사용하면 테스트 설정 코드를 대폭 줄일 수 있습니다.

-   **`@Mock`**: Mock 객체를 생성합니다.
-   **`@InjectMocks`**: 테스트 대상 객체(`SUT`, System Under Test)를 생성하고, `@Mock`으로 생성된 의존 객체들을 자동으로 주입해 줍니다.

```java
@ExtendWith(MockitoExtension.class)
class UserServiceImplTest {

    @Mock // 가짜 UserRepository를 자동으로 생성
    private UserRepository userRepository;

    @InjectMocks // userRepository Mock 객체를 사용하는 UserServiceImpl 인스턴스를 자동으로 생성
    private UserServiceImpl userService;

    // @BeforeEach에서 new UserServiceImpl(new MockUserRepository()) 같은 코드를 쓸 필요가 없음!

    @Test
    void someTest() {
        // 이제 userService와 userRepository를 바로 사용하면 됩니다.
        ...
    }
}
```

이 팁들을 잘 활용하면 더 깔끔하고, 읽기 쉽고, 유지보수하기 좋은 테스트 코드를 작성할 수 있습니다.