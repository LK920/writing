# 데이터베이스 파티셔닝 학습 자료

## 1. 파티셔닝이란?

파티셔닝(partitioning)은 데이터베이스 테이블의 데이터를 더 작은 논리적 조각(파티션)으로 나누는 기술입니다. 마치 커다란 책을 여러 챕터로 나누어 관리하는 것과 비슷합니다. 각 파티션은 동일한 테이블의 일부 데이터를 저장하며, 데이터베이스 엔진은 이를 독립적으로 처리할 수 있습니다.

### 비유
- 책장(테이블)에 책(데이터)이 너무 많아 찾기 힘들 때, 책을 주제별로 나눠 각각의 작은 책장에 보관하는 것과 같습니다.
- 예: 이커머스 프로젝트의 `event_logs` 테이블에 시스템 이벤트 로그가 수백만 행 쌓이면, 특정 날짜의 로그를 찾기 위해 전체 데이터를 스캔해야 합니다. 파티셔닝을 적용하면 특정 날짜별로 데이터를 나누어 빠르게 찾을 수 있습니다.

### 파티셔닝의 목적
- **성능 향상**: 특정 파티션만 스캔하여 쿼리 속도 개선.
- **관리 용이성**: 데이터 삭제, 백업, 아카이빙이 쉬워짐.
- **확장성**: 대량 데이터 처리 시 부하 분산.

---

## 2. 파티셔닝의 주요 유형

파티셔닝은 데이터를 나누는 기준에 따라 여러 유형으로 나뉩니다. 주요 유형은 다음과 같습니다:

### 2.1. 범위 파티셔닝 (Range Partitioning)
- 데이터를 특정 범위(예: 날짜, 숫자)로 나눕니다.
- **예시**: 이커머스 프로젝트의 `event_logs` 테이블을 `created_at` 날짜 기준으로 월별 파티션으로 나눔.
  ```sql
  CREATE TABLE event_logs (
      id BIGINT PRIMARY KEY,
      event_type VARCHAR,
      payload TEXT,
      status VARCHAR,
      created_at TIMESTAMP
  ) PARTITION BY RANGE (created_at);

  CREATE TABLE event_logs_202501 PARTITION OF event_logs 
      FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
  CREATE TABLE event_logs_202502 PARTITION OF event_logs 
      FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
  ```
- **사용 사례**: 2025년 1월 로그만 조회하려면 `event_logs_202501` 파티션만 스캔.

### 2.2. 리스트 파티셔닝 (List Partitioning)
- 특정 값 목록(예: 상태, 지역)으로 데이터를 나눕니다.
- **예시**: `orders` 테이블을 `status`(예: PENDING, COMPLETED, CANCELLED)로 나눔.
  ```sql
  CREATE TABLE orders (
      id BIGINT PRIMARY KEY,
      user_id BIGINT,
      total_amount DECIMAL,
      status VARCHAR,
      created_at TIMESTAMP
  ) PARTITION BY LIST (status);

  CREATE TABLE orders_pending PARTITION OF orders FOR VALUES IN ('PENDING');
  CREATE TABLE orders_completed PARTITION OF orders FOR VALUES IN ('COMPLETED');
  ```
- **사용 사례**: 완료된 주문만 조회 시 `orders_completed` 파티션만 접근.

### 2.3. 해시 파티셔닝 (Hash Partitioning)
- 데이터의 해시값을 기준으로 균등하게 나눕니다.
- **예시**: `users` 테이블을 `id`의 해시값으로 4개 파티션으로 나눔.
  ```sql
  CREATE TABLE users (
      id BIGINT PRIMARY KEY,
      name VARCHAR,
      created_at TIMESTAMP
  ) PARTITION BY HASH (id);

  CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
  CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
  ```
- **사용 사례**: 데이터가 균등 분포되어야 할 때, 부하 분산에 유용.

### 2.4. 복합 파티셔닝 (Composite Partitioning)
- 여러 파티셔닝 방법을 조합(예: 범위 + 해시).
- **예시**: `event_logs`를 먼저 `created_at`으로 범위 파티셔닝한 후, 각 파티션을 `event_type`으로 해시 파티셔닝.

---

## 3. 파티셔닝의 장점

1. **쿼리 성능 개선**:
   - 특정 파티션만 스캔하므로 전체 테이블 스캔 감소.
   - 예: `SELECT * FROM event_logs WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31'`은 `event_logs_202501`만 조회.

2. **데이터 관리 용이**:
   - 오래된 데이터를 삭제하거나 아카이빙할 때 파티션 단위로 처리.
   - 예: 2024년 로그 파티션을 삭제하면 전체 테이블을 스캔하지 않음.

3. **인덱스 효율성**:
   - 각 파티션에 독립적인 인덱스를 생성하여 인덱스 크기 감소.
   - 예: `event_logs_202501`의 `event_type`에 인덱스를 생성하면 특정 파티션 내 검색이 빨라짐.

---

## 4. 파티셔닝의 단점

1. **복잡성 증가**:
   - 테이블 관리(파티션 추가/삭제)가 복잡해짐.
   - 예: 매달 새로운 파티션을 생성해야 함.

2. **제약사항**:
   - 모든 데이터베이스 시스템이 모든 파티셔닝 유형을 지원하지 않음(예: MySQL은 범위/리스트 지원, PostgreSQL은 더 풍부한 지원).
   - 외래 키 제약이 파티션 간에 복잡할 수 있음.

3. **쿼리 설계 주의**:
   - 파티션 키(`created_at`, `status` 등)를 쿼리에 포함하지 않으면 모든 파티션을 스캔하여 성능 저하.

---

## 5. 이커머스 프로젝트에서의 적용 예시

### `event_logs` 테이블 파티셔닝
- **상황**: `event_logs` 테이블은 매일 수십만 행의 이벤트(예: 주문 생성, 결제 완료)가 추가되어 데이터 크기가 빠르게 증가.
- **적용**:
  - `created_at` 기준으로 월별 범위 파티셔닝.
  - 2025년 1월 로그만 조회하는 쿼리: `SELECT * FROM event_logs WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31'`.
- **효과**:
  - 조회 속도 향상: 전체 테이블 대신 특정 파티션만 스캔.
  - 관리 용이: 1년 지난 로그 파티션을 삭제하여 스토리지 절약.

### `orders` 테이블 파티셔닝
- **상황**: 주문 상태별로 자주 조회(예: 관리자가 완료된 주문만 확인).
- **적용**:
  - `status` 기준으로 리스트 파티셔닝.
  - 예: `orders_completed` 파티션만 조회.
- **효과**:
  - 상태별 조회 성능 향상.
  - 분석 쿼리(예: 완료된 주문 통계) 효율성 증가.

---

## 6. 파티셔닝 구현 시 고려사항

1. **파티션 키 선택**:
   - 자주 조회되는 컬럼(예: `created_at`, `status`)을 선택.
   - 이커머스에서는 날짜, 상태, 사용자 ID 등이 적합.

2. **파티션 크기 관리**:
   - 너무 작은 파티션은 관리 오버헤드 증가, 너무 큰 파티션은 성능 이점 감소.
   - 예: 월별 파티션은 적당한 크기, 일별은 관리 복잡.

3. **자동화**:
   - 새 파티션 생성 스크립트를 자동화(예: 매달 새로운 `event_logs` 파티션 생성).
   - 예: PostgreSQL의 `pg_cron`으로 주기적 파티션 생성.

4. **쿼리 최적화**:
   - 쿼리에 파티션 키를 포함하여 특정 파티션만 스캔하도록 설계.
   - 예: `WHERE created_at >= '2025-01-01'` 추가.

---

## 7. 학습자 팁

- **시작하기**: PostgreSQL 또는 MySQL에서 간단한 테이블에 범위 파티셔닝을 적용해보세요(예: 날짜 기반 파티션).
- **실습 예제**: `event_logs` 테이블을 월별로 파티셔닝하고, 특정 월의 데이터를 조회하는 쿼리를 작성.
- **도구**: PostgreSQL의 `EXPLAIN` 명령어로 쿼리가 파티션을 어떻게 사용하는지 확인.
  ```sql
  EXPLAIN SELECT * FROM event_logs WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31';
  ```

---

## 8. 요약

파티셔닝은 대규모 데이터를 효율적으로 관리하고 쿼리 성능을 향상시키는 강력한 도구입니다. 이커머스 프로젝트에서는 로그 데이터(`event_logs`)나 주문 데이터(`orders`)에 파티셔닝을 적용하여 성능과 관리 효율성을 높일 수 있습니다. 초보자는 범위 파티셔닝부터 시작하고, 실제 쿼리 패턴을 분석하여 적절한 파티션 키를 선택하는 연습을 해보세요.