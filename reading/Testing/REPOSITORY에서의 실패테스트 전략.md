---
tags:
  - study
  - TDD
  - test
---

# Repository Failure Case Testing 전략

> 본 문서는 **TDD 학습자**가 Repository 계층 단위 테스트를 설계할 때 고려해야 할 실패-케이스와 검증 포인트를 체계적으로 정리합니다. 서비스 계층의 *비즈니스 유효성 검사* 와 달리, Repository 계층은 **"데이터를 어떻게 안전하게 저장·조회하느냐"** 에 초점을 맞춥니다.

---

## 1. Repository 계층의 책임 다시보기

| 구분 | 책임 | 예시 |
|------|------|------|
| **비즈니스(서비스)** | *무엇을/왜* 저장·조회할지 결정 | "음수 금액이면 저장 불가", "중복 충전은 1회만 허용" |
| **저장소(Repository)** | 데이터를 *어떻게* 안전하게 **영속화**·**조회**·**갱신**할지 보장 | 트랜잭션, 동시성 제어, 무결성, 예외 변환 |

따라서 Repository 테스트는 **기술적 실패** 와 **동시성·경계조건** 위주로 설계합니다.

---

## 2. 실패-케이스 카테고리 & 왜 검증해야 하나?

| # | 카테고리 | 검증 이유 | 예시 시나리오 | 간단한 테스트 아이디어 |
|---|-----------|-----------|---------------|-----------------------|
| 1 | **존재하지 않는 데이터 조회** | 호출자가 `null`(또는 Empty Object) 처리 로직을 갖고 있다고 전제. Repository 가 *예상된 기본 응답*을 주는지 확인 | `findById(999)` → 빈 객체 | 빈 객체의 필드가 모두 기본값인지 검증 |
| 2 | **중복 PK / Unique 제약** | 중복 저장 시 남는 데이터가 명확해야 한다. *덮어쓰기* vs *예외* 정책 선명화 | 동일 `userId` 두 번 `save` | 마지막 값만 남는지 or 예외 발생하는지 |
| 3 | **동시성 & Race Condition** | 멀티스레드 환경에서 데이터 손실/스키ュー 방지 | 100 개의 병행 `save` | 모든 쓰기가 반영됐는지 카운트 확인 |
| 4 | **업데이트 실패 (행 개수 0)** | Optimistic-Lock 시 버전 불일치 → Silent 실패 방지 | `updateWhereVersionNotMatch` | 업데이트 안 됐으면 예외 또는 재시도 반환 |
| 5 | **데이터 무결성/타입 한계** | DB 칼럼 Range, `BIGINT` Overflow 등 | `amount = Long.MAX_VALUE + 1` | 예외 발생 또는 값 Clip 확인 |
| 6 | **트랜잭션 롤백 & 예외 매핑** | DB 예외를 서비스가 이해할 도메인 예외로 변환 | `SQLIntegrityConstraintViolationException` → `DuplicatePointHistoryException` | try/catch 후 기대 예외 타입 검증 |
| 7 | **타임아웃/느린 쿼리** | SLA 보장 & Circuit Breaker 연동 | DB 5초 Sleep | `TimeoutException` 발생 여부 |
| 8 | **외부 의존 장애(DB Down)** | Fail Fast / 재시도 정책 검증 | DB 연결 끊김 | 커넥션 예외 전파·랩핑 검증 |
| 9 | **대량 Batch 처리 한계** | 메모리/커넥션 폭주 방지 | 1백만건 Bulk Insert | 수행 시간/메모리 사용량 측정 & Assert |
| 10 | **정렬·페이징 오류** | 쿼리 `ORDER BY` 누락, OFFSET 계산 오류 | historyId DESC인데 ASC로 나옴 | 첫 Row 값이 가장 최신 ID 인지 검증 |

> 실제 프로젝트에서 **1,2,3,10** 번은 In-Memory 구조여도 손쉽게 재현 가능하므로 반드시 포함하세요.

---

## 3. 테스트 작성 전략

1. **Given / When / Then** 주석으로 역할 명확화  
2. **ParameterizedTest**로 입력 범위를 넓히고 중복 테스트 방지  
3. **동시성 테스트** 
   ```java
   ExecutorService pool = Executors.newFixedThreadPool(100);
   CountDownLatch latch = new CountDownLatch(100);
   for (int i = 0; i < 100; i++) {
       pool.submit(() -> {
           repository.save(i, 100L);
           latch.countDown();
       });
   }
   latch.await();
   ```
4. **실제 DB vs 인메모리**
   - **In-Memory**: 빠르지만 트랜잭션 & Lock 테스트 한계  
   - **TestContainers**: Real DB 행동 재현, 예외·제약조건 검증 가능  
5. **예외 메시지/타입 명확화**: 기대 예외를 정확히 명시해 *“왜 실패했는지”* 문서화

---

## 4. 예제 실패 테스트 스케치

```java
@Test
@DisplayName("실패: 동일 ID 두 번 저장 시 마지막 값만 남는다")
void save_duplicateId_overwrites() {
    // given
    long userId = 1L;
    repository.save(userId, 100L);

    // when
    repository.save(userId, 200L);

    // then
    assertThat(repository.findById(userId).point()).isEqualTo(200L);
}
```

---

## 5. 추가로 학습하면 좋은 키워드

| 키워드 | 간단 설명 |
|--------|-----------|
| **Optimistic Lock** | 버전 번호(Version) 기반 동시성 제어. 충돌 시 예외 → 재시도 필요 |
| **Pessimistic Lock** | DB Row 에 Lock 을 걸어 충돌 방지. 동시성 낮지만 안전 |
| **Isolation Level** | READ_COMMITTED, REPEATABLE_READ 등이 트랜잭션 간 간섭을 어떻게 막는지 |
| **ACID** | Atomicity, Consistency, Isolation, Durability – 트랜잭션 4대 특성 |
| **Idempotency** | 요청을 여러 번 보내도 결과가 동일(부작용 없음) 함을 보장 |
| **TestContainers** | Docker 로 실 DB 띄워 통합/단위 테스트 할 수 있는 라이브러리 |
| **Circuit Breaker** | 외부 시스템 장애 시 빠르게 실패해 시스템 전체 장애 확산 방지 |
| **Bulkhead Pattern** | 리소스 풀 분리로 장애 전파를 격리 |
| **Flyway / Liquibase** | DB 스키마 마이그레이션 관리 툴; 제약조건 테스트에도 유용 |
| **Profile 기반 설정** | `application-test.yml` 로 테스트용 DB, 커넥션 풀을 분리 관리 |

---

### 결론
Repository 실패-케이스 테스트는 **“기술적 안전망”** 을 검증하여 서비스 계층이 *비즈니스 로직*에만 집중할 수 있게 해줍니다. 위 표를 체크리스트로 삼아, 각 프로젝트 상황에 맞는 실패 시나리오를 꾸준히 추가·보완해 보세요. TDD 사이클 (Red → Green → Refactor) 내에서 **실패 시나리오 먼저** 쓰는 습관이 버그를 극적으로 감소시킵니다. 