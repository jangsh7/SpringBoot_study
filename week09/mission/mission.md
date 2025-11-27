# Mission – API & Paging 구현 정리

## 1. API 설계
### 1-1. 공통 규칙 – 페이징

- page 쿼리 스트링 (1 기반)

    - 프론트 요청 예: `GET /api/reviews/me?page=1`

- 내부에서는 0 기반으로 변환하여 PageRequest 생성

```java
int pageIndex = page - 1; // page는 1 이상이 검증된 상태
PageRequest pageRequest = PageRequest.of(pageIndex, 10, Sort.by("createdAt").descending());
```

### 1-2. 내가 작성한 리뷰 목록

**Endpoint**

- `GET /api/reviews/me?page={page}`

**기능**

- 로그인한 사용자(회원)의 작성 리뷰 목록을 최신순으로 페이징 조회

- 한 페이지당 10개

**주요 파라미터**

- `page` (query, required, 1 이상, `@ValidPage` 적용)

**Response (요약)**
```json
{
    "content": [
        {
            "reviewId": 1,
            "storeName": "반이학생마라탕마라반",
            "rating": 4.5,
            "comment": "음 너무 맛있어요...",
            "visitedDate": "2025-05-14",
            "createdAt": "2025-05-15T10:00:00"
        }
    ],
    "page": 1,
    "size": 10,
    "totalElements": 123,
    "totalPages": 13
}
```

**Converter 예시 (Stream + Builder)**

```java
public List<MyReviewResponse> toMyReviewResponses(List<Review> reviews) {
  return reviews.stream()
  .map(review -> MyReviewResponse.builder()
  .reviewId(review.getId())
  .storeName(review.getStore().getName())
  .rating(review.getRating())
  .comment(review.getComment())
  .visitedDate(review.getVisitedDate())
  .createdAt(review.getCreatedAt())
  .build()
  )
  .toList();
}
```

### 1-3. 특정 가게의 미션 목록

Endpoint

- `GET /api/stores/{storeId}/missions?page={page}`

기능

- 특정 가게에 대해 등록된 미션 목록을 페이징 조회

- 예: “12,000원 이상 식사 후 리뷰 남기기 (500P)”

주요 파라미터

- storeId (path)

- page (query, @ValidPage)

Response (요약)
```json
{
  "content": [
    {
      "missionId": 10,
      "storeId": 3,
      "storeName": "가게이름a",
      "rewardPoint": 500,
      "conditionText": "12,000원 이상의 식사를 하세요!",
      "status": "AVAILABLE"
    }
  ],
  "page": 1,
  "size": 10,
  "totalElements": 27,
  "totalPages": 3
}
```

### 1-4. 내가 진행중인 미션 목록

Endpoint

- `GET /api/missions/in-progress?page={page}`

기능

- 로그인한 사용자가 진행중(IN_PROGRESS) 상태인 미션 목록 조회

- 하단 탭 “미션” 목록 화면용

주요 파라미터

- page (query, @ValidPage)

Response (요약)
```json
{
  "content": [
    {
      "missionId": 11,
      "storeName": "가게이름a",
      "rewardPoint": 500,
      "status": "IN_PROGRESS",
      "conditionText": "12,000원 이상의 식사를 하세요!",
      "expiredAt": "2025-06-30T23:59:59"
    }
  ],
  "page": 1,
  "size": 10,
  "totalElements": 5,
  "totalPages": 1
}
```

### 1-5. 진행중인 미션 진행 완료로 변경

Endpoint

- `PATCH /api/missions/{missionId}/complete?page={page}`

기능

- 로그인한 사용자가 가진 진행중(IN_PROGRESS) 미션인지 검증

- 상태를 COMPLETED로 변경

- 변경된 미션의 상세 정보와, 변경 후 기준의 **진행중 미션 목록(해당 page)** 를 함께 반환

  - “진행완료” 버튼 클릭 후 리스트를 재조회하는 플로우를 한 번에 처리

요청

- missionId (path)

- page (query, @ValidPage)

Response (요약)
```json
{
  "updatedMission": {
    "missionId": 11,
    "storeName": "가게이름a",
    "rewardPoint": 500,
    "status": "COMPLETED",
    "completedAt": "2025-05-20T12:34:56"
  },
  "inProgressMissions": {
    "content": [
    ],
    "page": 1,
    "size": 10,
    "totalElements": 4,
    "totalPages": 1
  }
}
```

---

## 2. 커스텀 page 어노테이션 & 예외 처리
### 2-1. @ValidPage 커스텀 어노테이션

- 역할: 쿼리 스트링으로 들어오는 page 값이 1 미만인 경우 예외 발생

```java
@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint(validatedBy = PageValidator.class)
public @interface ValidPage {
    String message() default "page 파라미터는 1 이상이어야 합니다.";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### 2-2. PageValidator
```java
public class PageValidator implements ConstraintValidator<ValidPage, Integer> {

    @Override
    public boolean isValid(Integer value, ConstraintValidatorContext context) {
        if (value == null) return false;  // null 도 에러 처리
        return value >= 1;
    }
}
```

### 2-3. RestControllerAdvice 연동

- MethodArgumentNotValidException, ConstraintViolationException 등 처리

- 공통 에러 응답 포맷 예시:

```json
{
  "code": "INVALID_PAGE",
  "message": "page 파라미터는 1 이상이어야 합니다.",
  "status": 400,
  "timestamp": "2025-05-20T12:00:00"
}
```

---

## 3. Builder 패턴 사용

- DTO, Entity에서 `@Builder` 사용

- 예시 – MissionResponse

```java
@Getter
@Builder
public class MissionResponse {
    private Long missionId;
    private Long storeId;
    private String storeName;
    private int rewardPoint;
    private String conditionText;
    private MissionStatus status;
}
```
- Service/Converter에서 MissionResponse.builder() 형태로 일관되게 생성

- 필드가 많아도 가독성이 좋고, 선택적인 필드 세팅이 용이함

---

## 4. 마무리

- page 파라미터를 1 기반으로 받되, 내부에서는 0 기반으로 변환하는 로직을 명확히 분리해서 구현

- 커스텀 어노테이션 + RestControllerAdvice를 통해 입력 검증과 에러 응답을 표준화

- 리스트 변환 시 for문 대신 Stream + Builder 패턴으로 구현하여 코드 가독성과 요구사항을 동시에 만족

- 상태 변경 API(진행중 -> 진행완료)는 상태 변경 + 재조회를 한 번의 호출로 처리해, 실제 앱 화면 플로우와 맞게 설계