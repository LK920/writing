# Testcontainers: 내 테스트만을 위한 일회용 장난감 가게

## 1. Testcontainers가 뭔가요? 왜 쓰는 건가요?

상상해보세요. 여러분은 멋진 자동차 장난감(여러분의 애플리케이션)을 만들었습니다. 이 자동차가 잘 달리는지 테스트하려면 실제 도로(운영 환경)와 아주 비슷한 테스트용 도로가 필요합니다. 이 도로에는 신호등, 다른 자동차, 주차장(데이터베이스, 메시지 큐 등)이 있어야 합니다.

- **과거의 방식 (불편한 장난감 가게)**: 모든 장난감 자동차들이 하나의 거대한 테스트용 도로 세트장을 공유합니다. 다른 자동차가 도로를 망가뜨리거나, 내가 테스트하는 동안 다른 사람이 도로 설정을 바꿔버릴 수 있습니다. 테스트가 끝나도 내가 만든 쓰레기(테스트 데이터)를 직접 치워야 합니다.

- **H2 인메모리 DB (장난감 도로 스케치)**: 실제 도로와 비슷한 그림을 종이에 그려놓고 그 위에서 자동차를 굴리는 것과 같습니다. 빠르고 간편하지만, 실제 도로의 질감, 경사, 다른 자동차의 움직임(DB의 고유한 함수, 트랜잭션 동작 등)과는 전혀 다릅니다. 스케치 위에서는 잘 달리던 차가 실제 도로에서는 고장 날 수 있습니다.

- **Testcontainers (나만의 일회용 도로 세트)**: 테스트를 시작할 때마다, 나만을 위한 새로운 도로 세트장(Docker 컨테이너)을 즉석에서 만듭니다. 이 세트장에는 내가 원하는 사양의 신호등(MySQL, Redis 등)이 정확하게 설치됩니다. 내 테스트는 완전히 독립된 공간에서 진행되며, 다른 누구에게도 방해받지 않습니다. 테스트가 끝나면, 이 세트장은 **자동으로 사라집니다.** 청소할 필요도 없죠!

> **핵심**: Testcontainers는 JUnit 같은 테스트 라이브러리 안에서 **프로그래밍 방식으로 Docker 컨테이너를 제어**할 수 있게 해주는 도구입니다. 덕분에 실제 운영 환경과 거의 동일한 환경에서, 깨끗하고 독립적인 통합 테스트를 수행할 수 있습니다.

## 2. 왜 그냥 H2 DB를 쓰면 안 되나요?

H2 같은 인메모리 데이터베이스는 빠르고 간편하다는 큰 장점이 있지만, 결정적인 단점이 있습니다.

- **"내 컴퓨터에선 됐는데..."의 원인**: H2는 MySQL이나 PostgreSQL이 아닙니다. 데이터 타입, SQL 문법, 함수, 트랜잭션 처리 방식 등 미묘하지만 중요한 부분들이 다릅니다. H2에서 통과한 테스트가 실제 운영 DB에서는 실패할 수 있습니다.
- **비유**: 한국어와 일본어는 문법 구조가 비슷해서 배우기 쉽지만, 엄연히 다른 언어입니다. 일본어로 대화하는 연습만 하다가 갑자기 한국어로 중요한 발표를 하라면 실수할 수밖에 없는 것과 같습니다.

Testcontainers는 실제 운영에 사용할 **MySQL 8.0 Docker 이미지를 그대로 가져와 테스트 환경을 만들기 때문에** 이러한 환경 불일치 문제를 원천적으로 해결합니다.

## 3. Testcontainers, 어떻게 사용하나요?

Testcontainers를 사용하는 것은 마치 레고 블록을 조립하는 것과 같습니다.

**1단계: 필요한 블록(의존성) 가져오기**

`build.gradle.kts` 파일에 필요한 Testcontainers 라이브러리를 추가합니다. 레고 가게에서 필요한 부품을 사는 것과 같습니다.

```kotlin
// build.gradle.kts

dependencies {
    // ... 다른 의존성들
    testImplementation("org.springframework.boot:spring-boot-testcontainers") // 스프링 부트와 Testcontainers를 쉽게 연결해주는 접착제
    testImplementation("org.testcontainers:junit-jupiter") // JUnit 5에서 Testcontainers를 사용하기 위한 부품
    testImplementation("org.testcontainers:mysql") // MySQL 컨테이너를 사용하기 위한 특수 부품
}
```

**2단계: 나만의 테스트 장난감 가게(Configuration) 설계하기**

테스트를 실행할 때 어떤 종류의 컨테이너(도로 세트장)를 만들지 설계도를 그립니다. 이 프로젝트에서는 `TestcontainersConfiguration.java` 파일이 그 역할을 합니다.

```java
// src/test/java/kr/hhplus/be/server/TestcontainersConfiguration.java

@TestConfiguration // "이건 테스트 전용 설계도야!" 라고 알려주는 스티커
class TestcontainersConfiguration {

	@Bean // 스프링이 관리하는 특별한 부품(컨테이너)을 만들 거야
	@ServiceConnection // 엄청난 마법! 이 스티커 하나로 스프링 부트가 컨테이너 정보를 알아서 가져가. (JDBC URL, username, password 등)
	MySQLContainer<?> mysqlContainer() {
        // 1. "mysql:8.0" 이미지로 된 MySQL 컨테이너를 만들고
		return new MySQLContainer<>(DockerImageName.parse("mysql:8.0"))
            // 2. 컨테이너 안에 "hhplus" 라는 데이터베이스를 만들고
			.withDatabaseName("hhplus")
            // 3. 접속 계정은 "test" / "test" 로 설정해줘!
			.withUsername("test")
			.withPassword("test");
	}
}
```

- **`@TestConfiguration`**: 이 설정은 오직 테스트 환경에서만 로드됩니다.
- **`@Bean`**: 스프링의 테스트 컨텍스트에 `MySQLContainer` 객체를 등록합니다. 즉, 스프링이 이 컨테이너의 존재를 알게 됩니다.
- **`@ServiceConnection`**: **가장 중요한 마법의 어노테이션입니다.** 과거에는 컨테이너가 실행될 때마다 바뀌는 IP 주소와 포트 번호를 알아내서 `application.yml`의 `spring.datasource.url` 같은 설정에 수동으로 주입해줘야 했습니다. 하지만 `@ServiceConnection`을 붙이면, 스프링 부트가 테스트 컨테이너의 모든 연결 정보(JDBC URL, 사용자 이름, 비밀번호 등)를 **자동으로 감지하고** 테스트용 DataSource에 반영해줍니다. 더 이상 수동으로 설정할 필요가 없습니다!

**3단계: 테스트 코드에서 사용하기**

이제 테스트 클래스에 `@SpringBootTest` 어노테이션만 붙이면, 테스트가 실행되기 전에 다음의 일들이 자동으로 일어납니다.

1.  `TestcontainersConfiguration` 설계도를 읽어들입니다.
2.  Docker를 사용하여 MySQL 8.0 컨테이너를 실행시킵니다.
3.  `@ServiceConnection` 덕분에, 스프링 부트는 새로 만들어진 MySQL 컨테이너에 연결되도록 DataSource 설정을 자동으로 변경합니다.
4.  테스트 코드가 실행됩니다. 모든 DB 관련 로직은 방금 뜬 따끈따끈한 일회용 MySQL 컨테이너 위에서 수행됩니다.
5.  테스트가 모두 끝나면, Testcontainers가 해당 MySQL 컨테이너를 **자동으로 종료하고 삭제**합니다.

## 4. 이 프로젝트의 Testcontainers 설정 현황

- **설정 파일**: `build.gradle.kts`에 `spring-boot-testcontainers`, `testcontainers-junit-jupiter`, `testcontainers-mysql` 의존성이 모두 포함되어 있습니다.
- **구성 클래스**: `src/test/java/kr/hhplus/be/server/TestcontainersConfiguration.java`에 MySQL 8.0 컨테이너를 생성하고, `@ServiceConnection`을 통해 스프링 부트 테스트에 자동으로 연동되도록 설정되어 있습니다.
- **동작 방식**: `@SpringBootTest`가 붙은 통합 테스트(`BalanceTest`, `ProductTest` 등)가 실행될 때, H2 인메모리 DB 대신 **실제 MySQL 8.0 Docker 컨테이너가 동적으로 실행**되어 테스트 환경으로 사용됩니다. 이를 통해 운영 환경과 거의 100% 동일한 환경에서 신뢰도 높은 통합 테스트를 보장합니다.
