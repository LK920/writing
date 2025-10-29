# UUID 학습 가이드

## 📌 목적
UUID(Universally Unique Identifier)는 데이터베이스에서 고유 식별자로 사용되는 값입니다. 이 문서는 초보자를 위해 UUID의 개념, 특징, PK로 사용 시 장단점, 그리고 이커머스 도메인에서의 적합성을 설명합니다. 특히, `Long` 타입 PK와 비교해 클러스터링 인덱스와의 상호작용을 다룹니다.

---

## ✅ 1. UUID란?

### 1.1 정의
- **UUID**는 128비트(16바이트)로 구성된 고유 식별자입니다.
- 전 세계적으로 고유한 값을 생성하도록 설계, 충돌 가능성이 극히 낮음(약 2^128).
- **형식**: 36자 문자열(예: `550e8400-e29b-41d4-a716-446655440000`), 하이픈 포함.
- **비유**: 주민등록번호처럼, 모든 데이터에 고유한 "이름표"를 붙이는 방식.

### 1.2 주요 특징
- **고유성**: 동일한 UUID가 생성될 확률은 사실상 0에 가까움.
- **무작위성**: UUID는 순차적이지 않고 무작위로 생성(버전 4 기준).
- **저장 형식**:
  - 문자열: 36바이트(UTF-8 인코딩).
  - 바이너리: 16바이트(`BINARY(16)`로 저장 가능).

---

## ✅ 2. UUID의 생성과 사용

### 2.1 생성 방식
- **버전 4 (무작위)**: 가장 흔히 사용, 난수 기반.
  ```java
  import java.util.UUID;
  String uuid = UUID.randomUUID().toString(); // 예: "550e8400-e29b-41d4-a716-446655440000"
  ```
- **MySQL에서**:
  ```sql
  SELECT UUID(); -- 문자열 UUID 생성
  ```
  - `BINARY(16)`로 저장 시 `UNHEX(REPLACE(UUID(), '-', ''))` 사용.

### 2.2 PK로 사용
```sql
CREATE TABLE orders (
    order_id BINARY(16) PRIMARY KEY,
    customer_id BIGINT,
    order_date DATETIME
);
```
- `order_id`를 UUID로 설정, 클러스터링 인덱스로 사용.

---

## ✅ 3. UUID를 PK로 사용할 때의 장단점

### 3.1 장점
- **고유성**: 분산 시스템(예: 여러 데이터베이스)에서 충돌 없이 고유 ID 생성.
- **보안**: 예측 불가능한 값으로, ID 노출 시 추측 어려움.
- **독립성**: 서버 간 동기화 없이 ID 생성 가능.

### 3.2 단점
- **큰 크기**:
  - 문자열: 36바이트, `Long`(8바이트)보다 4배 이상 큼.
  - 바이너리: 16바이트, 여전히 `Long`보다 큼.
  - 클러스터링/비클러스터링 인덱스 크기 증가, 디스크 I/O 비용 상승.
- **느린 비교 연산**:
  - UUID는 문자열 또는 바이너리로 비교, `Long`의 정수 비교보다 느림.
  - B+ Tree 탐색 속도 저하.
- **무작위 삽입**:
  - UUID는 무작위 값, B+ Tree 중간에 삽입되어 페이지 분할 빈발.
  - 삽입 성능 저하, 프래그먼테이션 증가.
- **범위 쿼리 비효율**:
  - 무작위 값으로 정렬, `WHERE order_id BETWEEN ...` 같은 쿼리 비효율적.

---

## ✅ 4. `Long` 타입 PK와 비교

| **항목**          | **Long (BIGINT)**                     | **UUID**                             |
|-------------------|---------------------------------------|--------------------------------------|
| **크기**         | 8바이트                              | 16~36바이트                         |
| **비교 연산**    | 빠름 (정수)                         | 느림 (문자열/바이너리)             |
| **삽입 성능**    | 단조 증가, 페이지 분할 적음         | 무작위, 페이지 분할 빈발            |
| **인덱스 크기**  | 작음                                | 큼                                  |
| **범위 쿼리**    | 효율적                              | 비효율적                           |
| **확장성**       | `AUTO_INCREMENT`로 고유성 보장      | 분산 시스템에서 유리                |

### 4.1 이커머스에서의 적합성
- **Long**: 주문/상품 조회, 삽입이 빈번한 이커머스에 적합. 빠른 성능, 작은 인덱스 크기.
- **UUID**: 분산 시스템(예: 여러 데이터센터)에서 ID 생성이 필요한 경우 유용, 하지만 성능 저하로 단일 DB 환경에서는 비추천.

---

## ✅ 5. 이커머스 도메인에서의 UUID

- **사용 사례**:
  - 분산 시스템에서 주문 ID 생성(예: 마이크로서비스 아키텍처).
  - 보안이 중요한 경우(예: 외부에 노출되는 토큰).
- **문제점**:
  - 주문 조회(`SELECT * FROM orders WHERE order_id = ?`)가 느림.
  - 삽입 성능 저하로 대량 주문 처리 시 병목.
- **대안**:
  - `Long` PK + 별도 UUID 컬럼: 고유성은 `Long` PK로, 외부 노출은 UUID로 처리.
  ```sql
  CREATE TABLE orders (
      order_id BIGINT PRIMARY KEY AUTO_INCREMENT,
      order_uuid BINARY(16) UNIQUE,
      customer_id BIGINT
  );
  ```

---

## ✅ 6. 학습 가이드

### 6.1 학습 목표
- UUID의 생성과 PK 사용 시 장단점 이해.
- 이커머스에서 `Long` PK와 UUID의 적합성 비교.

### 6.2 학습 단계
1. **기본 개념**:
   - [UUID Wikipedia](https://en.wikipedia.org/wiki/Universally_unique_identifier)로 UUID 구조 학습.
   - MySQL 문서: [UUID Functions](https://dev.mysql.com/doc/refman/8.4/en/miscellaneous-functions.html#function_uuid).
2. **실습**:
   - `BIGINT` PK와 UUID PK 테이블 생성, 삽입/조회 성능 비교.
   - `EXPLAIN`으로 쿼리 실행 계획 분석.
3. **적용**:
   - 이커머스 프로젝트에서 UUID 사용 여부 검토(예: 보안 vs. 성능).

### 6.3 추천 자료
- 문서: [MySQL UUID Functions](https://dev.mysql.com/doc/refman/8.4/en/miscellaneous-functions.html#function_uuid).
- 블로그: [UUID vs. Auto-Increment](https://www.percona.com/blog/2019/11/22/uuids-are-popular-but-bad-for-performance-lets-discuss/).
- 도서: *High Performance MySQL* (O’Reilly).

---

## 🔚 요약
- **UUID**: 고유 식별자, 분산 시스템에 유리하지만 PK로 사용 시 성능 저하.
- **Long PK 우위**: 작은 크기, 빠른 비교, 단조 증가로 이커머스에 적합.
- **학습 팁**: UUID와 `Long` PK 비교 실습으로 성능 차이 체감.