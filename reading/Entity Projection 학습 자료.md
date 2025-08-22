
이 문서는 Spring, JPA, 클린 아키텍처(헥사고날 아키텍처)에서 **엔티티 프로젝션(Entity Projection)**의 개념, 사용 목적, 구현 방법을 다룹니다. 각 프레임워크와 아키텍처에서의 차이점을 비교하고, 실제 코드 예시를 제공하여 학습자가 실무에서 활용할 수 있도록 돕습니다.

## 1. 엔티티 프로젝션의 정의

**엔티티 프로젝션**은 데이터베이스의 엔티티(Entity)에서 **필요한 속성(필드)만 선택적으로 추출**하여 애플리케이션에서 사용하는 기술입니다. 이는 데이터 조회 성능을 최적화하고, 불필요한 데이터를 줄여 메모리 사용량과 네트워크 부하를 감소시키는 데 목적이 있습니다.

- **엔티티**: 데이터베이스 테이블과 매핑되는 객체(예: `Student`, `Order`)로, 업무에 필요한 데이터를 나타냅니다.
- **프로젝션**: 데이터의 특정 속성만 선택하여 반환하는 과정(예: SQL의 `SELECT id, name FROM student`).
- **핵심 목적**:
    - 성능 최적화: 대용량 필드(예: 이미지, 긴 텍스트) 제외.
    - 데이터 간소화: 클라이언트(예: API 호출자)에게 필요한 데이터만 제공.
    - 유연한 데이터 처리: 특정 용도에 맞는 데이터 구조로 변환.

## 2. Spring과 JPA에서의 엔티티 프로젝션

Spring과 JPA(Java Persistence API)에서는 엔티티 프로젝션을 주로 **Spring Data JPA**를 통해 구현합니다. JPA 엔티티는 데이터베이스 테이블과 직접 매핑되며, 프로젝션은 `Repository` 계층에서 특정 필드를 선택하여 반환합니다.

### 2.1. 구현 방식

Spring Data JPA는 엔티티 프로젝션을 위해 다음 네 가지 주요 방식을 제공합니다:

#### (1) 인터페이스 기반 프로젝션

- **설명**: 인터페이스를 정의하여 필요한 필드만 지정. JPA가 프록시 객체를 동적으로 생성하여 데이터를 매핑.
- **장점**: 간단하고 명시적, 동적 프로젝션 지원.
- **단점**: 복잡한 쿼리나 중첩 객체 처리 시 제한적.
- **예시**:
    
    ```java
    // 프로젝션 인터페이스
    public interface StudentProjection {
        Long getId();
        String getName();
    }
    
    // Repository
    public interface StudentRepository extends JpaRepository<Student, Long> {
        List<StudentProjection> findByGrade(String grade);
    }
    ```
    
    - 실행되는 SQL: `SELECT s.id, s.name FROM student s WHERE s.grade = ?`.

#### (2) 클래스 기반 프로젝션 (DTO)

- **설명**: DTO 클래스를 정의하여 JPQL로 데이터를 매핑.
- **장점**: 복잡한 쿼리나 조인 처리에 유연.
- **단점**: DTO 클래스를 별도로 작성해야 함.
- **예시**:
    
    ```java
    // DTO 클래스
    public class StudentDto {
        private Long id;
        private String name;
    
        public StudentDto(Long id, String name) {
            this.id = id;
            this.name = name;
        }
        // Getter, Setter
    }
    
    // Repository
    public interface StudentRepository extends JpaRepository<Student, Long> {
        @Query("SELECT new com.example.StudentDto(s.id, s.name) FROM Student s WHERE s.grade = :grade")
        List<StudentDto> findStudentDtosByGrade(@Param("grade") String grade);
    }
    ```
    

#### (3) 동적 프로젝션

- **설명**: 메서드 호출 시 프로젝션 타입을 동적으로 지정.
- **장점**: 유연성 제공(동일 메서드로 다양한 결과 반환).
- **예시**:
    
    ```java
    public interface StudentRepository extends JpaRepository<Student, Long> {
        <T> List<T> findByGrade(String grade, Class<T> type);
    }
    
    // 호출
    List<StudentProjection> projections = repository.findByGrade("A", StudentProjection.class);
    List<StudentDto> dtos = repository.findByGrade("A", StudentDto.class);
    ```
    

#### (4) JPQL/네이티브 쿼리 기반 프로젝션

- **설명**: JPQL 또는 네이티브 SQL로 특정 필드를 직접 조회.
- **예시**:
    
    ```java
    public interface StudentRepository extends JpaRepository<Student, Long> {
        @Query("SELECT s.id, s.name FROM Student s WHERE s.grade = :grade")
        List<Object[]> findByGrade(@Param("grade") String grade);
    }
    ```
    
    - 결과는 `Object[]` 배열로 반환(예: `[id, name]`).

### 2.2. 특징과 주의점

- **성능 최적화**: 불필요한 필드(예: 대용량 `profileImage`) 제외.
- **한계**:
    - 프로젝션 결과는 JPA의 영속성 컨텍스트에서 관리되지 않음.
    - 조인이나 관계 처리 시 N+1 쿼리 문제가 발생할 수 있음(해결: `@EntityGraph`, `fetch join`).
- **활용 사례**: REST API에서 클라이언트에 필요한 데이터만 반환(예: 학생 목록에서 `id`, `name`만 제공).

## 3. 클린 아키텍처(헥사고날)에서의 엔티티 프로젝션

클린 아키텍처는 **도메인 중심 설계**와 **기술 독립성**을 강조하며, 시스템을 계층화하여 비즈니스 로직을 외부 기술(JPA, 데이터베이스)로부터 분리합니다. 헥사고날 아키텍처(포트와 어댑터 패턴)를 통해 구현되며, 엔티티 프로젝션은 **어댑터 계층**에서 처리됩니다.

### 3.1. 클린 아키텍처의 구조

- **도메인 계층**: 비즈니스 로직과 도메인 엔티티(순수 자바 객체).
- **유스케이스 계층**: 비즈니스 로직의 실행 흐름 정의.
- **포트**: 도메인과 외부 시스템 간의 인터페이스(예: 데이터 액세스 포트).
- **어댑터**: 포트를 실제 기술(JPA, REST API 등)에 연결.

### 3.2. 엔티티 프로젝션의 구현

- **도메인 엔티티**: JPA 어노테이션을 포함하지 않으며, 비즈니스 로직에 집중.
- **JPA 엔티티**: 어댑터 계층에서 데이터베이스와 매핑.
- **프로젝션**: 어댑터에서 JPA를 사용해 데이터를 조회하고, 도메인 DTO나 객체로 변환하여 반환.

#### 예시: 학생 관리 시스템

##### (1) 도메인 계층

- 도메인 엔티티:
    
    ```java
    package domain;
    
    public class Student {
        private Long id;
        private String name;
        private String grade;
    
        public Student(Long id, String name, String grade) {
            this.id = id;
            this.name = name;
            this.grade = grade;
        }
        // Getter
    }
    ```
    
- 프로jek션용 DTO:
    
    ```java
    package domain;
    
    public class StudentSummary {
        private Long id;
        private String name;
    
        public StudentSummary(Long id, String name) {
            this.id = id;
            this.name = name;
        }
        // Getter
    }
    ```
    

##### (2) 유스케이스 계층

- 유스케이스 인터페이스:
    
    ```java
    package usecase;
    
    import domain.StudentSummary;
    
    public interface StudentUseCase {
        StudentSummary findStudentSummary(Long id);
    }
    ```
    

##### (3) 포트 계층

- 데이터 액세스 포트:
    
    ```java
    package port;
    
    import domain.StudentSummary;
    
    public interface StudentRepositoryPort {
        StudentSummary findStudentSummary(Long id);
    }
    ```
    

##### (4) 어댑터 계층

- JPA 엔티티:
    
    ```java
    package adapter.persistence;
    
    import javax.persistence.Entity;
    import javax.persistence.Id;
    
    @Entity
    public class JpaStudent {
        @Id
        private Long id;
        private String name;
        private String grade;
        private String address;
        // Getter, Setter
    }
    ```
    
- 프로젝션 인터페이스:
    
    ```java
    package adapter.persistence;
    
    public interface StudentProjection {
        Long getId();
        String getName();
    }
    ```
    
- JPA Repository:
    
    ```java
    package adapter.persistence;
    
    import org.springframework.data.jpa.repository.JpaRepository;
    
    public interface JpaStudentRepository extends JpaRepository<JpaStudent, Long> {
        StudentProjection findProjectionById(Long id);
    }
    ```
    
- 어댑터:
    
    ```java
    package adapter.persistence;
    
    import domain.StudentSummary;
    import port.StudentRepositoryPort;
    import org.springframework.stereotype.Component;
    
    @Component
    public class StudentRepositoryAdapter implements StudentRepositoryPort {
        private final JpaStudentRepository jpaRepository;
    
        public StudentRepositoryAdapter(JpaStudentRepository jpaRepository) {
            this.jpaRepository = jpaRepository;
        }
    
        @Override
        public StudentSummary findStudentSummary(Long id) {
            StudentProjection projection = jpaRepository.findProjectionById(id);
            if (projection == null) {
                return null;
            }
            return new StudentSummary(projection.getId(), projection.getName());
        }
    }
    ```
    

##### (5) 결과

- **도메인**: JPA에 의존하지 않음.
- **어댑터**: JPA를 사용해 프로젝션 수행, 도메인 DTO로 변환.
- **효과**: 데이터베이스 기술 변경(예: JPA → MyBatis) 시 도메인과 유스케이스 계층은 수정 불필요.

### 3.3. 특징과 주의점

- **장점**:
    - 기술 독립성: JPA를 어댑터로 분리하여 도메인 로직이 기술에 의존하지 않음.
    - 테스트 용이성: 도메인 로직은 순수 자바로 작성되어 단위 테스트가 쉬움.
    - 유연성: 다른 데이터 소스(예: NoSQL)로 전환 가능.
- **단점**:
    - 추가 계층(포트, 어댑터)으로 인해 초기 설계 복잡도 증가.
    - DTO와 도메인 객체 간 매핑 코드 추가 필요.

## 4. Spring/JPA와 클린 아키텍처의 비교

|**항목**|**Spring/JPA**|**클린 아키텍처 + JPA**|
|---|---|---|
|**엔티티 정의**|JPA 엔티티(`@Entity`)가 데이터베이스와 직접 매핑.|도메인 엔티티는 순수, JPA 엔티티는 어댑터에서 사용.|
|**프로젝션 위치**|`Repository`에서 처리.|어댑터에서 처리, 도메인은 DTO로 받음.|
|**기술 의존성**|JPA에 강하게 의존.|도메인과 유스케이스는 JPA 독립, 어댑터만 JPA 사용.|
|**구현 복잡도**|단순(Repository에서 직접 처리).|포트와 어댑터 추가로 약간 복잡.|
|**유연성**|JPA 기능에 제한됨.|기술 변경에 유연(예: JPA → MyBatis).|
|**테스트 용이성**|JPA 환경 필요, 테스트 복잡.|도메인 로직은 순수하여 테스트 쉬움.|

## 5. 실제 활용 사례

- **Spring/JPA**: REST API에서 학생 목록을 조회할 때 `id`, `name`만 반환하여 네트워크 부하 감소.
    
    ```java
    @RestController
    public class StudentController {
        @Autowired
        private StudentRepository repository;
    
        @GetMapping("/students")
        public List<StudentProjection> getStudentsByGrade(@RequestParam String grade) {
            return repository.findByGrade(grade);
        }
    }
    ```
    
- **클린 아키텍처**: 동일 요구사항을 처리하되, 도메인과 기술을 분리.
    
    ```java
    @RestController
    public class StudentController {
        private final StudentUseCase useCase;
    
        public StudentController(StudentUseCase useCase) {
            this.useCase = useCase;
        }
    
        @GetMapping("/students/{id}")
        public StudentSummary getStudentSummary(@PathVariable Long id) {
            return useCase.findStudentSummary(id);
        }
    }
    ```
    

## 6. 결론

- **Spring/JPA**: 엔티티 프로젝션은 `Repository`에서 직접 처리되며, 구현이 간단하지만 JPA에 강하게 의존합니다.
- **클린 아키텍처**: 도메인 중심 설계로 기술 독립성을 보장하며, 프로젝션은 어댑터에서 처리되어 유연성과 테스트 용이성을 높입니다.
- **공통점**: 두 접근 모두 성능 최적화와 데이터 간소화를 목표로 하며, JPA의 JPQL, 인터페이스, DTO를 활용.
- **선택 기준**:
    - 소규모 프로젝트: Spring/JPA의 단순한 구현이 적합.
    - 대규모/장기 프로젝트: 클린 아키텍처로 기술 독립성과 유지보수성 강화.

## 7. 추가 학습 자료

- Spring Data JPA 공식 문서: [spring.io/projects/spring-data-jpa](https://spring.io/projects/spring-data-jpa)
- 클린 아키텍처: Robert C. Martin, _Clean Architecture_ (책)
- 헥사고날 아키텍처: [alistair.cockburn.us/Hexagonal+architecture](https://alistair.cockburn.us/Hexagonal+architecture)