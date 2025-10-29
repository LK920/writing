# 캐시 공급 패턴 학습 자료

캐시 공급 방식은 캐시 미스 시 데이터베이스(DB)에서 값을 조회해 캐시에 저장하는 패턴으로, 소프트웨어 디자인 패턴의 일종으로 볼 수 있습니다. 이 자료에서는 `Supplier`, `Factory`, `Lazy Initialization` 패턴을 중심으로, 캐시 미스 시 DB 반영 로직을 어떻게 구현하는지 설명합니다. 각 패턴은 프레임워크(예: Spring Cache, Caffeine)와 통합될 때 변형되며, 멀티스레드 환경에서의 동기화와 확장성도 고려됩니다.

## 1. Supplier 패턴
`Supplier` 패턴은 캐시 미스 시 데이터를 생성하거나 DB에서 조회해 캐시에 저장하는 함수형 접근 방식입니다. Java의 `Supplier` 인터페이스나 프레임워크의 캐시 로딩 메커니즘을 활용하며, 지연 로딩(Lazy Loading)을 지원합니다.

### 특징
- **지연 로딩**: 캐시 값은 요청 시점에 생성/조회.
- **함수형 스타일**: 람다 표현식이나 메서드 참조로 간결히 구현.
- **프레임워크 통합**: Spring Cache의 `@Cacheable`이나 Caffeine의 `CacheLoader`에서 자주 사용.

### 예제 코드 (Spring Cache)
Spring Cache에서 `@Cacheable`을 사용해 캐시 미스 시 DB 조회 후 캐시 저장.

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class DataService {
    @Cacheable(value = "myCache", key = "#id", sync = true)
    public String getData(String id) {
        System.out.println("DB 조회: " + id);
        return "DB 데이터: " + id; // DB 조회 로직
    }

    public static void main(String[] args) {
        // Spring 컨텍스트에서 호출 가정
        DataService service = new DataService();
        System.out.println(service.getData("key1")); // DB 조회 후 캐시 저장
        System.out.println(service.getData("key1")); // 캐시에서 조회
    }
}
```

### 예제 코드 (Caffeine)
Caffeine의 `LoadingCache`를 사용해 `Supplier` 패턴 구현.

```java
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.LoadingCache;

public class CacheExample {
    public static void main(String[] args) {
        LoadingCache<String, String> cache = Caffeine.newBuilder()
                .build(key -> {
                    System.out.println("DB 조회: " + key);
                    return "DB 데이터: " + key; // DB 조회 로직
                });

        System.out.println(cache.get("key1")); // DB 조회 후 캐시 저장
        System.out.println(cache.get("key1")); // 캐시에서 조회
    }
}
```

### 장점
- 간결한 구현: 함수형 스타일로 코드 간소화.
- 프레임워크 통합: Spring, Caffeine 등에서 자동으로 캐시 저장 처리.
- 메모리 효율: 필요 시에만 데이터 로드.

### 단점
- 멀티스레드 환경에서 동기화 필요 (예: `sync = true` 또는 `ConcurrentHashMap`).
- 복잡한 데이터 생성 로직에는 추가 설계 필요.

---

## 2. Factory 패턴
`Factory` 패턴은 캐시 미스 시 DB 조회 로직을 별도의 팩토리 객체 또는 메서드에 캡슐화하여 처리합니다. 캐시 저장은 팩토리 외부의 캐시 관리자나 프레임워크가 담당할 수 있습니다.

### 특징
- **캡슐화**: 데이터 생성/조회 로직을 팩토리에 분리.
- **확장성**: 다양한 데이터 소스(DB, API 등)를 처리 가능.
- **명시적**: 캐시 미스 시 호출되는 로직이 명확.

### 예제 코드
팩토리에서 DB 조회, 캐시 관리자가 캐시 저장 처리.

```java
public class DataFactory {
    public String fetchDataFromDB(String key) {
        System.out.println("DB 조회: " + key);
        return "DB 데이터: " + key;
    }
}

public class CacheManager {
    private final DataFactory factory;
    private final Map<String, String> cache = new ConcurrentHashMap<>();

    public CacheManager(DataFactory factory) {
        this.factory = factory;
    }

    public String getData(String key) {
        return cache.computeIfAbsent(key, k -> factory.fetchDataFromDB(k));
    }

    public static void main(String[] args) {
        CacheManager manager = new CacheManager(new DataFactory());
        System.out.println(manager.getData("key1")); // DB 조회 후 캐시 저장
        System.out.println(manager.getData("key1")); // 캐시에서 조회
    }
}
```

### 장점
- 복잡한 생성 로직 캡슐화 가능.
- 다양한 데이터 소스 처리에 적합.
- 프레임워크와 통합 가능 (예: Spring의 `@Cacheable` 메서드를 팩토리로 사용).

### 단점
- 팩토리 객체 관리 필요.
- 단순한 캐싱에는 다소 복잡할 수 있음.

---

## 3. Lazy Initialization 패턴
`Lazy Initialization` 패턴은 캐시 값이 `null`일 때(캐시 미스) DB에서 값을 조회해 캐시에 저장하는 방식입니다. 간단한 구현이 특징이며, 프레임워크 없이도 적용 가능합니다.

### 특징
- **직관적**: 캐시 미스 시 초기화 로직이 명시적.
- **간단한 구현**: 별도의 인터페이스나 팩토리 없이 가능.
- **스레드 안전성**: `ConcurrentHashMap`이나 동기화 처리로 멀티스레드 지원.

### 예제 코드
`ConcurrentHashMap`을 사용해 캐시 미스 시 DB 조회.

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

public class LazyCache {
    private final Map<String, String> cache = new ConcurrentHashMap<>();

    public String getData(String key) {
        return cache.computeIfAbsent(key, k -> {
            System.out.println("DB 조회: " + key);
            return "DB 데이터: " + key; // DB 조회 로직
        });
    }

    public static void main(String[] args) {
        LazyCache cache = new LazyCache();
        System.out.println(cache.getData("key1")); // DB 조회 후 캐시 저장
        System.out.println(cache.getData("key1")); // 캐시에서 조회
    }
}
```

### 장점
- 구현이 간단하고 직관적.
- `ConcurrentHashMap`으로 스레드 안전성 확보 가능.
- 프레임워크 없이도 동작.

### 단점
- 복잡한 갱신 로직이나 TTL 설정은 추가 구현 필요.
- 프레임워크 통합 없이 사용할 경우 관리 부담 증가.

---

## 비교
| 패턴            | 캐시 미스 처리 방식                     | 장점                              | 단점                              | 적합한 사용 사례                     |
|----------------|---------------------------------------|----------------------------------|----------------------------------|------------------------------------|
| Supplier       | 함수형 인터페이스로 DB 조회 후 캐시 저장 | 간결, 프레임워크 통합 용이, 메모리 효율 | 동기화 처리 필요, 복잡한 로직 제한 | Spring/Caffeine과 통합된 간단한 캐싱 |
| Factory        | 팩토리 객체로 DB 조회 후 캐시 저장     | 캡슐화, 확장성 우수              | 팩토리 관리 필요, 다소 복잡       | 복잡한 데이터 생성 로직 필요 시      |
| Lazy Initialization | 캐시 미스 시 직접 초기화 및 저장   | 간단, 직관적, 스레드 안전성 지원 | 갱신 로직 관리 어려움, 프레임워크 미사용 시 부담 | 간단한 캐싱, 프레임워크 없는 환경   |

---

## 프레임워크와의 연관성
- **Spring Cache**: `@Cacheable`은 `Supplier` 패턴과 유사하며, 캐시 미스 시 메서드 실행 결과를 캐시에 저장. `sync = true`로 동기화 지원.
- **Caffeine/Guava**: `CacheLoader`는 `Supplier` 패턴을 기반으로 캐시 미스 시 데이터를 로드하고 저장. `Factory` 패턴도 커스텀 로직으로 통합 가능.
- **Redis/Ehcache**: 외부 캐시 저장소와 결합 시 `Supplier` 또는 `Factory` 패턴으로 데이터 조회 로직 제공, 캐시 저장은 프레임워크가 처리.
- **Lazy Initialization**: 프레임워크 없이 구현할 때는 `ConcurrentHashMap`을 활용, 프레임워크와 통합 시 `Supplier`로 대체될 수 있음.

---

## 결론
- **Supplier 패턴**은 캐시 미스 시 DB 조회 후 저장을 간결히 처리하며, Spring Cache나 Caffeine과 같은 프레임워크와 잘 맞습니다.
- **Factory 패턴**은 복잡한 데이터 생성 로직을 캡슐화하며, 다양한 소스에서 데이터를 가져올 때 유용합니다.
- **Lazy Initialization 패턴**은 간단한 캐시 미스 처리에 적합하며, 프레임워크 없이도 쉽게 구현 가능합니다.
- **패턴적 관점**: 이들은 모두 캐시 미스 시 데이터를 로드하고 저장하는 공통 목표를 가지며, 구현 방식(함수형, 객체 지향, 직접 초기화)과 프레임워크 통합 여부에 따라 달라집니다.

프로젝트 요구사항(예: 스레드 안전성, 캐시 만료 정책, 데이터 소스 복잡도)에 따라 적절한 패턴을 선택하고, 필요 시 프레임워크를 활용해 관리 부담을 줄이세요.