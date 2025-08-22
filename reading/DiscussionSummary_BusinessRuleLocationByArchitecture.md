# 토론 정리: 비즈니스 규칙의 구현 위치 (레이어드, 헥사고날, 클린 아키텍처 비교)

## 토론 배경
예약 시스템에서 사용자가 예약 버튼을 누를 때 "이 사람이 예약할 수 있는지 확인하는 코드"의 적절한 위치를 결정하는 것은 중요한 설계 선택입니다. 구체적인 비즈니스 규칙으로는 "하루 최대 3회 예약 가능"과 "취소는 24시간 전까지만 가능"이 제시되었습니다. 이를 레이어드 아키텍처, 헥사고날 아키텍처, 클린 아키텍처의 관점에서 분석하여 각 아키텍처에서 Controller, Service, Domain Entity, 또는 별도의 검증 클래스를 활용한 구현 방식을 비교합니다. 장단점, 테스트 작성 용이성, 유지보수성을 중심으로 논의합니다.

---

## 1. 레이어드 아키텍처에서의 비교

### 1.1 Controller에서 구현
- **방식**: `@PostMapping("/reserve")`에서 "하루 3회 제한", "24시간 취소" 로직 직접 구현.
- **장점**:
  - 초기 구현 간단, 요청-응답 흐름과 로직이 한 곳에 통합.
- **단점**:
  - 비즈니스 로직과 프레젠테이션 혼합, 관심사 분리 위배.
  - 단위 테스트 시 HTTP 컨텍스트 모의 필요, 복잡성 증가.
- **테스트 작성**: MockMvc 사용, 비즈니스 로직 단독 테스트 어려움 (예: 하루 3회 제한 테스트).
- **유지보수성**: 규칙 변경 시 모든 Controller 수정 필요, 중복 코드 위험.

### 1.2 Service 클래스에서 구현
- **방식**: `ReservationService.canReserve(User user, Reservation request)`에 로직 포함.
- **장점**:
  - Business Layer에 로직 집중, Controller는 요청/응답 처리에 집중.
  - 단위 테스트 용이, Mocking으로 의존성 격리 가능.
- **단점**:
  - Service 비대화 위험, 도메인 모델 약화(Anemic Model).
- **테스트 작성**: JUnit + Mockito로 `canReserve` 단독 테스트 (예: 3회 제한, 24시간 조건).
- **유지보수성**: Service 한 곳 수정으로 영향 최소화, 그러나 클래스 커짐 시 관리 어려움.

### 1.3 Domain Entity에서 구현
- **방식**: `User.canReserveToday()` 또는 `Reservation.isCancelable()`에 로직 포함.
- **장점**:
  - 도메인 중심 설계, Rich Domain Model 구현.
  - 로직 캡슐화로 재사용성 높음.
- **단점**:
  - Entity 복잡성 증가, JPA와의 영속성 충돌 가능.
- **테스트 작성**: DB 의존성 제거 후 Mock 사용, 단위 테스트 복잡 (예: `canReserveToday` 테스트).
- **유지보수성**: 규칙 변경 시 Entity 수정, ORM 매핑 문제 주의 필요.

### 1.4 별도의 검증 클래스에서 구현
- **방식**: `ReservationValidator.checkReservationRules(User, Reservation)` 생성.
- **장점**:
  - 검증 로직 모듈화, Service/Entity 부담 감소.
  - 단위 테스트 용이, 의존성 최소화.
- **단점**:
  - 추가 클래스 오버헤드, Service/Controller와 통합 로직 필요.
- **테스트 작성**: Validator 단독 테스트, Mocking으로 격리 (예: 모든 규칙 테스트).
- **유지보수성**: 규칙 변경 시 Validator 수정, 확장성 우수.

### 1.5 레이어드 권장 위치
- **제안**: Service 또는 별도 검증 클래스.
- **이유**: 레이어드의 계층적 구조에서 Business Layer가 비즈니스 로직의 적합한 위치. 초기에는 Service로 시작하되, 규칙 증가 시 `ReservationValidator`로 분리 추천.

---

## 2. 헥사고날 아키텍처에서의 비교

### 2.1 Controller (Inbound Adapter)에서 구현
- **방식**: Inbound Adapter(컨트롤러 역할)에서 포트 호출 전 규칙 검증.
- **장점**:
  - 요청 처리와 초기 검증 통합, 빠른 피드백 가능.
- **단점**:
  - 비즈니스 로직이 Adapter에 혼합, 도메인 독립성 위배.
  - 테스트 시 HTTP/Adapter 모의 필요.
- **테스트 작성**: Mock Adapter 사용, 단독 테스트 어려움.
- **유지보수성**: 규칙 변경 시 모든 Adapter 수정 필요.

### 2.2 Service (Application Layer)에서 구현
- **방식**: Inbound Port(예: `ReservationUseCase`)를 통해 Service에서 로직 처리.
- **장점**:
  - 도메인 로직이 Application Layer에 위치, 포트로 추상화.
  - 테스트 용이, Mock Port로 의존성 격리.
- **단점**:
  - Application Layer 비대화 위험, 도메인과 분리 시 오버헤드.
- **테스트 작성**: `ReservationUseCase` 단독 테스트, Mock Port 사용.
- **유지보수성**: 포트 인터페이스 변경 시 영향 최소화, 그러나 구현체 관리 필요.

### 2.3 Domain Entity (Domain Layer)에서 구현
- **방식**: 도메인 객체(예: `User.reserve()`)에 로직 포함.
- **장점**:
  - 도메인 중심 설계, 비즈니스 로직 순수성 유지.
  - 외부 의존성 없이 독립적 운영.
- **단점**:
  - 도메인 복잡성 증가, Adapter와 통합 시 추가 노력 필요.
- **테스트 작성**: 도메인 단독 테스트, 외부 의존성 제거.
- **유지보수성**: 규칙 변경 시 도메인만 수정, 장기적 안정성 우수.

### 2.4 별도의 검증 클래스 (Outbound Port/Adapter)에서 구현
- **방식**: Outbound Port(예: `ReservationCheckPort`)와 Adapter로 검증 로직 분리.
- **장점**:
  - 검증 로직 모듈화, 도메인 독립성 유지.
  - 어댑터 교체 용이, 테스트 용이.
- **단점**:
  - 포트/어댑터 추가로 설계 비용 증가.
- **테스트 작성**: Mock Port로 단독 테스트.
- **유지보수성**: 규칙 변경 시 Adapter만 수정, 확장성 우수.

### 2.5 헥사고날 권장 위치
- **제안**: Domain Layer 또는 Outbound Port/Adapter.
- **이유**: 헥사고날의 도메인 중심 설계에 맞게 `User.reserve()`로 구현하거나, 복잡한 규칙은 `ReservationCheckPort`와 Adapter로 분리. 도메인 순수성을 유지하며 유연성 확보.

---

## 3. 클린 아키텍처에서의 비교

### 3.1 Controller (Interface Adapter)에서 구현
- **방식**: Interface Adapter에서 Use Case 호출 전 규칙 검증.
- **장점**:
  - 요청-응답 변환과 초기 검증 통합.
- **단점**:
  - 비즈니스 로직이 Adapter에 혼합, 의존성 규칙 위배.
  - 테스트 시 Adapter 모의 필요.
- **테스트 작성**: Mock Adapter 사용, 단독 테스트 어려움.
- **유지보수성**: 규칙 변경 시 모든 Adapter 수정.

### 3.2 Service (Use Case)에서 구현
- **방식**: Use Case Interactor(예: `ReserveOrderInteractor`)에 로직 포함.
- **장점**:
  - 애플리케이션 특화 규칙이 Use Case에 적합.
  - 단위 테스트 용이, Mock Input/Output Port 사용.
- **단점**:
  - Use Case 비대화 위험, Entities와 분리 시 중복 가능성.
- **테스트 작성**: `ReserveOrderInteractor` 단독 테스트, Mock Port 사용.
- **유지보수성**: Use Case 수정으로 영향 최소화, 인터페이스 관리 필요.

### 3.3 Domain Entity (Entities)에서 구현
- **방점**: Entities(예: `User.reserve()`)에 핵심 비즈니스 로직 포함.
- **장점**:
  - 핵심 비즈니스 규칙이 Entities에 속함, 의존성 안으로 향.
  - 독립적 테스트 및 재사용성 우수.
- **단점**:
  - Entities 복잡성 증가, Adapter 통합 시 추가 노력.
- **테스트 작성**: Entities 단독 테스트, 외부 의존성 제거.
- **유지보수성**: 규칙 변경 시 Entities만 수정, 안정성 높음.

### 3.4 별도의 검증 클래스 (Use Case 지원)에서 구현
- **방식**: Use Case 내 별도 검증 로직(예: `ReservationRulesChecker`) 분리.
- **장점**:
  - 검증 로직 모듈화, Use Case 가독성 향상.
  - 테스트 용이, 의존성 격리.
- **단점**:
  - 추가 클래스 오버헤드, Use Case와 통합 로직 필요.
- **테스트 작성**: `ReservationRulesChecker` 단독 테스트, Mock 사용.
- **유지보수성**: 규칙 변경 시 Checker 수정, 확장성 우수.

### 3.5 클린 아키텍처 권장 위치
- **제안**: Entities 또는 Use Case 내 별도 검증 클래스.
- **이유**: 클린 아키텍처의 의존성 규칙에 따라 Entities에 핵심 로직, 복잡한 규칙은 Use Case 내 `ReservationRulesChecker`로 분리. 독립성과 유지보수성 확보.

---

## 4. 종합 비교 분석

| 아키텍처    | 권장 위치            | 테스트 용이성       | 유지보수성           | 적합 시나리오                  |
|-------------|----------------------|---------------------|----------------------|--------------------------------|
| 레이어드    | Service/Validator    | 중간~높음           | 중간~높음            | 중간 규모, 계층적 구조 선호    |
| 헥사고날    | Domain/Port+Adapter  | 높음                | 높음                 | 복잡한 도메인, 외부 통합       |
| 클린        | Entities/Use Case    | 높음                | 높음                 | 장기 프로젝트, 의존성 관리      |

## 5. 결론 및 제안
- **권장 접근**: 아키텍처에 따라 다름.
  - **레이어드**: `ReservationService` 또는 `ReservationValidator`로 시작, 필요 시 분리.
  - **헥사고날**: `User.reserve()` 또는 `ReservationCheckPort`+Adapter로 도메인 중심 설계.
  - **클린**: `User.reserve()` 또는 `ReserveOrderInteractor` 내 `ReservationRulesChecker`로 의존성 준수.
- **추가 고려**: 프로젝트 초기에는 Service/Use Case로 시작, 복잡성 증가 시 도메인/별도 클래스로 분리.
- **다음 단계**: 팀 내 합의 후 프로토타입 구현 및 테스트 결과 4:50까지 공유.