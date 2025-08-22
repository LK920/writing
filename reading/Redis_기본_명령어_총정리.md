# Redis 기본 명령어 총정리

## 📋 목차
1. [실무 사용빈도 높은 명령어 Top 10](#1-실무-사용빈도-높은-명령어-top-10)
2. [데이터 타입별 명령어 분류](#2-데이터-타입별-명령어-분류)
3. [실무 시나리오별 활용법](#3-실무-시나리오별-활용법)
4. [성능과 주의사항](#4-성능과-주의사항)
5. [명령어 조합 패턴](#5-명령어-조합-패턴)

---

## 1. 실무 사용빈도 높은 명령어 Top 10

### 🥇 1순위: SET/GET (기본 캐싱)

**명령어 어원:**
- **SET**: 영어 "설정하다" - 키에 값을 설정
- **GET**: 영어 "가져오다" - 키의 값을 가져옴
- **SETEX**: SET + EXpire - 설정과 동시에 만료시간 지정
- **NX**: Not eXists - 키가 존재하지 않을 때만
- **XX**: eXists - 키가 존재할 때만

**동작 방식:**
```redis
# 가장 기본적인 키-값 저장 (O(1) 시간복잡도)
SET user:1001 "김철수"                     # 메모리에 해시 테이블로 저장
GET user:1001                             # 해시 테이블에서 O(1) 조회

# TTL과 함께 (가장 많이 사용)
SET session:abc123 "user_data" EX 3600    # 설정과 동시에 TTL 지정 (원자적)
SETEX cache:product:1 300 "product_data"  # 위와 동일하지만 별도 명령어

# 조건부 설정 (원자적 연산)
SET counter:daily 0 NX                     # 키가 없을 때만 설정 (분산 락에 활용)
SET config:maintenance "true" XX           # 키가 있을 때만 설정 (업데이트 보장)

# 고급 옵션들
SET key value GET                          # 이전 값 반환하며 새 값 설정
SET key value KEEPTTL                      # 기존 TTL 유지하며 값만 변경
```

**내부 동작 세부사항:**
- Redis는 키를 해시 테이블(Dictionary)에 저장
- 메모리 내 포인터 기반 접근으로 O(1) 성능
- TTL 설정 시 별도 TTL 테이블에 만료시간 저장
- NX/XX 옵션은 원자적으로 체크 후 설정 (race condition 방지)

**실무 활용:**
- 세션 저장 (웹 세션, JWT 토큰)
- API 응답 캐싱 (DB 조회 결과)
- 설정값 저장 (feature flag, 설정 정보)
- 임시 데이터 보관 (OTP, 인증 코드)

### 🥈 2순위: INCR/DECR (카운터)

**명령어 어원:**
- **INCR**: INCRement - 증가시키다 (1씩)
- **DECR**: DECRement - 감소시키다 (1씩)
- **INCRBY**: INCREMENT BY - 지정값만큼 증가
- **DECRBY**: DECREMENT BY - 지정값만큼 감소
- **INCRBYFLOAT**: 실수(floating point) 증가

**동작 방식:**
```redis
# 원자적 증감 (동시성 100% 안전)
INCR page_views                    # 1 증가 (CPU 레벨에서 원자적)
DECR stock:product:1               # 1 감소 (락 없이도 안전)
INCRBY downloads 100               # 100 증가 (32비트/64비트 정수 연산)
DECRBY stock:product:1 5           # 5 감소

# 실무 예시
INCR user:1001:login_count         # 로그인 횟수 (동시 로그인 시에도 안전)
INCR api:requests:2025-08-19       # 일별 API 호출수 (정확한 통계)
DECR coupon:summer:remaining       # 쿠폰 잔여량 (선착순 보장)

# 실수 연산
INCRBYFLOAT product:1001:rating 0.5    # 평점 0.5 증가
```

**내부 동작 세부사항:**
- 키가 없으면 0으로 초기화 후 연산
- 문자열 값을 정수로 파싱 시도, 실패 시 에러
- CPU 레벨의 원자적 연산 (CAS - Compare And Swap)
- 64비트 정수 범위: -9,223,372,036,854,775,808 ~ 9,223,372,036,854,775,807
- 여러 클라이언트가 동시 실행해도 결과 일관성 보장

**Race Condition 방지 예시:**
```redis
# 잘못된 방식 (Race Condition 발생 가능)
GET stock:product:1     # Thread A: 10
GET stock:product:1     # Thread B: 10  
SET stock:product:1 9   # Thread A 실행
SET stock:product:1 9   # Thread B 실행 (잘못된 결과!)

# 올바른 방식 (원자적 연산)
DECR stock:product:1    # Thread A: 10 → 9
DECR stock:product:1    # Thread B: 9 → 8 (정확한 결과!)
```

**실무 활용:**
- 조회수 카운터 (페이지뷰, 비디오 조회수)
- 재고 관리 (선착순 상품, 쿠폰 발급)
- Rate Limiting (API 호출 횟수 제한)
- 통계 집계 (일별/월별 방문자 수)

### 🥉 3순위: HSET/HGET/HMGET (구조화 데이터)

**명령어 어원:**
- **H** prefix: **Hash** - 해시 테이블 (필드-값 쌍)
- **HSET**: Hash SET - 해시에 필드 설정
- **HGET**: Hash GET - 해시에서 필드 조회
- **HMGET**: Hash Multiple GET - 해시에서 여러 필드 조회
- **HGETALL**: Hash GET ALL - 해시의 모든 필드-값 조회
- **HINCRBY**: Hash INCREMENT BY - 해시 필드 값 증가

**동작 방식:**
```redis
# 객체 저장 (가장 유용함) - JSON 대신 Hash 사용
HSET user:1001 name "김철수" age 28 email "kim@example.com"  # O(1) per field
HGET user:1001 name                        # "김철수" - O(1) 조회
HMGET user:1001 name email                 # ["김철수", "kim@example.com"] - O(N) N=필드수
HGETALL user:1001                          # 전체 필드-값 - O(N) 전체 필드수

# 개별 필드 업데이트 (부분 업데이트 가능)
HSET user:1001 last_login "2025-08-19 10:30:00"    # 기존 필드만 업데이트
HINCRBY user:1001 login_count 1            # 숫자 필드 원자적 증가

# 고급 연산들
HDEL user:1001 temp_field                  # 특정 필드 삭제
HEXISTS user:1001 phone                    # 필드 존재 확인
HKEYS user:1001                            # 모든 필드명만 조회
HVALS user:1001                            # 모든 값만 조회
HLEN user:1001                             # 필드 개수
```

**내부 동작 세부사항:**
- Redis Hash는 내부적으로 두 가지 인코딩 사용:
  1. **ziplist** (필드 수 < 512개, 값 크기 < 64바이트): 메모리 효율적
  2. **hashtable** (큰 해시): 빠른 접근을 위한 실제 해시 테이블
- 각 필드는 별도의 키-값으로 저장되어 부분 업데이트 가능
- JSON 문자열 저장 대비 메모리 30-40% 절약 가능

**Hash vs JSON String 비교:**
```redis
# JSON String 방식 (비효율적)
SET user:1001 '{"name":"김철수","age":28,"email":"kim@example.com"}'
# 문제점: 부분 업데이트 불가, 파싱 오버헤드, 메모리 낭비

# Hash 방식 (효율적)
HSET user:1001 name "김철수" age 28 email "kim@example.com"
HSET user:1001 last_login "2025-08-19"     # 기존 데이터 손실 없이 추가
```

**메모리 최적화 설정:**
```redis
# redis.conf 설정으로 ziplist 최적화
hash-max-ziplist-entries 512    # 필드 수 제한
hash-max-ziplist-value 64       # 값 크기 제한 (바이트)
```

**실무 활용:**
- 사용자 프로필 (이름, 나이, 이메일 등 구조화 데이터)
- 상품 정보 (이름, 가격, 재고, 카테고리)
- 설정 그룹 (앱 설정, feature flag 그룹)
- 세션 데이터 (사용자 ID, 권한, 마지막 접근시간)

### 4순위: ZADD/ZRANGE (랭킹 시스템)

**명령어 어원:**
- **Z** prefix: **Z**set 또는 **Z**orted Set - 정렬된 집합 (알파벳 마지막 Z = 최고급)
- **ZADD**: Zset ADD - 정렬된 집합에 멤버 추가
- **ZRANGE**: Zset RANGE - 정렬된 집합 범위 조회 (오름차순)
- **ZREVRANGE**: Zset REVerse RANGE - 역순 범위 조회 (내림차순)
- **ZINCRBY**: Zset INCREMENT BY - 멤버의 점수 증가
- **ZRANK**: Zset RANK - 멤버의 순위 (0부터 시작)

**동작 방식:**
```redis
# 점수 기반 정렬 집합 (Skip List + Hash Table 조합)
ZADD leaderboard 1500 "player1" 1200 "player2" 1800 "player3"  # O(log N)

# 랭킹 조회 (Skip List 덕분에 빠른 범위 조회)
ZRANGE leaderboard 0 9                     # 하위 10명 (낮은 점수부터) - O(log N + M)
ZREVRANGE leaderboard 0 9                  # 상위 10명 (높은 점수부터) - O(log N + M)
ZREVRANGE leaderboard 0 9 WITHSCORES       # 점수와 함께 반환

# 점수 업데이트 (기존 멤버 점수 변경)
ZINCRBY leaderboard 100 "player1"          # player1 점수 100 증가 - O(log N)

# 순위 조회 (Hash Table 덕분에 빠른 멤버 검색)
ZREVRANK leaderboard "player1"             # player1의 순위 (0부터 시작) - O(log N)
ZSCORE leaderboard "player1"               # player1의 현재 점수 - O(1)

# 점수 범위 조회
ZRANGEBYSCORE leaderboard 1000 2000        # 1000~2000점 사이 멤버들 - O(log N + M)
ZCOUNT leaderboard 1000 2000               # 1000~2000점 사이 멤버 수 - O(log N)
```

**내부 동작 세부사항:**
- **Skip List**: 정렬된 순서 유지를 위한 확률적 자료구조
  - 평균 O(log N) 삽입/삭제/검색 성능
  - 연결 리스트의 확장판으로 여러 레벨의 포인터 유지
- **Hash Table**: 멤버명 → 점수 매핑으로 O(1) 점수 조회
- 동일 점수일 때는 멤버명의 사전순으로 정렬 (lexicographical order)

**Skip List 시각화:**
```
Level 3: 1 ---------> 7 ---------> NIL
Level 2: 1 -----> 4 -> 7 ---------> NIL  
Level 1: 1 -> 3 -> 4 -> 7 -> 9 -----> NIL
Level 0: 1 -> 3 -> 4 -> 7 -> 9 -> 12 -> NIL
```

**메모리 효율성:**
```redis
# 설정으로 메모리 최적화 가능
zset-max-ziplist-entries 128     # 엔트리 수가 적을 때 ziplist 사용
zset-max-ziplist-value 64        # 값이 작을 때 ziplist 사용
```

**동일 점수 처리:**
```redis
ZADD ranking 100 "alice" 100 "bob" 100 "charlie"
ZRANGE ranking 0 -1    # ["alice", "bob", "charlie"] - 알파벳 순
```

**실무 활용:**
- 게임 랭킹 (점수 기반 실시간 순위)
- 상품 인기순 (조회수, 판매량 기반)
- 실시간 차트 (음악, 동영상 등)
- 추천 시스템 (유사도 점수 기반)

### 5순위: SADD/SMEMBERS (집합 연산)

**명령어 어원:**
- **S** prefix: **S**et - 집합 (중복 없는 요소들)
- **SADD**: Set ADD - 집합에 멤버 추가
- **SMEMBERS**: Set MEMBERS - 집합의 모든 멤버 조회
- **SISMEMBER**: Set IS MEMBER - 멤버가 집합에 있는지 확인
- **SCARD**: Set CARDinality - 집합의 크기 (원소 개수)
- **SINTER**: Set INTERsection - 교집합
- **SUNION**: Set UNION - 합집합
- **SDIFF**: Set DIFFerence - 차집합

**동작 방식:**
```redis
# 중복 없는 집합 (Hash Table 기반)
SADD tags:article:1 "redis" "database" "nosql"     # O(1) per member
SADD user:1001:interests "programming" "music" "travel"

# 집합 조회
SMEMBERS tags:article:1                    # 모든 태그 - O(N) 주의!
SISMEMBER tags:article:1 "redis"           # "redis" 태그 존재 확인 - O(1)
SCARD tags:article:1                       # 태그 개수 - O(1)
SRANDMEMBER tags:article:1 2               # 랜덤 2개 선택 - O(1)

# 집합 연산 (수학의 집합 연산과 동일)
SINTER user:1001:interests user:1002:interests    # 교집합 (공통 관심사) - O(N*M)
SUNION user:1001:interests user:1002:interests    # 합집합 (전체 관심사) - O(N)
SDIFF user:1001:interests user:1002:interests     # 차집합 (user1만의 관심사) - O(N)

# 집합 연산 후 저장 (결과를 새로운 키에 저장)
SINTERSTORE common:interests user:1001:interests user:1002:interests
SUNIONSTORE all:interests user:1001:interests user:1002:interests
```

**내부 동작 세부사항:**
- Redis Set은 내부적으로 두 가지 인코딩 사용:
  1. **intset** (정수만 저장, 512개 이하): 메모리 효율적 배열
  2. **hashtable** (일반적인 경우): 해시 테이블로 O(1) 연산
- 순서 보장하지 않음 (순서가 중요하면 List 사용)
- 멤버는 유일성 보장 (중복 자동 제거)

**Set vs List vs Sorted Set 비교:**
```redis
# List: 순서 O, 중복 O
LPUSH mylist "A" "B" "A"    # [A, B, A]

# Set: 순서 X, 중복 X  
SADD myset "A" "B" "A"      # {A, B}

# Sorted Set: 순서 O, 중복 X, 점수 필요
ZADD myzset 1 "A" 2 "B"     # [(A,1), (B,2)]
```

**집합 연산 예시:**
```redis
# 사용자 관심사 분석
SADD user:1001:interests "programming" "music" "travel"
SADD user:1002:interests "programming" "art" "travel"

SINTER user:1001:interests user:1002:interests  # ["programming", "travel"]
SDIFF user:1001:interests user:1002:interests   # ["music"]
SUNION user:1001:interests user:1002:interests  # ["programming", "music", "travel", "art"]
```

**메모리 최적화:**
```redis
# redis.conf 설정
set-max-intset-entries 512    # 정수 집합 최적화
```

**주의사항:**
```redis
# SMEMBERS는 O(N) - 큰 집합에서 주의!
SMEMBERS large_set    # 위험: 백만 개 멤버 시 느림

# 대안: SSCAN 사용
SSCAN large_set 0 COUNT 100    # 안전: 배치로 조회
```

**실무 활용:**
- 태그 시스템 (블로그 태그, 상품 카테고리)
- 중복 제거 (방문자 집합, 구매자 목록)
- 관계 관리 (팔로워/팔로잉, 친구 관계)
- 권한 그룹 (사용자 권한, 역할 관리)

### 6순위: LPUSH/RPUSH/LPOP/RPOP (큐/스택)

**명령어 어원:**
- **L** prefix: **L**ist - 연결 리스트 (순서 보장)
- **LPUSH**: List PUSH Left - 리스트 왼쪽 끝에 삽입
- **RPUSH**: List PUSH Right - 리스트 오른쪽 끝에 삽입
- **LPOP**: List POP Left - 리스트 왼쪽 끝에서 제거
- **RPOP**: List POP Right - 리스트 오른쪽 끝에서 제거
- **LRANGE**: List RANGE - 리스트 범위 조회
- **BLPOP**: Blocking LPOP - 대기하며 왼쪽에서 제거

**동작 방식:**
```redis
# 리스트 조작 (Doubly Linked List 기반)
LPUSH queue:tasks "task1" "task2"          # 왼쪽에 추가 (스택 LIFO) - O(1)
RPUSH queue:logs "log1" "log2"             # 오른쪽에 추가 (큐 FIFO) - O(1)

LPOP queue:tasks                           # 왼쪽에서 제거 (LIFO) - O(1)
RPOP queue:logs                            # 오른쪽에서 제거 (FIFO) - O(1)

# 범위 조회
LRANGE queue:tasks 0 -1                    # 전체 조회 - O(S+N) S=start offset
LRANGE recent:actions 0 9                  # 최근 10개 - O(S+N)
LINDEX recent:actions 0                    # 첫 번째 요소 - O(N) 주의!

# 블로킹 연산 (Producer-Consumer 패턴)
BLPOP queue:urgent 10                      # 10초 대기하며 꺼내기 - Blocking
BRPOP queue:messages 0                     # 무한 대기 (timeout=0)

# 리스트 조작
LTRIM recent:logs 0 99                     # 최근 100개만 유지 - O(N)
LINSERT mylist BEFORE "pivot" "new"        # 특정 값 앞에 삽입 - O(N)
LREM mylist 2 "value"                      # 특정 값 2개 제거 - O(N)
```

**내부 동작 세부사항:**
- Redis List는 **Doubly Linked List**(양방향 연결 리스트) 구현
- 양 끝 삽입/삭제는 O(1), 중간 접근은 O(N)
- 내부적으로 두 가지 인코딩:
  1. **ziplist** (작은 리스트): 메모리 효율적 연속 배열
  2. **linkedlist** (큰 리스트): 포인터 기반 연결 리스트

**List 방향성 이해:**
```
HEAD (Left)  <->  [A]  <->  [B]  <->  [C]  <->  TAIL (Right)
               LPUSH        RPUSH
               LPOP         RPOP
```

**스택 vs 큐 구현:**
```redis
# 스택 (LIFO - Last In First Out)
LPUSH stack "A"     # [A]
LPUSH stack "B"     # [B, A]  
LPOP stack          # "B" 반환, [A] 남음

# 큐 (FIFO - First In First Out)  
RPUSH queue "A"     # [A]
RPUSH queue "B"     # [A, B]
LPOP queue          # "A" 반환, [B] 남음
```

**블로킹 연산의 장점:**
```redis
# 폴링 방식 (비효율적)
while True:
    item = LPOP queue
    if item is None:
        sleep(0.1)  # CPU 낭비
    else:
        process(item)

# 블로킹 방식 (효율적)
while True:
    item = BLPOP queue 5  # 5초 대기, CPU 사용량 최소
    if item:
        process(item)
```

**메모리 최적화 설정:**
```redis
# redis.conf
list-max-ziplist-size -2        # 8KB 이하일 때 ziplist 사용
list-compress-depth 0           # 압축 설정 (0=압축안함)
```

**성능 특성:**
- **빠른 연산**: LPUSH, RPUSH, LPOP, RPOP (O(1))
- **느린 연산**: LINDEX, LSET, LINSERT (O(N))
- **범위 조회**: LRANGE는 시작 위치에 따라 성능 차이

**Producer-Consumer 패턴:**
```redis
# Producer (메시지 생산자)
RPUSH job:queue '{"type":"email","to":"user@example.com"}'

# Consumer (메시지 소비자) 
BLPOP job:queue 30    # 30초 대기하며 작업 가져옴
```

**실무 활용:**
- 작업 큐 (백그라운드 잡 처리)
- 최근 활동 기록 (사용자 행동 로그)
- 메시지 큐 (실시간 채팅, 알림)
- 실시간 피드 (타임라인, 뉴스피드)

### 7순위: EXPIRE/TTL (만료 관리)

**명령어 어원:**
- **EXPIRE**: 영어 "만료시키다" - 키에 생존시간 설정
- **TTL**: **T**ime **T**o **L**ive - 남은 생존시간 조회
- **PEXPIRE**: Precision EXPIRE - 밀리초 단위 만료 설정
- **PTTL**: Precision TTL - 밀리초 단위 생존시간 조회
- **EXPIREAT**: EXPIRE AT - 특정 타임스탬프에 만료
- **PERSIST**: 영어 "지속시키다" - TTL 제거하여 영구 보관

**동작 방식:**
```redis
# TTL 설정 (상대 시간)
EXPIRE session:abc123 3600                 # 1시간 후 만료 - O(1)
PEXPIRE cache:temp 5000                    # 5초 후 만료 (밀리초) - O(1)

# TTL 설정 (절대 시간)
EXPIREAT config:daily 1692662400           # Unix 타임스탬프에 만료 - O(1)
PEXPIREAT cache:data 1692662400000         # 밀리초 단위 타임스탬프 - O(1)

# TTL 확인
TTL session:abc123                         # 남은 시간 (초) - O(1)
PTTL cache:temp                            # 남은 시간 (밀리초) - O(1)

# TTL 제거
PERSIST session:abc123                     # 영구 보관으로 변경 - O(1)

# TTL 반환값 의미
TTL mykey
# 양수: 남은 시간 (초)
# -1: 키는 존재하지만 TTL 없음 (영구)
# -2: 키가 존재하지 않음
```

**내부 동작 세부사항:**
- Redis는 별도의 **TTL 테이블**에서 만료시간 관리
- 만료 확인은 **능동적 + 수동적** 방식 조합:
  1. **능동적**: 백그라운드에서 주기적으로 만료된 키 삭제
  2. **수동적**: 키 접근 시 만료 여부 확인 후 즉시 삭제
- TTL 정밀도: 밀리초 단위 지원 (Redis 2.6+)

**TTL 동작 메커니즘:**
```redis
# 1. 키 생성
SET mykey "value"
TTL mykey    # -1 (TTL 없음)

# 2. TTL 설정
EXPIRE mykey 10
TTL mykey    # 10 (10초 남음)

# 3. 시간 경과
# ... 5초 후
TTL mykey    # 5 (5초 남음)

# 4. 만료 후
# ... 5초 더 경과
TTL mykey    # -2 (키 삭제됨)
GET mykey    # (nil)
```

**SET 명령어와 TTL 조합:**
```redis
# 원자적 설정 + TTL (권장)
SET session:123 "data" EX 3600      # 설정과 동시에 TTL 지정
SETEX session:123 3600 "data"       # 위와 동일

# 비원자적 방식 (권장하지 않음)
SET session:123 "data"
EXPIRE session:123 3600             # 두 번의 명령어 (race condition 가능)
```

**TTL과 데이터 타입:**
```redis
# 모든 Redis 데이터 타입에 TTL 적용 가능
SET string_key "value"
EXPIRE string_key 60

HSET hash_key field value
EXPIRE hash_key 60

ZADD zset_key 1 member
EXPIRE zset_key 60

# 주의: HSET 등으로 필드만 변경해도 TTL은 유지됨
TTL hash_key     # 60
HSET hash_key new_field new_value
TTL hash_key     # 여전히 카운트다운 중
```

**TTL 패턴과 최적화:**
```redis
# 슬라이딩 윈도우 TTL
GET session:123
if exists:
    EXPIRE session:123 1800    # 접근할 때마다 30분 연장

# 캐시 워밍 (미리 TTL 연장)
if TTL(cache:key) < 300:       # 5분 미만 남았을 때
    refresh_cache()
    EXPIRE cache:key 3600      # 1시간 연장
```

**만료 이벤트 모니터링:**
```redis
# redis.conf 설정
notify-keyspace-events Ex      # 만료 이벤트 활성화

# 클라이언트에서 만료 이벤트 구독
SUBSCRIBE __keyevent@0__:expired
```

**메모리 관리와 TTL:**
```redis
# maxmemory 정책과 TTL 조합
# redis.conf
maxmemory 1gb
maxmemory-policy allkeys-lru   # TTL 없는 키도 LRU로 삭제
# 또는
maxmemory-policy volatile-ttl  # TTL 있는 키만 삭제 (TTL 짧은 순)
```

**주의사항:**
```redis
# 주의: RENAME 시 TTL 유지
SET key1 "value"
EXPIRE key1 60
RENAME key1 key2
TTL key2        # 여전히 카운트다운

# 주의: 키 업데이트 시 TTL 제거 가능 (명령어별 다름)
SET key "value"
EXPIRE key 60
SET key "new_value"    # TTL 제거됨!
TTL key               # -1
```

**실무 활용:**
- 세션 관리 (로그인 세션 자동 만료)
- 캐시 만료 (API 응답 캐시 갱신)
- 임시 데이터 (OTP 코드, 임시 파일)
- 토큰 유효기간 (JWT, API 키 만료)

### 8순위: EXISTS/DEL (키 관리)

**명령어 어원:**
- **EXISTS**: 영어 "존재하다" - 키의 존재 여부 확인
- **DEL**: **DEL**ete - 키 삭제 (즉시 메모리에서 제거)
- **UNLINK**: 비동기 삭제 (백그라운드에서 메모리 해제)
- **KEYS**: 키 목록 조회 (패턴 매칭)
- **SCAN**: 커서 기반 키 스캔 (안전한 순회)
- **TYPE**: 키의 데이터 타입 조회

**동작 방식:**
```redis
# 키 존재 확인
EXISTS user:1001                           # 1 (존재) 또는 0 (없음) - O(1)
EXISTS user:1001 user:1002 user:1003       # 존재하는 키 개수 반환 - O(N)

# 키 삭제 (동기적 - 즉시 삭제)
DEL cache:old                              # 1 (삭제됨) 또는 0 (없었음) - O(1)
DEL session:abc session:def                # 여러 키 동시 삭제 - O(N)

# 키 삭제 (비동기적 - 백그라운드 삭제)
UNLINK large:hash large:zset               # 큰 데이터 구조 안전하게 삭제 - O(1)

# 키 목록 조회 (운영환경 절대 금지!)
KEYS session:*                             # 패턴 매칭 (전체 DB 스캔) - O(N)
KEYS *                                     # 모든 키 조회 (매우 위험!) - O(N)

# 안전한 키 스캔 (권장)
SCAN 0 MATCH session:* COUNT 100           # 커서 기반 안전한 스캔 - O(1)
SCAN 1234 MATCH user:* COUNT 50            # 이전 커서로 계속 스캔

# 키 정보 조회
TYPE user:1001                             # string, hash, list, set, zset 등
OBJECT ENCODING user:1001                  # 내부 인코딩 정보
```

**내부 동작 세부사항:**
- **EXISTS**: 해시 테이블에서 O(1) 조회
- **DEL**: 즉시 메모리에서 제거, 큰 객체는 블로킹 가능
- **UNLINK**: 키를 unlinking하고 백그라운드 스레드에서 실제 메모리 해제
- **KEYS**: 전체 키 공간을 순회하여 패턴 매칭 (운영환경 금지!)
- **SCAN**: 커서 기반으로 일부씩 스캔 (non-blocking)

**DEL vs UNLINK 비교:**
```redis
# 작은 데이터 - DEL 사용
SET small_key "value"
DEL small_key           # 즉시 삭제, 빠름

# 큰 데이터 - UNLINK 사용 (권장)
HSET big_hash field1 value1 field2 value2 ... # 백만 개 필드
UNLINK big_hash         # 논블로킹, 백그라운드에서 삭제

ZADD big_zset 1 member1 2 member2 ...         # 백만 개 멤버
UNLINK big_zset         # 메인 스레드 블로킹 방지
```

**SCAN 패턴 사용법:**
```redis
# 기본 스캔
redis> SCAN 0
1) "17"        # 다음 커서
2) 1) "key:12"  # 발견된 키들
   2) "key:8"
   3) "key:4"

# 패턴과 함께 스캔
redis> SCAN 0 MATCH session:* COUNT 1000
1) "0"         # 커서 0 = 스캔 완료
2) 1) "session:abc123"
   2) "session:def456"

# Java에서 SCAN 사용 예시
cursor = 0
do {
    ScanResult result = redis.scan(cursor, "session:*", 100);
    for (String key : result.getResult()) {
        processKey(key);
    }
    cursor = result.getCursor();
} while (cursor != 0);
```

**키 패턴 매칭:**
```redis
# KEYS 패턴 (운영환경 금지)
KEYS session:*          # session:으로 시작
KEYS *:temp             # :temp로 끝남
KEYS user:?001          # user:X001 형태 (X는 한 글자)
KEYS cache:[abc]*       # cache:a*, cache:b*, cache:c*로 시작

# SCAN에서도 동일한 패턴 사용 가능
SCAN 0 MATCH user:?001 COUNT 100
```

**메모리 사용량 체크:**
```redis
# 키별 메모리 사용량 (Redis 4.0+)
MEMORY USAGE user:1001                     # 바이트 단위 메모리 사용량
MEMORY USAGE large:hash SAMPLES 10        # 샘플링으로 추정

# 전체 메모리 정보
INFO memory
```

**배치 삭제 패턴:**
```redis
# 안전한 배치 삭제 (Lua Script)
EVAL "
local keys = redis.call('SCAN', 0, 'MATCH', ARGV[1], 'COUNT', ARGV[2])
for i=1, #keys[2] do
    redis.call('DEL', keys[2][i])
end
return #keys[2]
" 0 "session:*" 1000

# 또는 SCAN + 파이프라인 조합 (클라이언트)
```

**조건부 삭제:**
```redis
# 특정 조건일 때만 삭제 (Lua Script)
EVAL "
local value = redis.call('GET', KEYS[1])
if value == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
" 1 mykey "expected_value"
```

**성능 고려사항:**
```redis
# 빠른 연산 (O(1))
EXISTS single_key
DEL single_key
TYPE single_key

# 느린 연산 (운영환경 주의)
KEYS *              # 전체 DB 스캔
DEL large_hash      # 큰 객체 동기 삭제
EXISTS key1 key2 key3 ... key1000  # 대량 키 확인

# 최적화된 대안
SCAN 0 MATCH pattern COUNT 100  # 대신 SCAN 사용
UNLINK large_hash               # 대신 UNLINK 사용
MGET key1 key2 key3             # EXISTS 대신 MGET으로 일괄 확인
```

**실무 활용:**
- 데이터 정리 (만료된 세션, 임시 파일 삭제)
- 조건부 로직 (키 존재 여부에 따른 분기)
- 배치 삭제 (일괄 정리 작업)
- 모니터링 (키 개수, 메모리 사용량 추적)

### 9순위: XADD/XREAD (스트림 - 최신 기능)

**명령어 어원:**
- **X** prefix: e**X**tended (확장된) 또는 strea**X** (스트림의 X)
- **XADD**: Stream ADD - 스트림에 엔트리 추가
- **XREAD**: Stream READ - 스트림에서 엔트리 읽기
- **XGROUP**: Stream GROUP - 컨슈머 그룹 관리
- **XREADGROUP**: Stream READ GROUP - 그룹에서 읽기
- **XACK**: Stream ACKnowledge - 메시지 처리 확인
- **XPENDING**: Stream PENDING - 미처리 메시지 조회

**동작 방식:**
```redis
# 이벤트 스트림 추가 (자동 ID 생성)
XADD events:orders * orderId 1001 userId 123 amount 50000    # O(1)
# 반환: "1692664800000-0" (타임스탬프-시퀀스)

# 이벤트 스트림 추가 (수동 ID 지정)
XADD events:orders 1692664800001-0 orderId 1002 status completed

# 스트림 읽기 (기본)
XREAD STREAMS events:orders 0              # 처음부터 모든 엔트리 읽기
XREAD STREAMS events:orders $              # 새 메시지만 읽기 (현재 시점 이후)
XREAD COUNT 10 STREAMS events:orders 0     # 최대 10개만 읽기

# 블로킹 읽기 (실시간 처리)
XREAD BLOCK 1000 STREAMS events:orders $   # 1초 대기하며 새 메시지 읽기
XREAD BLOCK 0 STREAMS events:orders $      # 무한 대기

# 범위 읽기
XRANGE events:orders - +                   # 전체 범위
XRANGE events:orders 1692664800000 1692664900000  # 시간 범위
XREVRANGE events:orders + - COUNT 10       # 최신 10개 (역순)

# 컨슈머 그룹 생성
XGROUP CREATE events:orders processing $   # 현재 시점부터 처리
XGROUP CREATE events:orders processing 0   # 처음부터 처리

# 컨슈머 그룹에서 읽기
XREADGROUP GROUP processing consumer1 COUNT 5 STREAMS events:orders >
# > : 그룹에서 아직 전달되지 않은 메시지만
# 0 : 특정 컨슈머의 미처리 메시지부터

# 메시지 처리 확인
XACK events:orders processing 1692664800000-0 1692664800001-0
```

**내부 동작 세부사항:**
- **Radix Tree** 기반으로 시간순 정렬된 로그 저장
- 각 엔트리는 고유한 ID (타임스탬프-시퀀스) 가짐
- 메모리 효율적인 압축 저장 (listpack 인코딩)
- Consumer Group은 분산 처리를 위한 메시지 분배 메커니즘

**Stream ID 구조:**
```
1692664800000-0
│            │
│            └─ 시퀀스 번호 (같은 밀리초 내 순서)
└─ Unix 타임스탬프 (밀리초)

특수 ID:
- * : 자동 생성 (현재 시간 + 자동 시퀀스)
- $ : 현재 마지막 ID (새 메시지만 읽을 때)
- + : 가능한 가장 큰 ID
- - : 가능한 가장 작은 ID
- > : 컨슈머 그룹에서 새 메시지만
```

**Consumer Group 동작 방식:**
```redis
# 1. 컨슈머 그룹 생성
XGROUP CREATE mystream mygroup $

# 2. 여러 컨슈머가 경쟁적으로 메시지 소비
# Consumer 1
XREADGROUP GROUP mygroup consumer1 STREAMS mystream >

# Consumer 2  
XREADGROUP GROUP mygroup consumer2 STREAMS mystream >

# 3. 메시지는 한 번만 한 컨슈머에게 전달됨
# 4. 처리 완료 후 ACK 필요
XACK mystream mygroup 1692664800000-0
```

**장애 처리 및 복구:**
```redis
# 미처리 메시지 조회
XPENDING mystream mygroup                  # 그룹의 전체 미처리 현황
XPENDING mystream mygroup - + 10           # 상세 미처리 메시지 10개

# 다른 컨슈머의 미처리 메시지 재할당
XCLAIM mystream mygroup consumer2 60000 1692664800000-0
# 60초 이상 미처리 메시지를 consumer2에게 재할당

# 실패한 메시지 Dead Letter Queue로 이동
XREADGROUP GROUP mygroup consumer1 STREAMS mystream 0  # 미처리 재시도
```

**Stream vs List 비교:**
```redis
# List (기존 방식)
LPUSH myqueue "message"    # 단순 메시지
BRPOP myqueue 10           # 소비 후 사라짐, 복구 불가

# Stream (새로운 방식)
XADD mystream * msg "message" ts 1692664800  # 구조화된 데이터
XREAD STREAMS mystream $                     # 메시지 보존, 재읽기 가능
```

**메모리 관리:**
```redis
# 스트림 크기 제한 (메모리 보호)
XADD mystream MAXLEN 1000 * field value     # 최대 1000개 엔트리 유지
XADD mystream MAXLEN ~ 1000 * field value   # 대략적 크기 제한 (효율적)

# 수동 정리
XTRIM mystream MAXLEN 1000                  # 오래된 엔트리 삭제
XTRIM mystream MINID 1692664800000-0        # 특정 ID 이전 삭제
```

**실시간 모니터링:**
```redis
# 스트림 정보 조회
XINFO STREAM mystream                       # 스트림 메타데이터
XINFO GROUPS mystream                       # 컨슈머 그룹 정보
XINFO CONSUMERS mystream mygroup            # 컨슈머 상태

# 스트림 길이
XLEN mystream                               # 엔트리 개수
```

**성능 특성:**
```redis
# 매우 빠른 연산
XADD mystream * field value     # O(1) - 끝에 추가
XREAD STREAMS mystream $        # O(1) - 새 메시지만
XLEN mystream                   # O(1) - 길이 조회

# 상대적으로 느린 연산
XRANGE mystream - +             # O(N) - 전체 범위 스캔
XREADGROUP ... COUNT 10000      # O(N) - 대량 메시지 읽기
```

**실무 활용 패턴:**
```redis
# 1. 주문 이벤트 스트림
XADD orders * orderId 1001 status created userId 123 amount 50000
XADD orders * orderId 1001 status paid timestamp 1692664800
XADD orders * orderId 1001 status shipped carrier UPS

# 2. 마이크로서비스 간 통신
XADD user.events * type user.created userId 123 email "user@example.com"
XADD inventory.events * type stock.updated productId 456 quantity 100

# 3. 실시간 로그 수집
XADD app.logs * level ERROR message "DB connection failed" service auth
```

**실무 활용:**
- 이벤트 스트리밍 (도메인 이벤트 발행/구독)
- 메시지 큐 (안정적인 메시지 전달 보장)
- 실시간 로그 (구조화된 로그 수집 및 분석)
- 마이크로서비스 통신 (서비스 간 이벤트 기반 통신)

### 10순위: MULTI/EXEC (트랜잭션)

**명령어 어원:**
- **MULTI**: **MULTI**ple - 여러 명령어 시작 (트랜잭션 시작)
- **EXEC**: **EXEC**ute - 대기 중인 명령어들 실행 (트랜잭션 커밋)
- **DISCARD**: 트랜잭션 취소 (롤백)
- **WATCH**: 키 변경 감시 (낙관적 락)
- **UNWATCH**: 감시 해제

**동작 방식:**
```redis
# 기본 트랜잭션 (원자적 명령어 묶음 실행)
MULTI                          # 트랜잭션 시작
SET account:1001 1000         # 큐에 추가 (실행 안됨)
SET account:1002 2000         # 큐에 추가
INCR transaction:count        # 큐에 추가
EXEC                          # 모든 명령어 원자적 실행

# 트랜잭션 취소
MULTI
SET key1 "value1"
SET key2 "value2"
DISCARD                       # 트랜잭션 취소 (아무것도 실행 안됨)

# 조건부 트랜잭션 (낙관적 락)
WATCH account:1001            # account:1001 키 감시 시작
GET account:1001              # 현재 값 확인: 1000
MULTI
DECRBY account:1001 100       # 1000 - 100 = 900
INCRBY account:1002 100       # 2000 + 100 = 2100
EXEC                          # account:1001이 변경되지 않았으면 실행

# 실행 결과
# - 성공 시: [900, 2100] 반환
# - 실패 시: null 반환 (다른 클라이언트가 account:1001 변경)
```

**내부 동작 세부사항:**
- Redis 트랜잭션은 **단일 스레드**에서 순차 실행
- MULTI~EXEC 사이의 명령어들이 **큐에 저장**됨
- EXEC 시점에 **모든 명령어가 원자적으로 실행**
- 다른 클라이언트의 명령어가 중간에 끼어들 수 없음
- WATCH된 키가 변경되면 트랜잭션 전체가 **자동 취소**

**Redis 트랜잭션 특징 (ACID):**
```redis
# Atomicity (원자성): ✅ 모든 명령어가 함께 성공 또는 실패
MULTI
SET key1 "value1"
SET key2 "value2"
EXEC  # 둘 다 성공하거나 둘 다 실패

# Consistency (일관성): ✅ 데이터 무결성 유지
# Isolation (격리성): ✅ 트랜잭션 중간에 다른 명령어 실행 불가
# Durability (영속성): ⚠️ Redis 설정에 따라 다름 (AOF, RDB)
```

**트랜잭션 vs 일반 명령어:**
```redis
# 일반 명령어 (인터리빙 가능)
Client A: SET account:1001 1000
Client B: GET account:1001        # 1000
Client A: INCR account:1001
Client B: GET account:1001        # 1001

# 트랜잭션 (원자적 실행)
Client A: MULTI
Client A: SET account:1001 1000
Client A: INCR account:1001
Client B: GET account:1001        # 아직 1000 (변경 전)
Client A: EXEC                    # 이제 1001로 변경됨
```

**WATCH를 이용한 낙관적 락:**
```redis
# 계좌 이체 예시 (Race Condition 방지)
def transfer_money(from_account, to_account, amount):
    while True:
        # 1. 키 감시 시작
        WATCH from_account
        
        # 2. 현재 잔액 확인
        balance = GET from_account
        if balance < amount:
            UNWATCH
            return "잔액 부족"
        
        # 3. 트랜잭션 시작
        MULTI
        DECRBY from_account amount
        INCRBY to_account amount
        
        # 4. 실행 시도
        result = EXEC
        if result is not None:
            return "이체 성공"
        # 5. 다른 클라이언트가 잔액을 변경했으면 재시도
```

**파이프라인 vs 트랜잭션:**
```redis
# 파이프라인: 네트워크 최적화 (원자성 보장 X)
PIPELINE:
  SET key1 "value1"
  SET key2 "value2"
  GET key1
SEND  # 모든 명령어를 한 번에 전송 (성능 향상)

# 트랜잭션: 원자성 보장 (네트워크 왕복 필요)
MULTI
SET key1 "value1"
SET key2 "value2"
EXEC  # 원자적 실행 보장
```

**트랜잭션 + 파이프라인 조합:**
```redis
# Redis Cluster에서는 같은 슬롯의 키만 트랜잭션 가능
# 최적화: 파이프라인 + 트랜잭션
PIPELINE:
  MULTI
  SET {user:1001}:balance 1000    # {} 해시 태그로 같은 슬롯 보장
  SET {user:1001}:status active
  EXEC
SEND
```

**에러 처리:**
```redis
# 구문 오류 시 전체 트랜잭션 취소
MULTI
SET key1 "value1"
INVALID_COMMAND  # 구문 오류
SET key2 "value2"
EXEC  # 실행되지 않음, 에러 반환

# 런타임 오류 시 일부만 실패
MULTI
SET key1 "value1"
INCR string_key   # 문자열에 INCR 시도 (런타임 오류)
SET key2 "value2"
EXEC  # [OK, (error), OK] - 부분 실행
```

**Lua Script vs 트랜잭션:**
```redis
# 트랜잭션: 단순한 원자적 실행
MULTI
INCR counter
SET last_update timestamp
EXEC

# Lua Script: 복잡한 로직 + 원자성
EVAL "
local current = redis.call('GET', KEYS[1])
if tonumber(current) > tonumber(ARGV[1]) then
    redis.call('DECR', KEYS[1])
    return 1
else
    return 0
end
" 1 counter 10
```

**성능 특성:**
```redis
# 빠른 연산
MULTI / EXEC               # O(1) - 트랜잭션 관리 오버헤드 최소
WATCH key                  # O(1) - 키 감시 등록

# 주의할 점
WATCH large_amount_keys    # O(N) - 많은 키 감시 시 성능 저하
MULTI + 많은 명령어 + EXEC  # O(N) - 명령어 수에 비례
```

**실무 패턴:**
```redis
# 1. 계좌 이체 (원자적 잔액 변경)
WATCH account:from account:to
MULTI
DECRBY account:from 100
INCRBY account:to 100
SADD transaction:log "transfer:123"
EXEC

# 2. 카운터 + 로그 (원자적 통계 업데이트)
MULTI
INCR page:views
LPUSH page:view_log "timestamp:user_id"
EXPIRE page:view_log 86400
EXEC

# 3. 캐시 무효화 (관련 캐시 일괄 삭제)
MULTI
DEL cache:user:1001
DEL cache:user_posts:1001
DEL cache:user_followers:1001
EXEC
```

**주의사항:**
```redis
# 주의: 트랜잭션 중에는 다른 명령어 실행 불가
MULTI
SET key1 "value1"
GET key1                # 값 확인 불가! 큐에만 추가됨
SET key2 "value2"
EXEC

# 해결: WATCH + 트랜잭션 전에 미리 확인
GET key1                # 먼저 값 확인
MULTI
SET key1 "new_value"
SET key2 "value2" 
EXEC
```

**실무 활용:**
- 계좌 이체 (원자적 금액 변경)
- 일관성 보장 (관련 데이터 동기 업데이트)
- 배치 처리 (여러 연산을 하나로 묶기)
- 원자적 업데이트 (카운터 + 로그, 캐시 무효화)

---

## 2. 데이터 타입별 명령어 분류

### 📝 String (가장 기본)
```redis
# 설정/조회
SET key value [EX seconds] [PX milliseconds] [NX|XX]
GET key
MSET key1 value1 key2 value2              # 다중 설정
MGET key1 key2 key3                       # 다중 조회

# 조작
APPEND key value                          # 문자열 추가
STRLEN key                                # 길이 조회
GETRANGE key start end                    # 부분 문자열
SETRANGE key offset value                 # 부분 수정

# 숫자 연산
INCR key                                  # 1 증가
DECR key                                  # 1 감소
INCRBY key increment                      # 지정값 증가
INCRBYFLOAT key increment                 # 실수 증가
```

### 🗂 Hash (객체 저장)
```redis
# 설정/조회
HSET key field value [field value ...]   # 필드 설정
HGET key field                            # 필드 조회
HMSET key field value [field value ...]  # 다중 필드 설정 (deprecated)
HMGET key field [field ...]              # 다중 필드 조회
HGETALL key                               # 전체 조회

# 조작
HDEL key field [field ...]               # 필드 삭제
HEXISTS key field                         # 필드 존재 확인
HKEYS key                                 # 모든 필드명
HVALS key                                 # 모든 값
HLEN key                                  # 필드 개수

# 숫자 연산
HINCRBY key field increment               # 필드 값 증가
HINCRBYFLOAT key field increment          # 실수 증가
```

### 📊 Sorted Set (랭킹)
```redis
# 추가/수정
ZADD key score member [score member ...]  # 멤버 추가
ZINCRBY key increment member               # 점수 증가

# 조회 (인덱스 기반)
ZRANGE key start stop [WITHSCORES]         # 점수 오름차순
ZREVRANGE key start stop [WITHSCORES]      # 점수 내림차순

# 조회 (점수 기반)
ZRANGEBYSCORE key min max [WITHSCORES]     # 점수 범위 조회
ZREVRANGEBYSCORE key max min [WITHSCORES]  # 점수 범위 역순

# 정보 조회
ZSCORE key member                          # 멤버 점수
ZRANK key member                           # 순위 (오름차순)
ZREVRANK key member                        # 순위 (내림차순)
ZCARD key                                  # 멤버 수
ZCOUNT key min max                         # 점수 범위 내 멤버 수

# 삭제
ZREM key member [member ...]               # 멤버 삭제
ZREMRANGEBYRANK key start stop             # 순위 범위 삭제
ZREMRANGEBYSCORE key min max               # 점수 범위 삭제
```

### 🏷 Set (집합)
```redis
# 추가/삭제
SADD key member [member ...]               # 멤버 추가
SREM key member [member ...]               # 멤버 삭제

# 조회
SMEMBERS key                               # 모든 멤버
SISMEMBER key member                       # 멤버 존재 확인
SCARD key                                  # 멤버 수
SRANDMEMBER key [count]                    # 랜덤 멤버

# 집합 연산
SINTER key [key ...]                       # 교집합
SUNION key [key ...]                       # 합집합
SDIFF key [key ...]                        # 차집합

# 집합 연산 후 저장
SINTERSTORE destination key [key ...]      # 교집합 저장
SUNIONSTORE destination key [key ...]      # 합집합 저장
SDIFFSTORE destination key [key ...]       # 차집합 저장
```

### 📋 List (순서 보장)
```redis
# 추가
LPUSH key element [element ...]            # 왼쪽 추가
RPUSH key element [element ...]            # 오른쪽 추가
LINSERT key BEFORE|AFTER pivot element     # 특정 위치 삽입

# 제거
LPOP key                                   # 왼쪽 제거
RPOP key                                   # 오른쪽 제거
LREM key count element                     # 특정 값 제거
LTRIM key start stop                       # 범위 외 제거

# 조회
LRANGE key start stop                      # 범위 조회
LINDEX key index                           # 인덱스 조회
LLEN key                                   # 길이

# 블로킹 (대기열)
BLPOP key [key ...] timeout                # 대기하며 왼쪽 제거
BRPOP key [key ...] timeout                # 대기하며 오른쪽 제거
```

### 🌊 Stream (이벤트 스트림)
```redis
# 추가
XADD key ID field value [field value ...]  # 엔트리 추가

# 읽기
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] id [id ...]
XRANGE key start end [COUNT count]          # 범위 읽기
XREVRANGE key end start [COUNT count]       # 역순 읽기

# 컨슈머 그룹
XGROUP CREATE key groupname id [MKSTREAM]   # 그룹 생성
XREADGROUP GROUP group consumer [COUNT count] STREAMS key [key ...] ID [ID ...]

# 확인 및 정리
XACK key group ID [ID ...]                 # 처리 확인
XPENDING key group                          # 대기 중인 메시지
XTRIM key MAXLEN count                      # 크기 제한
```

---

## 3. 실무 시나리오별 활용법

### 🛒 전자상거래 시나리오

#### 3.1 상품 정보 캐싱
```redis
# 상품 기본 정보 (Hash)
HSET product:1001 name "iPhone 15" price 1200000 stock 50 category "smartphone"

# 상품 태그 (Set)
SADD product:1001:tags "apple" "smartphone" "5g" "premium"

# 상품 조회수 (String + INCR)
INCR product:1001:views

# 상품 평점 (Sorted Set)
ZADD product:1001:reviews 4.5 "user123" 5.0 "user456" 3.5 "user789"
```

#### 3.2 장바구니 시스템
```redis
# 사용자 장바구니 (Hash)
HSET cart:user123 product:1001 2 product:1002 1 product:1003 3

# 장바구니 TTL (세션 기반)
EXPIRE cart:user123 86400

# 장바구니 상품 수량 조회
HGET cart:user123 product:1001
```

#### 3.3 실시간 랭킹
```redis
# 실시간 판매량 랭킹
ZINCRBY products:bestseller 5 "product:1001"    # 5개 판매
ZINCRBY products:bestseller 3 "product:1002"    # 3개 판매

# TOP 10 베스트셀러 조회
ZREVRANGE products:bestseller 0 9 WITHSCORES
```

### 👤 사용자 관리 시나리오

#### 3.4 세션 관리
```redis
# 로그인 세션 (Hash + TTL)
HMSET session:abc123 userId 1001 loginTime "2025-08-19T10:30:00" role "user"
EXPIRE session:abc123 3600

# 중복 로그인 방지 (Set)
SADD user:1001:sessions "session:abc123" "session:def456"
```

#### 3.5 사용자 활동 로그
```redis
# 최근 활동 (List)
LPUSH user:1001:activities "viewed product:1001" "added to cart" "purchased"
LTRIM user:1001:activities 0 99    # 최근 100개만 유지

# 일별 활동 횟수 (String + INCR)
INCR user:1001:daily:2025-08-19
EXPIRE user:1001:daily:2025-08-19 86400
```

### 📊 분석 및 모니터링 시나리오

#### 3.6 API Rate Limiting
```redis
# 사용자별 API 호출 제한
INCR ratelimit:user123:2025-08-19-14        # 시간대별 카운트
EXPIRE ratelimit:user123:2025-08-19-14 3600 # 1시간 TTL

# 현재 호출 수 확인
GET ratelimit:user123:2025-08-19-14
```

#### 3.7 실시간 통계
```redis
# 동시 접속자 수 (Set)
SADD online:users "user123" "user456" "user789"
SCARD online:users                           # 현재 접속자 수

# 실시간 메트릭 (Hash)
HSET metrics:realtime concurrent_users 1250 active_sessions 890 cpu_usage 75.5
```

---

## 4. 성능과 주의사항

### ⚡ 고성능 명령어 (O(1))
```redis
# 매우 빠른 명령어들
SET key value                    # O(1)
GET key                          # O(1)
HSET key field value            # O(1)
HGET key field                  # O(1)
INCR key                        # O(1)
SADD key member                 # O(1)
SISMEMBER key member            # O(1)
LPUSH key element               # O(1)
RPOP key                        # O(1)
```

### 🐌 주의해야 할 명령어
```redis
# 느린 명령어들 (데이터 크기에 비례)
KEYS *                          # O(N) - 절대 운영환경 금지!
SMEMBERS large_set              # O(N) - 큰 Set은 위험
HGETALL large_hash              # O(N) - 큰 Hash는 위험
ZRANGE large_zset 0 -1          # O(N) - 전체 조회 위험

# 안전한 대안들
SCAN 0 MATCH pattern            # 대신 SCAN 사용
SSCAN key 0                     # Set 순회
HSCAN key 0                     # Hash 순회
ZSCAN key 0                     # Sorted Set 순회
```

### 🛡 운영환경 주의사항

#### 4.1 메모리 관리
```redis
# TTL 항상 설정
SETEX cache:data 3600 "value"   # 1시간 TTL
EXPIRE session:abc123 1800      # 30분 TTL

# 큰 데이터 구조 주의
LTRIM recent:logs 0 999         # List 크기 제한
ZREMRANGEBYRANK ranking 100 -1  # Sorted Set 크기 제한
```

#### 4.2 원자성 보장
```redis
# 트랜잭션 사용
MULTI
DECR stock:product1
INCR orders:count
EXEC

# Lua Script로 복잡한 원자 연산
EVAL "local stock = redis.call('GET', KEYS[1]); if tonumber(stock) > 0 then redis.call('DECR', KEYS[1]); return 1; else return 0; end" 1 stock:product1
```

---

## 5. 명령어 조합 패턴

### 🎯 자주 사용하는 패턴

#### 5.1 카운터 + TTL 패턴
```redis
# 일별 카운터
INCR daily:visits:2025-08-19
EXPIRE daily:visits:2025-08-19 86400

# 시간대별 카운터  
INCR hourly:api:2025-08-19-14
EXPIRE hourly:api:2025-08-19-14 3600
```

#### 5.2 캐시 + 백업 패턴
```redis
# 캐시 조회 → 없으면 DB 조회 → 캐시 저장
GET cache:user:1001
# 없으면 DB에서 조회 후
SETEX cache:user:1001 3600 "user_data"
```

#### 5.3 리더보드 패턴
```redis
# 점수 업데이트
ZINCRBY leaderboard:daily 100 "player123"

# 상위 10명 조회
ZREVRANGE leaderboard:daily 0 9 WITHSCORES

# 특정 사용자 순위
ZREVRANK leaderboard:daily "player123"
```

#### 5.4 분산 락 패턴
```redis
# 락 획득 시도
SET lock:resource123 "token456" NX EX 30

# 락 해제 (Lua Script)
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:resource123 "token456"
```

#### 5.5 메시지 큐 패턴
```redis
# 메시지 추가
LPUSH queue:emails "email_data"

# 워커에서 처리
BRPOP queue:emails 10           # 10초 대기

# 처리 중 큐 (실패 시 복구용)
LMOVE queue:emails queue:processing RIGHT LEFT
```

---

## 📚 정리 및 핵심 포인트

### ✅ 명령어 선택 가이드
1. **단순 캐싱**: SET/GET + TTL
2. **구조화 데이터**: HSET/HGET 
3. **카운터**: INCR/DECR
4. **랭킹**: ZADD/ZREVRANGE
5. **집합 연산**: SADD/SISMEMBER
6. **큐/스택**: LPUSH/RPOP
7. **이벤트**: XADD/XREAD

### 🎯 성능 최적화
1. **O(1) 명령어** 우선 사용
2. **TTL 필수** 설정으로 메모리 관리
3. **큰 데이터 구조** 피하기
4. **KEYS 명령어** 절대 금지
5. **Pipeline/Transaction** 활용

### 🚀 실무 적용
1. **용도별 키 네이밍** 규칙 준수
2. **적절한 데이터 타입** 선택
3. **원자성 보장** 고려
4. **모니터링** 및 알림 설정
5. **장애 대응** 계획 수립

이제 Redis 기본 명령어를 활용하여 다양한 실무 시나리오를 해결할 수 있습니다!