# Controller RateLimit 단위테스트 & 구현 가이드

---

## 1. 목적 및 요구사항

- 컨트롤러에서 userId별로 1초 내 5회 초과 요청 시 429(Too Many Requests) 반환
- 다른 userId는 정상 처리(200 OK)
- 동시성 환경(멀티스레드/멀티요청)에서 정책이 정확히 동작하는지 검증

---

## 2. RateLimit 동시성 테스트 핵심 패턴

- **ExecutorService**: 스레드풀로 동시 요청 실행
- **CountDownLatch**: 스레드 동기화(동시 시작/종료)
- **Future**: 비동기 결과/예외 수집
- **MockMvc**: Spring Web Layer 단위테스트
- **ObjectMapper**: 응답 JSON 파싱
- **테스트 독립성**: 상태 초기화(@BeforeEach)

---

## 3. 실전 코드 예시 (userId별 결과 합산 검증)

```java
@Test
@DisplayName("실패: 1초 내 같은 userId 5회 초과 충전 요청 시 429 Too Many Requests 반환, 다른 userId는 정상 처리")
void concurrentChargePoint() throws Exception {
    // given
    int thread1Count = 10;
    int thread2Count = 5;
    List<Long> userIds = new ArrayList<>();
    for (int i = 0; i < thread1Count; i++) userIds.add(1L);
    for (int i = 0; i < thread2Count; i++) userIds.add(2L);
    int totalRequests = userIds.size();

    ExecutorService executor = Executors.newFixedThreadPool(totalRequests);
    CountDownLatch readyLatch = new CountDownLatch(totalRequests);
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(totalRequests);
    List<Future<MvcResult>> results = new ArrayList<>();

    for (Long userId : userIds) {
        results.add(executor.submit(() -> {
            readyLatch.countDown();
            try {
                startLatch.await();
                return mockMvc.perform(patch("/point/" + userId + "/charge")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("100"))
                    .andReturn();
            } catch (Exception e) {
                throw new RuntimeException(e);
            } finally {
                doneLatch.countDown();
            }
        }));
    }
    readyLatch.await();
    startLatch.countDown();
    doneLatch.await();
    executor.shutdown();

    // then
    int user1_429 = 0, user2_429 = 0, user2_200 = 0;
    for (int i = 0; i < results.size(); i++) {
        Long userId = userIds.get(i);
        MvcResult result = results.get(i).get();
        int status = result.getResponse().getStatus();
        if (userId == 1L) {
            if (status == 429) {
                user1_429++;
                String content = new String(result.getResponse().getContentAsByteArray(), StandardCharsets.UTF_8);
                JsonNode json = objectMapper.readTree(content);
                assertThat(json.get("code").asText()).isEqualTo(ErrorCode.TOO_MANY_REQUESTS.getCode());
                assertThat(json.get("message").asText()).isEqualTo(ErrorCode.TOO_MANY_REQUESTS.getMessage());
            }
        } else if (userId == 2L) {
            if (status == 200) user2_200++;
            else if (status == 429) user2_429++;
        }
    }
    assertThat(user1_429).isGreaterThan(0);
    assertThat(user2_200).isEqualTo(5);
    assertThat(user2_429).isEqualTo(0);
}
```

---

## 4. 동기화/스레드풀/예외처리 패턴 (템플릿)

```java
ExecutorService executor = Executors.newFixedThreadPool(threadCount);
CountDownLatch readyLatch = new CountDownLatch(threadCount);
CountDownLatch startLatch = new CountDownLatch(1);
CountDownLatch doneLatch = new CountDownLatch(threadCount);

for (int i = 0; i < threadCount; i++) {
    executor.submit(() -> {
        readyLatch.countDown();
        startLatch.await();
        // ...테스트 로직...
        doneLatch.countDown();
    });
}
readyLatch.await();
startLatch.countDown();
doneLatch.await();
executor.shutdown();
```

---

## 5. 테스트 독립성 보장 (상태 초기화)

```java
@Autowired
private PointController pointController;

@BeforeEach
void resetRateLimitMap() throws Exception {
    Field field = PointController.class.getDeclaredField("rateLimitMap");
    field.setAccessible(true);
    ((ConcurrentHashMap<?, ?>) field.get(pointController)).clear();
}
```

---

## 6. 한글 인코딩 문제 해결

- MockMvc 응답은 반드시
  `new String(result.getResponse().getContentAsByteArray(), StandardCharsets.UTF_8)`
  로 변환해서 파싱
- gradle.properties, build.gradle, IDE 실행 옵션에
  `-Dfile.encoding=UTF-8`
  추가

---

## 7. 실전 팁/보완 사항

- **테스트별 상태 초기화**: 전역 상태(rateLimitMap 등)는 반드시 초기화. 안 하면 테스트 간 간섭 발생
- **예외/경계상황**: userId null/음수/미등록, 요청 body 이상 등도 테스트에 포함
- **멀티유저/멀티스레드**: userId 여러 개, 스레드 수/요청 수 다양화해서 경계상황 검증
- **429 응답 상세 검증**: code/message 등도 assert
- **테스트 불안정 시**: threadCount, latch, sleep 등 조정
- **코드블록 언어 항상 지정**: ```java, ```text 등
- **파일 저장은 UTF-8**

---

## 8. 결론

- 동시성/RateLimit 테스트는 반드시 여러 스레드, 동기화, 결과 합산 검증 구조로 작성
- Future로 각 요청의 예외/응답을 안전하게 수집, 검증
- MockMvc, ObjectMapper 등 Spring 표준 도구 적극 활용
- 다양한 케이스로 확장, 실전에서 바로 쓸 수 있는 구조로 작성

---