# MariaDB 학습 가이드

MariaDB는 MySQL에서 포크된 오픈소스 관계형 데이터베이스 관리 시스템(RDBMS)입니다. 이 문서는 기초부터 고급까지 MariaDB 사용법을 다룹니다.

---

## 목차
0. [Docker로 MariaDB 시작하기](#0-docker로-mariadb-시작하기)
1. [기초 (Basics)](#1-기초-basics)
2. [데이터베이스 및 테이블 관리](#2-데이터베이스-및-테이블-관리)
3. [데이터 타입](#3-데이터-타입)
4. [기본 SQL 명령어](#4-기본-sql-명령어)
5. [조건문과 필터링](#5-조건문과-필터링)
6. [JOIN 연산](#6-join-연산)
7. [집계 함수와 그룹화](#7-집계-함수와-그룹화)
8. [인덱스](#8-인덱스)
9. [사용자 및 권한 관리](#9-사용자-및-권한-관리)
10. [트랜잭션](#10-트랜잭션)
11. [뷰(View)](#11-뷰view)
12. [저장 프로시저와 함수](#12-저장-프로시저와-함수)
13. [트리거](#13-트리거)
14. [고급 기능](#14-고급-기능)
15. [성능 최적화](#15-성능-최적화)
16. [MariaDB 11.8 LTS (2025) 주요 변경사항 정리](#16-mariadb-118-lts-2025-주요-변경사항-정리)

---

## 0. Docker로 MariaDB 시작하기

### Docker 설치 확인
```bash
# Docker 버전 확인
docker --version

# Docker가 실행 중인지 확인
docker ps
```

### MariaDB 컨테이너 실행

#### 기본 실행
```bash
# 최신 버전 MariaDB 실행
docker run --name mariadb-container \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -p 3306:3306 \
  -d mariadb:latest

# 설명:
# --name: 컨테이너 이름 지정
# -e MYSQL_ROOT_PASSWORD: root 사용자 비밀번호 설정
# -p 3306:3306: 포트 매핑 (호스트:컨테이너)
# -d: 백그라운드 실행
# mariadb:latest: 사용할 이미지
```

#### Spring Boot 프로젝트용 설정
```bash
# 데이터베이스와 사용자를 함께 생성하며 실행
docker run --name mariadb-bootex \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=bootex \
  -e MYSQL_USER=bootuser \
  -e MYSQL_PASSWORD=bootpass \
  -p 3306:3306 \
  -d mariadb:latest \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci

# 설명:
# MYSQL_DATABASE: 시작 시 자동으로 생성될 데이터베이스
# MYSQL_USER: 생성할 사용자 이름
# MYSQL_PASSWORD: 사용자 비밀번호
# --character-set-server: 문자셋 설정
# --collation-server: 정렬 규칙 설정
```

#### 데이터 영구 저장 (볼륨 마운트)
```bash
# 호스트 디렉토리에 데이터 저장
docker run --name mariadb-persistent \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=bootex \
  -e MYSQL_USER=bootuser \
  -e MYSQL_PASSWORD=bootpass \
  -p 3306:3306 \
  -v ~/mariadb-data:/var/lib/mysql \
  -d mariadb:latest \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci

# -v 옵션으로 호스트의 ~/mariadb-data 디렉토리에 데이터 저장
# 컨테이너를 삭제해도 데이터가 유지됩니다
```

#### Docker Compose 사용 (권장)
```yaml
# docker-compose.yml 파일 생성
version: '3.8'

services:
  mariadb:
    image: mariadb:latest
    container_name: mariadb-bootex
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: bootex
      MYSQL_USER: bootuser
      MYSQL_PASSWORD: bootpass
    ports:
      - "3306:3306"
    volumes:
      - mariadb-data:/var/lib/mysql
      - ./init-scripts:/docker-entrypoint-initdb.d
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password

volumes:
  mariadb-data:
    driver: local
```

```bash
# Docker Compose로 실행
docker-compose up -d

# 중지
docker-compose down

# 중지 및 볼륨 삭제
docker-compose down -v
```

### 컨테이너 관리

#### 컨테이너 상태 확인
```bash
# 실행 중인 컨테이너 확인
docker ps

# 모든 컨테이너 확인 (중지된 것 포함)
docker ps -a

# 컨테이너 로그 확인
docker logs mariadb-container

# 실시간 로그 확인
docker logs -f mariadb-container
```

#### 컨테이너 시작/중지/재시작
```bash
# 컨테이너 중지
docker stop mariadb-container

# 컨테이너 시작
docker start mariadb-container

# 컨테이너 재시작
docker restart mariadb-container

# 컨테이너 삭제 (중지 후)
docker stop mariadb-container
docker rm mariadb-container

# 실행 중인 컨테이너 강제 삭제
docker rm -f mariadb-container
```

### MariaDB 접속

#### 컨테이너 내부에서 접속
```bash
# 컨테이너 내부 bash 접속
docker exec -it mariadb-container bash

# MariaDB 접속 (컨테이너 내부에서)
mysql -u root -p
# 비밀번호 입력: rootpassword

# 또는 한 번에
docker exec -it mariadb-container mysql -u root -p
```

#### 호스트에서 직접 접속
```bash
# 호스트에 mysql 클라이언트가 설치되어 있다면
mysql -h 127.0.0.1 -P 3306 -u root -p

# 또는 bootuser로 접속
mysql -h 127.0.0.1 -P 3306 -u bootuser -p bootex
```

#### GUI 도구로 접속
```plaintext
호스트: localhost (또는 127.0.0.1)
포트: 3306
사용자: bootuser
비밀번호: bootpass
데이터베이스: bootex

추천 GUI 도구:
- DBeaver (무료, 크로스 플랫폼)
- HeidiSQL (무료, Windows)
- MySQL Workbench (무료, 크로스 플랫폼)
- DataGrip (유료, JetBrains)
```

### 초기 스크립트 실행

#### 방법 1: 컨테이너 시작 시 자동 실행
```bash
# init-scripts 디렉토리 생성
mkdir -p init-scripts

# SQL 스크립트 작성 (init-scripts/01-init.sql)
cat > init-scripts/01-init.sql << 'EOF'
-- 테이블 생성
CREATE TABLE IF NOT EXISTS users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 샘플 데이터 삽입
INSERT INTO users (username, email) VALUES
('admin', 'admin@example.com'),
('user1', 'user1@example.com');

-- 인덱스 생성
CREATE INDEX idx_username ON users(username);
EOF

# 컨테이너 실행 시 초기 스크립트 마운트
docker run --name mariadb-with-init \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=bootex \
  -e MYSQL_USER=bootuser \
  -e MYSQL_PASSWORD=bootpass \
  -p 3306:3306 \
  -v ./init-scripts:/docker-entrypoint-initdb.d \
  -d mariadb:latest

# /docker-entrypoint-initdb.d 디렉토리의 .sql, .sh 파일이 자동 실행됩니다
```

#### 방법 2: 실행 중인 컨테이너에 SQL 파일 실행
```bash
# SQL 파일을 컨테이너에 복사
docker cp my-script.sql mariadb-container:/tmp/

# 컨테이너 내에서 실행
docker exec -i mariadb-container mysql -u root -p bootex < my-script.sql

# 또는 직접 실행
docker exec -i mariadb-container mysql -u root -p -e "USE bootex; SOURCE /tmp/my-script.sql;"
```

### 데이터 백업 및 복원

#### 백업
```bash
# 데이터베이스 전체 백업
docker exec mariadb-container mysqldump -u root -proot bootex > bootex_backup.sql

# 모든 데이터베이스 백업
docker exec mariadb-container mysqldump -u root -proot --all-databases > all_backup.sql

# 압축 백업
docker exec mariadb-container mysqldump -u root -proot bootex | gzip > bootex_backup.sql.gz
```

#### 복원
```bash
# 백업 파일 복원
docker exec -i mariadb-container mysql -u root -proot bootex < bootex_backup.sql

# 압축 파일 복원
gunzip < bootex_backup.sql.gz | docker exec -i mariadb-container mysql -u root -proot bootex
```

### 설정 파일 커스터마이징

#### 방법 1: 커맨드 옵션으로 설정
```bash
docker run --name mariadb-custom \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  -d mariadb:latest \
  --max-connections=200 \
  --innodb-buffer-pool-size=1G \
  --character-set-server=utf8mb4 \
  --collation-server=utf8mb4_unicode_ci
```

#### 방법 2: 설정 파일 마운트
```bash
# my.cnf 파일 생성
cat > my.cnf << 'EOF'
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
max_connections=200
innodb_buffer_pool_size=1G
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log
long_query_time=2

[mysql]
default-character-set=utf8mb4

[client]
default-character-set=utf8mb4
EOF

# 설정 파일 마운트하여 실행
docker run --name mariadb-custom-config \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  -v ./my.cnf:/etc/mysql/conf.d/custom.cnf \
  -d mariadb:latest

# 💡 MariaDB 11.8+ (2025 LTS): 최신 버전 사용 권장
# MariaDB 11.8은 2025년 LTS 버전으로 Vector 지원 등 새로운 기능 포함
docker run --name mariadb-latest \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  -v ./my.cnf:/etc/mysql/conf.d/custom.cnf \
  -d mariadb:11.8
```

### 실전 예제: Spring Boot 개발 환경 구성

#### docker-compose.yml (완전한 개발 환경)
```yaml
version: '3.8'

services:
  mariadb:
    image: mariadb:11.2
    container_name: mariadb-dev
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: bootex
      MYSQL_USER: bootuser
      MYSQL_PASSWORD: bootpass
      TZ: Asia/Seoul
    ports:
      - "3306:3306"
    volumes:
      - mariadb-data:/var/lib/mysql
      - ./init-scripts:/docker-entrypoint-initdb.d
      - ./conf.d:/etc/mysql/conf.d
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password
      - --lower-case-table-names=1
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mariadb-data:
    driver: local
```

#### application.yml 설정
```yaml
spring:
  application:
    name: ex2
  datasource:
    driver-class-name: org.mariadb.jdbc.Driver
    url: jdbc:mariadb://localhost:3306/bootex
    username: bootuser
    password: bootpass
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MariaDBDialect
```

### 유용한 Docker 명령어 모음

```bash
# 컨테이너 리소스 사용량 확인
docker stats mariadb-container

# 컨테이너 상세 정보 확인
docker inspect mariadb-container

# 컨테이너 내부 파일 시스템 확인
docker exec mariadb-container ls -la /var/lib/mysql

# 특정 MariaDB 버전 사용
docker run --name mariadb-11 -d mariadb:11.2
docker run --name mariadb-10 -d mariadb:10.11

# 사용하지 않는 이미지/컨테이너 정리
docker system prune -a

# MariaDB 이미지 업데이트
docker pull mariadb:latest
docker stop mariadb-container
docker rm mariadb-container
# 새 이미지로 재실행

# 네트워크 생성 (여러 컨테이너 연결 시)
docker network create bootex-network
docker run --network bootex-network --name mariadb -d mariadb:latest
```

### 문제 해결 (Troubleshooting)

```bash
# 1. 포트 충돌 (3306 포트가 이미 사용 중)
# 해결: 다른 포트 사용
docker run -p 3307:3306 ...

# 2. 컨테이너가 즉시 종료됨
# 원인 확인
docker logs mariadb-container

# 3. 연결 거부 (Can't connect)
# MariaDB가 준비될 때까지 대기
docker logs -f mariadb-container | grep "ready for connections"

# 4. 비밀번호 분실
# 컨테이너 재생성 또는
docker exec -it mariadb-container mysql -u root
# (비밀번호 없이 접속 시도)

# 5. 데이터 초기화가 필요한 경우
docker stop mariadb-container
docker rm mariadb-container
docker volume rm mariadb-data
# 새로 시작

# 6. 권한 문제
# 볼륨 디렉토리 권한 확인
ls -la ~/mariadb-data
# 필요시 권한 변경
sudo chown -R $(whoami) ~/mariadb-data
```

### Docker에서 MariaDB 사용 시 장점

1. **빠른 설정**: 몇 초 만에 MariaDB 환경 구성
2. **격리된 환경**: 호스트 시스템에 영향 없음
3. **버전 관리**: 여러 버전을 쉽게 테스트 가능
4. **이식성**: 같은 설정을 다른 환경에서 재현 가능
5. **쉬운 초기화**: 컨테이너 삭제 후 재생성으로 깨끗한 상태 유지

---

## 1. 기초 (Basics)

### MariaDB 접속
```bash
# root 사용자로 접속
mysql -u root -p

# 특정 사용자로 접속
mysql -u username -p

# 특정 데이터베이스에 직접 접속
mysql -u username -p database_name

# 원격 서버 접속
mysql -h hostname -u username -p

# ⚠️ MariaDB 11+ 변경사항: mysql 명령어는 deprecated 예정
# 향후 버전에서는 mariadb 명령어 사용 권장
mariadb -u root -p
mariadb -u username -p database_name
```

### 기본 명령어
```sql
-- 현재 사용자 확인
SELECT USER();

-- MariaDB 버전 확인
SELECT VERSION();

-- 현재 날짜와 시간
SELECT NOW();

-- 데이터베이스 목록 보기
SHOW DATABASES;

-- 현재 사용 중인 데이터베이스 확인
SELECT DATABASE();

-- MariaDB 종료
EXIT;
-- 또는
QUIT;
```

---

## 2. 데이터베이스 및 테이블 관리

### 데이터베이스 생성 및 삭제
```sql
-- 데이터베이스 생성
CREATE DATABASE bootex;

-- 문자셋을 지정하여 생성 (UTF-8 권장)
CREATE DATABASE bootex
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

-- 💡 MariaDB 11+ 추가: utf8mb4_0900_ai_ci 콜레이션 지원
-- MySQL 8.0과 호환되는 콜레이션 (MariaDB에서는 utf8mb4_uca1400_ai_ci의 별칭)
CREATE DATABASE bootex_modern
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;

-- 데이터베이스가 없을 때만 생성
CREATE DATABASE IF NOT EXISTS bootex;

-- 데이터베이스 선택
USE bootex;

-- 데이터베이스 삭제
DROP DATABASE bootex;

-- 데이터베이스가 존재할 때만 삭제
DROP DATABASE IF EXISTS bootex;

-- 데이터베이스 목록 조회
SHOW DATABASES;

-- 특정 패턴의 데이터베이스 조회
SHOW DATABASES LIKE 'boot%';
```

### 테이블 생성
```sql
-- 기본 테이블 생성
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 더 복잡한 테이블 생성 예제
CREATE TABLE products (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(100) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INT DEFAULT 0,
    category_id INT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_category FOREIGN KEY (category_id)
        REFERENCES categories(category_id)
        ON DELETE SET NULL
        ON UPDATE CASCADE
);

-- 다른 테이블 구조 복사
CREATE TABLE users_backup LIKE users;

-- 데이터까지 포함하여 복사
CREATE TABLE users_backup AS SELECT * FROM users;
```

### 테이블 수정
```sql
-- 컬럼 추가
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 여러 컬럼 동시 추가
ALTER TABLE users
ADD COLUMN age INT,
ADD COLUMN address VARCHAR(200);

-- 컬럼 수정
ALTER TABLE users MODIFY COLUMN phone VARCHAR(30);

-- 컬럼 이름 변경
ALTER TABLE users CHANGE COLUMN phone phone_number VARCHAR(30);

-- 컬럼 삭제
ALTER TABLE users DROP COLUMN age;

-- 제약조건 추가
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);

-- 외래키 추가
ALTER TABLE orders
ADD CONSTRAINT fk_user
FOREIGN KEY (user_id) REFERENCES users(id);

-- 인덱스 추가
ALTER TABLE users ADD INDEX idx_username (username);

-- 테이블 이름 변경
ALTER TABLE users RENAME TO customers;
-- 또는
RENAME TABLE customers TO users;
```

### 테이블 조회 및 삭제
```sql
-- 현재 데이터베이스의 테이블 목록
SHOW TABLES;

-- 특정 패턴의 테이블 조회
SHOW TABLES LIKE 'user%';

-- 테이블 구조 확인
DESCRIBE users;
-- 또는
DESC users;
-- 또는
SHOW COLUMNS FROM users;

-- 테이블 생성 쿼리 확인
SHOW CREATE TABLE users;

-- 테이블 삭제
DROP TABLE users;

-- 테이블이 존재할 때만 삭제
DROP TABLE IF EXISTS users;

-- 테이블 데이터만 삭제 (구조는 유지)
TRUNCATE TABLE users;
```

---

## 3. 데이터 타입

### 숫자형 데이터 타입

#### 정수형
```sql
-- TINYINT: -128 ~ 127 (또는 0 ~ 255 UNSIGNED)
CREATE TABLE test_tiny (value TINYINT);

-- SMALLINT: -32,768 ~ 32,767 (또는 0 ~ 65,535 UNSIGNED)
CREATE TABLE test_small (value SMALLINT);

-- MEDIUMINT: -8,388,608 ~ 8,388,607
CREATE TABLE test_medium (value MEDIUMINT);

-- INT: -2,147,483,648 ~ 2,147,483,647
CREATE TABLE test_int (value INT);

-- BIGINT: 매우 큰 정수
CREATE TABLE test_bigint (value BIGINT);

-- UNSIGNED (양수만)
CREATE TABLE counters (
    page_views INT UNSIGNED,
    likes BIGINT UNSIGNED
);
```

#### 실수형
```sql
-- DECIMAL(M, D): 정확한 소수점 (M: 전체 자릿수, D: 소수점 이하)
CREATE TABLE prices (
    product_id INT,
    price DECIMAL(10, 2)  -- 최대 99999999.99
);

-- FLOAT: 단정밀도 부동소수점
CREATE TABLE measurements (
    temperature FLOAT
);

-- DOUBLE: 배정밀도 부동소수점
CREATE TABLE scientific_data (
    measurement DOUBLE
);
```

### 문자열 데이터 타입
```sql
-- CHAR(n): 고정 길이 문자열 (최대 255)
CREATE TABLE codes (
    country_code CHAR(2),  -- 항상 2바이트 사용
    zip_code CHAR(5)
);

-- VARCHAR(n): 가변 길이 문자열 (최대 65,535)
CREATE TABLE users (
    username VARCHAR(50),
    email VARCHAR(100),
    bio VARCHAR(500)
);

-- TEXT 타입들
CREATE TABLE articles (
    title VARCHAR(200),
    summary TEXT,           -- 최대 65,535 바이트
    content MEDIUMTEXT,     -- 최대 16,777,215 바이트
    full_content LONGTEXT   -- 최대 4,294,967,295 바이트
);

-- ENUM: 미리 정의된 값 목록
CREATE TABLE orders (
    order_id INT,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled')
);

-- SET: 여러 값을 동시에 선택 가능
CREATE TABLE permissions (
    user_id INT,
    rights SET('read', 'write', 'delete', 'admin')
);
```

### 날짜/시간 데이터 타입
```sql
CREATE TABLE events (
    event_id INT PRIMARY KEY AUTO_INCREMENT,

    -- DATE: 날짜만 (YYYY-MM-DD)
    event_date DATE,

    -- TIME: 시간만 (HH:MM:SS)
    event_time TIME,

    -- DATETIME: 날짜와 시간 (YYYY-MM-DD HH:MM:SS)
    event_datetime DATETIME,

    -- TIMESTAMP: 날짜와 시간 (자동 업데이트 가능)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- YEAR: 연도만
    event_year YEAR
);

-- 예제 데이터 삽입
INSERT INTO events (event_date, event_time, event_datetime, event_year)
VALUES ('2024-12-25', '14:30:00', '2024-12-25 14:30:00', 2024);
```

### 바이너리 데이터 타입
```sql
CREATE TABLE files (
    file_id INT PRIMARY KEY AUTO_INCREMENT,
    file_name VARCHAR(255),

    -- BINARY/VARBINARY: 바이너리 데이터
    checksum BINARY(16),

    -- BLOB 타입들
    thumbnail BLOB,         -- 최대 65,535 바이트
    image MEDIUMBLOB,       -- 최대 16MB
    video LONGBLOB          -- 최대 4GB
);
```

### JSON 데이터 타입
```sql
CREATE TABLE user_settings (
    user_id INT PRIMARY KEY,
    settings JSON
);

-- JSON 데이터 삽입
INSERT INTO user_settings VALUES
(1, '{"theme": "dark", "notifications": true, "language": "ko"}');

-- JSON 데이터 조회
SELECT
    user_id,
    JSON_EXTRACT(settings, '$.theme') as theme,
    JSON_EXTRACT(settings, '$.language') as language
FROM user_settings;
```

### VECTOR 데이터 타입 (MariaDB 11.8+)
```sql
-- 💡 MariaDB 11.8 LTS (2025) 신규 기능: Vector 데이터 타입
-- AI/ML 애플리케이션을 위한 벡터 임베딩 저장 및 검색 지원

-- Vector 테이블 생성
CREATE TABLE embeddings (
    doc_id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    content TEXT NOT NULL,

    -- VECTOR(차원): 벡터 데이터 저장 (예: OpenAI 임베딩은 1536차원)
    embedding VECTOR(1536) NOT NULL,

    -- Vector 인덱스 생성 (M: 인접 노드 수, DISTANCE: 거리 계산 방식)
    VECTOR INDEX (embedding) M=8 DISTANCE=cosine
);

-- Vector 데이터 삽입 (텍스트 형식)
INSERT INTO embeddings (content, embedding) VALUES
('AI 기술 문서', VEC_FromText('[0.1, 0.2, 0.3, ...]'));

-- Vector 데이터 삽입 (바이너리 형식)
INSERT INTO embeddings (content, embedding) VALUES
('머신러닝 가이드', x'6ca1d43e9df91b3fe580da3e1c247d3f');

-- 유사도 검색: Cosine 거리 (값이 작을수록 유사)
SELECT
    doc_id,
    content,
    VEC_DISTANCE_COSINE(embedding, VEC_FromText('[0.15, 0.25, 0.35, ...]')) AS similarity
FROM embeddings
ORDER BY similarity ASC
LIMIT 10;

-- 유사도 검색: Euclidean 거리
SELECT
    doc_id,
    content,
    VEC_DISTANCE_EUCLIDEAN(embedding, VEC_FromText('[0.1, 0.2, 0.3, ...]')) AS distance
FROM embeddings
ORDER BY distance ASC
LIMIT 5;

-- 자동 거리 계산 (인덱스 타입에 따라 자동 선택)
SELECT
    doc_id,
    VEC_DISTANCE(embedding, VEC_FromText('[...]')) AS distance
FROM embeddings
ORDER BY distance
LIMIT 10;

-- RAG(Retrieval-Augmented Generation) 예제
-- 특정 사용자의 문서에서 가장 유사한 콘텐츠 검색
SELECT
    doc_id,
    content,
    VEC_DISTANCE_COSINE(embedding, VEC_FromText(?)) AS relevance
FROM embeddings
WHERE user_id = ?
ORDER BY relevance ASC
LIMIT 5;
```

---

## 4. 기본 SQL 명령어

### SELECT - 데이터 조회
```sql
-- 모든 컬럼 조회
SELECT * FROM users;

-- 특정 컬럼만 조회
SELECT username, email FROM users;

-- DISTINCT: 중복 제거
SELECT DISTINCT city FROM users;

-- LIMIT: 결과 개수 제한
SELECT * FROM users LIMIT 10;

-- LIMIT with OFFSET: 페이징
SELECT * FROM users LIMIT 10 OFFSET 20;  -- 21~30번째 행

-- ORDER BY: 정렬
SELECT * FROM users ORDER BY created_at DESC;
SELECT * FROM users ORDER BY username ASC, created_at DESC;

-- AS: 별칭 사용
SELECT
    username AS 사용자명,
    email AS 이메일,
    YEAR(created_at) AS 가입년도
FROM users;
```

### INSERT - 데이터 삽입
```sql
-- 단일 행 삽입
INSERT INTO users (username, email)
VALUES ('john_doe', 'john@example.com');

-- 여러 행 동시 삽입
INSERT INTO users (username, email) VALUES
('alice', 'alice@example.com'),
('bob', 'bob@example.com'),
('charlie', 'charlie@example.com');

-- 모든 컬럼에 값을 넣을 때 (컬럼명 생략 가능)
INSERT INTO users VALUES
(NULL, 'david', 'david@example.com', NOW());

-- SELECT 결과를 삽입
INSERT INTO users_backup
SELECT * FROM users WHERE created_at < '2024-01-01';

-- ON DUPLICATE KEY UPDATE: 중복 시 업데이트
INSERT INTO users (id, username, email)
VALUES (1, 'john', 'john@example.com')
ON DUPLICATE KEY UPDATE email = 'john@example.com';
```

### UPDATE - 데이터 수정
```sql
-- 특정 조건의 데이터 수정
UPDATE users
SET email = 'newemail@example.com'
WHERE username = 'john_doe';

-- 여러 컬럼 동시 수정
UPDATE users
SET
    email = 'updated@example.com',
    updated_at = NOW()
WHERE id = 1;

-- 모든 행 수정 (주의!)
UPDATE products SET price = price * 1.1;

-- 계산식 사용
UPDATE products
SET price = price * 0.9
WHERE category_id = 3;

-- LIMIT 사용
UPDATE users
SET status = 'inactive'
WHERE last_login < '2023-01-01'
LIMIT 100;
```

### DELETE - 데이터 삭제
```sql
-- 특정 조건의 데이터 삭제
DELETE FROM users WHERE id = 5;

-- 여러 조건
DELETE FROM orders
WHERE status = 'cancelled'
  AND created_at < '2023-01-01';

-- LIMIT 사용
DELETE FROM logs
WHERE created_at < '2024-01-01'
LIMIT 1000;

-- 모든 데이터 삭제 (주의!)
DELETE FROM temp_table;

-- TRUNCATE: 테이블의 모든 데이터 삭제 (빠름, 롤백 불가)
TRUNCATE TABLE temp_table;
```

---

## 5. 조건문과 필터링

### WHERE 절
```sql
-- 기본 비교 연산자
SELECT * FROM products WHERE price > 1000;
SELECT * FROM products WHERE price >= 1000 AND price <= 5000;
SELECT * FROM products WHERE category_id = 3;
SELECT * FROM products WHERE category_id != 3;
SELECT * FROM products WHERE category_id <> 3;  -- != 와 동일

-- IN: 여러 값 중 하나
SELECT * FROM users WHERE id IN (1, 3, 5, 7);
SELECT * FROM products WHERE category_id IN (1, 2, 3);

-- BETWEEN: 범위
SELECT * FROM products WHERE price BETWEEN 1000 AND 5000;
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- LIKE: 패턴 매칭
SELECT * FROM users WHERE username LIKE 'john%';     -- john으로 시작
SELECT * FROM users WHERE username LIKE '%smith';    -- smith로 끝남
SELECT * FROM users WHERE username LIKE '%admin%';   -- admin 포함
SELECT * FROM users WHERE email LIKE '_@%.com';      -- _는 한 글자

-- IS NULL / IS NOT NULL
SELECT * FROM users WHERE phone IS NULL;
SELECT * FROM users WHERE phone IS NOT NULL;

-- AND, OR, NOT
SELECT * FROM products
WHERE (price > 1000 AND category_id = 1)
   OR (price > 5000 AND category_id = 2);

SELECT * FROM users WHERE NOT (status = 'inactive');
```

### CASE 문
```sql
-- 기본 CASE 문
SELECT
    product_name,
    price,
    CASE
        WHEN price < 1000 THEN '저가'
        WHEN price BETWEEN 1000 AND 5000 THEN '중가'
        WHEN price > 5000 THEN '고가'
        ELSE '미분류'
    END AS price_category
FROM products;

-- 간단한 CASE
SELECT
    username,
    CASE status
        WHEN 'active' THEN '활성'
        WHEN 'inactive' THEN '비활성'
        WHEN 'banned' THEN '정지'
        ELSE '알 수 없음'
    END AS status_kr
FROM users;

-- UPDATE에서 CASE 사용
UPDATE products
SET discount = CASE
    WHEN price > 10000 THEN 0.15
    WHEN price > 5000 THEN 0.10
    WHEN price > 1000 THEN 0.05
    ELSE 0
END;
```

### IF 함수
```sql
SELECT
    product_name,
    price,
    IF(price > 5000, '고가', '일반') AS price_type
FROM products;

SELECT
    username,
    IF(email IS NOT NULL, '이메일 있음', '이메일 없음') AS email_status
FROM users;
```

---

## 6. JOIN 연산

### INNER JOIN
```sql
-- 기본 INNER JOIN: 양쪽 테이블에 모두 존재하는 데이터만
SELECT
    users.username,
    orders.order_id,
    orders.total_amount
FROM users
INNER JOIN orders ON users.id = orders.user_id;

-- 테이블 별칭 사용
SELECT
    u.username,
    o.order_id,
    o.total_amount,
    o.created_at
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- 여러 테이블 조인
SELECT
    u.username,
    o.order_id,
    p.product_name,
    oi.quantity,
    oi.price
FROM users u
INNER JOIN orders o ON u.id = o.user_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id;
```

### LEFT JOIN (LEFT OUTER JOIN)
```sql
-- 왼쪽 테이블의 모든 데이터 + 오른쪽 테이블의 매칭 데이터
SELECT
    u.username,
    o.order_id,
    o.total_amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 주문하지 않은 사용자 찾기
SELECT
    u.username,
    u.email
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.order_id IS NULL;

-- 여러 LEFT JOIN
SELECT
    u.username,
    COUNT(o.order_id) as order_count,
    SUM(o.total_amount) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;
```

### RIGHT JOIN (RIGHT OUTER JOIN)
```sql
-- 오른쪽 테이블의 모든 데이터 + 왼쪽 테이블의 매칭 데이터
SELECT
    o.order_id,
    u.username,
    o.total_amount
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;
```

### CROSS JOIN
```sql
-- 모든 조합 생성 (Cartesian Product)
SELECT
    colors.color_name,
    sizes.size_name
FROM colors
CROSS JOIN sizes;

-- 실용 예제: 날짜와 상품의 모든 조합
SELECT
    dates.date,
    products.product_name
FROM (
    SELECT DATE('2024-01-01') + INTERVAL n DAY as date
    FROM (SELECT 0 as n UNION SELECT 1 UNION SELECT 2) numbers
) dates
CROSS JOIN products;
```

### SELF JOIN
```sql
-- 같은 테이블을 자기 자신과 조인
-- 예: 직원과 매니저 관계
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(50),
    manager_id INT
);

SELECT
    e.emp_name AS employee,
    m.emp_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;

-- 같은 카테고리의 다른 상품 찾기
SELECT
    p1.product_name AS product1,
    p2.product_name AS product2,
    p1.category_id
FROM products p1
INNER JOIN products p2
    ON p1.category_id = p2.category_id
    AND p1.product_id < p2.product_id;
```

### UNION
```sql
-- 두 쿼리 결과를 합치기 (중복 제거)
SELECT username, email FROM customers
UNION
SELECT username, email FROM suppliers;

-- UNION ALL: 중복 허용
SELECT product_id FROM orders_2023
UNION ALL
SELECT product_id FROM orders_2024;

-- 다른 구조의 데이터를 통합
SELECT 'customer' as type, name, email FROM customers
UNION
SELECT 'supplier' as type, company_name, contact_email FROM suppliers;
```

---

## 7. 집계 함수와 그룹화

### 기본 집계 함수
```sql
-- COUNT: 개수
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT city) FROM users;
SELECT COUNT(email) FROM users;  -- NULL 제외

-- SUM: 합계
SELECT SUM(total_amount) FROM orders;
SELECT SUM(quantity * price) as revenue FROM order_items;

-- AVG: 평균
SELECT AVG(price) FROM products;
SELECT AVG(total_amount) as average_order FROM orders;

-- MIN, MAX: 최소값, 최대값
SELECT MIN(price), MAX(price) FROM products;
SELECT MIN(created_at) as first_order FROM orders;

-- 여러 집계 함수 동시 사용
SELECT
    COUNT(*) as total_products,
    AVG(price) as avg_price,
    MIN(price) as min_price,
    MAX(price) as max_price,
    SUM(stock_quantity) as total_stock
FROM products;
```

### GROUP BY
```sql
-- 기본 그룹화
SELECT
    category_id,
    COUNT(*) as product_count
FROM products
GROUP BY category_id;

-- 여러 컬럼으로 그룹화
SELECT
    category_id,
    is_active,
    COUNT(*) as count,
    AVG(price) as avg_price
FROM products
GROUP BY category_id, is_active;

-- 날짜 기준 그룹화
SELECT
    DATE(created_at) as order_date,
    COUNT(*) as order_count,
    SUM(total_amount) as daily_revenue
FROM orders
GROUP BY DATE(created_at)
ORDER BY order_date DESC;

-- 월별 집계
SELECT
    YEAR(created_at) as year,
    MONTH(created_at) as month,
    COUNT(*) as order_count,
    SUM(total_amount) as monthly_revenue
FROM orders
GROUP BY YEAR(created_at), MONTH(created_at)
ORDER BY year DESC, month DESC;
```

### HAVING 절
```sql
-- GROUP BY 결과를 필터링 (WHERE는 그룹화 전, HAVING은 그룹화 후)
SELECT
    category_id,
    COUNT(*) as product_count,
    AVG(price) as avg_price
FROM products
GROUP BY category_id
HAVING COUNT(*) > 5;

-- 여러 조건
SELECT
    user_id,
    COUNT(*) as order_count,
    SUM(total_amount) as total_spent
FROM orders
GROUP BY user_id
HAVING COUNT(*) >= 3 AND SUM(total_amount) > 100000;

-- WHERE와 HAVING 함께 사용
SELECT
    category_id,
    AVG(price) as avg_price
FROM products
WHERE is_active = TRUE
GROUP BY category_id
HAVING AVG(price) > 10000;
```

### WITH ROLLUP
```sql
-- 소계와 총계 포함
SELECT
    category_id,
    COUNT(*) as product_count,
    SUM(price) as total_price
FROM products
GROUP BY category_id WITH ROLLUP;

-- 여러 레벨의 소계
SELECT
    YEAR(created_at) as year,
    MONTH(created_at) as month,
    SUM(total_amount) as revenue
FROM orders
GROUP BY year, month WITH ROLLUP;
```

---

## 8. 인덱스

### 인덱스 생성
```sql
-- 단일 컬럼 인덱스
CREATE INDEX idx_username ON users(username);
CREATE INDEX idx_email ON users(email);

-- 복합 인덱스 (여러 컬럼)
CREATE INDEX idx_name_email ON users(username, email);
CREATE INDEX idx_category_price ON products(category_id, price);

-- UNIQUE 인덱스
CREATE UNIQUE INDEX idx_unique_email ON users(email);

-- 테이블 생성 시 인덱스 정의
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    category_id INT,
    price DECIMAL(10, 2),
    INDEX idx_category (category_id),
    INDEX idx_price (price),
    INDEX idx_category_price (category_id, price)
);

-- FULLTEXT 인덱스 (전문 검색용)
CREATE FULLTEXT INDEX idx_fulltext_title ON articles(title);
CREATE FULLTEXT INDEX idx_fulltext_content ON articles(title, content);

-- 💡 MariaDB 11.8+ VECTOR 인덱스 (AI/ML 벡터 검색용)
CREATE TABLE embeddings (
    doc_id BIGINT UNSIGNED PRIMARY KEY,
    embedding VECTOR(1536) NOT NULL,
    -- VECTOR 인덱스: M=인접 노드 수(기본 16), DISTANCE=거리 계산 방식
    VECTOR INDEX (embedding) M=8 DISTANCE=cosine    -- 또는 DISTANCE=euclidean
);

-- 기존 테이블에 VECTOR 인덱스 추가
ALTER TABLE embeddings ADD VECTOR INDEX idx_vec (embedding) M=16 DISTANCE=euclidean;
```

### 인덱스 조회 및 삭제
```sql
-- 테이블의 인덱스 확인
SHOW INDEX FROM users;
SHOW KEYS FROM products;

-- 인덱스 삭제
DROP INDEX idx_username ON users;
ALTER TABLE users DROP INDEX idx_email;

-- 인덱스 재생성
ALTER TABLE users DROP INDEX idx_username;
CREATE INDEX idx_username ON users(username);
```

### 인덱스 사용 확인
```sql
-- EXPLAIN: 쿼리 실행 계획 확인
EXPLAIN SELECT * FROM users WHERE username = 'john';

EXPLAIN SELECT * FROM products
WHERE category_id = 3 AND price > 1000;

-- EXPLAIN EXTENDED: 더 자세한 정보
EXPLAIN EXTENDED SELECT * FROM orders WHERE user_id = 100;

-- ANALYZE: 실제 실행 통계
ANALYZE TABLE users;
```

### 인덱스 최적화 팁
```sql
-- 좋은 예: 인덱스 활용
SELECT * FROM products WHERE category_id = 3;  -- category_id에 인덱스 있음

-- 나쁜 예: 인덱스 무효화
SELECT * FROM products WHERE YEAR(created_at) = 2024;  -- 함수 사용

-- 개선: 인덱스 활용 가능하도록 변경
SELECT * FROM products
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- 복합 인덱스 활용: (category_id, price) 인덱스가 있을 때
SELECT * FROM products
WHERE category_id = 3 AND price > 1000;  -- 인덱스 완전 활용

SELECT * FROM products
WHERE category_id = 3;  -- 인덱스 부분 활용

SELECT * FROM products
WHERE price > 1000;  -- 인덱스 활용 불가 (첫 번째 컬럼이 아님)
```

---

## 9. 사용자 및 권한 관리

### 사용자 생성
```sql
-- 기본 사용자 생성
CREATE USER 'bootuser'@'localhost' IDENTIFIED BY 'password123';

-- 모든 호스트에서 접근 가능한 사용자
CREATE USER 'bootuser'@'%' IDENTIFIED BY 'password123';

-- 특정 IP에서만 접근 가능
CREATE USER 'bootuser'@'192.168.1.100' IDENTIFIED BY 'password123';

-- 비밀번호 없는 사용자 (권장하지 않음)
CREATE USER 'guest'@'localhost';

-- 비밀번호 만료 설정
CREATE USER 'tempuser'@'localhost'
IDENTIFIED BY 'temp123'
PASSWORD EXPIRE INTERVAL 30 DAY;
```

### 사용자 조회 및 수정
```sql
-- 모든 사용자 조회
SELECT User, Host FROM mysql.user;

-- 현재 사용자 확인
SELECT USER(), CURRENT_USER();

-- 사용자 비밀번호 변경
ALTER USER 'bootuser'@'localhost' IDENTIFIED BY 'newpassword456';
SET PASSWORD FOR 'bootuser'@'localhost' = PASSWORD('newpassword456');

-- 현재 사용자의 비밀번호 변경
SET PASSWORD = PASSWORD('mynewpassword');

-- 사용자 이름 변경
RENAME USER 'oldname'@'localhost' TO 'newname'@'localhost';

-- 사용자 삭제
DROP USER 'bootuser'@'localhost';
DROP USER IF EXISTS 'tempuser'@'localhost';
```

### 권한 부여 (GRANT)
```sql
-- 특정 데이터베이스의 모든 권한
GRANT ALL PRIVILEGES ON bootex.* TO 'bootuser'@'localhost';

-- 특정 테이블의 SELECT 권한만
GRANT SELECT ON bootex.users TO 'readonly'@'localhost';

-- 여러 권한 동시 부여
GRANT SELECT, INSERT, UPDATE ON bootex.* TO 'appuser'@'localhost';

-- 특정 컬럼에만 권한 부여
GRANT SELECT (username, email), UPDATE (email)
ON bootex.users TO 'limiteduser'@'localhost';

-- 전역 권한 부여
GRANT CREATE, DROP ON *.* TO 'admin'@'localhost';

-- WITH GRANT OPTION: 권한을 다른 사용자에게 부여할 수 있는 권한
GRANT ALL PRIVILEGES ON bootex.* TO 'manager'@'localhost'
WITH GRANT OPTION;

-- 권한 즉시 적용
FLUSH PRIVILEGES;
```

### 권한 조회
```sql
-- 특정 사용자의 권한 확인
SHOW GRANTS FOR 'bootuser'@'localhost';

-- 현재 사용자의 권한 확인
SHOW GRANTS;
SHOW GRANTS FOR CURRENT_USER;

-- 모든 사용자와 권한 조회
SELECT User, Host, Select_priv, Insert_priv, Update_priv, Delete_priv
FROM mysql.user;

SELECT * FROM mysql.db WHERE User = 'bootuser';
```

### 권한 회수 (REVOKE)
```sql
-- 특정 권한 회수
REVOKE INSERT, UPDATE ON bootex.* FROM 'bootuser'@'localhost';

-- 모든 권한 회수
REVOKE ALL PRIVILEGES ON bootex.* FROM 'bootuser'@'localhost';

-- GRANT OPTION 회수
REVOKE GRANT OPTION ON bootex.* FROM 'manager'@'localhost';

-- 권한 즉시 적용
FLUSH PRIVILEGES;
```

### 주요 권한 종류
```sql
-- 데이터 관련 권한
-- SELECT: 데이터 조회
-- INSERT: 데이터 삽입
-- UPDATE: 데이터 수정
-- DELETE: 데이터 삭제

-- 구조 관련 권한
-- CREATE: 데이터베이스/테이블 생성
-- ALTER: 테이블 구조 변경
-- DROP: 데이터베이스/테이블 삭제
-- INDEX: 인덱스 생성/삭제

-- 관리 권한
-- CREATE USER: 사용자 생성
-- GRANT OPTION: 권한 부여 권한
-- RELOAD: FLUSH 명령 사용
-- SHUTDOWN: 서버 종료
-- PROCESS: 프로세스 조회
-- SUPER: 슈퍼유저 권한

-- 예제: 일반 애플리케이션 사용자
GRANT SELECT, INSERT, UPDATE, DELETE ON bootex.* TO 'appuser'@'localhost';

-- 예제: 읽기 전용 사용자
GRANT SELECT ON bootex.* TO 'readonly'@'localhost';

-- 예제: 백업 전용 사용자
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER
ON bootex.* TO 'backup'@'localhost';

-- 예제: 개발자 계정
GRANT ALL PRIVILEGES ON bootex_dev.* TO 'developer'@'localhost';
```

### 실전 예제: Spring Boot 애플리케이션용 사용자 설정
```sql
-- 1. 데이터베이스 생성
CREATE DATABASE bootex
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

-- 2. 사용자 생성
CREATE USER 'bootuser'@'localhost' IDENTIFIED BY 'Boot@2024!';

-- 3. 필요한 권한만 부여 (최소 권한 원칙)
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, ALTER
ON bootex.* TO 'bootuser'@'localhost';

-- 4. 권한 적용
FLUSH PRIVILEGES;

-- 5. 확인
SHOW GRANTS FOR 'bootuser'@'localhost';
```

---

## 10. 트랜잭션

### 트랜잭션 기본
```sql
-- 트랜잭션 시작
START TRANSACTION;
-- 또는
BEGIN;

-- SQL 명령 실행
UPDATE accounts SET balance = balance - 1000 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE user_id = 2;

-- 트랜잭션 확정 (COMMIT)
COMMIT;

-- 트랜잭션 취소 (ROLLBACK)
ROLLBACK;
```

### 트랜잭션 실전 예제
```sql
-- 계좌 이체 예제
START TRANSACTION;

-- 송금자 계좌에서 출금
UPDATE accounts
SET balance = balance - 50000
WHERE account_id = 1;

-- 잔액 확인
SELECT balance FROM accounts WHERE account_id = 1;

-- 수취인 계좌에 입금
UPDATE accounts
SET balance = balance + 50000
WHERE account_id = 2;

-- 거래 내역 기록
INSERT INTO transactions (from_account, to_account, amount, transaction_date)
VALUES (1, 2, 50000, NOW());

-- 모든 작업이 성공하면 확정
COMMIT;
-- 문제가 있으면 롤백
-- ROLLBACK;
```

### SAVEPOINT
```sql
-- 복잡한 트랜잭션에서 중간 지점 설정
START TRANSACTION;

INSERT INTO orders (user_id, total_amount) VALUES (1, 50000);
SET @order_id = LAST_INSERT_ID();

SAVEPOINT sp1;

INSERT INTO order_items (order_id, product_id, quantity)
VALUES (@order_id, 101, 2);

SAVEPOINT sp2;

UPDATE products SET stock_quantity = stock_quantity - 2 WHERE product_id = 101;

-- sp2 시점으로 롤백 (재고 업데이트만 취소)
ROLLBACK TO SAVEPOINT sp2;

-- 또는 sp1 시점으로 롤백 (order_items 삽입도 취소)
ROLLBACK TO SAVEPOINT sp1;

-- 최종 확정
COMMIT;
```

### 자동 커밋 설정
```sql
-- 자동 커밋 상태 확인
SELECT @@autocommit;

-- 자동 커밋 비활성화
SET autocommit = 0;

-- 자동 커밋 활성화
SET autocommit = 1;

-- 자동 커밋이 비활성화된 상태에서 작업
SET autocommit = 0;
UPDATE users SET status = 'active' WHERE id = 1;
COMMIT;  -- 명시적으로 커밋 필요
```

### 트랜잭션 격리 수준
```sql
-- 현재 격리 수준 확인
SELECT @@transaction_isolation;

-- 격리 수준 설정
-- READ UNCOMMITTED: 커밋되지 않은 데이터 읽기 가능
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- READ COMMITTED: 커밋된 데이터만 읽기 가능
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- REPEATABLE READ: 트랜잭션 내에서 일관된 읽기 (MariaDB 기본값)
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- SERIALIZABLE: 가장 엄격한 격리 수준
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 전역 설정
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### 잠금 (Locking)
```sql
-- FOR UPDATE: 쓰기 잠금
START TRANSACTION;
SELECT * FROM products WHERE product_id = 1 FOR UPDATE;
UPDATE products SET stock_quantity = stock_quantity - 1 WHERE product_id = 1;
COMMIT;

-- LOCK IN SHARE MODE: 읽기 잠금
START TRANSACTION;
SELECT * FROM products WHERE product_id = 1 LOCK IN SHARE MODE;
-- 다른 트랜잭션이 읽을 수는 있지만 수정은 불가
COMMIT;

-- 테이블 잠금
LOCK TABLES products WRITE;
UPDATE products SET price = price * 1.1;
UNLOCK TABLES;

-- 읽기 잠금
LOCK TABLES products READ;
SELECT * FROM products;
UNLOCK TABLES;
```

### 대용량 트랜잭션 성능 개선 (MariaDB 11.8+)
```sql
-- 💡 MariaDB 11.8 성능 개선: 대용량 트랜잭션 커밋 시 다른 트랜잭션 차단 해소
-- 이전 버전: binlog가 활성화된 환경에서 대용량 트랜잭션 커밋 시 모든 트랜잭션이 대기
-- MariaDB 11.8+: 대용량 트랜잭션 커밋 중에도 다른 트랜잭션이 정상적으로 처리됨

-- 대용량 데이터 처리 예제 (11.8+에서 성능 향상)
START TRANSACTION;

-- 수백만 건의 데이터 삽입/수정
INSERT INTO large_table SELECT * FROM source_table WHERE created_at > '2024-01-01';
UPDATE large_table SET status = 'processed' WHERE batch_id = 12345;

-- 이전 버전: COMMIT 시 다른 세션들이 freeze
-- MariaDB 11.8+: COMMIT 중에도 다른 트랜잭션들이 정상 작동
COMMIT;
```

---

## 11. 뷰(View)

### 뷰 생성
```sql
-- 기본 뷰 생성
CREATE VIEW active_users AS
SELECT id, username, email, created_at
FROM users
WHERE status = 'active';

-- 복잡한 조인을 뷰로 단순화
CREATE VIEW order_details AS
SELECT
    o.order_id,
    u.username,
    u.email,
    o.total_amount,
    o.status,
    o.created_at
FROM orders o
INNER JOIN users u ON o.user_id = u.id;

-- 집계 결과를 뷰로
CREATE VIEW product_stats AS
SELECT
    p.category_id,
    c.category_name,
    COUNT(*) as product_count,
    AVG(p.price) as avg_price,
    MIN(p.price) as min_price,
    MAX(p.price) as max_price
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
GROUP BY p.category_id, c.category_name;

-- OR REPLACE: 뷰가 있으면 대체
CREATE OR REPLACE VIEW active_users AS
SELECT id, username, email, phone, created_at
FROM users
WHERE status = 'active';
```

### 뷰 사용
```sql
-- 일반 테이블처럼 사용
SELECT * FROM active_users;

SELECT * FROM order_details
WHERE created_at > '2024-01-01'
ORDER BY total_amount DESC;

-- 뷰를 조인에 사용
SELECT
    od.*,
    p.product_name
FROM order_details od
INNER JOIN order_items oi ON od.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id;
```

### 뷰 조회 및 관리
```sql
-- 데이터베이스의 뷰 목록
SHOW FULL TABLES WHERE table_type = 'VIEW';

-- 뷰 정의 확인
SHOW CREATE VIEW active_users;

-- 뷰 삭제
DROP VIEW active_users;
DROP VIEW IF EXISTS order_details;

-- 여러 뷰 동시 삭제
DROP VIEW IF EXISTS view1, view2, view3;
```

### 뷰 수정
```sql
-- ALTER VIEW로 수정
ALTER VIEW active_users AS
SELECT id, username, email, phone, address, created_at
FROM users
WHERE status = 'active' AND email IS NOT NULL;

-- CREATE OR REPLACE로 수정
CREATE OR REPLACE VIEW product_stats AS
SELECT
    p.category_id,
    c.category_name,
    COUNT(*) as product_count,
    AVG(p.price) as avg_price,
    SUM(p.stock_quantity) as total_stock
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
GROUP BY p.category_id, c.category_name;
```

### 업데이트 가능한 뷰
```sql
-- 단순한 뷰는 INSERT, UPDATE, DELETE 가능
CREATE VIEW simple_users AS
SELECT id, username, email
FROM users;

-- 뷰를 통한 데이터 수정
UPDATE simple_users SET email = 'new@example.com' WHERE id = 1;
INSERT INTO simple_users (username, email) VALUES ('newuser', 'new@example.com');
DELETE FROM simple_users WHERE id = 10;

-- 복잡한 뷰는 읽기 전용 (조인, 집계 함수, DISTINCT 등 포함)
-- 이런 뷰는 UPDATE, INSERT, DELETE 불가
```

---

## 12. 저장 프로시저와 함수

### 저장 프로시저 생성
```sql
-- 구분자 변경 (프로시저 내부에서 세미콜론 사용하기 위함)
DELIMITER //

-- 기본 프로시저
CREATE PROCEDURE GetAllUsers()
BEGIN
    SELECT * FROM users;
END //

DELIMITER ;

-- 프로시저 실행
CALL GetAllUsers();
```

### 매개변수가 있는 프로시저
```sql
DELIMITER //

-- IN 매개변수: 입력
CREATE PROCEDURE GetUserById(IN user_id INT)
BEGIN
    SELECT * FROM users WHERE id = user_id;
END //

-- OUT 매개변수: 출력
CREATE PROCEDURE GetUserCount(OUT user_count INT)
BEGIN
    SELECT COUNT(*) INTO user_count FROM users;
END //

-- INOUT 매개변수: 입출력
CREATE PROCEDURE IncreasePrice(INOUT price DECIMAL(10,2), IN increase_rate DECIMAL(3,2))
BEGIN
    SET price = price * (1 + increase_rate);
END //

-- 💡 MariaDB 11.8+ 신규 기능: 기본값이 있는 매개변수
CREATE PROCEDURE GetUsersByStatus(
    IN user_status VARCHAR(20) DEFAULT 'active',  -- 기본값 지정
    IN result_limit INT DEFAULT 10
)
BEGIN
    SELECT * FROM users
    WHERE status = user_status
    LIMIT result_limit;
END //

DELIMITER ;

-- 실행 예제
CALL GetUserById(1);

SET @count = 0;
CALL GetUserCount(@count);
SELECT @count;

SET @price = 1000;
CALL IncreasePrice(@price, 0.1);
SELECT @price;  -- 1100

-- 💡 MariaDB 11.8+ 기본값 매개변수 사용 예제
CALL GetUsersByStatus();                    -- 기본값 사용: 'active', 10
CALL GetUsersByStatus('inactive');          -- 첫 번째 매개변수만 지정
CALL GetUsersByStatus('banned', 20);        -- 모든 매개변수 지정
CALL GetUsersByStatus(DEFAULT, 5);          -- 첫 번째는 기본값, 두 번째만 지정
```

### 복잡한 프로시저 예제
```sql
DELIMITER //

CREATE PROCEDURE ProcessOrder(
    IN p_user_id INT,
    IN p_product_id INT,
    IN p_quantity INT,
    OUT p_order_id INT,
    OUT p_status VARCHAR(50)
)
BEGIN
    DECLARE v_stock INT;
    DECLARE v_price DECIMAL(10,2);
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_status = 'ERROR';
    END;

    START TRANSACTION;

    -- 재고 확인
    SELECT stock_quantity, price INTO v_stock, v_price
    FROM products
    WHERE product_id = p_product_id
    FOR UPDATE;

    -- 재고 부족 체크
    IF v_stock < p_quantity THEN
        SET p_status = 'OUT_OF_STOCK';
        ROLLBACK;
    ELSE
        -- 주문 생성
        INSERT INTO orders (user_id, total_amount, status, created_at)
        VALUES (p_user_id, v_price * p_quantity, 'pending', NOW());

        SET p_order_id = LAST_INSERT_ID();

        -- 주문 아이템 추가
        INSERT INTO order_items (order_id, product_id, quantity, price)
        VALUES (p_order_id, p_product_id, p_quantity, v_price);

        -- 재고 감소
        UPDATE products
        SET stock_quantity = stock_quantity - p_quantity
        WHERE product_id = p_product_id;

        COMMIT;
        SET p_status = 'SUCCESS';
    END IF;
END //

DELIMITER ;

-- 실행
SET @order_id = 0;
SET @status = '';
CALL ProcessOrder(1, 101, 2, @order_id, @status);
SELECT @order_id, @status;
```

### 저장 함수
```sql
DELIMITER //

-- 함수 생성 (프로시저와 달리 값을 반환)
CREATE FUNCTION CalculateDiscount(price DECIMAL(10,2), discount_rate DECIMAL(3,2))
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    DECLARE discounted_price DECIMAL(10,2);
    SET discounted_price = price * (1 - discount_rate);
    RETURN discounted_price;
END //

-- 더 복잡한 함수
CREATE FUNCTION GetUserOrderCount(user_id INT)
RETURNS INT
READS SQL DATA
BEGIN
    DECLARE order_count INT;
    SELECT COUNT(*) INTO order_count
    FROM orders
    WHERE orders.user_id = user_id;
    RETURN order_count;
END //

DELIMITER ;

-- 함수 사용
SELECT CalculateDiscount(10000, 0.15);  -- 8500
SELECT product_name, price, CalculateDiscount(price, 0.2) as sale_price
FROM products;

SELECT username, GetUserOrderCount(id) as order_count
FROM users;

-- 💡 MariaDB 11.8+ ROW 데이터 타입 반환 함수
DELIMITER //

CREATE FUNCTION GetUserInfo(user_id INT)
RETURNS ROW(username VARCHAR(50), email VARCHAR(100), total_orders INT)
READS SQL DATA
BEGIN
    DECLARE result ROW(username VARCHAR(50), email VARCHAR(100), total_orders INT);

    SELECT u.username, u.email, COUNT(o.order_id)
    INTO result.username, result.email, result.total_orders
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id = user_id
    GROUP BY u.id;

    RETURN result;
END //

DELIMITER ;

-- ROW 타입 함수 사용
SELECT GetUserInfo(1).username;
SELECT GetUserInfo(1).email;
SELECT GetUserInfo(1).total_orders;
```

### 프로시저/함수 관리
```sql
-- 프로시저 목록
SHOW PROCEDURE STATUS WHERE Db = 'bootex';

-- 함수 목록
SHOW FUNCTION STATUS WHERE Db = 'bootex';

-- 프로시저 정의 확인
SHOW CREATE PROCEDURE GetAllUsers;

-- 함수 정의 확인
SHOW CREATE FUNCTION CalculateDiscount;

-- 프로시저 삭제
DROP PROCEDURE IF EXISTS GetAllUsers;

-- 함수 삭제
DROP FUNCTION IF EXISTS CalculateDiscount;
```

### 제어문
```sql
DELIMITER //

-- IF 문
CREATE PROCEDURE CheckStock(IN product_id INT)
BEGIN
    DECLARE stock INT;
    SELECT stock_quantity INTO stock FROM products WHERE product_id = product_id;

    IF stock > 100 THEN
        SELECT '재고 충분' as status;
    ELSEIF stock > 10 THEN
        SELECT '재고 보통' as status;
    ELSE
        SELECT '재고 부족' as status;
    END IF;
END //

-- CASE 문
CREATE PROCEDURE GetPriceCategory(IN price DECIMAL(10,2))
BEGIN
    DECLARE category VARCHAR(20);

    CASE
        WHEN price < 1000 THEN SET category = '저가';
        WHEN price < 5000 THEN SET category = '중가';
        ELSE SET category = '고가';
    END CASE;

    SELECT category;
END //

-- LOOP
CREATE PROCEDURE InsertNumbers(IN max_num INT)
BEGIN
    DECLARE counter INT DEFAULT 1;

    my_loop: LOOP
        INSERT INTO numbers (num) VALUES (counter);
        SET counter = counter + 1;

        IF counter > max_num THEN
            LEAVE my_loop;
        END IF;
    END LOOP;
END //

-- WHILE
CREATE PROCEDURE InsertNumbersWhile(IN max_num INT)
BEGIN
    DECLARE counter INT DEFAULT 1;

    WHILE counter <= max_num DO
        INSERT INTO numbers (num) VALUES (counter);
        SET counter = counter + 1;
    END WHILE;
END //

-- REPEAT
CREATE PROCEDURE InsertNumbersRepeat(IN max_num INT)
BEGIN
    DECLARE counter INT DEFAULT 1;

    REPEAT
        INSERT INTO numbers (num) VALUES (counter);
        SET counter = counter + 1;
    UNTIL counter > max_num
    END REPEAT;
END //

DELIMITER ;
```

---

## 13. 트리거

### 트리거 생성
```sql
DELIMITER //

-- BEFORE INSERT 트리거
CREATE TRIGGER before_user_insert
BEFORE INSERT ON users
FOR EACH ROW
BEGIN
    -- 이메일 소문자 변환
    SET NEW.email = LOWER(NEW.email);
    -- 생성 시간 자동 설정
    SET NEW.created_at = NOW();
END //

-- AFTER INSERT 트리거
CREATE TRIGGER after_order_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    -- 주문 생성 시 로그 기록
    INSERT INTO order_logs (order_id, action, created_at)
    VALUES (NEW.order_id, 'ORDER_CREATED', NOW());
END //

-- BEFORE UPDATE 트리거
CREATE TRIGGER before_product_update
BEFORE UPDATE ON products
FOR EACH ROW
BEGIN
    -- 가격 변경 시 이력 저장
    IF NEW.price != OLD.price THEN
        INSERT INTO price_history (product_id, old_price, new_price, changed_at)
        VALUES (OLD.product_id, OLD.price, NEW.price, NOW());
    END IF;

    -- 수정 시간 자동 업데이트
    SET NEW.updated_at = NOW();
END //

-- AFTER UPDATE 트리거
CREATE TRIGGER after_user_update
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    -- 상태 변경 알림
    IF NEW.status != OLD.status THEN
        INSERT INTO notifications (user_id, message, created_at)
        VALUES (NEW.id, CONCAT('상태가 ', OLD.status, '에서 ', NEW.status, '로 변경됨'), NOW());
    END IF;
END //

-- BEFORE DELETE 트리거
CREATE TRIGGER before_user_delete
BEFORE DELETE ON users
FOR EACH ROW
BEGIN
    -- 삭제 전 아카이브
    INSERT INTO users_archive SELECT * FROM users WHERE id = OLD.id;
END //

-- AFTER DELETE 트리거
CREATE TRIGGER after_product_delete
AFTER DELETE ON products
FOR EACH ROW
BEGIN
    -- 삭제 로그
    INSERT INTO deletion_logs (table_name, record_id, deleted_at)
    VALUES ('products', OLD.product_id, NOW());
END //

DELIMITER ;
```

### 실전 트리거 예제
```sql
DELIMITER //

-- 재고 자동 업데이트
CREATE TRIGGER update_stock_after_order
AFTER INSERT ON order_items
FOR EACH ROW
BEGIN
    UPDATE products
    SET stock_quantity = stock_quantity - NEW.quantity
    WHERE product_id = NEW.product_id;
END //

-- 주문 총액 자동 계산
CREATE TRIGGER calculate_order_total
BEFORE INSERT ON order_items
FOR EACH ROW
BEGIN
    DECLARE item_total DECIMAL(10,2);
    SET item_total = NEW.quantity * NEW.price;

    UPDATE orders
    SET total_amount = total_amount + item_total
    WHERE order_id = NEW.order_id;
END //

-- 감사 추적 (Audit Trail)
CREATE TRIGGER audit_user_changes
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (
        table_name,
        record_id,
        action,
        old_values,
        new_values,
        changed_by,
        changed_at
    ) VALUES (
        'users',
        NEW.id,
        'UPDATE',
        JSON_OBJECT('username', OLD.username, 'email', OLD.email, 'status', OLD.status),
        JSON_OBJECT('username', NEW.username, 'email', NEW.email, 'status', NEW.status),
        USER(),
        NOW()
    );
END //

DELIMITER ;
```

### 트리거 관리
```sql
-- 트리거 목록 조회
SHOW TRIGGERS;
SHOW TRIGGERS FROM bootex;
SHOW TRIGGERS LIKE 'user%';

-- 특정 테이블의 트리거 조회
SHOW TRIGGERS WHERE `Table` = 'users';

-- 트리거 정의 확인
SHOW CREATE TRIGGER before_user_insert;

-- 트리거 삭제
DROP TRIGGER IF EXISTS before_user_insert;

-- 트리거 비활성화는 불가능 (삭제만 가능)
-- 필요시 조건문으로 우회
DELIMITER //
CREATE TRIGGER optional_trigger
BEFORE INSERT ON some_table
FOR EACH ROW
BEGIN
    -- 세션 변수로 트리거 활성화 여부 제어
    IF @disable_trigger IS NULL OR @disable_trigger = 0 THEN
        -- 트리거 로직
        SET NEW.created_at = NOW();
    END IF;
END //
DELIMITER ;

-- 사용
SET @disable_trigger = 1;  -- 트리거 비활성화
INSERT INTO some_table ...;
SET @disable_trigger = 0;  -- 트리거 활성화
```

---

## 14. 고급 기능

### 서브쿼리
```sql
-- WHERE 절의 서브쿼리
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- IN 사용
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders WHERE total_amount > 100000);

-- EXISTS 사용
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- NOT EXISTS
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- FROM 절의 서브쿼리 (인라인 뷰)
SELECT category, avg_price
FROM (
    SELECT
        category_id,
        AVG(price) as avg_price
    FROM products
    GROUP BY category_id
) AS category_stats
WHERE avg_price > 10000;

-- SELECT 절의 서브쿼리
SELECT
    username,
    email,
    (SELECT COUNT(*) FROM orders WHERE user_id = users.id) as order_count,
    (SELECT SUM(total_amount) FROM orders WHERE user_id = users.id) as total_spent
FROM users;
```

### 공통 테이블 표현식 (CTE - WITH 절)
```sql
-- 기본 CTE
WITH active_users AS (
    SELECT * FROM users WHERE status = 'active'
)
SELECT * FROM active_users WHERE created_at > '2024-01-01';

-- 여러 CTE
WITH
high_value_orders AS (
    SELECT * FROM orders WHERE total_amount > 50000
),
premium_users AS (
    SELECT user_id, COUNT(*) as order_count
    FROM high_value_orders
    GROUP BY user_id
    HAVING COUNT(*) > 5
)
SELECT
    u.username,
    p.order_count
FROM users u
INNER JOIN premium_users p ON u.id = p.user_id;

-- 재귀 CTE (조직도, 카테고리 트리 등)
WITH RECURSIVE category_tree AS (
    -- 최상위 카테고리
    SELECT category_id, category_name, parent_id, 1 as level
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- 하위 카테고리
    SELECT c.category_id, c.category_name, c.parent_id, ct.level + 1
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT * FROM category_tree ORDER BY level, category_name;
```

### 윈도우 함수
```sql
-- ROW_NUMBER: 행 번호
SELECT
    product_name,
    category_id,
    price,
    ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) as price_rank
FROM products;

-- RANK: 순위 (동일 값은 동일 순위, 다음 순위는 건너뜀)
SELECT
    username,
    total_spent,
    RANK() OVER (ORDER BY total_spent DESC) as spending_rank
FROM (
    SELECT u.username, SUM(o.total_amount) as total_spent
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    GROUP BY u.id, u.username
) user_totals;

-- DENSE_RANK: 밀집 순위 (동일 값은 동일 순위, 다음 순위는 연속)
SELECT
    product_name,
    price,
    DENSE_RANK() OVER (ORDER BY price DESC) as price_rank
FROM products;

-- NTILE: N개 그룹으로 분할
SELECT
    username,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) as quartile
FROM user_spending;

-- LAG: 이전 행 값
SELECT
    order_date,
    daily_revenue,
    LAG(daily_revenue, 1) OVER (ORDER BY order_date) as prev_day_revenue
FROM daily_stats;

-- LEAD: 다음 행 값
SELECT
    order_date,
    daily_revenue,
    LEAD(daily_revenue, 1) OVER (ORDER BY order_date) as next_day_revenue
FROM daily_stats;

-- FIRST_VALUE, LAST_VALUE
SELECT
    product_name,
    category_id,
    price,
    FIRST_VALUE(price) OVER (PARTITION BY category_id ORDER BY price) as min_price,
    LAST_VALUE(price) OVER (
        PARTITION BY category_id
        ORDER BY price
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as max_price
FROM products;

-- SUM, AVG, COUNT (윈도우 함수로)
SELECT
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_7day_sum,
    AVG(daily_revenue) OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as rolling_7day_avg
FROM daily_stats;
```

### 문자열 함수
```sql
-- CONCAT: 문자열 연결
SELECT CONCAT(first_name, ' ', last_name) as full_name FROM users;
SELECT CONCAT_WS('-', year, month, day) as date_string FROM dates;

-- SUBSTRING: 부분 문자열
SELECT SUBSTRING(email, 1, 3) as email_prefix FROM users;
SELECT SUBSTRING(phone, -4) as last_4_digits FROM users;

-- LENGTH, CHAR_LENGTH
SELECT username, LENGTH(username), CHAR_LENGTH(username) FROM users;

-- UPPER, LOWER
SELECT UPPER(username), LOWER(email) FROM users;

-- TRIM, LTRIM, RTRIM
SELECT TRIM('  hello  ');  -- 'hello'
SELECT LTRIM('  hello  ');  -- 'hello  '
SELECT RTRIM('  hello  ');  -- '  hello'

-- REPLACE
SELECT REPLACE(description, 'old', 'new') FROM products;

-- LEFT, RIGHT
SELECT LEFT(username, 3), RIGHT(username, 3) FROM users;

-- LOCATE, POSITION
SELECT LOCATE('@', email) as at_position FROM users;
SELECT POSITION('com' IN email) as com_position FROM users;
```

### 날짜/시간 함수
```sql
-- 현재 날짜/시간
SELECT NOW(), CURDATE(), CURTIME();
SELECT CURRENT_TIMESTAMP(), CURRENT_DATE(), CURRENT_TIME();

-- 날짜 추출
SELECT
    YEAR(created_at) as year,
    MONTH(created_at) as month,
    DAY(created_at) as day,
    HOUR(created_at) as hour,
    MINUTE(created_at) as minute,
    SECOND(created_at) as second
FROM orders;

-- 날짜 포맷
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');
SELECT DATE_FORMAT(created_at, '%Y년 %m월 %d일') FROM orders;

-- 날짜 연산
SELECT DATE_ADD(NOW(), INTERVAL 7 DAY);
SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH);
SELECT created_at + INTERVAL 1 YEAR FROM users;

-- 날짜 차이
SELECT DATEDIFF(NOW(), created_at) as days_ago FROM users;
SELECT TIMESTAMPDIFF(HOUR, created_at, NOW()) as hours_ago FROM orders;

-- 날짜 변환
SELECT STR_TO_DATE('2024-12-25', '%Y-%m-%d');
SELECT UNIX_TIMESTAMP(NOW());
SELECT FROM_UNIXTIME(1703505600);
```

### 조건 및 변환 함수
```sql
-- IFNULL, COALESCE
SELECT username, IFNULL(phone, '전화번호 없음') FROM users;
SELECT username, COALESCE(phone, email, '연락처 없음') FROM users;

-- NULLIF
SELECT NULLIF(stock_quantity, 0) FROM products;  -- 0이면 NULL 반환

-- IF 함수
SELECT product_name, IF(stock_quantity > 0, '재고있음', '품절') as status
FROM products;

-- CASE
SELECT
    product_name,
    price,
    CASE
        WHEN price < 1000 THEN '저가'
        WHEN price < 5000 THEN '중가'
        ELSE '고가'
    END as price_category
FROM products;
```

---

## 15. 성능 최적화

### 쿼리 최적화 기본
```sql
-- EXPLAIN으로 쿼리 분석
EXPLAIN SELECT * FROM users WHERE username = 'john';

EXPLAIN SELECT
    u.username,
    COUNT(o.order_id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- EXPLAIN EXTENDED로 더 자세한 정보
EXPLAIN EXTENDED SELECT * FROM products WHERE category_id = 3;
SHOW WARNINGS;

-- EXPLAIN 결과 해석
-- type: ALL (최악) < index < range < ref < eq_ref < const (최상)
-- key: 사용된 인덱스
-- rows: 검사할 예상 행 수
-- Extra: 추가 정보 (Using index, Using temporary, Using filesort 등)
```

### 인덱스 최적화
```sql
-- 적절한 인덱스 생성
-- WHERE 절에 자주 사용되는 컬럼
CREATE INDEX idx_status ON orders(status);

-- JOIN에 사용되는 외래키
CREATE INDEX idx_user_id ON orders(user_id);

-- ORDER BY에 사용되는 컬럼
CREATE INDEX idx_created_at ON orders(created_at);

-- 복합 인덱스 (순서 중요!)
CREATE INDEX idx_category_price ON products(category_id, price);

-- 커버링 인덱스: 인덱스만으로 쿼리 해결
CREATE INDEX idx_covering ON users(username, email);
SELECT username, email FROM users WHERE username = 'john';  -- 테이블 접근 불필요

-- 불필요한 인덱스 제거
-- 중복 인덱스 확인
SELECT
    TABLE_NAME,
    INDEX_NAME,
    GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) as columns
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'bootex'
GROUP BY TABLE_NAME, INDEX_NAME;
```

### 쿼리 작성 팁
```sql
-- ❌ SELECT * 피하기
SELECT * FROM users;  -- 나쁜 예

-- ✅ 필요한 컬럼만 선택
SELECT id, username, email FROM users;  -- 좋은 예

-- ❌ 함수를 WHERE 절에 사용하면 인덱스 무효화
SELECT * FROM orders WHERE YEAR(created_at) = 2024;  -- 나쁜 예

-- ✅ 범위 조건으로 변경
SELECT * FROM orders
WHERE created_at >= '2024-01-01'
  AND created_at < '2025-01-01';  -- 좋은 예

-- ❌ OR 조건은 인덱스 효율 저하
SELECT * FROM products
WHERE category_id = 1 OR category_id = 2;  -- 나쁜 예

-- ✅ IN 사용
SELECT * FROM products
WHERE category_id IN (1, 2);  -- 좋은 예

-- ❌ LIKE '%keyword'는 인덱스 사용 불가
SELECT * FROM products WHERE product_name LIKE '%phone';  -- 나쁜 예

-- ✅ LIKE 'keyword%'는 인덱스 사용 가능
SELECT * FROM products WHERE product_name LIKE 'phone%';  -- 좋은 예

-- ❌ 서브쿼리 중복 실행
SELECT
    username,
    (SELECT COUNT(*) FROM orders WHERE user_id = users.id) as order_count
FROM users;  -- 나쁜 예

-- ✅ JOIN 사용
SELECT
    u.username,
    COUNT(o.order_id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;  -- 좋은 예
```

### 테이블 파티셔닝
```sql
-- RANGE 파티셔닝 (날짜별)
CREATE TABLE orders_partitioned (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    total_amount DECIMAL(10,2),
    created_at DATETIME
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- LIST 파티셔닝
CREATE TABLE users_by_country (
    user_id INT,
    username VARCHAR(50),
    country VARCHAR(2)
)
PARTITION BY LIST COLUMNS(country) (
    PARTITION p_asia VALUES IN ('KR', 'JP', 'CN'),
    PARTITION p_america VALUES IN ('US', 'CA', 'MX'),
    PARTITION p_europe VALUES IN ('UK', 'DE', 'FR')
);

-- HASH 파티셔닝
CREATE TABLE logs (
    log_id BIGINT,
    log_message TEXT,
    created_at TIMESTAMP
)
PARTITION BY HASH(log_id)
PARTITIONS 10;
```

### 테이블 최적화
```sql
-- 테이블 분석
ANALYZE TABLE users;
ANALYZE TABLE orders, products, categories;

-- 테이블 최적화
OPTIMIZE TABLE users;

-- 테이블 검사
CHECK TABLE users;

-- 테이블 복구
REPAIR TABLE users;

-- 테이블 통계 정보 조회
SHOW TABLE STATUS LIKE 'users';

-- 인덱스 통계
SHOW INDEX FROM users;
```

### 캐싱 및 설정
```sql
-- 쿼리 캐시 확인 (MariaDB 10.1 이하)
SHOW VARIABLES LIKE 'query_cache%';

-- 버퍼 풀 크기 확인
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- 연결 수 확인
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';

-- 슬로우 쿼리 로그 활성화
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- 2초 이상 걸리는 쿼리 로깅
SHOW VARIABLES LIKE 'slow_query%';

-- 실행 중인 프로세스 확인
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- 프로세스 종료
KILL [process_id];
```

### 배치 작업 최적화
```sql
-- ❌ 루프에서 개별 INSERT
-- 매우 느림
INSERT INTO logs VALUES (1, 'message1');
INSERT INTO logs VALUES (2, 'message2');
-- ...

-- ✅ 배치 INSERT
-- 빠름
INSERT INTO logs VALUES
(1, 'message1'),
(2, 'message2'),
(3, 'message3'),
-- ... (1000개씩 배치)
(1000, 'message1000');

-- 대용량 UPDATE/DELETE는 분할 실행
-- ❌ 한 번에 전체
UPDATE huge_table SET status = 'processed';  -- 테이블 잠금 오래 유지

-- ✅ LIMIT으로 나눠서
UPDATE huge_table SET status = 'processed' WHERE status = 'pending' LIMIT 1000;
-- 여러 번 반복

-- 트랜잭션 크기 조절
START TRANSACTION;
-- 적절한 크기의 작업만
UPDATE products SET price = price * 1.1 WHERE category_id = 1 LIMIT 100;
COMMIT;
```

### 모니터링
```sql
-- 데이터베이스 크기
SELECT
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables
GROUP BY table_schema;

-- 테이블별 크기
SELECT
    table_name AS 'Table',
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)',
    table_rows AS 'Rows'
FROM information_schema.tables
WHERE table_schema = 'bootex'
ORDER BY (data_length + index_length) DESC;

-- 인덱스 사용률
SELECT
    object_schema,
    object_name,
    index_name,
    count_star,
    count_read,
    count_insert,
    count_update,
    count_delete
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'bootex'
ORDER BY count_star DESC;
```

---

## 부록: 실전 예제 및 모범 사례

### 데이터베이스 설계 원칙
```sql
-- 1. 적절한 데이터 타입 선택
CREATE TABLE users (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,  -- UNSIGNED로 범위 2배
    username VARCHAR(50) NOT NULL,  -- 고정 크기가 아니면 VARCHAR
    email VARCHAR(100) NOT NULL UNIQUE,
    is_active BOOLEAN DEFAULT TRUE,  -- 불린은 BOOLEAN
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  -- 자동 타임스탬프
    INDEX idx_username (username),
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;  -- InnoDB, UTF-8

-- 2. 정규화 vs 비정규화
-- 정규화: 중복 제거, 데이터 일관성
-- 비정규화: 조회 성능 향상

-- 정규화된 설계
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    created_at TIMESTAMP
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10,2)
);

-- 비정규화 예시: 자주 조회하는 총액을 미리 계산
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);

-- 3. 적절한 제약조건
CREATE TABLE products (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock_quantity INT NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES categories(category_id)
        ON DELETE SET NULL
        ON UPDATE CASCADE
);
```

### 보안 모범 사례
```sql
-- 1. 최소 권한 원칙
CREATE USER 'readonly_user'@'localhost' IDENTIFIED BY 'password';
GRANT SELECT ON bootex.* TO 'readonly_user'@'localhost';

CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'password';
GRANT SELECT, INSERT, UPDATE, DELETE ON bootex.* TO 'app_user'@'localhost';

-- 2. SQL Injection 방지
-- ❌ 위험: 동적 SQL
SET @username = 'admin';
SET @sql = CONCAT('SELECT * FROM users WHERE username = "', @username, '"');
PREPARE stmt FROM @sql;
EXECUTE stmt;

-- ✅ 안전: Prepared Statement
PREPARE stmt FROM 'SELECT * FROM users WHERE username = ?';
SET @username = 'admin';
EXECUTE stmt USING @username;

-- 3. 민감한 데이터 암호화
-- AES 암호화
INSERT INTO sensitive_data (user_id, data)
VALUES (1, AES_ENCRYPT('sensitive info', 'encryption_key'));

SELECT AES_DECRYPT(data, 'encryption_key') as decrypted_data
FROM sensitive_data WHERE user_id = 1;

-- 비밀번호는 애플리케이션 레벨에서 해시 (bcrypt, SHA-256 등)
```

### 백업 및 복구
```bash
# 전체 데이터베이스 백업
mysqldump -u root -p bootex > bootex_backup.sql

# 특정 테이블만 백업
mysqldump -u root -p bootex users orders > tables_backup.sql

# 구조만 백업
mysqldump -u root -p --no-data bootex > bootex_structure.sql

# 데이터만 백업
mysqldump -u root -p --no-create-info bootex > bootex_data.sql

# 복구
mysql -u root -p bootex < bootex_backup.sql

# 압축 백업
mysqldump -u root -p bootex | gzip > bootex_backup.sql.gz

# 압축 복구
gunzip < bootex_backup.sql.gz | mysql -u root -p bootex
```

### 일반적인 실수와 해결책
```sql
-- 1. N+1 쿼리 문제
-- ❌ 나쁜 예: 각 사용자마다 쿼리 실행
SELECT * FROM users;
-- 애플리케이션 루프에서
-- SELECT * FROM orders WHERE user_id = ?

-- ✅ 좋은 예: JOIN으로 한 번에
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 2. 인덱스 없는 외래키
-- ❌ 외래키에 인덱스 없음
ALTER TABLE orders ADD FOREIGN KEY (user_id) REFERENCES users(id);

-- ✅ 외래키에 인덱스 추가
CREATE INDEX idx_user_id ON orders(user_id);
ALTER TABLE orders ADD FOREIGN KEY (user_id) REFERENCES users(id);

-- 3. 대용량 데이터 한번에 처리
-- ❌ 전체 삭제
DELETE FROM logs WHERE created_at < '2023-01-01';

-- ✅ 배치로 나눠서 삭제
DELETE FROM logs WHERE created_at < '2023-01-01' LIMIT 1000;
-- 여러 번 반복
```

---

## 16. MariaDB 11.8 LTS (2025) 주요 변경사항 정리

### 💡 새로운 기능

#### 1. Vector 데이터 타입 및 AI/ML 지원
```sql
-- 벡터 임베딩 저장 및 검색
CREATE TABLE embeddings (
    doc_id BIGINT PRIMARY KEY,
    embedding VECTOR(1536) NOT NULL,
    VECTOR INDEX (embedding) M=8 DISTANCE=cosine
);

-- 유사도 검색 함수
SELECT VEC_DISTANCE_COSINE(embedding, VEC_FromText('[...]')) FROM embeddings;
SELECT VEC_DISTANCE_EUCLIDEAN(embedding, VEC_FromText('[...]')) FROM embeddings;
```

#### 2. 저장 프로시저 기본 매개변수
```sql
-- 기본값이 있는 매개변수 지원
CREATE PROCEDURE GetUsers(
    IN status VARCHAR(20) DEFAULT 'active',
    IN limit_count INT DEFAULT 10
)
BEGIN
    SELECT * FROM users WHERE status = status LIMIT limit_count;
END;

-- 호출 시 기본값 사용 가능
CALL GetUsers();                    -- 모든 기본값 사용
CALL GetUsers('inactive');          -- 일부만 지정
CALL GetUsers(DEFAULT, 20);         -- DEFAULT 키워드 사용
```

#### 3. ROW 데이터 타입 함수 반환
```sql
-- 함수가 ROW 타입을 반환할 수 있음
CREATE FUNCTION GetUserInfo(user_id INT)
RETURNS ROW(username VARCHAR(50), email VARCHAR(100), total_orders INT)
READS SQL DATA
BEGIN
    DECLARE result ROW(username VARCHAR(50), email VARCHAR(100), total_orders INT);
    -- 로직
    RETURN result;
END;

-- 사용
SELECT GetUserInfo(1).username;
```

#### 4. utf8mb4_0900_ai_ci 콜레이션 지원
```sql
-- MySQL 8.0 호환 콜레이션 (utf8mb4_uca1400_ai_ci의 별칭)
CREATE DATABASE mydb
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;
```

### ⚡ 성능 개선

#### 1. 대용량 트랜잭션 커밋 개선
```sql
-- 이전: binlog 활성화 시 대용량 트랜잭션 커밋 중 모든 트랜잭션 freeze
-- MariaDB 11.8+: 커밋 중에도 다른 트랜잭션 정상 처리
START TRANSACTION;
INSERT INTO large_table SELECT * FROM source WHERE ...;  -- 수백만 건
COMMIT;  -- 다른 트랜잭션에 영향 최소화
```

#### 2. MySQL 8.0 Binlog 이벤트 지원
- PARTIAL_UPDATE_ROWS_EVENT
- TRANSACTION_PAYLOAD_EVENT
- HEARTBEAT_LOG_EVENT_V2

### ⚠️ 변경사항 및 Deprecation

#### 1. mysql 명령어 Deprecated
```bash
# 기존 (deprecated)
mysql -u root -p

# 권장 (MariaDB 11+)
mariadb -u root -p
```

### 🎯 호환성

- **LTS 지원**: 5년간 지원 (2025-2030)
- **MySQL 호환성**: MySQL 8.0 binlog 이벤트 지원으로 복제 호환성 향상
- **플랫폼**: SIMD 하드웨어 최적화 (Intel AVX2/AVX512, ARM, IBM Power10)

### 📚 권장 버전
- **프로덕션**: MariaDB 11.8 LTS (2025 LTS)
- **안정성**: MariaDB 10.11 LTS (2023 LTS, 2028년까지 지원)
- **레거시**: MariaDB 10.6 (2026년까지 지원)

### 🔄 업그레이드 고려사항
```sql
-- 업그레이드 전 호환성 확인
CHECK TABLE table_name FOR UPGRADE;

-- 데이터베이스 업그레이드
mysql_upgrade -u root -p

-- 또는 (MariaDB 11+)
mariadb-upgrade -u root -p
```

---

## 마치며

이 문서는 MariaDB의 기초부터 고급 기능까지 다루는 포괄적인 학습 자료입니다.

### 학습 순서 추천
1. **기초 단계**: 섹션 1-5 (기본 SQL 명령어, 데이터 타입, 조건문)
2. **중급 단계**: 섹션 6-9 (JOIN, 집계 함수, 인덱스, 사용자 관리)
3. **고급 단계**: 섹션 10-15 (트랜잭션, 뷰, 저장 프로시저, 트리거, 최적화)

### 추가 학습 자료
- MariaDB 공식 문서: https://mariadb.com/kb/en/
- MariaDB 한국 커뮤니티
- SQL 온라인 실습: SQL Fiddle, DB Fiddle

### 실습 팁
- 항상 개발/테스트 환경에서 먼저 실습하세요
- 백업을 생활화하세요
- EXPLAIN으로 쿼리 성능을 확인하는 습관을 들이세요
- 인덱스는 적절히 사용하되, 과도하게 만들지 마세요
- 보안을 항상 염두에 두세요

**Happy Learning! 즐거운 MariaDB 학습 되세요!** 🚀
