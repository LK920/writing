
## 1. 개요

**Look-Aside Cache Pattern**은 애플리케이션이 캐시와 데이터 저장소(DB)를 함께 사용할 때, **캐시 미스(Cache Miss)**가 발생했을 때의 데이터 조회 전략을 정의하는 패턴입니다.  
주로 **읽기 중심의 서비스**에서 성능 향상과 DB 부하 감소를 위해 사용됩니다.

Redisson을 사용할 때도 RMapCache, RLocalCachedMap 등을 쓰다 보면, 캐시에서 키를 찾지 못하는 경우(미스)가 발생하는데, 이때 Look-Aside 패턴을 적용하면 DB 조회와 캐시 업데이트를 안정적으로 처리할 수 있습니다.

---

## 2. 동작 흐름

1. **읽기 요청 발생**
    
    - 애플리케이션이 먼저 **캐시**를 조회.
        
2. **캐시 히트(Cache Hit)**
    
    - 캐시에 데이터가 있으면 그대로 응답.
        
3. **캐시 미스(Cache Miss)**
    
    - DB(혹은 다른 원천 데이터 소스)에서 데이터를 읽어옴.
        
    - 읽어온 데이터를 캐시에 저장(TTL 포함).
        
4. **응답 반환**
    
    - 이후 같은 요청이 오면 캐시에서 바로 응답.
        

---

## 3. 시각적 구조

```
[Client]
   |
   v
[Application]
   |
   |----> 캐시 조회 ----> [Cache]
   |          |
   |          └─(미스)──> [Database] → 캐시에 저장 → 반환
   |
   └─(히트)──> 캐시 값 반환
```

---

## 4. 왜 사용하는가? (사용 이유)

- **DB 부하 감소**  
    캐시를 먼저 조회하므로 동일한 데이터를 반복적으로 DB에서 읽는 부하를 줄임.
    
- **응답 속도 향상**  
    메모리 기반 캐시(Redis, Memcached)는 DB보다 훨씬 빠른 응답 제공.
    
- **유연성**  
    캐시 갱신 시점을 애플리케이션에서 제어 가능.
    
- **데이터 최신성 유지**  
    TTL(만료 시간) 설정으로 오래된 데이터를 자동 갱신.
    

---

## 5. 장점과 단점

### 장점

- 구현이 단순하고 직관적.
    
- 캐시 장애가 발생해도 DB를 통해 서비스는 지속 가능.
    
- 캐시 갱신 주기(TTL)를 자유롭게 조절 가능.
    

### 단점

- 캐시 미스 시 지연이 발생(DB 조회 필요).
    
- TTL이 짧으면 캐시 미스 빈도 증가 → DB 부하 증가.
    
- 쓰기 작업이 많은 경우에는 캐시와 DB의 데이터 불일치 가능성 존재.
    

---

## 6. Redisson에서의 활용 예시

```java
public String getUserProfile(String userId) {
    RMapCache<String, String> cache = redissonClient.getMapCache("userProfiles");
    
    // 1. 캐시 조회
    String profile = cache.get(userId);
    if (profile != null) {
        return profile; // 캐시 히트
    }
    
    // 2. 캐시 미스 → DB 조회
    profile = userRepository.findById(userId)
                            .map(User::toJson)
                            .orElse(null);
    
    // 3. 캐시에 저장 (TTL 30분)
    if (profile != null) {
        cache.put(userId, profile, 30, TimeUnit.MINUTES);
    }
    
    return profile;
}
```

---

## 7. 내부 동작과 고려사항

- **캐시 키 설계**
    
    - 키 충돌 방지를 위해 네임스페이스(prefix) 사용: `"userProfiles:{userId}"`
        
- **TTL(만료 시간)**
    
    - 너무 길면 오래된 데이터 제공, 너무 짧으면 캐시 미스 증가.
        
- **동시성 제어**
    
    - 동일 키 요청이 동시에 들어와 모두 DB를 조회하는 **Cache Stampede** 문제 발생 가능.
        
    - Redisson RLock과 함께 사용해 캐시 미스 시 DB 접근을 직렬화 가능.
        
- **데이터 불일치**
    
    - 쓰기 시 캐시와 DB를 함께 갱신(Write-Through 패턴과 비교).
        
- **예외 처리**
    
    - 캐시 장애 시 DB를 fallback 경로로 사용.
        

---

## 8. Look-Aside vs 다른 캐시 패턴

| 패턴            | 특징                       | 장점                            | 단점             |
| ------------- | ------------------------ | ----------------------------- | -------------- |
| Look-Aside    | 애플리케이션이 캐시와 DB를 직접 조회·갱신 | 구현 단순, 캐시 장애 시 DB fallback 가능 | 캐시 미스 시 지연 발생  |
| Write-Through | 쓰기 시 캐시와 DB 동시에 갱신       | 데이터 일관성 높음                    | 쓰기 지연 가능       |
| Write-Behind  | 쓰기 시 캐시에만 반영 후 비동기 DB 갱신 | 쓰기 성능 매우 빠름                   | 장애 시 데이터 유실 위험 |

---

## 9. 실무 적용 팁

1. **TTL + RLock 조합**
    
    - 캐시 미스 시 다수의 쓰레드가 동시에 DB를 조회하지 않게 잠금 적용.
        
2. **Hot Key 모니터링**
    
    - 특정 키가 지나치게 자주 조회되는 경우 캐시 갱신 주기 조정.
        
3. **백업 경로 준비**
    
    - 캐시 장애 시 DB를 fallback 경로로 사용하되, 트래픽이 몰리지 않게 Rate Limit 적용.
        
4. **Lazy Loading**
    
    - 미스가 발생할 때만 캐시 적재.
        

---

## 10. 결론

Look-Aside 패턴은 캐시와 DB를 함께 사용하는 가장 기본적이고 널리 쓰이는 접근 방식입니다.  
Redisson 같은 Redis 클라이언트를 사용하면 TTL 설정, 분산 락(RLock), 자동 캐시 무효화 등과 결합하여 성능과 안정성을 크게 높일 수 있습니다.  
캐시 미스가 발생했을 때 **무엇을 조회하고, 어떻게 캐시에 다시 채울지**를 정의하는 것이 핵심입니다.

---

