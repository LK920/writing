# 자바 동시성 Set 완전정복 (실무/학습 가이드)

---

## 1. HashSet을 동시성 환경에서 쓰면 안되는 이유
- **HashSet은 동기화(스레드 안전) 보장 X**
    - 여러 스레드가 동시에 add/remove 등 변경 작업 시 데이터 손상, ConcurrentModificationException, 무한루프 등 심각한 버그 발생
    - 예: 두 스레드가 동시에 add하면 내부 구조가 꼬여서 데이터 유실/중복/예외 발생
- **실무에서 HashSet을 멀티스레드 환경에 그냥 쓰는 건 금지**

---

## 2. 동시성 Set 주요 종류/패턴

### 2.1. Collections.synchronizedSet(new HashSet<>())
- **설명:**
    - 기존 HashSet을 감싸서 모든 메서드에 synchronized(락) 걸어줌
    - JDK 표준, 가장 간단하게 쓸 수 있는 동기화 Set
- **장점:**
    - 구현/이해 쉽고, 기존 코드와 호환성 높음
    - 작은 규모/낮은 동시성에서는 충분히 안전
- **단점:**
    - 모든 메서드에 락 → 성능 저하(특히 동시 add/remove 많을 때)
    - iterator 사용 시 별도 락 필요 (아래 예시 참고)
- **실무 팁:**
    - 반복문 돌릴 때 반드시 `synchronized(set) { for(...) ... }`로 감싸야 안전
    - 대량 동시성/성능 중요하면 다른 Set 고려
- **예시:**
    ```java
    Set<String> set = Collections.synchronizedSet(new HashSet<>());
    // add/remove는 안전
    set.add("a");
    // 반복문은 반드시 락!
    synchronized(set) {
        for (String s : set) { ... }
    }
    ```

### 2.2. ConcurrentSkipListSet
- **설명:**
    - 내부적으로 SkipList(정렬된 트리 구조) 기반, JDK에서 제공하는 진짜 동시성 Set
    - 모든 연산이 lock-free/비교적 빠름, 정렬 지원
- **장점:**
    - iterator도 동시성 안전 (fail-safe)
    - add/remove/contains 등 대부분 lock-free, 성능 우수
    - 정렬 필요할 때 유용
- **단점:**
    - 메모리 사용량 많음, HashSet보다 느릴 수 있음(특히 정렬 불필요할 때)
    - null 저장 불가
- **실무 팁:**
    - 동시성+정렬 필요하면 무조건 이거
    - 단순 중복제거/순서 상관없으면 HashSet 기반이 더 빠름
    - 대량 데이터/성능 중요하면 벤치마크 필수
    - 예시:
    ```java
    Set<String> set = new ConcurrentSkipListSet<>();
    set.add("a");
    for (String s : set) { ... } // 별도 락 필요 없음
    ```

### 2.3. CopyOnWriteArraySet
- **설명:**
    - 내부적으로 CopyOnWriteArrayList 기반, add/remove 시마다 전체 배열 복사
    - 읽기(read) 위주 환경에서 매우 빠름
- **장점:**
    - iterator 완전 안전, 락 필요 없음
    - 읽기 많은 환경에서 성능 우수
- **단점:**
    - add/remove 등 변경 연산이 매우 느림(전체 복사)
    - 대량 데이터/변경 많은 환경엔 부적합
- **실무 팁:**
    - 거의 읽기만 하고, 변경은 드물 때만 사용
    - 예시:
    ```java
    Set<String> set = new CopyOnWriteArraySet<>();
    set.add("a");
    for (String s : set) { ... } // 락 필요 없음
    ```

---

## 3. 각 방식의 단점 극복/실무 대안

- **synchronizedSet 단점(전체 락, iterator 별도 락)**
    - 대량 동시성/성능 중요: ConcurrentSkipListSet, ConcurrentHashMap.newKeySet() 등으로 대체
    - 반복문 돌릴 때 반드시 synchronized 블록 사용
- **ConcurrentSkipListSet 단점(정렬 불필요, 메모리/성능)**
    - 정렬 필요 없으면 ConcurrentHashMap.newKeySet() 추천
- **CopyOnWriteArraySet 단점(변경 느림)**
    - 변경 많으면 절대 쓰지 말 것, 읽기 위주에만 사용

---

## 4. 실무에서 자주 쓰는 동시성 Set 패턴

### 4.1. ConcurrentHashMap.newKeySet()
- **설명:**
    - JDK8+에서 지원, 내부적으로 ConcurrentHashMap의 keySet view
    - 동시성/성능/메모리 효율 모두 우수, 정렬 불필요한 Set에 최적
- **장점:**
    - add/remove/contains 모두 lock-free, 매우 빠름
    - iterator도 동시성 안전 (fail-safe)
    - null 저장 불가
- **예시:**
    ```java
    Set<String> set = ConcurrentHashMap.newKeySet();
    set.add("a");
    for (String s : set) { ... } // 락 필요 없음
    ```
- **실무 팁:**
    - 실무에서 동시성 Set 필요하면 거의 이거 씀 (정렬 필요 없을 때)

---

## 5. 실무적으로 꼭 알아야 할 주의점/패턴

- HashSet은 멀티스레드 환경에서 절대 단독 사용 금지
- synchronizedSet은 반복문 돌릴 때 반드시 synchronized 블록 사용
- CopyOnWriteArraySet은 변경 적고 읽기 많은 환경에서만
- ConcurrentSkipListSet은 정렬 필요할 때만
- ConcurrentHashMap.newKeySet()은 정렬 불필요한 동시성 Set의 실무 표준
- null 값은 어떤 동시성 Set에서도 저장 불가 (NPE 발생)
- 동시성 Set도 복잡한 트랜잭션/원자적 복합 연산이 필요하면 별도 락/동기화 로직 추가 필요

---

## 6. 요약/실무 선택 가이드

| 용도/상황                | 추천 Set                        |
|--------------------------|---------------------------------|
| 단순 동시성, 정렬 불필요 | ConcurrentHashMap.newKeySet()   |
| 동시성 + 정렬 필요       | ConcurrentSkipListSet           |
| 읽기 위주, 변경 적음     | CopyOnWriteArraySet             |
| 기존 코드 호환/간단      | Collections.synchronizedSet      |

- newKeySet() > SkipListSet > CopyOnWrite 순으로 더 장점이 많음
- 각 방식의 한계/특성을 반드시 이해하고, 상황에 맞게 선택할 것 

---

## 7. 실무에서 '정렬'이 필요한 이유와 데드락 방지 패턴

### 7.1. 정렬이 필요한 대표적 상황
- **여러 리소스(락, 계좌, 파일 등)를 동시에 처리할 때**
    - 예: 여러 계좌 이체, 여러 파일 동시 접근, 여러 객체 락 동시 획득 등
- **순서가 중요한 데이터 처리(예: 우선순위 큐, 타임라인, 정렬된 결과 필요)**
- **동일한 순서로 반복/처리해야 일관성 보장되는 경우**

### 7.2. 데드락 방지와 정렬의 관계
- **여러 스레드가 여러 리소스를 동시에 락(lock) 걸 때, 락 획득 순서가 다르면 데드락 발생 가능**
    - 예: 스레드A가 1→2, 스레드B가 2→1 순서로 락을 걸면 서로 대기하다가 영원히 멈춤
- **해결법: 항상 '정렬된 순서'로 락을 획득**
    - 모든 스레드가 리소스 id, 객체 hash, 이름 등 기준으로 정렬해서 락을 건다
    - 대표 패턴: 락 걸기 전 Set/리스트를 정렬(예: new TreeSet, ConcurrentSkipListSet 등)
- **실제 예시:**
    ```java
    // 여러 계좌를 동시에 이체할 때, deadlock 방지용 정렬
    List<Account> accounts = ...;
    // 계좌 id 기준 정렬
    accounts.sort(Comparator.comparing(Account::getId));
    for (Account acc : accounts) {
        acc.lock(); // 항상 같은 순서로 락
    }
    // ... 이체 처리 ...
    for (Account acc : accounts) {
        acc.unlock();
    }
    ```
    - 위 패턴을 동시성 Set으로 구현할 때 ConcurrentSkipListSet 사용

### 7.3. 실무에서 ConcurrentSkipListSet이 정렬 때문에 쓰이는 케이스
- **여러 리소스/락/ID를 항상 정렬된 순서로 관리해야 할 때**
- **동시성 환경에서 반복문(for-each) 돌릴 때도 항상 정렬된 순서 보장**
- **예시:**
    - deadlock 방지용 락 id 집합
    - 우선순위가 중요한 작업 큐
    - 타임스탬프/ID 순서대로 처리해야 하는 이벤트 집합

### 7.4. 요약
- 정렬이 필요한 대표적 이유는 '데드락 방지'와 '일관된 순서 보장'임
- 단순히 중복제거/동시성만 필요하면 정렬 불필요(ConcurrentHashMap.newKeySet)
- 여러 리소스 락, 우선순위, 순차처리 등 '순서'가 중요하면 반드시 정렬 지원 Set(ConcurrentSkipListSet 등) 사용 

---

### 7.5. 실무 공식: 언제 정렬이 '필수'고 언제 '불필요'한가?

- **정렬이 반드시 필요한 경우**
    - 동시에 **2개 이상 락(리소스)을 잡아야 할 때**
        - 예: 계좌 2개 이상 이체, 여러 파일/객체 동시 접근 등
        - → 데드락 방지 위해 항상 같은 기준(정렬된 순서)으로 락을 잡아야 함
- **정렬이 필요 없는 경우**
    - 락이 1개만 필요할 때(단일 계좌, 단일 파일 등)
    - 단순히 Set에 값만 넣고 빼는 등, 순서가 전혀 중요하지 않을 때
    - 단순 동시성 처리(ConcurrentHashMap.newKeySet 등)
- **실무 공식 요약**
    - “동시에 잡는 락이 2개 이상이면 무조건 정렬”
    - “1개면 신경 안 써도 됨”
- **이 원칙만 지키면 데드락 걱정 거의 없음!** 

---

## 8. 실전 선택 사례 및 의사결정 가이드 (대화 요약)

### 8.1. 테스트/구현에서 어떤 Set을 왜 선택해야 하는가?

- **실제 구현(서비스/레포지토리 등):**
    - 동시성 제어(1건만 처리, 나머지 중복 에러)는 반드시 실제 구현에만 있어야 함
    - 실무적으로는 `ConcurrentHashMap.newKeySet()`이 lock-free, 반복문 안전, 성능 우수라서 더 권장됨
- **테스트 코드:**
    - 테스트에서는 동시성 Set(newKeySet 등)을 직접 사용하지 말 것
    - 테스트는 실제 구현의 동작(동시 요청 시 1건만 처리, 나머지는 중복 에러)을 검증만 해야 함
    - 테스트에서 Mock/Stub/임시 Set으로 동시성 제어를 시뮬레이션하면 잘못된 테스트가 됨
- **최종 결론:**
    - 동시성 Set(`ConcurrentHashMap.newKeySet()` 등)은 반드시 실제 구현에만 사용
    - 테스트는 동시성 Set을 직접 쓰지 않고, 실제 구현의 동작을 검증하는 데 집중
    - 더 장점이 많은 방식: "동시성 제어는 실제 구현에, 테스트는 검증만"

### 8.2. synchronizedSet과 newKeySet의 차이와 선택 기준

- **synchronizedSet(new HashSet<>())**
    - 모든 메서드에 락, 반복문은 별도 락 필요
    - add/remove만 쓸 때는 안전하지만, 반복문/복합 연산/성능 요구에는 한계
    - 테스트/레거시/간단한 코드에선 쓸 수 있음
- **ConcurrentHashMap.newKeySet()**
    - lock-free, 반복문도 안전, 성능 우수
    - 실무/프로덕션/확장성/성능까지 고려하면 이게 표준
    - 테스트와 실제 구현 모두 이걸 쓰는 게 가장 안전

### 8.3. 실무적 권장사항

- **동시성 컬렉션(newKeySet 등)은 실제 구현에만 사용**
- **테스트는 동시성 제어를 직접 구현하지 않고, 실제 구현의 동작을 검증**
- **테스트에서 동시성 Set을 직접 쓰면 잘못된 테스트가 됨**
- **실제 구현과 테스트의 책임 분리(구현=동시성 제어, 테스트=검증)를 명확히 할 것**

### 8.4. 결론 요약

- “동시성 Set(`ConcurrentHashMap.newKeySet()` 등)은 실제 구현에만 사용하고, 테스트는 반드시 그 동작(1건만 처리, 나머지는 중복 에러)을 검증해야 한다.”
- “테스트에서 동시성 Set을 직접 쓰면 잘못된 테스트가 된다.”
- “실제 구현과 테스트의 책임 분리, 동작 일치성 원칙을 반드시 지킬 것.”

---

## 9. 예시 코드/설명 (실무적 책임 분리)

### 9.1. 실제 구현 예시 (동시성 Set 사용)
```java
// 서비스/레포지토리 등 실제 구현에서만 동시성 Set 사용
private final Set<Long> processingUserIds = ConcurrentHashMap.newKeySet();

public void charge(long userId, long amount) {
    if (!processingUserIds.add(userId)) {
        throw new DuplicateRequestException(ErrorCode.DUPLICATE_REQUEST.getMessage());
    }
    try {
        // 실제 충전 로직
    } finally {
        processingUserIds.remove(userId);
    }
}
```

### 9.2. 테스트 예시 (동시성 Set 직접 사용 금지)
```java
// 테스트에서는 동시성 Set을 직접 사용하지 않고, 실제 구현의 동작을 검증
@DisplayName("동일 userId로 여러 스레드가 동시에 요청 시 1건만 처리, 나머지는 중복 요청 에러 발생")
@Test
void 동시성_중복요청_테스트() throws Exception {
    // given: 여러 스레드 준비
    // when: 동시에 charge 호출
    // then: 1건만 성공, 나머지는 ErrorCode.DUPLICATE_REQUEST 예외 발생 검증
}
```

---

## 10. 실무 공식 요약

- 동시성 Set(newKeySet 등)은 반드시 실제 구현에만 사용
- 테스트는 동시성 Set을 직접 쓰지 않고, 실제 구현의 동작(1건만 처리, 나머지는 중복 에러)을 검증
- 테스트에서 동시성 Set을 직접 쓰면 잘못된 테스트가 됨
- 실제 구현과 테스트의 책임 분리, 동작 일치성 원칙을 반드시 지킬 것 