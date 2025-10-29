## JPQL이란?

JPQL은 Java Persistence API(JPA)에서 사용되는 객체 지향 쿼리 언어입니다. SQL과 유사하지만, 데이터베이스 테이블이 아닌 **엔티티 객체**와 그들의 속성을 대상으로 쿼리를 작성합니다. JPQL은 JPA 구현체(예: Hibernate, EclipseLink)에서 데이터베이스에 접근할 때 사용되며, 데이터베이스 독립적인 쿼리를 작성할 수 있도록 설계되었습니다.

### 주요 특징
- **객체 지향**: 테이블과 컬럼이 아닌 엔티티 클래스와 필드를 대상으로 쿼리 작성.
- **데이터베이스 독립성**: JPQL은 특정 DBMS(MySQL, PostgreSQL 등)에 종속되지 않음. JPA 구현체가 쿼리를 DBMS에 맞게 변환.
- **SQL과 유사**: SQL 문법과 비슷하지만, 엔티티와 속성 중심으로 동작.
- **동적/정적 쿼리 지원**: 정적 쿼리(`@NamedQuery`)와 동적 쿼리(`EntityManager.createQuery`) 모두 가능.

## JPQL이 사용되는 상황

JPQL은 주로 **JPA를 사용하는 Java 애플리케이션**에서 데이터베이스 작업을 수행할 때 사용됩니다. 주요 사용 사례:
- **복잡한 데이터 조회**: 단순한 CRUD보다 복잡한 조건으로 데이터 조회.
- **엔티티 간 관계 처리**: `@OneToMany`, `@ManyToOne` 등 엔티티 관계 기반 데이터 조회.
- **동적 쿼리 생성**: 사용자 입력에 따라 쿼리 조건이 동적으로 변하는 경우.
- **배치 처리**: 특정 조건에 맞는 데이터 일괄 조회, 수정, 삭제.
- **집계 쿼리**: `COUNT`, `SUM`, `AVG` 등 집계 함수를 사용한 데이터 분석.

## MySQL SQL과의 주요 차이점

| **항목**            | **JPQL**                                                                 | **MySQL SQL**                                                       |
|---------------------|--------------------------------------------------------------------------|--------------------------------------------------------------------|
| **대상**           | 엔티티 객체와 속성 (예: `User.name`)                                      | 테이블과 컬럼 (예: `users.name`)                                   |
| **문법**           | 객체 지향적, 클래스명과 필드명 사용 (예: `SELECT u FROM User u`)         | 테이블 중심, 테이블명과 컬럼명 사용 (예: `SELECT * FROM users`)     |
| **DB 독립성**      | DBMS에 독립적, JPA가 쿼리를 DBMS별 SQL로 변환                            | MySQL 전용 문법, 다른 DBMS에서 동작하지 않을 수 있음                |
| **조인**           | 엔티티 관계 기반 조인 (예: `u.address`)                                 | 명시적 조인 필요 (예: `JOIN addresses ON users.id = addresses.user_id`) |
| **함수**           | 제한된 내장 함수 (`UPPER`, `CONCAT` 등), JPA 구현체에 따라 확장 가능     | MySQL 고유 함수 사용 가능 (예: `IFNULL`, `JSON_EXTRACT`)            |
| **서브쿼리**       | 제한적 지원, 복잡한 서브쿼리는 구현체에 따라 동작 여부 다름              | 더 자유로운 서브쿼리 작성 가능                                      |
| **네이티브 쿼리**  | JPQL은 네이티브 SQL 실행 가능 (`createNativeQuery`), 하지만 권장되지 않음 | 기본적으로 네이티브 SQL로 동작                                      |

### 예시: JPQL vs MySQL SQL
- **MySQL SQL**: `SELECT * FROM users WHERE age > 30`
- **JPQL**: `SELECT u FROM User u WHERE u.age > 30`

JPQL은 `User` 엔티티를 대상으로 하며, 테이블명(`users`)이 아닌 엔티티 클래스명(`User`)을 사용합니다.

## 실무에서 중요한 JPQL 개념 (빈도 순)

다음은 실무에서 자주 사용되며 학습 우선순위가 높은 JPQL 개념들입니다. SQL을 알고 있는 학습자를 위해 MySQL과의 차이를 강조하며 정리했습니다.

### 1. **기본 SELECT 쿼리**
- **설명**: 엔티티를 조회하는 가장 기본적인 쿼리. SQL의 `SELECT *`와 비슷하지만, 엔티티 객체 반환.
- **문법**:
  ```jpql
  SELECT e FROM EntityName e WHERE e.property = :param
  ```
- **MySQL과의 차이**: JPQL은 컬럼명 대신 엔티티의 속성명 사용.
- **예시**:
  ```jpql
  SELECT u FROM User u WHERE u.name = :name
  ```
  MySQL: `SELECT * FROM users WHERE name = ?`

### 2. **조인 (JOIN)**
- **설명**: 엔티티 간 관계를 기반으로 데이터 조회. `@OneToMany`, `@ManyToOne` 등의 관계 활용.
- **종류**: `INNER JOIN`, `LEFT JOIN`, `FETCH JOIN` 등.
- **문법**:
  ```jpql
  SELECT u FROM User u JOIN u.address a WHERE a.city = :city
  ```
  - `FETCH JOIN`: 관련 엔티티를 즉시 로딩 (성능 최적화).
    ```jpql
    SELECT u FROM User u JOIN FETCH u.address
    ```
- **MySQL과의 차이**: MySQL은 테이블 간 외래키를 명시적으로 지정해야 하지만, JPQL은 엔티티 관계를 자동 처리.
- **예시**:
  ```jpql
  SELECT u FROM User u LEFT JOIN u.orders o WHERE o.status = 'PENDING'
  ```
  MySQL: `SELECT * FROM users u LEFT JOIN orders o ON u.id = o.user_id WHERE o.status = 'PENDING'`

### 3. **파라미터 바인딩**
- **설명**: 동적 쿼리에서 사용자 입력을 안전하게 처리. SQL 인젝션 방지.
- **문법**:
  - 명명 파라미터: `:paramName`
  - 위치 파라미터: `?1`, `?2`
  ```jpql
  SELECT u FROM User u WHERE u.age > :age AND u.name = ?1
  ```
- **MySQL과의 차이**: MySQL은 `?`를 주로 사용하며, 명명 파라미터 미지원.
- **예시**:
  ```jpql
  SELECT u FROM User u WHERE u.email = :email
  ```

### 4. **집계 함수**
- **설명**: `COUNT`, `SUM`, `AVG`, `MAX`, `MIN` 등으로 데이터 집계.
- **문법**:
  ```jpql
  SELECT COUNT(u), AVG(u.age) FROM User u GROUP BY u.department
  ```
- **MySQL과의 차이**: 문법은 유사하지만, JPQL은 엔티티 속성을 대상.
- **예시**:
  ```jpql
  SELECT d.name, COUNT(u) FROM User u JOIN u.department d GROUP BY d.name
  ```

### 5. **서브쿼리**
- **설명**: 복잡한 조건을 위해 서브쿼리 사용. JPQL은 서브쿼리 지원이 제한적 (예: `SELECT` 절 서브쿼리 미지원).
- **문법**:
  ```jpql
  SELECT u FROM User u WHERE u.age > (SELECT AVG(u2.age) FROM User u2)
  ```
- **MySQL과의 차이**: MySQL은 서브쿼리 작성 자유도가 높지만, JPQL은 `WHERE`, `HAVING` 절에서만 서브쿼리 지원.
- **예시**:
  ```jpql
  SELECT u FROM User u WHERE u.salary > (SELECT AVG(u2.salary) FROM User u2 WHERE u2.department = u.department)
  ```

### 6. **업데이트/삭제 쿼리**
- **설명**: 엔티티 데이터를 일괄적으로 수정하거나 삭제. 트랜잭션 내에서 실행해야 함.
- **문법**:
  ```jpql
  UPDATE User u SET u.status = :status WHERE u.id = :id
  DELETE FROM User u WHERE u.status = :status
  ```
- **MySQL과의 차이**: 문법은 비슷하지만, JPQL은 엔티티 중심이고 영속성 컨텍스트를 고려해야 함.
- **예시**:
  ```jpql
  UPDATE User u SET u.salary = u.salary + 1000 WHERE u.department = :dept
  ```

### 7. **FETCH JOIN과 성능 최적화**
- **설명**: 연관 엔티티를 한 번에 로딩하여 N+1 문제 방지.
- **문법**:
  ```jpql
  SELECT u FROM User u JOIN FETCH u.address JOIN FETCH u.orders
  ```
- **MySQL과의 차이**: MySQL은 조인만으로 데이터를 가져오지만, JPQL은 엔티티 그래프를 로딩하는 방식이 다름.

## JPQL의 인덱스 및 쿼리 최적화

### **장점**
- **데이터베이스 독립성**: JPQL은 JPA 구현체가 쿼리를 최적화된 SQL로 변환하므로, 동일한 쿼리로 여러 DBMS에서 동작 가능.
- **엔티티 중심 최적화**: 엔티티 관계를 기반으로 쿼리를 작성하므로, 복잡한 조인 조건을 간소화.
- **캐싱 지원**: JPA의 1차/2차 캐시를 활용해 반복적인 쿼리 성능 향상.
- **FETCH JOIN**: N+1 문제를 해결하여 연관 엔티티를 효율적으로 로딩.

### **단점**
- **제한된 최적화 제어**: JPQL은 DBMS별 최적화(예: MySQL의 `FORCE INDEX`)를 직접 제어하기 어려움.
- **복잡한 쿼리 제한**: 복잡한 서브쿼리나 고급 SQL 기능(예: MySQL의 `PARTITION`)은 지원하지 않음.
- **N+1 문제 위험**: 잘못된 쿼리 설계 시, 다중 쿼리가 실행되어 성능 저하 발생.
- **인덱스 직접 관리 불가**: JPQL은 테이블 인덱스를 직접 지정할 수 없으며, JPA 구현체와 DBMS에 의존.

### **개선법**
1. **FETCH JOIN 활용**:
   - N+1 문제를 방지하기 위해 연관 엔티티를 즉시 로딩.
   - 예: `SELECT u FROM User u JOIN FETCH u.address`
   - **주의**: 과도한 `FETCH JOIN`은 쿼리 결과 크기를 증가시켜 성능 저하 가능.
2. **엔티티 그래프 사용**:
   - `@NamedEntityGraph`를 사용해 필요한 연관 엔티티만 선택적으로 로딩.
   - 예:
     ```java
     @NamedEntityGraph(name = "User.withAddress", attributeNodes = @NamedAttributeNode("address"))
     @Entity
     public class User { ... }
     ```
3. **쿼리 힌트 사용**:
   - Hibernate 등 JPA 구현체에서 제공하는 쿼리 힌트로 캐시 설정.
   - 예: `query.setHint("org.hibernate.cacheable", true)`
4. **배치 크기 설정**:
   - `@BatchSize` 어노테이션으로 지연 로딩 시 한 번에 로딩할 엔티티 수 지정.
   - 예: `@BatchSize(size = 100)`
5. **인덱스 간접 관리**:
   - JPQL은 인덱스를 직접 지정할 수 없지만, 엔티티의 자주 조회되는 필드(예: `name`, `email`)에 `@Index`를 추가해 테이블 생성 시 인덱스 적용.
   - 예:
     ```java
     @Table(indexes = {@Index(columnList = "email")})
     @Entity
     public class User { ... }
     ```
6. **쿼리 최적화 분석**:
   - Hibernate의 `show_sql`과 `format_sql` 설정으로 생성된 SQL 확인.
   - MySQL의 `EXPLAIN` 명령으로 실제 쿼리 실행 계획 분석.
   - 예: `spring.jpa.properties.hibernate.show_sql=true`

### **관리 방법**
- **쿼리 로그 모니터링**: 애플리케이션 실행 중 생성되는 SQL을 로깅하여 비효율적인 쿼리 식별.
- **엔티티 설계 최적화**: 불필요한 연관 관계(`@ManyToMany` 등)를 최소화하고, 단방향 관계를 우선 고려.
- **캐시 전략**:
  - JPA 2차 캐시(Ehcache, Infinispan 등)를 활성화해 자주 조회되는 데이터 캐싱.
  - 예: `spring.jpa.properties.hibernate.cache.use_second_level_cache=true`
- **네이티브 쿼리 최소화**: JPQL 대신 네이티브 SQL을 사용할 경우, DBMS 종속성이 증가하므로 주의.
- **트랜잭션 관리**: `UPDATE`, `DELETE` 쿼리는 트랜잭션 내에서 실행하고, 영속성 컨텍스트 동기화 주의.

### **사용 시 주의사항 (하지 말아야 할 것)**
- **과도한 FETCH JOIN**: 여러 엔티티를 한 번에 로딩하면 메모리 사용량 증가.
- **복잡한 동적 쿼리 남발**: 동적 쿼리는 유지보수가 어렵고 성능 예측이 어려움. 가능하면 `@NamedQuery` 사용.
- **인덱스 무시**: 자주 검색되는 필드에 인덱스가 없으면 성능 저하. 엔티티 설계 시 인덱스 고려.
- **불필요한 전체 조회**: `SELECT u FROM User u`처럼 조건 없는 조회는 대량 데이터 조회로 성능 문제 발생.
- **서브쿼리 과다 사용**: JPQL은 서브쿼리 지원이 제한적이므로, 복잡한 로직은 네이티브 SQL이나 애플리케이션 단에서 처리.

## 자주 사용되는 JPQL 문법

### 1. **기본 SELECT**
```jpql
SELECT u FROM User u WHERE u.age BETWEEN :minAge AND :maxAge
```

### 2. **조인**
```jpql
SELECT u, o FROM User u JOIN u.orders o WHERE o.total > :amount
```

### 3. **집계 쿼리**
```jpql
SELECT d, COUNT(u), AVG(u.salary) FROM User u JOIN u.department d GROUP BY d HAVING COUNT(u) > 5
```

### 4. **정렬**
```jpql
SELECT u FROM User u ORDER BY u.age DESC, u.name ASC
```

### 5. **IN 절**
```jpql
SELECT u FROM User u WHERE u.department.id IN (:deptIds)
```

### 6. **LIKE 검색**
```jpql
SELECT u FROM User u WHERE u.name LIKE :pattern
```
예: `u.name LIKE '%John%'`

### 7. **NULL 처리**
```jpql
SELECT u FROM User u WHERE u.email IS NULL
```

### 8. **동적 쿼리 예시 (Java 코드와 함께)**
```java
String jpql = "SELECT u FROM User u WHERE u.status = :status";
Query query = entityManager.createQuery(jpql);
query.setParameter("status", "ACTIVE");
List<User> users = query.getResultList();
```

## 학습 팁
1. **기본 SELECT부터 시작**: 간단한 엔티티 조회 쿼리 작성으로 문법 익히기.
2. **조인 연습**: `@OneToMany`, `@ManyToOne` 관계 기반 조인 쿼리 작성.
3. **성능 최적화 실습**: `FETCH JOIN`, 엔티티 그래프, 캐시 설정 적용.
4. **SQL 로그 분석**: Hibernate의 `show_sql`로 생성된 SQL 확인하고, MySQL `EXPLAIN`으로 실행 계획 분석.
5. **JPA 구현체 문서 참고**: Hibernate 등에서 제공하는 JPQL 확장 기능 확인.

## 참고 자료
- **JPA 공식 문서**: [Java Persistence API](https://docs.oracle.com/javaee/7/api/javax/persistence/package-summary.html)
- **Hibernate 문서**: [Hibernate ORM](https://hibernate.org/orm/documentation/)
- **책 추천**: *Java Persistence with Hibernate* (Christian Bauer 외)