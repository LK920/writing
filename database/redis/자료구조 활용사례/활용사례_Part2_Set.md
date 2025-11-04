---
aliases:
  - Redis 좋아요 구현
  - Set 활용사례
tags:
  - Redis
  - Set
  - 좋아요기능
  - 중복방지
  - 태그시스템
  - 집합연산
  - 해시태그
  - SNS개발
  - 백엔드
  - 데이터베이스
---

# Redis 자료구조 활용사례 Part 2: 좋아요 중복 방지, Redis Set으로 간단하게 해결

> 이 시리즈는 책에서 소개된 다양한 자료구조를 실전에서 활용하는 방법을 소개합니다.
> - Part 1: 게임 랭킹, RDBMS 대신 Redis로 구현하면? (Sorted Set)
> - **Part 2: 좋아요 중복 방지, Redis Set으로 간단하게 해결** (현재 글)
> - Part 3: 1000만 사용자 DAU를 1.2MB로 추적하는 방법 (Bitmap)
> - Part 4: 12KB로 무제한 카운팅? HyperLogLog의 마법

---

## 1. Set을 이용한 태그 기능

### 개요
Set 자료구조는 중복을 허용하지 않는 특성을 활용하여 태그 기능을 구현하는 데 적합합니다.

특정 태그를 포함한 게시물을 빠르게 검색하고, 여러 태그의 교집합/합집합 등의 집합 연산을 효율적으로 수행할 수 있습니다.

### 사용 방법

#### 1. 게시물별 태그 현황
각 게시물이 어떤 태그를 가지고 있는지 저장합니다.

```redis
# 게시물 222에 IT, DATABASE, REDIS 태그 추가
SADD post:222:tags IT DATABASE REDIS

# 게시물 221에 IT, PYTHON 태그 추가
SADD post:221:tags IT PYTHON
```

#### 2. 태그별 게시물 현황
각 태그를 가진 게시물 목록을 저장합니다.

```redis
# IT 태그가 있는 게시물: 222, 221
SADD tag:IT:posts 222 221

# DATABASE 태그가 있는 게시물: 222
SADD tag:DATABASE:posts 222

# REDIS 태그가 있는 게시물: 222
SADD tag:REDIS:posts 222

# PYTHON 태그가 있는 게시물: 221
SADD tag:PYTHON:posts 221
```

#### 3. 특정 태그 조합을 가진 게시물 검색

**교집합 (AND 조건)**: IT와 DATABASE 태그를 모두 가진 게시물
```redis
SINTER tag:IT:posts tag:DATABASE:posts
# 결과: 222
```

**합집합 (OR 조건)**: IT 또는 DATABASE 태그를 가진 게시물
```redis
SUNION tag:IT:posts tag:DATABASE:posts
# 결과: 222, 221
```

**차집합**: IT 태그는 있지만 DATABASE 태그는 없는 게시물
```redis
SDIFF tag:IT:posts tag:DATABASE:posts
# 결과: 221
```

### 활용 예시
- 블로그 태그 시스템
- SNS 해시태그 검색
- 상품 필터링 (카테고리, 브랜드, 색상 등)
- 기술 스택 기반 개발자 매칭

---

## 2. Set을 이용한 좋아요 처리하기

### 개요
댓글, 게시글 등에서 좋아요 기능을 구현할 때 좋아요 개수 카운팅과 중복 방지가 필요합니다.

### RDBMS의 한계

관계형 데이터베이스에서 좋아요 클릭 시:
- 특정 행에서 좋아요 개수 데이터를 증감하는 UPDATE 처리 필요
- DB에 읽기/쓰기 부하가 발생하여 성능에 직접적인 영향
- 동시성 제어 필요 (race condition 발생 가능)

### Redis Set을 이용한 구현

Set 자료구조의 특성을 활용하여 간단하고 효율적으로 구현할 수 있습니다.

#### 좋아요 추가
```redis
# 유저 345, 25, 967이 댓글 ID 12554에 좋아요를 누름
SADD comment-like:12554 345 25 967
```

#### 좋아요 개수 조회
```redis
# 댓글 ID 12554의 좋아요를 누른 유저 수 반환
SCARD comment-like:12554
# 결과: 3
```

#### 특정 유저의 좋아요 여부 확인
```redis
# 유저 345가 댓글 12554에 좋아요를 눌렀는지 확인
SISMEMBER comment-like:12554 345
# 결과: 1 (존재함) 또는 0 (존재하지 않음)
```

#### 좋아요 취소
```redis
# 유저 345가 좋아요 취소
SREM comment-like:12554 345
```

#### 좋아요를 누른 유저 목록 조회
```redis
# 댓글 12554에 좋아요를 누른 모든 유저 조회
SMEMBERS comment-like:12554
# 결과: 345, 25, 967
```

### 장점
- 중복 방지가 자동으로 처리됨 (같은 유저가 여러 번 좋아요 불가)
- 빠른 조회 및 업데이트 성능
- 좋아요를 누른 유저 목록을 쉽게 관리

### 활용 예시
- SNS 게시물 좋아요
- 유튜브 좋아요/싫어요
- 댓글 추천 시스템
- 북마크 기능

---

## 3. 랜덤 데이터 추출

### 개요
RDBMS의 `ORDER BY RAND()`는 성능 이슈가 있습니다. Redis는 각 자료구조별 랜덤 추출 전용 커맨드를 제공합니다.

### 관계형 데이터베이스의 랜덤 추출

RDBMS에서는 `ORDER BY RAND()`를 사용하여 랜덤 추출을 수행합니다.

**처리 과정:**
1. 조건 절에서 모든 행을 읽음
2. 임시 테이블 생성
3. 랜덤하게 정렬
4. LIMIT에 해당하는 만큼 추출

**문제점:**
- 부하가 많이 발생
- 1만 건 이상일 경우 심각한 성능 이슈 발생

### Redis의 RANDOM 커맨드

Redis는 각 자료구조별로 랜덤 추출을 위한 전용 커맨드를 제공합니다.

#### 주요 커맨드
- `RANDOMKEY`: 저장된 전체 키 중 하나를 무작위로 반환
- `HRANDFIELD`: Hash에서 랜덤 필드 추출
- `SRANDMEMBER`: Set에서 랜덤 멤버 추출
- `ZRANDMEMBER`: Sorted Set에서 랜덤 멤버 추출

#### count 옵션: 원하는 개수만큼 추출
```redis
# 양수: 중복 허용 안 함 (유니크한 값들)
SRANDMEMBER myset 3

# 음수: 중복 허용 (같은 값이 여러 번 나올 수 있음)
SRANDMEMBER myset -3
```

#### withvalues/withscores 옵션: 값이나 점수도 함께 반환
```redis
# Hash의 경우 필드와 값을 함께 반환
HRANDFIELD myhash 2 WITHVALUES

# Sorted Set의 경우 멤버와 점수를 함께 반환
ZRANDMEMBER myzset 2 WITHSCORES
```

### 활용 예시
- 랜덤 추천 상품
- 랜덤 배너 광고
- 랜덤 퀴즈 문제 출제
- 랜덤 플레이리스트 생성

---

## 정리

Set은 **중복 불가**와 **집합 연산**이 핵심입니다:

- **태그 기능**: 교집합, 합집합, 차집합으로 복잡한 태그 검색 가능
- **좋아요 처리**: 중복 자동 제거로 간단한 좋아요 시스템 구현
- **랜덤 추출**: RDBMS 대비 월등한 성능으로 랜덤 데이터 추출

Set을 사용하면 중복 방지와 집합 연산을 자동으로 처리할 수 있어, 복잡한 로직을 간단하게 구현할 수 있습니다.

---

> **다음 글 예고**: Part 3에서는 Hash를 활용한 읽지 않은 메시지 수 관리와 Bitmap을 활용한 DAU(일간 활성 사용자) 추적을 다룹니다.
