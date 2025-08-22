# Service Layer 동시성 단위테스트 전략 (PointServiceImplTest 기준)

---

## 1. 질문/답변 대화 구조

### 질문
- 서비스 계층(PointServiceImpl)에서 동시성(Concurrent) 상황을 단위테스트로 어떻게 검증해야 하는지, 실전 코드, 스레드/동기화 도구, 트랜잭션/락/Mock 전략, 검증 구조, 실무적 이슈/트러블슈팅까지 모두 설명해달라고 요청

### 답변
- 동시성 기준, 테스트 구조, 스레드/동기화 도구(ExecutorService, CountDownLatch 등) 설명
- 실전 코드 예시 제공 (여러 스레드가 동시에 충전/사용/이력 저장 시나리오)
- Mock 기반 동시성 테스트 한계와 실무적 대안(통합테스트/DB락/Atomic 등) 안내
- 실무적 리팩토링/개선 포인트, 트러블슈팅, 검증 구조 상세 설명

---

## 2. Service 동시성 단위테스트 실전 작성법

### 2.1. 테스트 목적
- 여러 스레드가 동시에 같은 유저의 포인트를 충전/사용/조회할 때 데이터 정합성(Atomicity, Consistency)이 깨지지 않는지 검증
- 동시성 환경에서 race condition, lost update, dirty read 등 문제 발생 여부 확인
- Mock 기반 단위테스트로는 한계가 있으나, 최소한 서비스 로직의 흐름/락 호출/예외처리 검증

### 2.2. 주요 사용 기술/도구
- **ExecutorService**: 스레드풀 생성 및 동시 요청 실행
- **CountDownLatch**: 스레드 동기화(동시 시작/종료 보장)
- **Future**: 비동기 실행 결과 수집 및 예외 처리
- **Mockito**: Repository/DB 계층 Mocking (실제 DB 락/트랜잭션 효과는 없음)
- **AtomicLong/AtomicReference**: Mock 내부에서 상태를 원자적으로 변경할 때 활용 가능

---

## 3. 스레드/동기화 도구 상세 설명 (실전/실무 중심)

### 3.1. ExecutorService (스레드풀)
- 컨트롤러 레이어와 동일하게, 동시 요청 수만큼 스레드풀 생성
- 테스트 끝나면 반드시 shutdown() 호출

### 3.2. CountDownLatch (동기화 도구)
- readyLatch, startLatch, doneLatch 패턴으로 모든 스레드가 동시에 출발/종료하도록 보장

### 3.3. Future (비동기 결과/예외 수집)
- 각 요청을 Future로 받아서, 나중에 순서대로 get()으로 결과/예외를 안전하게 수집

---

## 4. 실전 코드 예시 (Mock 기반 동시성 테스트)

```java
@Test
@DisplayName("실패: 여러 스레드가 동시에 같은 유저 포인트를 충전할 때 데이터 정합성 깨짐 여부 검증")
void concurrentChargePoint() throws Exception {
    int threadCount = 10;
    long userId = 1L;
    long initialPoint = 1000L;
    long chargeAmount = 100L;
    // AtomicLong으로 Mock 내부 상태를 원자적으로 관리
    AtomicLong pointState = new AtomicLong(initialPoint);
    when(userPointRepository.findById(userId)).thenAnswer(invocation ->
        new UserPoint(userId, pointState.get(), System.currentTimeMillis()));
    when(userPointRepository.save(eq(userId), anyLong())).thenAnswer(invocation -> {
        long newPoint = invocation.getArgument(1);
        pointState.set(newPoint);
        return new UserPoint(userId, newPoint, System.currentTimeMillis());
    });
    mockActiveUser(userId);
    mockHistorySaveSuccess(userId, chargeAmount, TransactionType.CHARGE);

    ExecutorService executor = Executors.newFixedThreadPool(threadCount);
    CountDownLatch readyLatch = new CountDownLatch(threadCount);
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(threadCount);
    List<Future<UserPoint>> results = new ArrayList<>();
    for (int i = 0; i < threadCount; i++) {
        results.add(executor.submit(() -> {
            readyLatch.countDown();
            startLatch.await();
            try {
                return pointService.charge(userId, chargeAmount);
            } finally {
                doneLatch.countDown();
            }
        }));
    }
    readyLatch.await();
    startLatch.countDown();
    doneLatch.await();
    executor.shutdown();

    // then: 최종 포인트가 기대값(초기+threadCount*chargeAmount)과 일치하는지 검증
    assertThat(pointState.get()).isEqualTo(initialPoint + threadCount * chargeAmount);
}
```

---

## 5. 검증 구조 및 실무 팁

- **AtomicLong/AtomicReference**로 Mock 내부 상태를 원자적으로 관리해야 race condition을 일부 재현 가능
- 실제 DB/트랜잭션/락이 없는 Mock 기반 단위테스트는 한계가 명확함(실제 동시성 버그는 통합테스트/DB에서만 검출 가능)
- 동시성 테스트는 반드시 여러 스레드, 동기화, 결과 합산 검증 구조로 작성
- Future를 통해 각 요청의 예외/응답을 안전하게 수집, 검증
- 실전에서는 DB 락, Synchronized, Optimistic Lock, Version 필드 등과 연계 필요
- Mock 기반 테스트는 서비스 로직의 흐름/락 호출/예외처리 검증에만 한정

---

## 6. 실전 주의사항/트러블슈팅

### 1. Mock 기반 동시성 테스트의 한계
- 실제 DB 락/트랜잭션 효과가 없으므로, race condition/dirty read/lost update 등은 완벽히 재현 불가
- Mock 내부 상태를 Atomic으로 관리해도, 진짜 동시성 이슈는 통합테스트/실DB에서만 검증 가능
- **실무 팁:**
    - 단위테스트는 서비스 로직의 흐름/락 호출/예외처리만 검증
    - 진짜 동시성/정합성 검증은 통합테스트(@SpringBootTest+실DB+멀티스레드)에서 수행

### 2. 테스트별 상태 초기화
- Mock 객체의 내부 상태(AtomicLong 등)는 각 테스트마다 초기화해야 함
- @BeforeEach에서 상태 리셋 필수

### 3. 예외/실패 케이스 검증
- 여러 스레드가 동시에 예외(예: 포인트 한도 초과, 잔액 부족 등)를 발생시키는 상황도 검증
- Future.get()에서 예외 발생 시 ExecutionException.getCause()로 원인 확인

---

## 7. 결론/실무 적용 포인트

- 서비스 계층 동시성 단위테스트는 Mock 기반 한계 내에서 최대한 race condition/정합성 검증 구조로 작성
- 실제 동시성/정합성 보장은 통합테스트/실DB/락/트랜잭션에서만 가능
- 실무에서는 단위테스트+통합테스트를 모두 병행해야 안전
- 실전 코드/패턴/트러블슈팅을 반드시 참고해서 작성할 것 