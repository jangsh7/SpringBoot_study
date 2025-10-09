## 1. 계층형 구조 vs 도메인형 구조

### 개념
- **계층형 구조 (Layered Architecture)**
  - `controller → service → repository` 처럼 기술 계층별로 패키지를 나누는 방식이다.
    - 예: `controller/`, `service/`, `repository/`, `dto/`, `entity/`
- **도메인형 구조 (Domain-(by-feature) Architecture)**
  - 기능(도메인)별로 묶는다. 각 도메인 폴더 안에 controller/service/repository가 함께 있다.
    - 예: `user/`, `order/`, `product/`(각 폴더 안에 `UserController`, `UserService` 등)

### 장단점 비교
| 구분 | 계층형 구조 | 도메인형 구조 |
|---|---|---|
| 장점 | 기술별 책임이 명확, 입문 난이도 낮음 | 변경 범위가 도메인 폴더에 국한, 기능 단위 확장/이동 용이, 대규모 팀에 유리 |
| 단점 | 기능 추가 시 여러 계층 디렉터리 동시 편집, 모듈화/마이크로서비스 전환 난해 | 설계 초기에 도메인 경계 고민 필요, 공통 코드 중복 가능 |
| 추천 상황 | 소규모, 학습/프로토타입, 단순 CRUD | 중/대규모, 기능 단위 배포·확장, 팀별 도메인 책임 분리 |

### 실전 팁
- **하이브리드**: `global(common)/config/infra`는 공용, 나머지는 `domain/*`로 기능별 분리
- 도메인 폴더 예시
```
  src/main/java/com/example/app
  ├─ domain
  │  ├─ user
  │  │  ├─ api (controller)
  │  │  ├─ application (service/usecase)
  │  │  ├─ domain (entity/aggregate/repository-interface)
  │  │  └─ infra (jpa-adapter, repository-impl)
  │  └─ quest
  └─ global
  ├─ config
  ├─ error
  └─ util
```

- **패키지 의존 규칙**: `api → application → domain` 방향으로만 참조(역참조 금지). `infra`는 `domain`을 구현.

---

## 2. JPA

### 엔티티 기본
```java
@Entity
@Table(name = "users")
@Getter @Setter
public class User {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, length = 50)
  private String name;

  // 양방향 연관관계 시 주인(owner) 쪽에서 FK 관리
  @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<UserQuest> userQuests = new ArrayList<>();
}
```

### 연관관계 매핑 베스트 프랙티스
- 다대다(@ManyToMany) **지양** → **조인 엔티티**(예: `UserQuest`)로 풀기
- 지연로딩(LAZY) 기본, 즉시(EAGER) 지양
- `mappedBy`로 **연관관계의 주인**을 명확히, 주인만 FK 갱신
- 컬렉션은 `new ArrayList<>()`로 즉시 초기화(NullPointer 방지)

### 트랜잭션 & 영속성 컨텍스트
- 같은 트랜잭션 내에서 **1차 캐시**와 **더티 체킹**으로 변경 자동 반영
- 서비스 계층에 `@Transactional` 기본, **쓰기**는 꼭 트랜잭션 안에서

### Spring Data JPA Repository
```java
public interface UserRepository extends JpaRepository<User, Long> {
  Optional<User> findByName(String name);

  @EntityGraph(attributePaths = "userQuests") // N+1 회피 예시
  Optional<User> findWithUserQuestsById(Long id);
}
```

---

## 3. N+1 문제

- 한 번의 조회(N=1) 후, 연관된 엔티티를 **컬렉션/프록시 지연로딩**이 **행마다 추가 쿼리(N)**로 가져와 **총 N+1 번**의 SQL이 발생하는 현상.

### 재현 예시
```java
// users 100명 조회 (1쿼리)
// 각 user.getUserQuests() 접근 시 user_id별로 추가 쿼리 100회 → 총 101 쿼리
List<User> users = userRepository.findAll();
users.forEach(u -> u.getUserQuests().size());
```

### 해결 전략 (우선순위별)
1) **JPQL fetch join**
```java
@Query("select u from User u join fetch u.userQuests")
List<User> findAllWithQuests();
```

- 장점: 한 번에 로딩, 단순/효율
- 주의: 페이징 불가(컬렉션 fetch join), 데이터 중복(row 중복) 주의 → distinct, DTO 매핑 고려

2) **@EntityGraph**
```java
@EntityGraph(attributePaths = {"userQuests", "userQuests.quest"})
List<User> findAll();
```

- 장점: 메서드 선언만으로 로딩 계획 지정
- 주의: 조인 전략은 구현체(Hibernate)가 결정, 복잡한 그래프는 과다 조인 가능

3) **배치 사이즈 (IN 쿼리 뭉치기)**
```java
# application.yml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 100
```

- 효과: 지연로딩 시 FK를 모아 IN으로 묶어 **N 쿼리를 1~몇 개로 축소**
- 장점: 페이징과 병행 가능, 단일 쿼리 폭증 방지
- 주의: 너무 크게 잡으면 메모리/SQL 길이 부담

4) **DTO 프로젝션**

```java
@Query("select new com.example.api.UserSummary(u.id, u.name, q.title) " +
       "from User u join u.userQuests uq join uq.quest q")
List<UserSummary> fetchUserQuestSummaries();
```

- 화면용 데이터만 정확히 가져와 전송량 최소화

5) **캐시**(2차 캐시/Query Cache): 읽기 성격 강할 때 고려

### 탐지/확인
- Hibernate SQL 로그 활성화, p6spy 사용
- 특정 요청 시 쿼리 횟수/응답시간 모니터링(APM)

---

## 4. 기본 키 생성 전략

### JPA 전략 종류
- **GenerationType.IDENTITY**
    - DB의 AUTO_INCREMENT/IDENTITY 사용(MySQL 등)
    - 장점: 설정 간단
    - 단점: **영속성 컨텍스트에 flush 시점에 키 확정** → **JDBC batch insert 제약**(대량 insert 성능 불리)
- **GenerationType.SEQUENCE**
    - **시퀀스** 사용(PostgreSQL, Oracle, H2)
    - `@SequenceGenerator`로 시퀀스 이름/`allocationSize` 설정
    - 장점: **미리 식별자 할당** 가능 → 배치 insert/쓰기 성능 유리
    - 고급: Hi/Lo, pooled optimizer로 시퀀스 호출 최소화
- **GenerationType.TABLE**
    - 키 전용 테이블로 시퀀스 흉내
    - 장점: 모든 DB 호환
    - 단점: 성능 낮고 락 경합 가능(특별한 사유 없으면 비추천)
- **GenerationType.AUTO**
    - DB 방언에 따라 위 전략 중 자동 선택(명시적 선택 권장)

### 실전 설정 예시

**MySQL (IDENTITY)**
```java
@Entity
public class User {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
}
```

- 장점: 간단
- 주의: 대량 저장 성능이 중요하면 UUID/외부 키 생성기 고려

**PostgreSQL/Oracle (SEQUENCE)**
```java
@Entity
@SequenceGenerator(name = "user_seq_gen", sequenceName = "user_seq", allocationSize = 50)
public class User {
  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq_gen")
  private Long id;
}
```

- `allocationSize`를 50~100 등으로 올리면 시퀀스 호출 감소(성능 ↑)
- **장점**: 배치 insert, 멀티스레드 대량 처리에 유리

**UUID 키**
```java
@Entity
public class Document {
  @Id
  @GeneratedValue
  @JdbcTypeCode(SqlTypes.VARCHAR) // Hibernate 6
  private UUID id;
}
```

- 장점: 분산 시스템/샤딩 친화, 사전 키 생성 가능
- 단점: 인덱스/스토리지 비용 증가(특히 랜덤 UUID v4). v7(시간 기반) 사용 시 인덱스 친화 ↑

### 선택하는 법
- **쓰기 성능·배치 중요 + 시퀀스 지원 DB** → SEQUENCE (+ allocationSize)
- **MySQL/InnoDB 기반 단순 CRUD** → IDENTITY
- **분산·오프라인 키 발급 필요** → UUID (v7 권장)
- **모든 DB 호환**이 절대 필요 → TABLE(가능하면 피함)

---
