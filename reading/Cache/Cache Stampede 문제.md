좋아요. 그럼 **Cache Stampede 방지 전략**을 포함해서 **Redisson 전용 Look-Aside 패턴 심화 가이드**를 md 파일로 작성해 드리겠습니다.

---

# Cache Stampede 문제

### 1.1 문제 설명

- **Cache Stampede**는 특정 키의 캐시가 만료되었을 때, 동시에 다수의 요청이 해당 키를 조회하려고 DB에 몰리는 현상입니다.
    
- 대량의 DB 쿼리 발생 → DB 과부하 → 서비스 지연/장애 가능.
    

### 1.2 원인

- 캐시 TTL 만료 시점이 동일.
    
- 캐시 미스 시 모든 요청이 DB를 직접 조회.
    

---

## 2. Cache Stampede 방지 전략

### 2.1 분산 락(RLock) 사용

- 캐시 미스 발생 시, **첫 번째 요청만 DB 조회** → 다른 요청은 대기 후 캐시에서 조회.
    
- Redisson의 RLock은 분산 환경에서도 동작.
    

### 2.2 캐시 만료 시간 랜덤화

- TTL을 일정 범위에서 랜덤하게 설정 → 동일 시각에 대규모 캐시 만료 방지.
    

### 2.3 Soft TTL + 백그라운드 갱신

- Soft TTL: 캐시 값이 만료되었지만 일정 기간 동안은 사용 가능.
    
- 백그라운드 스레드에서 만료된 값 갱신.
    

---

## 4. Redisson 전용 Look-Aside 패턴 구현 예시

```java
public String getUserProfile(String userId) {
    String cacheKey = "userProfiles:" + userId;
    RMapCache<String, String> cache = redissonClient.getMapCache("userProfiles");

    // 1. 캐시 조회
    String profile = cache.get(cacheKey);
    if (profile != null) {
        return profile; // Cache Hit
    }

    // 2. 캐시 미스 → 분산 락 획득
    RLock lock = redissonClient.getLock("lock:" + cacheKey);
    try {
        // 다른 요청이 이미 락을 잡았으면 대기
        if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
            // 락을 잡은 이후 캐시를 다시 확인 (Double Check)
            profile = cache.get(cacheKey);
            if (profile != null) {
                return profile;
            }

            // DB 조회
            profile = userRepository.findById(userId)
                                    .map(User::toJson)
                                    .orElse(null);

            // 캐시에 저장 (TTL + 랜덤화 적용)
            if (profile != null) {
                long ttl = 30 + ThreadLocalRandom.current().nextInt(0, 10); // 30~40분
                cache.put(cacheKey, profile, ttl, TimeUnit.MINUTES);
            }
        } else {
            // 락을 못 잡으면 대기 후 캐시 재조회
            Thread.sleep(100);
            return cache.get(cacheKey);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }

    return profile;
}
```

---

## 6. 적용 시 고려사항

- **락 범위 최소화**: DB 조회와 캐시 갱신 로직에만 락 적용.
    
- **락 타임아웃 설정**: Deadlock 방지.
    
- **TTL 랜덤화**: 동시 만료 방지.
    
- **모니터링**: 캐시 히트율, DB 부하, 락 획득 실패율 모니터링.
    

---

## 7. 장점

- DB 부하 감소 (Cache Stampede 방지).
    
- 캐시와 DB의 데이터 일관성 유지.
    
- 장애 시에도 캐시 없이 DB 조회 가능.
    

---

## 8. 결론

Redisson 기반 Look-Aside 패턴은 단순히 캐시를 읽고 DB에서 채우는 수준을 넘어,  
**분산 락 + TTL 랜덤화 + Soft TTL** 등을 적용하면 **고부하 환경에서도 안정적으로 동작**할 수 있습니다.  
이 패턴은 API 서버, 전자상거래, 로그성 데이터 조회 등 읽기 비중이 높은 서비스에 특히 유용합니다.
