## 1. DB Join이란?

- 둘 이상의 테이블을 공통 컬럼 기준으로 묶어 한 번에 조회하는 기능이다.
- 즉, 테이블에 흩어져 있는 관련 데이터를 연결해 필요한 결과를 한 질의에서 가져올 수 있다.

## 2. Join 종류들

#### 2-1. (INNER) JOIN

- 조인하는 테이블의 ON 절의 조건이 일치하는 결과만 출력
- 표준 SQL과는 달리 MySQL에서는 JOIN, INNER JOIN, CROSS JOIN이 모두 같은 의미로 사용된다.

```sql
select u.userid, name
from usertbl as u inner join buytbl as b
on u.userid=b.userid
where u.userid="111" // join을 완료하고 그다음 조건을 따진다.
```

- 단순히 from 절에 콤마 쓰면 inner join 으로 치부된다.

```sql
select u.userid, name
from usertbl u, buytbl b
where u.userid=b.userid and u.userid="111"
```

### OUTER JOIN

- OUTER JOIN은 조인하는 테이블의 ON 절의 조건 중 한쪽의 데이터를 모두 가져온다.
- OUTER JOIN은 LEFT OUTER JOIN, RIGHT OUTER JOIN, FULL OUTER JOIN으로 분류할 수 있다.

#### 2-2. LEFT JOIN

- 두 테이블이 있는 경우, 첫 번째 테이블을 기준으로 두 번째 테이블을 조합하는 JOIN이다.

```sql
// 예) 3학년 학생의 이름과 지도교수명을 출력하라.
// 단, 지도교수가 지정되지 않은 학생도 출력되게 하라.
SELECT STUDENT.NAME, PROFESSOR.NAME
FROM STUDENT LEFT OUTER JOIN PROFESSOR // STUDENT를 기준으로 왼쪽 조인
ON STUDENT.PID = PROFESSOR.ID
WHERE GRADE = 3
```

#### 2-3. RIGHT JOIN

- 두 테이블이 있을 경우, 두 번째 테이블을 기준으로 첫 번째 테이블을 조합하는 JOIN이다.

```sql
// 예) 3학년 학생의 이름과 지도교수명을 출력하라.
// 단, 지도교수가 지정되지 않은 학생도 출력되게 하라.
SELECT STUDENT.NAME, PROFESSOR.NAME
FROM STUDENT RIGHT OUTER JOIN PROFESSOR // PROFESSOR를 기준으로 오른쪽 조인
ON STUDENT.PID = PROFESSOR.ID
WHERE GRADE = 3
```

#### 2-4. FULL OUTER JOIN

- LEFT OUTER JOIN을 거의 대부분 사용하여, FULL OUTER JOIN은 성능상 거의 사용하지 않는다.
```sql
select *
from topic FULL OUTER JOIN autor
on topic.auther_id = authoer.id
```

## 3. 트랜잭션이란?

- DB 트랜잭션(Transaction)은 **데이터베이스에서 하나의 작업 단위로 묶인 여러 SQL 명령을 모두 성공하거나 모두 실패시키는 기능**이다.
- 즉, 여러 쿼리를 하나의 불가분한 처리 단위로 묶어 데이터의 일관성과 무결성을 보장한다.

트랜잭션의 4대 특징(ACID)은 다음과 같다.
- **Atomicity(원자성)** : 모든 작업이 전부 수행되거나 전부 취소되어야 한다.
- **Consistency(일관성)** : 트랜잭션 전후로 데이터가 규칙과 제약 조건을 항상 만족해야 한다.
- **Isolation(격리성)** : 동시에 실행되는 트랜잭션이 서로 간섭하지 않도록 보호한다.
- **Durability(지속성)** : 트랜잭션이 성공적으로 완료되면 결과가 영구히 저장된다.

예를 들어 계좌 이체에서
- _① 출금 → ② 입금_ 이 두 쿼리를 하나의 트랜잭션으로 처리하면
  출금만 되고 입금이 실패하는 상황을 막을 수 있다.

정리하면, 트랜잭션은 데이터를 안전하게 처리하기 위한 최소 작업 단위이다.

## 4. Join on 과 where의 차이

#### 4-1. JOIN ON

- 역할 : **두(또는 그 이상) 테이블을 어떻게 연결할지**를 지정한다.
- 적용 시점 : 테이블을 합칠 때 먼저 적용되어, 어떤 행끼리 묶을지 결정한다.
- 예시

```sql
SELECT *
FROM 주문 o
JOIN 고객 c
  ON o.customer_id = c.customer_id;
```

→ `o.customer_id = c.customer_id` 조건으로 **조인 대상 행**을 결정한다.

#### 4-2. WHERE

- 역할 : **이미 JOIN으로 묶인 결과 집합에서** 필터링을 수행한다.
- 적용 시점 : 조인 후 생성된 결과에 조건을 걸어 추가로 줄인다.
- 예시

```sql
SELECT *
FROM 주문 o
JOIN 고객 c
  ON o.customer_id = c.customer_id
WHERE c.city = 'Seoul';
```

→ 먼저 `고객–주문`을 합친 뒤, 그 결과에서 `Seoul`에 사는 고객만 남긴다.

#### 4-3. 차이점 비교하기

|  구분  |  JOIN ON  |  WHERE  |
| :---: | :---: | :---: |
|   주목적   |   테이블 간 연결 조건   |   결과 행 필터링   |
|   실행 단계   |   테이블 합치기 이전   |   합친 뒤 이후   |
|   영향   |   조인 방식(Inner/Outer) 자체를 결정   |   결과 집합을 제한   |
|   주의   |   OUTER JOIN에서 WHERE를 잘못 쓰면 NULL 행이 제거되어 사실상 INNER JOIN처럼 될 수 있다   |      |

- **ON** : “테이블을 이렇게 묶어라.”
- **WHERE** : “묶인 결과 중 이 조건만 보여라.”
