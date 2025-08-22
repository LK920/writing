# MyBatis에서 DB 락 제어 가이드

이 문서는 MyBatis를 사용하는 환경에서 데이터베이스 락(DB Lock)을 효과적으로 제어하는 방법을 다룹니다. DB 락은 데이터 일관성과 동시성 문제를 해결하기 위해 필수적이며, MyBatis에서는 SQL 쿼리와 트랜잭션 설정을 통해 이를 구현합니다. 아래는 개념, 코드 예제, 그리고 사고를 유도하기 위한 질문을 포함한 단계별 가이드입니다.

## 1. DB 락의 필요성

DB 락은 여러 트랜잭션이 동시에 데이터를 수정하려 할 때 **데이터 무결성**을 보장합니다. 예를 들어, 전자상거래 시스템에서 주문 상태를 업데이트할 때, 다른 트랜잭션이 동일한 주문을 동시에 수정하지 못하도록 락을 사용할 수 있습니다.

- **질문**: 당신의 프로젝트에서 동시성 문제가 발생할 가능성이 있는 데이터는 무엇인가요? 예를 들어, 재고, 주문, 사용자 정보 등 어떤 시나리오를 다루고 있나요?
- **힌트**: 락은 필요한 경우에만 사용해야 합니다. 불필요한 락은 성능 저하를 초래할 수 있습니다.

## 2. MyBatis에서 락 제어의 핵심 원칙

MyBatis는 자체적으로 락 메커니즘을 제공하지 않으며, **DB 엔진**(예: MySQL, PostgreSQL)의 락 기능과 **JDBC**를 통해 락을 제어합니다. 따라서 락은 주로 **SQL 쿼리**와 **트랜잭션 설정**에서 구현됩니다.

### 주요 락의 종류
- **공유 락(Shared Lock)**: 데이터를 읽을 수 있지만 수정은 불가능. (예: MySQL의 `SELECT ... LOCK IN SHARE MODE`)
- **배타적 락(Exclusive Lock)**: 데이터를 읽거나 수정할 수 없도록 제한. (예: MySQL의 `SELECT ... FOR UPDATE`)

- **질문**: 당신의 시나리오에서 공유 락과 배타적 락 중 어떤 것이 더 적합할까요? 왜 그런 선택을 했나요?

## 3. 코드 예제: MyBatis에서 배타적 락 구현

아래는 MySQL과 Spring+MyBatis 환경에서 주문 데이터를 조회하고 업데이트하는 과정에서 배타적 락을 사용하는 예제입니다.

### 가정
- 테이블: `orders` (컬럼: `order_id`, `status`, `updated_at`)
- 목표: 특정 주문(`order_id`)을 조회하고, 다른 트랜잭션이 해당 주문을 수정하지 못하도록 락을 건 후 상태를 업데이트.

### 3.1. MyBatis 매퍼 XML (OrderMapper.xml)

```xml
<mapper namespace="com.example.OrderMapper">
    <!-- 특정 주문 조회 with 배타적 락 -->
    <select id="selectOrderWithLock" parameterType="int" resultType="com.example.Order">
        SELECT * FROM orders WHERE order_id = #{orderId} FOR UPDATE
    </select>

    <!-- 주문 상태 업데이트 -->
    <update id="updateOrderStatus" parameterType="com.example.Order">
        UPDATE orders 
        SET status = #{status}, updated_at = NOW()
        WHERE order_id = #{orderId}
    </update>
</mapper>
```

- **힌트**: `FOR UPDATE`는 트랜잭션 내에서 실행되어야 락이 유지됩니다. 트랜잭션 범위를 어떻게 설정할지 생각해보세요.

### 3.2. 서비스 계층 (Spring)

Spring의 `@Transactional`을 사용해 트랜잭션과 격리 수준을 설정합니다.

```java
package com.example.service;

import com.example.mapper.OrderMapper;
import com.example.model.Order;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {
    @Autowired
    private OrderMapper orderMapper;

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public void processOrder(int orderId, String newStatus) {
        // 1. 주문을 조회하면서 배타적 락 걸기
        Order order = orderMapper.selectOrderWithLock(orderId);
        if (order == null) {
            throw new IllegalStateException("주문이 존재하지 않습니다.");
        }

        // 2. 비즈니스 로직 처리
        if (order.getStatus().equals("CANCELLED")) {
            throw new IllegalStateException("이미 취소된 주문입니다.");
        }

        // 3. 주문 상태 업데이트
        order.setStatus(newStatus);
        orderMapper.updateOrderStatus(order);
    }
}
```

- **질문**: 위 코드에서 `@Transactional(isolation = Isolation.REPEATABLE_READ)`를 사용한 이유는 무엇일까요? 다른 격리 수준(예: `SERIALIZABLE`)을 사용하면 어떤 차이가 있을까요?
- **힌트**: `REPEATABLE_READ`는 MySQL의 기본 격리 수준으로, `FOR UPDATE`와 함께 사용하면 배타적 �락이 잘 작동합니다. 하지만 `SERIALIZABLE`은 더 엄격한 락을 걸 수 있습니다.

### 3.3. 트랜잭션 없이 MyBatis 단독 사용 시

Spring을 사용하지 않고 MyBatis 단독으로 트랜잭션을 관리하려면 `SqlSession`을 통해 트랜잭션을 수동으로 제어해야 합니다.

```java
public void processOrderWithMyBatis(int orderId, String newStatus) {
    SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.SIMPLE, false);
    try {
        OrderMapper mapper = sqlSession.getMapper(OrderMapper.class);
        Order order = mapper.selectOrderWithLock(orderId);
        if (order == null) {
            throw new IllegalStateException("주문이 존재하지 않습니다.");
        }

        if (order.getStatus().equals("CANCELLED")) {
            throw new IllegalStateException("이미 취소된 주문입니다.");
        }

        order.setStatus(newStatus);
        mapper.updateOrderStatus(order);

        sqlSession.commit();
    } catch (Exception e) {
        sqlSession.rollback();
        throw e;
    } finally {
        sqlSession.close();
    }
}
```

- **질문**: 수동 트랜잭션 관리에서 `commit`과 `rollback`을 명시적으로 호출하는 이유는 무엇일까요? 이 방식의 장단점은 무엇일까요?

## 4. 락 사용 시 주의사항

1. **데드락(Deadlock) 방지**
   - 락을 걸 때 항상 같은 순서로 테이블/로우를 접근하세요.
   - 예: 항상 `order_id`를 오름차순으로 락을 건다면 데드락 가능성을 줄일 수 있습니다.
   - **질문**: 두 트랜잭션이 서로 다른 순서로 `order_id=1`과 `order_id=2`를 락 걸려고 하면 어떤 문제가 생길까요?

2. **성능 최적화**
   - 트랜잭션 범위를 최소화하여 락 유지 시간을 줄이세요.
   - 불필요한 락은 시스템 성능을 저하시킬 수 있습니다.
   - **힌트**: 락이 필요한 최소한의 로우만 대상으로 `WHERE` 조건을 명확히 작성하세요.

3. **DB별 락 차이**
   - MySQL: `FOR UPDATE`, `LOCK IN SHARE MODE`
   - PostgreSQL: `FOR UPDATE`, `FOR SHARE`
   - **질문**: PostgreSQL을 사용한다면 위 예제의 `FOR UPDATE`가 동일하게 작동할까요? DB별로 다른 점은 무엇일까요?

## 5. 응용 과제: 스스로 설계해보기

이제 위 예제를 바탕으로, 당신의 프로젝트에 맞는 락 처리 로직을 설계해보세요. 다음 질문에 답하면서 구체화해보세요:

1. 어떤 테이블과 컬럼에 락을 걸고 싶나요?
2. 락을 걸기 위한 SQL 쿼리를 MyBatis 매퍼에 어떻게 작성할 건가요?
3. 트랜잭션 범위는 어떻게 설정할 건가요? (예: 서비스 메서드 전체? 특정 블록만?)

- **힌트**: 예를 들어, 재고 관리 시스템이라면 `inventory` 테이블의 `stock` 컬럼에 락을 걸고 업데이트하는 로직을 작성해볼 수 있습니다.

## 6. 추가 학습 리소스

- **MyBatis 공식 문서**: 트랜잭션 관리 (https://mybatis.org/mybatis-3/sqlmap-xml.html)
- **MySQL 락 문서**: `SELECT ... FOR UPDATE` (https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- **PostgreSQL 락 문서**: `FOR UPDATE`, `FOR SHARE` (https://www.postgresql.org/docs/current/explicit-locking.html)
- **Spring 트랜잭션**: `@Transactional` 설정 (https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)

## 7. 메타인지 질문

- 지금 이 자료에서 가장 이해하기 어려운 부분은 무엇인가요?
- 이 자료를 바탕으로, 당신의 프로젝트에 어떻게 적용할 수 있을지 구체적인 계획을 세워볼래요?
- 다른 접근법(예: 낙관적 락)을 고려해본다면 어떤 장단점이 있을까요?

---

이 문서를 통해 MyBatis에서 DB 락을 제어하는 방법을 이해하고, 실제 코드로 구현해볼 수 있을 것입니다. 추가로 궁금한 점이나 구체화하고 싶은 부분이 있다면 알려주세요!