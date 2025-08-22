# 프록시 패턴과 동작 방식 학습 자료

## 1. 프록시 패턴이란?

### 정의
- **프록시 패턴(Proxy Pattern)**은 디자인 패턴 중 하나로, 어떤 객체(타겟 객체)에 대한 접근을 제어하거나 추가적인 작업을 수행하기 위해 중간에 대리인(프록시) 객체를 두는 패턴입니다.
- 프록시는 타겟 객체의 대리인 역할을 하며, 클라이언트가 타겟 객체에 직접 접근하지 않고 프록시를 통해 간접적으로 접근하도록 합니다.
- **핵심 아이디어**: 프록시는 타겟 객체의 호출을 가로채어 추가 로직(예: 로깅, 보안 체크, 캐싱, 트랜잭션 관리)을 수행한 후, 필요에 따라 타겟 객체의 메소드를 호출합니다.

### 주요 용도
- **접근 제어**: 민감한 데이터나 리소스에 대한 접근을 제한.
- **지연 로딩(Lazy Loading)**: 필요한 시점까지 객체 생성을 미룸.
- **로깅/모니터링**: 메소드 호출 전/후에 로그를 남김.
- **기능 추가**: 트랜잭션 관리, 캐싱 등 추가 기능 제공.
- **예시**: Spring에서 `@Transactional` 어노테이션을 사용할 때, 프록시 객체가 트랜잭션 시작/커밋/롤백을 처리.

---

## 2. 프록시 패턴의 동작 방식

### 기본 구조
프록시 패턴은 다음과 같은 구성 요소로 이루어집니다:
1. **Subject**: 클라이언트가 호출하는 인터페이스(타겟 객체와 프록시가 구현).
2. **RealSubject**: 실제 작업을 수행하는 타겟 객체.
3. **Proxy**: 클라이언트와 RealSubject 사이에 위치하며, 추가 로직을 수행하고 RealSubject를 호출.
4. **Client**: 프록시를 통해 RealSubject에 접근.

### 동작 흐름
1. 클라이언트가 Subject 인터페이스의 메소드를 호출.
2. 프록시가 호출을 가로채고, 추가 로직(예: 권한 확인, 로깅)을 수행.
3. 프록시가 필요에 따라 RealSubject의 메소드를 호출.
4. RealSubject가 실제 작업을 수행하고 결과를 반환.
5. 프록시가 결과를 클라이언트에게 반환(필요 시 추가 처리).

### 예시 다이어그램
```
Client → Proxy → RealSubject
  |        |         |
  |        |--- 추가 로직 (예: 로깅)
  |        |--- RealSubject 호출
  |        |--- 결과 처리
  |--- 결과 반환
```

---

## 3. 프록시 패턴의 종류

프록시 패턴은 용도에 따라 여러 형태로 구현됩니다:
1. **가상 프록시(Virtual Proxy)**:
   - 객체 생성 비용이 클 때, 실제 객체가 필요할 때까지 생성을 지연.
   - 예: 이미지 로딩 - 이미지가 화면에 표시될 때까지 로딩을 미룸.
2. **원격 프록시(Remote Proxy)**:
   - 원격 객체(다른 서버에 있는 객체)에 대한 로컬 대리인 역할.
   - 예: RMI(Remote Method Invocation)에서 사용.
3. **보호 프록시(Protection Proxy)**:
   - 접근 제어를 위해 사용. 권한이 있는 클라이언트만 타겟 객체에 접근 가능.
   - 예: 보안 시스템에서 특정 사용자만 데이터 수정 가능.
4. **스마트 프록시(Smart Proxy)**:
   - 추가적인 로직(로깅, 캐싱, 트랜잭션 관리 등)을 수행.
   - 예: Spring의 AOP 프록시.

---

## 4. Spring에서의 프록시 동작 방식

Spring 프레임워크에서 프록시 패턴은 주로 **AOP(Aspect-Oriented Programming)**를 구현하는 데 사용됩니다. 특히 `@Transactional`, `@Cacheable`, `@Async` 같은 어노테이션은 프록시를 통해 동작합니다.

### Spring 프록시의 구현 방식
Spring은 두 가지 프록시를 사용합니다:
1. **JDK 동적 프록시**:
   - 인터페이스를 구현한 클래스에 대해 프록시를 생성.
   - `java.lang.reflect.Proxy`를 사용해 런타임에 동적으로 프록시 객체를 생성.
   - 제한: 타겟 클래스가 인터페이스를 구현해야 함.
2. **CGLIB 프록시**:
   - 인터페이스가 없는 클래스에 대해 프록시를 생성.
   - 바이트코드를 조작해 타겟 클래스의 서브클래스를 생성.
   - 제한: `final` 클래스나 메소드는 프록시로 처리 불가.

### 동작 방식 예시 (Spring의 `@Transactional`)
1. **빈 생성**:
   - Spring이 `@Transactional`이 붙은 클래스의 빈을 생성할 때, 프록시 객체를 생성.
   - 예: `MyService` 클래스의 프록시 객체가 생성됨.
2. **메소드 호출**:
   - 클라이언트가 `MyService`의 메소드(예: `saveData()`)를 호출하면, 프록시가 호출을 가로챔.
3. **추가 로직 수행**:
   - 프록시가 트랜잭션을 시작(예: `TransactionManager.begin()`).
   - 타겟 객체(`MyService`)의 `saveData()`를 호출.
   - 호출이 성공하면 트랜잭션 커밋, 실패하면 롤백.
4. **결과 반환**:
   - 프록시가 결과를 클라이언트에게 반환.

### 코드 예시
```java
public interface MyService {
    void saveData();
}

@Service
public class MyServiceImpl implements MyService {
    @Transactional
    public void saveData() {
        System.out.println("데이터 저장 중...");
        // 실제 비즈니스 로직
    }
}
```
- **동작**:
  1. Spring이 `MyServiceImpl`의 프록시를 생성.
  2. 클라이언트가 `myService.saveData()`를 호출하면 프록시가 호출을 가로챔.
  3. 프록시가 트랜잭션을 시작하고, `MyServiceImpl.saveData()`를 호출.
  4. 작업 완료 후 트랜잭션 커밋/롤백.

---

## 5. 프록시 패턴의 장점과 단점

### 장점
- **접근 제어**: 타겟 객체에 대한 접근을 제한하거나 관리 가능.
- **기능 분리**: 비즈니스 로직과 공통 관심사(로깅, 트랜잭션 등)를 분리.
- **지연 로딩**: 리소스 사용을 최적화.
- **확장성**: 프록시에 새로운 기능 추가 가능.

### 단점
- **복잡도 증가**: 프록시 추가로 시스템 구조가 복잡해질 수 있음.
- **성능 오버헤드**: 프록시를 거치면서 약간의 성능 저하 가능.
- **제한사항**: Spring의 프록시는 셀프 인보케이션(Self-Invocation)에서 동작하지 않음(이전 대화 참조).

---

## 6. 실습 예제: 간단한 프록시 구현
프록시 패턴을 이해하기 위해 간단한 JDK 동적 프록시를 구현해보겠습니다.

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

// Subject 인터페이스
interface Service {
    void doWork();
}

// RealSubject
class RealService implements Service {
    public void doWork() {
        System.out.println("실제 작업 수행");
    }
}

// 프록시 핸들러
class LoggingProxy implements InvocationHandler {
    private final Object target;

    LoggingProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("프록시: 메소드 호출 전 로깅");
        Object result = method.invoke(target, args);
        System.out.println("프록시: 메소드 호출 후 로깅");
        return result;
    }
}

// 클라이언트
public class ProxyExample {
    public static void main(String[] args) {
        RealService realService = new RealService();
        Service proxy = (Service) Proxy.newProxyInstance(
            Service.class.getClassLoader(),
            new Class[]{Service.class},
            new LoggingProxy(realService)
        );
        proxy.doWork();
    }
}
```

- **출력**:
  ```
  프록시: 메소드 호출 전 로깅
  실제 작업 수행
  프록시: 메소드 호출 후 로깅
  ```
- **설명**:
  - `LoggingProxy`가 `RealService`의 호출을 가로채어 로깅을 추가.
  - `Proxy.newProxyInstance`로 동적 프록시 생성.

---

## 7. 질문으로 이해도 확인
1. 프록시 패턴의 주요 목적은 무엇이라고 생각하나요?
2. Spring에서 `@Transactional`이 프록시를 통해 어떻게 동작한다고 생각하나요?
3. 위 코드 예제에서 `LoggingProxy`가 어떤 역할을 하는지 설명해볼 수 있나요?
4. 프록시 패턴을 실제 프로젝트에서 어디에 적용해보면 좋을 것 같나요?

---

## 8. 추가 학습 제안
- **Spring AOP 문서**: Spring 공식 문서에서 AOP와 프록시 섹션 읽기.
- **실습**: Spring Boot 프로젝트에서 `@Transactional`을 사용해 프록시 동작 확인.
- **심화**: JDK 동적 프록시와 CGLIB의 차이점 학습. CGLIB로 프록시 구현해보기.
- **탐구**: 프록시 패턴과 다른 디자인 패턴(데코레이터, 어댑터)의 차이점 비교.