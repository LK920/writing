# Spring의 AOP, 프록시, 셀프 인보케이션, 그리고 문제점 학습 자료

이 자료는 "Spring의 어노테이션은 AOP로 되어있다 -> AOP는 프록시로 구현되어있다 -> 프록시라서 셀프 인보케이션이 안된다 -> 이게 왜 문제인가"라는 흐름을 통해 Spring의 AOP 동작 방식, 셀프 인보케이션 문제, 그로 인한 문제점, 그리고 해결 방법을 통합적으로 설명합니다.

## 1. Spring의 어노테이션은 AOP로 되어있다

### AOP란?
- **AOP(Aspect-Oriented Programming, 관점 지향 프로그래밍)**은 공통 관심사(예: 로깅, 트랜잭션 관리, 보안, 캐싱)를 비즈니스 로직과 분리해 모듈화하는 프로그래밍 패러다임입니다.
- Spring에서는 `@Transactional`, `@Cacheable`, `@Async` 같은 어노테이션이 AOP를 통해 구현됩니다.
- **어떻게 동작하나?**:
  - 예: `@Transactional`은 메소드 실행 전/후에 트랜잭션을 시작하거나 커밋/롤백하는 로직을 추가합니다.
  - 이를 통해 비즈니스 로직(데이터 저장)과 공통 로직(트랜잭션 관리)을 분리해 코드 중복을 줄이고 유지보수를 쉽게 합니다.
- **예시**:
  ```java
  @Service
  public class MyService {
      @Transactional
      public void saveData() {
          System.out.println("데이터 저장");
          // 데이터베이스 작업
      }
  }
  ```
  - `@Transactional`은 AOP를 통해 트랜잭션 시작 → 비즈니스 로직 실행 → 트랜잭션 커밋/롤백을 처리합니다.

### 질문으로 확인
- `@Transactional` 같은 어노테이션을 써본 적이 있나요? 어떤 동작을 기대했나요?

---

## 2. AOP는 프록시로 구현되어있다

### 프록시 패턴이란?
- **프록시 패턴**은 타겟 객체(실제 작업을 수행하는 객체)에 대한 접근을 제어하거나 추가 로직을 수행하기 위해 중간에 "대리인" 객체(프록시)를 두는 디자인 패턴입니다.
- Spring에서 AOP는 **프록시 패턴**을 통해 구현됩니다. 프록시는 타겟 객체를 감싸서 클라이언트의 호출을 가로채고, AOP 로직(예: 트랜잭션 관리)을 실행한 후 타겟 객체를 호출합니다.

### Spring의 프록시 구현 방식
Spring은 두 가지 프록시를 사용합니다:
1. **JDK 동적 프록시**:
   - 인터페이스를 구현한 클래스에 대해 프록시를 생성.
   - `java.lang.reflect.Proxy`를 사용해 런타임에 동적으로 생성.
   - 제한: 타겟 클래스가 인터페이스를 구현해야 함.
2. **CGLIB 프록시**:
   - 인터페이스가 없는 클래스에 대해 프록시를 생성.
   - 바이트코드를 조작해 타겟 클래스의 서브클래스를 만듦.
   - 제한: `final` 클래스나 메소드는 프록시로 처리 불가.

### 프록시 동작 흐름
1. Spring이 `@Transactional`이 붙은 빈(예: `MyService`)을 생성할 때 프록시 객체를 만듦.
2. 클라이언트가 `myService.saveData()`를 호출하면 프록시가 호출을 가로챔.
3. 프록시가 트랜잭션 시작 → 타겟 객체의 `saveData()` 호출 → 트랜잭션 커밋/롤백.
4. 결과를 클라이언트에게 반환.

### 코드 예시
```java
public interface Service {
    void saveData();
}

@Service
public class MyService implements Service {
    @Transactional
    public void saveData() {
        System.out.println("데이터 저장");
    }
}
```
- **동작**:
  - Spring이 `MyService`의 프록시를 생성.
  - 클라이언트가 `service.saveData()`를 호출하면 프록시가 트랜잭션 로직을 추가로 실행.

### 질문으로 확인
- 프록시를 "중개인"이라고 생각하면 어떤 역할을 한다고 생각하나요?

---

## 3. 프록시라서 셀프 인보케이션이 안된다

### 셀프 인보케이션이란?
- **셀프 인보케이션(Self-Invocation)**은 클래스 내부에서 자신의 다른 메소드를 직접 호출하는 것을 말합니다. 즉, `this` 키워드를 사용해 같은 객체의 메소드를 호출하는 경우입니다.
- **예시**:
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

### 왜 셀프 인보케이션에서 문제가 되나?
- **프록시의 한계**:
  - Spring AOP는 프록시를 통해 동작하므로, 외부에서 타겟 객체의 메소드를 호출할 때만 AOP 로직(예: 트랜잭션 관리)이 적용됩니다.
  - 셀프 인보케이션(`this.methodB()`)은 프록시를 거치지 않고 타겟 객체의 메소드를 직접 호출합니다.
  - **결과**: `methodB()`의 `@Transactional`이 동작하지 않아 트랜잭션이 시작되지 않습니다.
- **쉽게 비유**:
  - 프록시는 집 앞 경비원이라고 생각하세요. 외부에서 집(`MyService`)에 들어갈 때 경비원이 신분 확인(트랜잭션 시작)을 하지만, 집 안에서 방(`methodA`)에서 다른 방(`methodB`)으로 이동할 때는 경비원을 안 거칩니다. 그래서 `methodB()`의 AOP 로직이 적용 안 됩니다.

### 동작 흐름
1. 클라이언트가 `myService.methodA()`를 호출.
2. 프록시가 호출을 가로채 `@Transactional`로 트랜잭션 시작.
3. `methodA()`가 실행되며 `this.methodB()`를 호출.
4. `this`는 프록시가 아닌 실제 `MyService` 객체를 가리키므로, `methodB()`가 프록시를 우회해 직접 호출됨.
5. `methodB()`의 `@Transactional`이 적용되지 않아 트랜잭션 없이 실행.

### 코드 예시 (문제 재현)
```java
@Service
public class MyService {
    @Transactional
    public void methodA() {
        System.out.println("methodA: 트랜잭션 = " + TransactionSynchronizationManager.isActualTransactionActive());
        this.methodB(); // 셀프 인보케이션
    }

    @Transactional
    public void methodB() {
        System.out.println("methodB: 트랜잭션 = " + TransactionSynchronizationManager.isActualTransactionActive());
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
  methodA: 트랜잭션 = true
  methodB: 트랜잭션 = false
  ```
- **설명**: `methodA()`는 프록시를 통해 호출되어 트랜잭션이 시작되지만, `this.methodB()`는 프록시를 우회하므로 트랜잭션이 적용되지 않음.

### 질문으로 확인
- 셀프 인보케이션에서 왜 `@Transactional`이 동작하지 않는다고 생각하나요?

---

## 4. 셀프 인보케이션 문제가 왜 중요한가?

셀프 인보케이션으로 인해 AOP 로직이 적용되지 않으면, 아래와 같은 심각한 문제가 발생할 수 있습니다:

### (1) 데이터 무결성 깨짐 (@Transactional의 경우)
- **상황**: `@Transactional`이 붙은 메소드가 트랜잭션 없이 실행되면, 데이터베이스 작업이 제대로 관리되지 않습니다.
- **예시**:
  ```java
  @Service
  public class MyService {
      @Autowired
      private EntityManager em;

      @Transactional
      public void methodA() {
          System.out.println("methodA 호출");
          em.persist(new Entity("데이터1")); // 데이터 저장
          this.methodB(); // 셀프 인보케이션
      }

      @Transactional
      public void methodB() {
          System.out.println("methodB 호출");
          em.persist(new Entity("데이터2"));
          throw new RuntimeException("에러 발생"); // 롤백 유발
      }
  }
  ```
  - **기대**: `methodB()`에 `@Transactional`이 있으니, 에러 발생 시 `데이터2`가 롤백되어야 함.
  - **실제**: `methodB()`가 프록시를 거치지 않으므로 트랜잭션이 시작되지 않음. `데이터2`가 데이터베이스에 남아 데이터 무결성이 깨짐.
- **문제**: 데이터베이스에 잘못된 데이터가 저장되거나, 롤백이 필요한 상황에서 롤백이 안 되어 시스템 상태가 일관되지 않음.
- **실제 사례**:
  - 전자상거래 시스템에서 재고 업데이트 메소드가 트랜잭션 없이 실행되면, 재고가 잘못 업데이트되어 고객이 없는 재고를 주문할 수 있음.
  - 금융 시스템에서 계좌 이체 메소드가 트랜잭션 없이 실행되면, 이체 실패 시 롤백이 안 되어 계좌 잔액이 불일치할 수 있음.

### (2) 캐싱 실패 (@Cacheable의 경우)
- **상황**: `@Cacheable`이 붙은 메소드가 셀프 인보케이션으로 호출되면 캐싱 로직이 동작하지 않음.
- **예시**:
  ```java
  @Service
  public class MyService {
      @Cacheable("myCache")
      public String methodA() {
          System.out.println("methodA 호출");
          return this.methodB(); // 셀프 인보케이션
      }

      @Cacheable("myCache")
      public String methodB() {
          System.out.println("methodB 호출");
          return "캐시된 결과"; // 비용이 큰 계산 또는 DB 조회
      }
  }
  ```
  - **기대**: `methodB()`의 결과가 캐시에 저장되어, 이후 호출 시 데이터베이스 조회 없이 캐시에서 결과를 가져옴.
  - **실제**: `methodB()`가 프록시를 거치지 않으므로 캐싱 로직이 적용되지 않아, 매번 데이터베이스를 조회하거나 계산을 수행함.
- **문제**: 성능 저하(캐싱의 이점을 잃음)와 불필요한 리소스 낭비.
- **실제 사례**:
  - 전자상거래 시스템에서 상품 목록 조회가 캐싱되지 않으면, 데이터베이스 부하가 증가해 응답 시간이 길어짐.

### (3) 비동기 처리 실패 (@Async의 경우)
- **상황**: `@Async`가 붙은 메소드가 셀프 인보케이션으로 호출되면 비동기 실행이 되지 않음.
- **예시**:
  ```java
  @Service
  public class MyService {
      @Async
      public void methodA() {
          System.out.println("methodA 호출");
          this.methodB(); // 셀프 인보케이션
      }

      @Async
      public void methodB() {
          System.out.println("methodB 호출");
          // 시간이 오래 걸리는 작업
      }
  }
  ```
  - **기대**: `methodB()`가 별도 스레드에서 비동기적으로 실행됨.
  - **실제**: `methodB()`가 프록시를 거치지 않으므로 같은 스레드에서 동기적으로 실행됨.
- **문제**: 비동기 처리로 얻을 수 있는 성능 향상(예: 작업 병렬화)이 없어지고, 응답 시간이 길어질 수 있음.
- **실제 사례**:
  - 알림 전송 시스템에서 비동기적으로 처리해야 할 알림 전송이 동기적으로 실행되면, 사용자 요청 처리 시간이 길어져 사용자 경험이 나빠짐.

### (4) 디버깅 어려움
- 개발자가 셀프 인보케이션 문제를 모르면, 코드가 의도대로 동작하지 않는 이유를 찾기 어려움.
- 예: `@Transactional`이 동작하지 않아 데이터가 롤백되지 않으면, 문제 원인을 프록시나 AOP로 의심하기까지 시간이 걸릴 수 있음.
- **문제**: 디버깅 시간 증가, 개발 생산성 저하.

### 왜 중요한가?
- **시스템 신뢰성**: 데이터 무결성, 성능, 비동기 처리 등 핵심 기능이 의도대로 동작하지 않으면 시스템 신뢰도가 떨어짐.
- **사용자 경험**: 예를 들어, 전자상거래에서 재고 오류로 주문이 잘못 처리되면 사용자 불만이 증가.
- **개발 생산성**: 문제를 모르면 디버깅에 많은 시간이 소요됨.

### 질문으로 확인
- 셀프 인보케이션 때문에 트랜잭션이 적용되지 않으면 어떤 문제가 생길 것 같나요?

---

## 5. 셀프 인보케이션 문제 해결 방법

셀프 인보케이션 문제를 해결하려면 프록시를 통해 메소드가 호출되도록 해야 합니다. 주요 방법은 다음과 같습니다:

### (1) 별도 서비스로 분리
- `methodB()`를 다른 서비스 클래스에 옮기고 주입받아 호출.
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
    }
}
```
- **장점**: 프록시가 정상적으로 동작해 AOP 기능 적용.
- **단점**: 코드 분리로 구조가 복잡해질 수 있음.

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
    }
}
```
- **장점**: 동일 클래스 내에서 프록시 호출 가능.
- **단점**: 순환 참조(Circular Dependency) 위험이 있으므로 주의 필요.

### (3) AspectJ 사용
- Spring AOP 대신 AspectJ를 사용하면 프록시 없이 바이트코드를 직접 조작해 AOP를 적용하므로 셀프 인보케이션 문제가 없음.
- **설정**: `spring-aspects` 의존성 추가, Load-Time Weaving 또는 Compile-Time Weaving 설정.
- **장점**: 셀프 인보케이션 문제 해결, 프록시 의존성 제거.
- **단점**: 설정이 복잡하고, Spring AOP보다 학습 곡선이 높음.

### 질문으로 확인
- 셀프 인보케이션 문제를 해결하기 위해 어떤 방법을 선호하나요? 이유는 뭔가요?

---

## 6. 질문으로 이해도 확인
1. AOP가 프록시로 구현된다는 게 어떤 의미인지 설명해볼 수 있나요?
2. 셀프 인보케이션에서 왜 `@Transactional`이 동작하지 않는다고 생각하나요?
3. 위 예시 중 어떤 상황(트랜잭션, 캐싱, 비동기)이 가장 심각한 문제라고 생각하나요?
4. 실제 프로젝트에서 셀프 인보케이션 문제를 만난다면 어떻게 디버깅할 건가요?

---

## 7. 추가 학습 제안
- **Spring AOP 문서**: Spring 공식 문서에서 AOP, 프록시, 셀프 인보케이션 관련 섹션 읽기.
- **실습**: Spring Boot 프로젝트에서 위 예제를 실행해 셀프 인보케이션 문제를 재현하고, 별도 서비스 또는 자기 자신 주입으로 해결해보기.
- **심화**: AspectJ를 설정해 프록시 없이 AOP를 적용해보고, Spring AOP와 비교.
- **탐구**: 프록시 패턴과 셀프 인보케이션이 다른 프레임워크(예: Java EE)에서 어떻게 동작하는지 조사.