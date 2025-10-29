# Redis 학습 자료: 구조와 빠른 성능의 비밀

## 1. Redis란 무엇인가?

Redis(Remote Dictionary Server)는 **인메모리(In-Memory) 키-값(Key-Value) 데이터베이스**로, 캐싱, 실시간 분석, 메시지 브로커 등 다양한 용도로 사용됩니다. 전통적인 디스크 기반 데이터베이스와 달리 데이터를 **RAM(메모리)**에 저장해 초고속 읽기/쓰기 성능을 제공합니다. NoSQL 데이터베이스로, 단순한 키-값 쌍뿐만 아니라 다양한 **데이터 구조**를 지원해 유연한 데이터 모델링이 가능합니다.[](https://www.geeksforgeeks.org/system-design/introduction-to-redis-server/)[](https://dev.to/rijultp/what-is-redis-a-beginners-guide-to-blazing-fast-databases-1c36)[](https://en.wikipedia.org/wiki/Redis)

---

## 2. Redis의 구조: 어떻게 동작할까?

Redis는 **단일 스레드(single-threaded)**와 **논블로킹 이벤트 루프(non-blocking event loop)**를 사용해 동작합니다. 이 구조가 Redis의 빠른 성능의 핵심입니다. 아래는 동작 과정을 간단히 설명한 내용입니다:

1. **클라이언트 요청**: 클라이언트가 Redis 서버에 명령(예: `SET`, `GET`)을 보냅니다.
2. **이벤트 루프 처리**: Redis의 단일 스레드가 **이벤트 루프**를 통해 요청을 감지하고 순차적으로 처리합니다. 대부분의 명령은 메모리에서 처리되므로 매우 빠릅니다.
3. **응답 반환**: 처리된 결과는 즉시 클라이언트 소켓에 기록되며, 논블로킹 소켓을 사용해 다른 요청을 바로 처리할 수 있습니다.
4. **다음 이벤트로 이동**: 이벤트 루프는 다음 클라이언트 요청이나 내부 이벤트(예: 타이머)를 처리합니다.[](https://redis.io/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/)

### **이벤트 루프란?**
이벤트 루프는 Redis가 여러 클라이언트 요청을 효율적으로 처리하도록 돕는 메커니즘입니다. 마치 레스토랑의 웨이터가 여러 테이블의 주문을 순차적으로 처리하는 것처럼, Redis는 단일 스레드로 모든 요청을 빠르게 처리합니다. 이 과정에서 **논블로킹 I/O**를 사용해 요청이 대기하지 않고 즉시 처리됩니다.

### **Lua 스크립트 주의점**
Redis는 Lua 스크립트를 지원하지만, 복잡하거나 오래 걸리는 스크립트는 단일 이벤트 루프를 막아 다른 요청의 지연(latency)을 유발할 수 있습니다. 따라서 Lua 스크립트는 **최소화**하고, 간단한 연산에만 사용하는 것이 좋습니다.[](https://redis.io/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/)

---

## 3. 왜 Redis가 빠른가?

Redis의 빠른 성능은 다음과 같은 이유에서 비롯됩니다:

1. **인메모리 저장소**: 데이터를 디스크가 아닌 **RAM**에 저장해 읽기/쓰기 속도가 매우 빠릅니다. 디스크 I/O가 없으므로 전통적인 데이터베이스보다 10~100배 빠릅니다.[](https://www.geeksforgeeks.org/system-design/introduction-to-redis-server/)[](https://dev.to/rijultp/what-is-redis-a-beginners-guide-to-blazing-fast-databases-1c36)
2. **단일 스레드**: 복잡한 멀티스레딩 대신 단일 스레드를 사용해 **Race Condition** 걱정 없이 순차적으로 명령을 처리합니다. 이는 락(lock) 관리의 오버헤드를 제거해 성능을 높입니다.[](https://redis.io/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/)
3. **효율적인 데이터 구조**: Redis는 **최적화된 데이터 구조**(문자열, 리스트, 해시 등)를 사용하며, 이를 메모리에서 직접 처리해 속도가 빠릅니다.[](https://thelinuxcode.com/redis-database-basics-how-the-redis-cli-works-and-key-concepts/)
4. **간단한 프로토콜(RESP)**: Redis는 **RESP(Redis Serialization Protocol)**라는 경량 프로토콜을 사용해 네트워크 통신의 오버헤드를 최소화합니다.[](https://www.geeksforgeeks.org/system-design/introduction-to-redis-server/)
5. **논블로킹 I/O**: 이벤트 루프와 논블로킹 소켓을 통해 다수의 클라이언트 요청을 동시에 처리할 수 있습니다.[](https://redis.io/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/)

**비유**: Redis는 마치 초고속 계산원이 한 명 있는 편의점과 같습니다. 모든 주문(요청)을 순차적으로 처리하지만, 계산원이 워낙 빠르고 줄이 짧아 기다릴 필요가 거의 없습니다.

---

## 4. Redis의 주요 데이터 구조

Redis는 단순한 키-값 저장소를 넘어 다양한 데이터 구조를 지원해 복잡한 문제를 쉽게 해결할 수 있습니다. 주요 데이터 구조는 다음과 같습니다:[](https://thelinuxcode.com/redis-database-basics-how-the-redis-cli-works-and-key-concepts/)[](https://dev.to/rijultp/what-is-redis-a-beginners-guide-to-blazing-fast-databases-1c36)

1. **문자열(Strings)**:
   - 가장 기본적인 데이터 구조로, 키에 하나의 문자열(또는 바이너리 데이터)을 저장.
   - 예: `SET name "Alice"` → `GET name` (결과: "Alice")
   - 용도: 캐싱, 카운터(예: `INCR visits`로 방문자 수 증가).

2. **리스트(Lists)**:
   - 순서가 있는 문자열 컬렉션(Linked List).
   - 예: `LPUSH tasks "Task1"` → `LRANGE tasks 0 -1` (결과: ["Task1"])
   - 용도: 작업 큐, 최근 활동 피드.

3. **세트(Sets)**:
   - 순서 없는 고유 문자열 컬렉션.
   - 예: `SADD tags "redis"` → `SMEMBERS tags` (결과: ["redis"])
   - 용도: 태그 관리, 중복 제거, 집합 연산(합집합, 교집합).

4. **해시(Hashes)**:
   - 키-값 쌍을 저장하는 구조(딕셔너리와 유사).
   - 예: `HSET user:1 name "Alice" age 25` → `HGET user:1 name` (결과: "Alice")
   - 용도: 객체 표현(예: 사용자 정보 저장).

5. **정렬된 세트(Sorted Sets)**:
   - 고유 문자열에 점수(score)를 부여해 정렬된 컬렉션.
   - 예: `ZADD leaderboard 100 "Alice"` → `ZRANGE leaderboard 0 1` (결과: ["Alice"])
   - 용도: 리더보드, 순위 계산.

**비유**: Redis의 데이터 구조는 주방의 다양한 도구와 같습니다. 문자열은 단일 접시, 리스트는 재료 순서 리스트, 세트는 중복 없는 재료 바구니, 해시는 레시피 북, 정렬된 세트는 점수 순으로 정렬된 대회 참가자 명단입니다.

---

## 5. Redis의 구조적 장점

1. **Race Condition 없음**: 단일 스레드 기반이므로 쓰기/읽기 명령이 순차적으로 처리되어 동시성 문제가 발생하지 않습니다.[](https://redis.io/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/)
2. **높은 성능과 예측 가능성**: 복잡한 락 메커니즘이 없어 처리 속도가 빠르고, 성능이 일정하게 유지됩니다.[](https://redis.io/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/)
3. **유연한 데이터 모델링**: 다양한 데이터 구조를 지원해 이커머스(재고 관리), 실시간 채팅, 리더보드 등 다양한 시나리오에 적합합니다.[](https://thelinuxcode.com/redis-database-basics-how-the-redis-cli-works-and-key-concepts/)

---

## 6. Sentinel vs Cluster: 고가용성(HA) 구조

Redis는 고가용성을 보장하기 위해 두 가지 주요 구조를 제공합니다:

1. **Sentinel**:
   - **구조**: 단일 마스터와 여러 슬레이브로 구성. Sentinel 노드가 마스터/슬레이브의 장애를 감지하고, 장애 발생 시 마스터를 자동으로 교체(failover).
   - **용도**: 소규모 시스템에서 고가용성과 자동 복구를 제공.
   - **예**: 이커머스에서 주문 데이터를 저장하는 Redis 서버에 Sentinel을 적용해 장애 복구.

2. **Cluster**:
   - **구조**: 멀티 마스터(샤딩) 구조로, 데이터를 여러 노드에 분산 저장. 각 노드가 독립적으로 동작하며, 장애 시 다른 노드가 처리.
   - **용도**: 대규모 트래픽과 데이터 처리가 필요한 환경(예: 대형 이커머스 플랫폼).
   - **예**: 상품 데이터를 샤드별로 분산 저장해 처리 속도 향상.[](https://redis.io/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/)

**비유**: Sentinel은 한 명의 주방장(마스터)과 보조 요리사(슬레이브)를 감시하는 매니저가 있는 식당이고, Cluster는 여러 주방장이 각자의 주방에서 동시에 요리를 만드는 대형 레스토랑 체인입니다.

---

## 7. 주요 키워드 정리

- **인메모리(In-Memory)**: 데이터를 RAM에 저장해 빠른 읽기/쓰기 제공.
- **키-값(Key-Value) 저장소**: 데이터를 키와 값 쌍으로 저장하는 간단한 구조.
- **이벤트 루프(Event Loop)**: 단일 스레드가 요청을 순차적으로 처리하는 메커니즘.
- **논블로킹 I/O**: 요청 처리 중 다른 요청을 대기시키지 않고 즉시 처리.
- **Race Condition**: 여러 스레드가 동시에 데이터에 접근해 발생하는 충돌. Redis는 단일 스레드로 이를 방지.
- **Sentinel**: 마스터-슬레이브 구조에서 장애 감지 및 복구를 담당.
- **Cluster**: 데이터를 분산 저장해 대규모 트래픽 처리.
- **데이터 구조(Data Structures)**: 문자열, 리스트, 세트, 해시, 정렬된 세트 등 Redis가 지원하는 다양한 데이터 형식.
- **RESP**: Redis의 경량 통신 프로토콜로, 네트워크 오버헤드 최소화.

---

## 8. Redis 사용 예시 (Python)

```python
import redis

# Redis 연결
r = redis.Redis(host='localhost', port=6379, db=0)

# 문자열: 캐싱 예시
r.set('name', 'Alice')
print(r.get('name').decode('utf-8'))  # 출력: Alice

# 리스트: 작업 큐 예시
r.lpush('tasks', 'Task1')
print(r.lrange('tasks', 0, -1))  # 출력: [b'Task1']

# 해시: 사용자 정보 저장
r.hset('user:1', mapping={'name': 'Alice', 'age': '25'})
print(r.hgetall('user:1'))  # 출력: {b'name': b'Alice', b'age': b'25'}

# 정렬된 세트: 리더보드
r.zadd('leaderboard', {'Alice': 100})
print(r.zrange('leaderboard', 0, -1))  # 출력: [b'Alice']
```

---

## 9. 결론

Redis는 **인메모리 저장소**, **단일 스레드 이벤트 루프**, **다양한 데이터 구조**를 통해 초고속 성능과 유연성을 제공합니다. 단일 스레드 구조로 Race Condition을 방지하고, Sentinel과 Cluster로 고가용성을 지원하며, 캐싱, 실시간 분석, 메시지 큐 등 다양한 시나리오에 적합합니다. 이커머스에서 재고 관리, 세션 저장, 리더보드 구현 등에 특히 유용합니다.[](https://www.geeksforgeeks.org/system-design/introduction-to-redis-server/)[](https://thelinuxcode.com/redis-database-basics-how-the-redis-cli-works-and-key-concepts/)

**다음 단계**:
- Redis 공식 문서(https://redis.io/docs/)를 통해 명령어와 고급 기능 학습.
- 실제 프로젝트(예: 캐싱, 리더보드)로 실습.
- Redis Cluster와 Sentinel을 로컬에서 설정해 고가용성 테스트.