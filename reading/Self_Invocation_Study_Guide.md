# 셀프 인보케이션(Self-Invocation) 학습 자료

## 1. 셀프 인보케이션이란?

### 정의
- **셀프 인보케이션(Self-Invocation)**은 클래스 내부에서 자신의 다른 메소드를 직접 호출하는 것을 말합니다. 즉, 객체가 `this` 키워드를 사용해 자신의 메소드를 호출하는 경우입니다.
- **핵심 아이디어**: 객체 내부에서 메소드를 호출하는 것은 일반적인 프로그래밍 패턴이지만, Spring 프레임워크에서 AOP(Aspect-Oriented Programming)와 프록시를 사용할 때 문제가 될 수 있습니다.
- **예시**:
  ```java
  public class MyService {
      public void methodA() {
          System.out.println("methodA 호출");
          this.methodB(); // 셀프 인보케이션
      }

      public void methodB() {
          System.out.println("methodB 호출");
      }
  }
  ```
  - `methodA()`가 `this.methodB()`를 호출하는 것이 셀프 인보케이션입니다.

### 언제 사용하는가?
- 클래스 내부에서 로직을 재사용하거나, 특정 작업을 단계적으로 처리할 때 자주 사용됩니다.
- 예: 한 메소드에서 데이터 검증 후 다른 메소드를 호출해 데이터를 저장하는 경우.

---

## 2. 셀프 인보케이션과 Spring AOP의 문제

Spring에서 AOP는 프록시 패턴을 통해 구현되며, `@Transactional`, `@Cacheable`, `@Async` 같은 어노테이션은 프록시 객체가 추가 로직(트랜잭션 관리, 캐싱 등)을 처리합니다. 하지만 셀프 인보케이션은 프록시를 우회하기 때문에 문제가 됩니다.

### 왜 문제가 되나?
1. **프록시의 동작 방식**:
   - Spring AOP는 타겟 객체를 감싸는 **프록시 객체**를 생성합니다.
   - 클라이언트가 타겟 객체의 메소드를 호출하면, 프록시가 호출을 가로채어 추가 로직(예: 트랜잭션 시작)을 수행한 후 타겟 객체의 메소드를 호출합니다.
   - 하지만 **셀프 인보케이션**은 프록시를 거치지 않고, 타겟 객체의 내부에서 직접 `this.methodB()`를 호출합니다.
2. **결과**:
   - 프록시가 적용한 AOP 로직(예: `@Transactional`의 트랜잭션 관리)이 동작하지 않습니다.
   - 예: `@Transactional`이 붙은 `methodB()`를 셀프 인보케이션으로 호출하면 트랜잭션이 시작되지 않습니다.

### 코드 예시
```java
@Service
public class MyService {
    @Transactional
    public void methodA() {
        System.out.println("methodA 호출");
        this.methodB(); // 셀프 인보케이션
    }

    @Transactional
    public void methodB() {
        System.out.println("methodB 호출");
        // 데이터베이스 작업
    }
}
```
- **문제**:
  - 외부에서 `myService.methodA()`를 호출하면 프록시가 `@Transactional`을 적용해 트랜잭션을 시작합니다.
  - 하지만 `methodA()` 내부에서 `this.methodB()`를 호출하면, 프록시를 거치지 않고 `MyService` 객체의 `methodB()`가 직접 호출되므로 `@Transactional`이 동작하지 않습니다.
- **결과**:
  - `methodB()`는 트랜잭션 없이 실행되어 데이터 무결성 문제가 발생할 수 있습니다.

---

## 3. 셀프 인보케이션 문제의 동작 방식

### 동작 흐름
1. **프록시 생성**:
   - Spring이 `MyService` 빈을 생성할 때, 프록시 객체를 생성합니다.
   - 클라이언트가 `myService.methodA()`를 호출하면 프록시가 호출을 가로챕니다.
2. **methodA 호출**:
   - 프록시가 트랜잭션을 시작하고, `MyService.methodA()`를 호출.
3. **셀프 인보케이션**:
   - `methodA()` 내부에서 `this.methodB()`를 호출.
   - `this`는 프록시가 아닌 실제 `MyService` 객체를 가리키므로, `methodB()`가 직접 호출됨.
   - 프록시의 `@Transactional` 로직이 적용되지 않음.
4. **결과**:
   - `methodB()`는 트랜잭션 없이 실행되어 예기치 않은 동작(예: 롤백 불가)이 발생할 수 있음.

### 다이어그램
```
Client → Proxy → MyService.methodA()
                   |
                   |--- this.methodB() (프록시 우회)
                   |
                   MyService.methodB() (AOP 적용 안 됨)
```

---

## 4. 셀프 인보케이션 문제 해결 방법

셀프 인보케이션 문제를 해결하려면 프록시를 통해 메소드가 호출되도록 해야 합니다. 주요 해결 방법은 다음과 같습니다:

### (1) 별도 서비스로 분리
- `methodB()`를 다른 서비스 클래스에 옮기고, 해당 서비스를 주입받아 호출.
```java
@Service
public class MyService {
    @Autowired
    private OtherService otherService;

    @Transactional
    public void methodA() {
        System.out.println("methodA 호출");
        otherService.methodB(); // 프록시를 통해 호출
    }
}

@Service
public class OtherService {
    @Transactional
    public void methodB() {
        System.out.println("methodB 호출");
        // 데이터베이스 작업
    }
}
```
- **장점**: 프록시가 정상적으로 동작해 AOP 기능이 적용됨.
- **단점**: 코드가 분리되어 구조가 복잡해질 수 있음.

### (2) 자기 자신 주입(Self-Injection)
- 자기 자신의 빈을 주입받아 프록시를 통해 호출.
```java
@Service
public class MyService {
    @Autowired
    private MyService self; // 자기 자신 주입

    @Transactional
    public void methodA() {
        System.out.println("methodA 호출");
        self.methodB(); // 프록시를 통해 호출
    }

    @Transactional
    public void methodB() {
        System.out.println("methodB 호출");
        // 데이터베이스 작업
    }
}
```
- **장점**: 동일 클래스 내에서 프록시 호출 가능.
- **단점**: 순환 참조(Circular Dependency) 위험이 있으므로 주의 필요.

### (3) AspectJ 사용
- Spring AOP 대신 AspectJ를 사용하면 프록시 없이 바이트코드를 직접 조작하므로 셀프 인보케이션 문제가 발생하지 않음.
- **설정**: `spring-aspects` 의존성을 추가하고, AspectJ의 Load-Time Weaving 또는 Compile-Time Weaving을 설정.
- **장점**: 셀프 인보케이션 문제 해결, 프록시 의존성 제거.
- **단점**: 설정이 복잡하고, Spring AOP보다 학습 곡선이 높음.

---

## 5. 왜 중요한가?
- **데이터 무결성**: `@Transactional`이 적용되지 않으면 트랜잭션이 시작되지 않아 데이터베이스 작업이 롤백되지 않을 수 있음.
- **예상치 못한 동작**: 개발자가 AOP가 적용될 거라 기대했지만, 셀프 인보케이션으로 인해 동작하지 않으면 버그 발생.
- **디버깅 어려움**: 프록시와 셀프 인보케이션의 동작 방식을 모르면 문제 원인을 찾기 어려움.

---

## 6. 실습 예제: 셀프 인보케이션 문제 재현
간단한 Spring Boot 프로젝트에서 셀프 인보케이션 문제를 재현해보는 예제입니다.

```java
@Service
public class MyService {
    @Transactional
    public void methodA() {
        System.out.println("methodA: 트랜잭션 활성화 = " + TransactionSynchronizationManager.isActualTransactionActive());
        this.methodB(); // 셀프 인보케이션
    }

    @Transactional
    public void methodB() {
        System.out.println("methodB: 트랜잭션 활성화 = " + TransactionSynchronizationManager.isActualTransactionActive());
        // 데이터베이스 작업
    }
}

@Component
public class Runner implements CommandLineRunner {
    @Autowired
    private MyService myService;

    @Override
    public void run(String... args) {
        myService.methodA();
    }
}
```

- **출력**:
  ```
  methodA: 트랜잭션 활성화 = true
  methodB: 트랜잭션 활성화 = false
  ```
- **설명**:
  - `methodA()`는 프록시를 통해 호출되므로 트랜잭션이 활성화됨.
  - `methodB()`는 셀프 인보케이션으로 프록시를 우회하므로 트랜잭션이 활성화되지 않음.

---

## 7. 질문으로 이해도 확인
1. 셀프 인보케이션이란 무엇인지, 간단히 설명해볼 수 있나요?
2. 셀프 인보케이션 때문에 `@Transactional`이 동작하지 않는 이유는 무엇이라고 생각하나요?
3. 위 예제에서 `methodB()`가 트랜잭션 없이 실행되면 어떤 문제가 생길 수 있을까요?
4. 셀프 인보케이션 문제를 해결하기 위해 어떤 방법을 사용하고 싶나요? 그 이유는 뭔가요?

---

## 8. 추가 학습 제안
- **Spring AOP 문서**: Spring 공식 문서에서 AOP와 프록시, 셀프 인보케이션 관련 섹션 읽기.
- **실습**: Spring Boot 프로젝트에서 위 예제를 실행하고, 셀프 인보케이션 문제를 재현한 후, 해결 방법(별도 서비스 또는 자기 자신 주입)을 적용해보기.
- **심화**: AspectJ를 설정해 셀프 인보케이션 문제를 해결해보고, Spring AOP와 비교.
- **탐구**: 프록시 패턴과 셀프 인보케이션이 다른 프레임워크(예: Java EE)에서 어떻게 동작하는지 비교.