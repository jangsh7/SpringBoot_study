## 1. QueryDSL에서 Fetch Join 하는 법

### Fetch Join이란?
- **JPA N+1 문제 해결**을 위해 연관된 엔티티를 한 번에 조회하는 방법
- JPQL에서 `join fetch`와 동일한 기능을 QueryDSL로 수행

### 사용 방법
```java
List<Member> result = queryFactory
    .selectFrom(member)
    .join(member.team, team).fetchJoin()
    .fetch();
```

### 언제 필요할까
| 상황                                   | Fetch Join 필요 여부 |
| ------------------------------------ | ---------------- |
| Lazy 로딩 상태에서 연관 엔티티를 즉시 화면에 보여줘야 할 때 | 필요             |
| 데이터가 크고, 조회 빈도가 낮아 Lazy로 가져와도 되는 경우  | 비추천            |

### 한계점 & 주의점

- **페이징 시 사용 불가**: 컬렉션 Fetch Join은 Hibernate가 메모리에서 페이징 처리 → 성능 저하 위험
- ToOne 관계는 페이징 허용되지만 ToMany는 매우 비효율적

> 개인 의견:
> - Fetch Join은 만능 키처럼 남용하기보다, 조회용 쿼리(읽기 전용)에서 선택적으로 쓰는 게 좋다고 생각한다.
> - 특히 API 설계 시 DTO 매핑을 병행하면 Fetch Join 의존도가 줄어든다.

---

## 2. DTO 매핑 방식 (+ DTO 안에 DTO)

### DTO 매핑 방법 3가지
| 방식                                          | 특징                | 장점          | 단점                        |
| ------------------------------------------- | ----------------- | ----------- | ------------------------- |
| **Setter/생성자 기반 매핑**                        | select로 필요 필드만 조회 | 가장 많이 쓰임    | DTO가 QueryDSL 의존성 생길 수 있음 |
| **@QueryProjection**                        | DTO 생성자에 Q파일 생성   | 컴파일 시 타입 체크 | DTO가 QueryDSL에 종속         |
| **Projections (fields, constructor, bean)** | QueryDSL에서 제공     | 유연함, 종속성 적음 | 필드명/타입 실수하는 경우 런타임 오류     |

#### 예시: DTO안에 DTO (Nested DTO)
```java
public record MemberDto(
    Long id,
    String name,
    TeamDto team
) {
    public record TeamDto(Long id, String name) {}
}
```

#### QueryDSL로 매핑
```java
List<MemberDto> result = queryFactory
    .select(Projections.constructor(MemberDto.class,
        member.id,
        member.name,
        Projections.constructor(MemberDto.TeamDto.class,
            team.id,
            team.name
        )
    ))
    .from(member)
    .join(member.team, team)
    .fetch();
```

> DTO 안에 DTO를 쓰면 계층 구조를 깔끔하게 유지할 수 있어서 API 응답용 DTO 구조 설계에 매우 유용하다.
특히 REST API 명세와 Front와의 협업에서 구조가 명확해져서 추천한다.

---

## 3. 커스텀 페이지네이션

### 왜 커스텀 페이지네이션을 쓸까
- Spring Data JPA의 기본 Page<>는 기능은 좋지만 복잡한 쿼리에서 성능 저하 발생 가능
- 별도 count 쿼리가 필요한 경우가 많고, count 쿼리 최적화 필요

### 기본 구조

```java
List<Member> content = queryFactory
    .selectFrom(member)
    .offset(pageable.getOffset())
    .limit(pageable.getPageSize())
    .fetch();

long count = queryFactory
    .select(member.count())
    .from(member)
    .fetchOne();

return new PageImpl<>(content, pageable, count);
```
### 성능 최적화 팁

- count 쿼리는 join 제거 버전으로 수행하여 속도 향상
- content 쿼리와 count 쿼리 분리

> 실무에서는 PageImpl 보다는, Slice로 충분한 경우 Slice가 사용자 경험과 성능 모두 좋다.
(무한 스크롤 → Slice 권장)

---

## 4. transform - groupBy

### 역할
- QueryDSL의 transform().groupBy()는 DB 결과를 메모리 상에서 그룹화하고 묶어서 구조화하는 기능

### 사용 예시
```java
Map<Long, MemberGroupDto> result = queryFactory
    .from(team)
    .join(team.members, member)
    .transform(
        groupBy(team.id).as(
            Projections.constructor(MemberGroupDto.class,
                team.name,
                list(member.name)
            )
        )
    );
```

### 언제 사용할까

- 1:N 관계를 DTO로 그룹핑하여 한 번에 변환하고 싶을 때
- DB에서는 flatten된 row가 나오지만, 애플리케이션에선 묶어야 할 때

> transform(groupBy)는 편리하지만, 내부적으로 Java 메모리에서 grouping이 이뤄지기 때문에 데이터가 매우 많을 때는 비효율적이다.
데이터 양이 크면 차라리 DB에서 group_concat, JSON aggregation 쓰는게 낫다고 본다.

---

## 5. ORDER BY NULL

### 개념
- 정렬 기준 없이 데이터를 가져오고 싶을 때 명시적으로 사용
- DB가 불필요한 정렬 작업을 수행하지 않도록 제어

### 언제 사용할까
- 불필요한 ORDER BY가 붙어 성능 저하될 때
- 이미 index-covered order로 충분할 때
- 데이터 순서를 아예 보장할 필요 없을 때

### QueryDSL 적용
```java
orderBy(Expressions.nullExpression().asc())
```

> “ORDER BY NULL"은 잘 쓰면 성능 미세 최적화에 좋지만 과한 미세 튜닝은 지양하는 입장이다.
그러나 대용량 통계 데이터나 batch 조회에서는 분명 의미가 있다.

---

## 6. 마무리

- QueryDSL은 타입 안정성 + 직관적 쿼리 빌딩이라는 큰 장점이 있지만, 특정 기능(특히 transform, fetch join)은 무작정 쓰면 성능 리스크가 있다.
- DTO 분리와 QueryDSL 기반 조회는 확실히 유지보수를 수월하게 하고, 팀 협업 시 API 스펙이 명확해져서 좋다.
- 개인적으로는 QueryDSL은 "JPA의 한계를 보완해주는 쿼리 최적화 도구"로 받아들이는 것이 이상적이라고 생각한다.
즉, QueryDSL 자체를 목적화하지 말고 도구적 관점에서 선택적으로 활용해야 한다.