# JUnit 5 `@ExtendWith` 확장(Extension) 총정리

> **목표** : 실무에서 자주 쓰는 Extension 의 역할·기능·적절한 사용 예시를 한눈에 파악한다.

---
## 1. 왜 Extension 이 필요한가?
JUnit5 는 JUnit4 의 **Runner / Rule** 을 통합·확장한 개념으로 *Extension* 을 제공한다.
`@ExtendWith(SomeExtension.class)` 를 달면 JUnit 엔진은 테스트 라이프사이클 각 지점(before all, parameter resolve 등)에 훅을 걸어 **부가 기능**을 실행한다.

```
@BeforeEach  → BeforeEachCallback
@parameter   → ParameterResolver
예외 처리     → TestExecutionExceptionHandler
```
즉, 어노테이션만으론 아무 일도 안 일어나고 **Extension 구현체가 실제 기능**을 수행한다.

---
## 2. 실무 빈도 TOP 7 Extension (요약)
| Extension | 컨테이너 필요 | 대표 기능 | 권장 용도 |
|-----------|--------------|-----------|-----------|
| `MockitoExtension` | ❌ | `@Mock/@InjectMocks` 자동 초기화 | 순수 단위 테스트 (가장 빠름) |
| `SpringExtension` | ✅ | TestContext, DI, 트랜잭션 | Spring 기능 필요한 단위·슬라이스 테스트 |
| `@SpringBootTest` *(내포)* | ✅ | 전체 Boot 컨텍스트, Auto-Config | 컨트롤러~DB 통합 테스트 |
| `RestDocumentationExtension` | ✅ | 테스트 결과 → AsciiDoc | REST API 문서화 |
| `TestcontainersExtension` | ✅ | `@Container` Docker lifecycle | 실제 DB/브로커 통합 테스트 |
| **Custom** `TimingExtension` | ❌/✅ | 실행 시간 측정 | 느린 테스트 식별 |
| 내장 `TempDirectory` | ❌ | 임시 디렉터리 자동 생성 | 파일 I/O 테스트 |

---
## 3. Extension별 실전 설명 & 예시

### 3.1 `MockitoExtension`
- **주요 기능**: `@Mock`, `@InjectMocks`, `@Captor` 등 Mockito 어노테이션을 자동으로 초기화. Strict Stubbing(불필요한 Mock 호출 감지) 지원.
- **장점**: Spring 컨텍스트 없이 가장 빠른 테스트. 순수 자바 객체 단위의 행위 기반 검증에 최적.
- **단점**: DI, @Value, 트랜잭션 등 Spring 기능은 전혀 지원 안 함.
- **실전 예시**:
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @InjectMocks UserService service; // 테스트 대상
    @Mock UserRepository repo;       // 의존성 Mock

    @Test void saveCallsRepo() {
        service.save("kim");
        verify(repo).save("kim");
    }
}
```
- **언제 쓰나?**
  - 비즈니스 로직만 검증, 외부 의존성 없는 순수 단위 테스트
  - Spring 컨텍스트 띄우기 부담스러울 때

---
### 3.2 `SpringExtension`
- **주요 기능**: Spring TestContext Framework 구동. DI(@Autowired), @Value, 트랜잭션, 롤백, @MockBean/@SpyBean 등 Spring 기능 지원.
- **장점**: 실제 빈 주입, 프로퍼티 주입, 트랜잭션 롤백 등 Spring 환경 그대로 재현 가능. 슬라이스 테스트(@WebMvcTest, @DataJpaTest 등) 내부에 이미 포함.
- **단점**: 컨텍스트 기동 필요(속도↓), Boot 전체는 아님.
- **실전 예시**:
```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = ServiceOnlyConfig.class)
@TestPropertySource(properties = "feature.on=true")
class ServiceTest {
    @Autowired MyService svc;
    @MockBean UserRepository repo;
    @Value("${feature.on}") boolean featureOn;
}
```
- **언제 쓰나?**
  - Spring DI, 트랜잭션, 프로퍼티 주입 등 일부 Spring 기능만 필요한 테스트
  - Boot 전체 띄우기 부담스러울 때

---
### 3.3 `@SpringBootTest` (SpringExtension 포함)
- **주요 기능**: Boot Auto-Configuration, 내장 H2 DB, @Value, @ConfigurationProperties, @MockBean, @SpyBean 등 전체 Spring Boot 환경 지원.
- **장점**: 실제 운영 환경과 거의 동일한 통합 테스트 가능. REST, JPA, 트랜잭션, 프로퍼티 등 모두 검증.
- **단점**: 컨텍스트 기동 비용 큼(가장 느림), 테스트 간 격리 주의 필요.
- **실전 예시**:
```java
@SpringBootTest
class IntegrationTest {
    @Autowired OrderService svc;
    @MockBean PaymentGateway gateway;
    @Value("${point.max-point}") long maxPoint;
}
```
- **언제 쓰나?**
  - 컨트롤러~DB까지 end-to-end 통합 테스트
  - 실제 환경과 최대한 유사하게 검증하고 싶을 때

---
### 3.4 `RestDocumentationExtension`
- **주요 기능**: 테스트 결과를 AsciiDoc 스니펫으로 생성, Spring REST Docs와 연동.
- **장점**: API 문서 자동화, MockMvc와 결합해 실제 응답 기반 문서화.
- **실전 예시**:
```java
@ExtendWith(RestDocumentationExtension.class)
@WebMvcTest(ArticleController.class)
class RestDocsTest {
    @Autowired MockMvc mvc;
    @Test void doc() throws Exception {
        mvc.perform(get("/articles/1"))
           .andDo(document("article-get"));
    }
}
```
- **언제 쓰나?**
  - REST API 문서 자동화 필요할 때

---
### 3.5 `TestcontainersExtension`
- **주요 기능**: `@Container`로 선언한 Docker 컨테이너의 lifecycle 자동 관리. DB, MQ 등 외부 시스템 통합 테스트.
- **장점**: 실제 Postgres, Redis 등 환경에서 통합 테스트 가능. 포트 자동 할당, 동적 프로퍼티 주입 지원.
- **실전 예시**:
```java
@Testcontainers
@SpringBootTest
class WithPgTest {
    @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:15");
    @Autowired OrderRepository repo;
}
```
- **언제 쓰나?**
  - 실제 DB, MQ 등 외부 시스템과 통합 테스트
  - Testcontainers 라이브러리 의존성 필요

---
### 3.6 커스텀 `TimingExtension`
- **주요 기능**: 각 테스트 실행 전후 시간 측정, 로그 출력 등 커스텀 콜백.
- **실전 예시**:
```java
public class TimingExtension implements BeforeTestExecutionCallback, AfterTestExecutionCallback {
    public void beforeTestExecution(ExtensionContext ctx) {
        ctx.getStore(Namespace.GLOBAL).put("start", System.currentTimeMillis());
    }
    public void afterTestExecution(ExtensionContext ctx) {
        long start = ctx.getStore(Namespace.GLOBAL).remove("start", long.class);
        System.out.printf("%s took %dms\n", ctx.getDisplayName(), System.currentTimeMillis()-start);
    }
}
@ExtendWith(TimingExtension.class)
class AnyTest { /* ... */ }
```
- **언제 쓰나?**
  - 느린 테스트 자동 탐지, 커스텀 라이프사이클 후킹

---
### 3.7 내장 `TempDirectory`
- **주요 기능**: `@TempDir` 파라미터로 임시 디렉터리/파일 자동 생성·정리.
- **실전 예시**:
```java
@Test void fileTest(@TempDir Path tmp) throws IOException {
    Path file = Files.createFile(tmp.resolve("foo.txt"));
    Files.writeString(file, "hello");
    assertThat(Files.readString(file)).isEqualTo("hello");
}
```
- **언제 쓰나?**
  - 파일 I/O, 임시 자원 테스트

---
## 4. 선택 가이드 & 팁
- **가장 빠른 단위** → `MockitoExtension`만.
- **Spring DI 필요** → `SpringExtension` + 최소 Bean 구성 (@ContextConfiguration).
- **MVC 슬라이스** → `@WebMvcTest` (=SpringExtension 포함) + `@MockBean`.
- **JPA/DB** → `@DataJpaTest` (+TestcontainersExtension).
- **End-to-End** → `@SpringBootTest` + Testcontainers.
- Extension 중복 선언 가능하지만 meta-annotation에 이미 포함되어 있는지 먼저 확인.
