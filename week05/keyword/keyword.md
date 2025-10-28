## 지연 로딩(LAZY) vs 즉시 로딩(EAGER)

- **지연 로딩 (Lazy Loading)**
    - 연관된 엔티티를 **실제로 사용할 때 쿼리로 불러오는 방식**
    - 예: `@ManyToOne(fetch = FetchType.LAZY)`
    - 필요할 때 SELECT 쿼리가 발생하므로 **성능 최적화**에 유리함
    - 하지만 영속성 컨텍스트가 닫힌 후 접근하면 `LazyInitializationException` 발생 가능

- **즉시 로딩 (Eager Loading)**
    - 연관된 엔티티를 **즉시 함께 로드하는 방식**
    - 예: `@OneToOne(fetch = FetchType.EAGER)`
    - JOIN 쿼리를 통해 즉시 데이터를 가져오지만, 사용하지 않는 데이터까지 조회되어 **N+1 문제**나 **불필요한 쿼리 증가** 초래 가능

> **결론:** 대부분의 경우 `LAZY`를 기본으로 설정하고, 필요한 경우에만 `fetch join` 등으로 즉시 로딩을 제어한다.

---

## JPQL (Java Persistence Query Language)

- JPA가 제공하는 **객체 지향 쿼리 언어**
- SQL과 비슷하지만 **테이블이 아닌 엔티티 객체를 대상으로 쿼리**
- 예시:
  ```java
  SELECT m FROM Member m WHERE m.age > 20
  ```
- SQL은 데이터베이스 테이블 기준이지만, JPQL은 엔티티 필드 및 객체 관계를 기반으로 동작한다.

- 특징:
  - 컴파일 시점 문법 검사 가능 (TypedQuery)
  - JOIN, GROUP BY, HAVING, ORDER BY 모두 지원
  - Spring Data JPA에서는 `@Query` 어노테이션을 통해 직접 작성 가능

---

## Fetch Join

- 연관된 엔티티를 한 번의 쿼리로 함께 조회하는 JPQL 문법
- N+1 문제 해결의 핵심 방법 중 하나
- 예시:
    ```java
        SELECT m FROM Member m JOIN FETCH m.team
    ```
- 위 쿼리는 Member와 Team을 한 번에 조인(fetch) 하여 가져온다.
- 즉, 이후 `m.getTeam()`을 호출해도 추가 쿼리가 발생하지 않는다.

> **주의점:**
> - 페이징(setFirstResult, setMaxResults)과 함께 사용 시 데이터 부풀림 가능
> - 컬렉션 fetch join(`@OneToMany`)은 1개만 사용 권장

---

## @EntityGraph

- 지연로딩 엔티티를 즉시로딩처럼 한 번에 조회하기 위한 기능
- SQL을 직접 작성하지 않고도 fetch join 효과를 낼 수 있다.
- 예시:
    ```java
    @EntityGraph(attributePaths = {"team"})
    @Query("SELECT m FROM Member m")
    List<Member> findAllWithTeam();
    ```
- JPA가 내부적으로 JOIN FETCH를 사용하여 team을 함께 가져온다.

> **장점:**
> - JPQL 없이도 fetch join처럼 동작
> - 공통적으로 자주 사용하는 조회 패턴을 재사용 가능

---

## commit vs flush

| 구분    | commit          | flush                             |
| ----- | --------------- | --------------------------------- |
| 시점    | 트랜잭션 종료 시       | 영속성 컨텍스트 → DB 동기화 시               |
| 역할    | DB에 실제로 트랜잭션 반영 | 변경 내용을 SQL로 DB에 반영하지만 커밋은 아님      |
| 호출 시점 | 트랜잭션 커밋 시 자동    | JPQL 실행 전, 수동 호출(`em.flush()`) 가능 |

> **정리**
> - `flush()`는 쓰기 지연 저장소의 SQL을 DB에 반영하지만, 트랜잭션은 여전히 진행 중이다.
> - `commit()`은 flush 후 트랜잭션을 종료하며 변경 사항이 확정된다.

---

## QueryDSL & OpenFeign의 QueryDSL

- **QueryDSL**
  - 타입 안정성이 보장되는 동적 쿼리 빌더
  - 문자열 기반이 아닌 코드 기반으로 JPQL 작성
  - **예시:**
  ```java
    QMember m = QMember.member;
    List<Member> result = queryFactory
    .selectFrom(m)
    .where(m.age.gt(20))
    .fetch();
    ```
  - **장점:** IDE 자동완성, 리팩토링 용이, 동적 조건 처리에 강함

- **OpenFeign의 QueryDSL**
  - OpenFeign에서 사용되는 "QueryDSL"은 JPA용이 아닌, HTTP 요청 파라미터(Query Parameter) 생성 DSL이다.
  - API 호출 시 query string을 동적 조합하기 위한 도구이며, JPA의 QueryDSL과는 목적이 다르다.

---

## N+1 문제 및 해결 방안

**문제 개념**

- 한 번의 조회로 N개의 연관 데이터를 가져올 때, 각 연관 엔티티마다 추가 쿼리가 N번 발생하는 문제.
- → 성능 급격히 저하

**원인**
- 즉시 로딩(EAGER) 또는 지연 로딩(LAZY) + 반복 접근

**해결 방법**

1. **Fetch Join 사용**
```java
SELECT m FROM Member m JOIN FETCH m.team
```

2. **@EntityGraph 사용**
```java
@EntityGraph(attributePaths = {"team"})
List<Member> findAll();
```

3. **Batch Size 설정**
```java
@BatchSize(size = 100)
```

4. **DTO Projection (필요 데이터만 조회)**
- JPQL 또는 QueryDSL로 필요한 컬럼만 선택

> 실무에서는 상황에 맞게 Fetch Join + BatchSize 조합을 자주 사용한다.

---

## 영속 상태의 종류

| 상태                  | 설명                                      |
| ------------------- | --------------------------------------- |
| 비영속 (New/Transient) | 영속성 컨텍스트에 저장되지 않은 상태 (`new Member()`)   |
| 영속 (Managed)        | `em.persist()`로 컨텍스트에 저장되어 1차 캐시에 관리됨   |
| 준영속 (Detached)      | 컨텍스트에서 분리되어 변경 감지 불가능 (`em.detach()` 등) |
| 삭제 (Removed)        | 삭제 예약 상태 (`em.remove()`)                |

- **변경 감지(Dirty Checking)**
  - 영속 상태의 엔티티가 변경되면, 트랜잭션 커밋 시점에 자동으로 UPDATE 쿼리 수행

---

## 5주차 시니어 미션
[내 벨로그 바로가기](https://velog.io/@jangsh7/JPA-SQL-%EB%A1%9C%EA%B7%B8-%EB%B6%84%EC%84%9D%EA%B3%BC-QueryDSL-%EB%A6%AC%ED%8C%A9%ED%86%A0%EB%A7%81%EC%9D%84-%ED%86%B5%ED%95%9C-%EC%84%B1%EB%8A%A5-%ED%96%A5%EC%83%81)