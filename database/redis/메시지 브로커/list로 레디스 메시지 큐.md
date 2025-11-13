

## list의 LPUSHX/RPUSHX 기능 (조건부 PUSH)

키가 **이미 존재할 경우에만** 데이터를 추가하는 방식입니다.

### 일반 PUSH vs X 계열 PUSH 비교

```
┌─────────────────────────────────────────────────────────────┐
│  LPUSH/RPUSH (일반)                                          │
├─────────────────────────────────────────────────────────────┤
│  • 키가 없으면 → 새로운 리스트 생성 후 추가                          │
│  • 키가 있으면 → 데이터 추가                                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  LPUSHX/RPUSHX (조건부)                                       │
├─────────────────────────────────────────────────────────────┤
│  • 키가 없으면 → 아무 작업도 하지 않음 (0 반환)                      │
│  • 키가 있으면 → 데이터 추가                                      │
└─────────────────────────────────────────────────────────────┘
```

### 예시

```bash
# Case 1: 키가 존재하지 않는 경우
LPUSH myqueue "item1"     # → (integer) 1  (새 리스트 생성)
LPUSHX newqueue "item2"   # → (integer) 0  (아무 일도 안 일어남)

# Case 2: 키가 존재하는 경우
LPUSH myqueue "item1"     # → (integer) 1
LPUSHX myqueue "item2"    # → (integer) 2  (정상 추가됨)
```

### 사용 이유

```
┌──────────────────────────────────────────────────────────────┐
│  Before (애플리케이션에서 확인)                                   │
├──────────────────────────────────────────────────────────────┤
│  1. EXISTS myqueue 실행                                       │
│  2. 결과 확인                                                  │
│  3. LPUSH myqueue "data" 실행                                 │
│     ➜ 2번의 Redis 통신 필요                                     │
│     ➜ Race Condition 가능성                                    │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  After (LPUSHX 사용)                                          │
├──────────────────────────────────────────────────────────────┤
│  1. LPUSHX myqueue "data" 실행                                │
│     ➜ 1번의 Redis 통신                                         │
│     ➜ 원자성(Atomic) 보장                                       │
│     ➜ 애플리케이션 로직 단순화                                     │
└──────────────────────────────────────────────────────────────┘
```
## list 블로킹 기능

### 폴링 방식의 문제점

**이벤트 루프**: 이벤트 큐에서 새 이벤트가 있는지 반복해서 체크
**폴링**: 정해진 시간 동안 대기한 뒤 다시 확인하는 과정

```
┌─────────────────────────────────────────────────────────────┐
│  전통적인 폴링 방식                                            │
└─────────────────────────────────────────────────────────────┘

Consumer                           Redis Queue
   │                                    │
   ├──── LPOP queue:a ─────────────────>│  (비어있음)
   │<────── nil ────────────────────────┤
   │                                    │
   │  (0.5초 대기)                      │
   │                                    │
   ├──── LPOP queue:a ─────────────────>│  (비어있음)
   │<────── nil ────────────────────────┤
   │                                    │
   │  (0.5초 대기)                      │
   │                                    │
   ├──── LPOP queue:a ─────────────────>│  (데이터 있음!)
   │<────── "message" ──────────────────┤

   ❌ 문제점:
      • 불필요한 Redis 호출 발생
      • 즉시 처리 불가능 (최대 0.5초 지연)
      • 네트워크 및 CPU 자원 낭비
```

### 블로킹 방식의 장점

**BLPOP/BRPOP** 커맨드를 사용하면 데이터가 들어올 때까지 대기하다가 **즉시 반환**합니다.

```
┌─────────────────────────────────────────────────────────────┐
│  블로킹 방식 (BLPOP 사용)                                      │
└─────────────────────────────────────────────────────────────┘

Consumer                           Redis Queue
   │                                    │
   ├──── BLPOP queue:a 10 ─────────────>│  (비어있음)
   │                                    │
   │  (연결 유지, 대기 중...)            │
   │                                    │
   │                     Producer       │
   │                        │           │
   │                        ├─ LPUSH ──>│  "message"
   │<────── "message" ──────────────────┤  (즉시 전달!)

   ✅ 장점:
      • 데이터 도착 즉시 처리
      • 불필요한 호출 없음
      • 자원 효율적
```

### BLPOP 동작 방식

```bash
# 문법
BLPOP key [key ...] timeout

# 예시
BLPOP queue:a 10   # queue:a에서 최대 10초 대기
BLPOP queue:a 0    # queue:a에서 무한정 대기
```

#### Case 1: 데이터가 있는 경우

```
Redis List: queue:a
┌─────────┬─────────┬─────────┐
│ "msg3"  │ "msg2"  │ "msg1"  │  ← HEAD (왼쪽)
└─────────┴─────────┴─────────┘

> BLPOP queue:a 10

반환값:
1) "queue:a"      ← 키 이름
2) "msg1"         ← 데이터 값

남은 List:
┌─────────┬─────────┐
│ "msg3"  │ "msg2"  │
└─────────┴─────────┘
```

#### Case 2: 데이터가 없는 경우

```
> BLPOP queue:a 10

Redis는 10초 동안 대기...
┌──────────────────────────────────┐
│  3초 후 데이터 도착!               │
│  LPUSH queue:a "new-msg"         │
└──────────────────────────────────┘

즉시 반환:
1) "queue:a"
2) "new-msg"

만약 10초 동안 데이터가 안 오면:
(nil)
```

### 여러 클라이언트 동시 블로킹

```
┌─────────────────────────────────────────────────────────────┐
│  여러 Consumer가 동일한 큐를 대기                              │
└─────────────────────────────────────────────────────────────┘

Redis Queue (비어있음)
        │
        │ BLPOP queue:a 0
        ├────── Consumer 1 (대기 중...) ← 첫 번째 도착
        │
        │ BLPOP queue:a 0
        ├────── Consumer 2 (대기 중...) ← 두 번째 도착
        │
        │ BLPOP queue:a 0
        ├────── Consumer 3 (대기 중...) ← 세 번째 도착
        │
        │
        │ LPUSH queue:a "message1"
        ├────── Producer
        │
        └────> Consumer 1 받음! (FIFO)

        │ LPUSH queue:a "message2"
        ├────── Producer
        │
        └────> Consumer 2 받음!

        │ LPUSH queue:a "message3"
        ├────── Producer
        │
        └────> Consumer 3 받음!
```

**중요**: 큐 자체는 잠기지 않으며, 가장 먼저 대기한 클라이언트가 우선순위를 가집니다.

### 블로킹 커맨드 반환값

BLPOP/BRPOP은 **항상 2개의 값**을 배열로 반환합니다:

```bash
# 단일 큐 대기
> BLPOP queue:a 5
1) "queue:a"      # 어느 큐에서 받았는지
2) "my-message"   # 실제 데이터

# 여러 큐 대기 (우선순위 순서로)
> BLPOP queue:a queue:b queue:c 5

# queue:b에 먼저 데이터가 들어왔다면:
1) "queue:b"      # 어느 큐에서 받았는지 알려줌
2) "message"      # 실제 데이터
```
## 리스트를 이용한 원형 큐 (Circular Queue)

특정 아이템을 **계속해서 순환하며 접근**해야 하는 경우, 원형 큐 방식으로 처리할 수 있습니다.

### RPOPLPUSH 커맨드

**RPOPLPUSH source destination**: 소스 리스트의 오른쪽 끝 요소를 꺼내서 목적지 리스트의 왼쪽 끝에 추가

```bash
# 문법
RPOPLPUSH source destination

# 같은 리스트를 source와 destination으로 사용하면 원형 큐!
RPOPLPUSH myqueue myqueue
```

### 동작 방식 시각화

#### 일반적인 큐 (소모형)

```
초기 상태: tasks
┌────────┬────────┬────────┬────────┐
│ Task1  │ Task2  │ Task3  │ Task4  │
└────────┴────────┴────────┴────────┘

RPOP tasks → Task4 제거
┌────────┬────────┬────────┐
│ Task1  │ Task2  │ Task3  │
└────────┴────────┴────────┘

RPOP tasks → Task3 제거
┌────────┬────────┐
│ Task1  │ Task2  │
└────────┴────────┘

❌ 한번 처리하면 사라짐!
```

#### 원형 큐 (순환형)

```
초기 상태: workers
┌──────────┬──────────┬──────────┬──────────┐
│ Worker1  │ Worker2  │ Worker3  │ Worker4  │  ← HEAD (왼쪽)
└──────────┴──────────┴──────────┴──────────┘
                                        ↑ TAIL (오른쪽)

RPOPLPUSH workers workers
→ Worker4를 오른쪽에서 꺼내서 왼쪽에 추가

┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Worker4  │ Worker1  │ Worker2  │ Worker3  │ Worker4  │
└──────────┴──────────┴──────────┴──────────┴──────────┘

아니, 잘못됐습니다! 올바른 동작:

RPOPLPUSH workers workers
→ Worker4를 반환하고, 왼쪽에 추가

┌──────────┬──────────┬──────────┬──────────┐
│ Worker4  │ Worker1  │ Worker2  │ Worker3  │
└──────────┴──────────┴──────────┴──────────┘
반환: "Worker4"

다시 RPOPLPUSH workers workers
┌──────────┬──────────┬──────────┬──────────┐
│ Worker3  │ Worker4  │ Worker1  │ Worker2  │
└──────────┴──────────┴──────────┴──────────┘
반환: "Worker3"

✅ 계속 순환하면서 모든 항목에 접근 가능!
```

### 실제 사용 예시 1: 라운드 로빈 작업 분배

```bash
# 워커 풀 초기화
RPUSH worker:pool "worker-1" "worker-2" "worker-3"

┌──────────┬──────────┬──────────┐
│ worker-1 │ worker-2 │ worker-3 │
└──────────┴──────────┴──────────┘

# 작업 1 할당
> RPOPLPUSH worker:pool worker:pool
"worker-3"  ← 이 워커에 작업 할당

┌──────────┬──────────┬──────────┐
│ worker-3 │ worker-1 │ worker-2 │
└──────────┴──────────┴──────────┘

# 작업 2 할당
> RPOPLPUSH worker:pool worker:pool
"worker-2"  ← 이 워커에 작업 할당

┌──────────┬──────────┬──────────┐
│ worker-2 │ worker-3 │ worker-1 │
└──────────┴──────────┴──────────┘

# 작업 3 할당
> RPOPLPUSH worker:pool worker:pool
"worker-1"  ← 이 워커에 작업 할당

┌──────────┬──────────┬──────────┐
│ worker-1 │ worker-2 │ worker-3 │
└──────────┴──────────┴──────────┘

# 계속 순환...
```

### 실제 사용 예시 2: 신뢰성 있는 큐 (Reliable Queue)

작업을 처리 중 실패하더라도 복구할 수 있는 패턴:

```
┌─────────────────────────────────────────────────────────────┐
│  Processing Queue Pattern                                    │
└─────────────────────────────────────────────────────────────┘

작업 대기 큐: tasks:pending
┌────────┬────────┬────────┐
│ Task A │ Task B │ Task C │
└────────┴────────┴────────┘

처리 중 큐: tasks:processing
(비어있음)


1. 작업 가져오기
> RPOPLPUSH tasks:pending tasks:processing

tasks:pending               tasks:processing
┌────────┬────────┐        ┌────────┐
│ Task A │ Task B │        │ Task C │
└────────┴────────┘        └────────┘

2. Task C 처리 시도...
   ✅ 성공 → LREM tasks:processing 1 "Task C"

   tasks:processing
   (비어있음)

   또는

   ❌ 실패 (서버 다운 등)
   → tasks:processing에 Task C가 남아있음!
   → 다른 워커가 복구 가능
```

### BRPOPLPUSH: 블로킹 버전

**BRPOPLPUSH**는 RPOPLPUSH의 블로킹 버전입니다:
- **source**: 데이터를 가져올 원본 리스트의 키
- **destination**: 데이터를 이동시킬 목적지 리스트의 키
- **timeout**: 대기 시간 (0 = 무한정 대기)

**동작**: source 리스트의 **오른쪽 끝(RPOP)** 에서 데이터를 꺼내서 destination 리스트의 **왼쪽 끝(LPUSH)** 에 추가하고, 그 값을 반환합니다.

```bash
# 문법
BRPOPLPUSH source destination timeout

# 예시: source에 데이터가 없으면 대기, 있으면 즉시 이동 & 반환
BRPOPLPUSH tasks:pending tasks:processing 0
```

#### 상세 동작 과정

```
[초기 상태]

tasks:pending (비어있음)
tasks:processing (비어있음)


[Consumer가 BRPOPLPUSH 실행]

> BRPOPLPUSH tasks:pending tasks:processing 0

Consumer: "tasks:pending에 데이터가 올 때까지 기다릴게요..."
          (연결 유지, 블로킹 상태)


[Producer가 데이터 추가]

> LPUSH tasks:pending "Task-X"

tasks:pending
┌─────────┐
│ Task-X  │  ← HEAD (왼쪽)
└─────────┘
     ↑ TAIL (오른쪽)


[BRPOPLPUSH 즉시 실행됨!]

1. tasks:pending의 오른쪽 끝에서 "Task-X"를 꺼냄 (RPOP)
2. tasks:processing의 왼쪽 끝에 "Task-X"를 추가 (LPUSH)
3. "Task-X"를 Consumer에게 반환

tasks:pending               tasks:processing
(비어있음)                  ┌─────────┐
                            │ Task-X  │
                            └─────────┘

Consumer가 받는 값: "Task-X"
```

#### 실전 예시: 안전한 작업 큐

```bash
# 1. Consumer가 작업 대기
> BRPOPLPUSH tasks:pending tasks:processing 0
(대기 중...)

# 2. Producer가 작업 3개 추가
> LPUSH tasks:pending "job1" "job2" "job3"

tasks:pending
┌──────┬──────┬──────┐
│ job3 │ job2 │ job1 │
└──────┴──────┴──────┘

# 3. BRPOPLPUSH가 즉시 실행됨
# job1을 오른쪽에서 꺼내서 processing으로 이동

tasks:pending          tasks:processing
┌──────┬──────┐       ┌──────┐
│ job3 │ job2 │       │ job1 │
└──────┴──────┘       └──────┘

반환값: "job1"

# 4. Consumer가 job1 처리 시작
# 만약 처리 중 서버가 죽으면?
# → job1은 tasks:processing에 남아있음!
# → 다른 워커가 복구 가능

# 5. 처리 성공 시
> LREM tasks:processing 1 "job1"
# tasks:processing에서 제거
```

#### 왜 이렇게 사용할까?

```
┌─────────────────────────────────────────────────────────────┐
│  일반 BLPOP 사용 시                                           │
└─────────────────────────────────────────────────────────────┘

> BLPOP tasks:pending 0
← "job1" 받음

❌ 문제: job1 처리 중 서버 다운 → 작업 유실!


┌─────────────────────────────────────────────────────────────┐
│  BRPOPLPUSH 사용 시                                           │
└─────────────────────────────────────────────────────────────┘

> BRPOPLPUSH tasks:pending tasks:processing 0
← "job1" 받음
+ tasks:processing에 백업 저장됨!

✅ 장점: job1 처리 중 서버 다운 → tasks:processing에서 복구 가능!

# 복구 방법
# 다른 워커가 처리 중인 작업 확인
> LRANGE tasks:processing 0 -1
# 오래된 작업 다시 처리
```

### 사용 사례 요약

| 패턴 | 커맨드 | 사용 사례 |
|------|--------|----------|
| **원형 큐** | `RPOPLPUSH queue queue` | 라운드 로빈 워커 선택, 캐러셀 |
| **안전한 큐** | `RPOPLPUSH pending processing` | 작업 처리 중 실패 대비 |
| **블로킹 원형** | `BRPOPLPUSH queue queue 0` | 이벤트 루프, 실시간 처리 |
