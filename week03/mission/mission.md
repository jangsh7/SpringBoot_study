# API 명세서

## 공통

- Base URL: `https://api.example.com/v1`
- Header
    - `Content-Type: application/json`
    - `Authorization: Bearer {JWT_TOKEN}` (로그인 이후 필요한 API에 한함)

---

## 1. 회원가입

### Endpoint

`POST /auth/signup`

### Request Body

```json
{
  "email": "user@example.com",
  "password": "password123",
  "name": "홍길동"
}
```

### Response

```json
{
  "userId": 1,
  "email": "user@example.com",
  "name": "홍길동",
  "createdAt": "2025-09-30T12:34:56"
}
```

- **이유:**
    - 신규 유저의 계정 생성을 위한 엔드포인트라서 `POST` 방식이 적합하다.
    - Request Body에 `email`, `password`, `name`을 넣은 이유는 필수 계정 식별 정보이기 때문이다.

---

## 2. 로그인

### Endpoint

`POST /auth/login`

### Request Body

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

### Response

```json
{
  "token": "jwt.access.token",
  "userId": 1
}
```

- **이유:**
    - 보안 상 로그인 시도는 서버에 인증을 요청하는 작업이므로 `POST`가 적절하다.
    - 응답으로 `JWT token`을 반환하도록 한 이유는 이후 API 호출에서 인증을 위해 Header에 넣을 수 있기 때문이다.

---

## 3. 홈 화면 (받은 미션 조회)

### Endpoint

`GET /missions/received`

### Headers

- `Authorization: Bearer {JWT_TOKEN}`

### Query String

- `status` (optional) : `in-progress`, `completed`

  (기본값: 전체 조회)


### Response

```json
[
  {
    "missionId": 101,
    "title": "김가네 방문하기",
    "status": "in-progress",
    "deadline": "2025-10-05"
  }
]
```

- **이유:**
    - 홈 화면은 내가 받은 미션만 보여주므로 `/received`라는 하위 리소스로 구분했다.
    - `GET` 방식은 데이터 조회에 사용되며, 추가적인 Body가 필요하지 않는다.
    - Query String(`status`)을 둔 이유는 "전체/진행중/완료"를 필터링할 수 있게 하기 위함이다.

---

## 4. 미션 목록 조회

### Endpoint

`GET /missions`

### Headers

- `Authorization: Bearer {JWT_TOKEN}`

### Query String

- `status=in-progress` → 진행중 미션
- `status=completed` → 완료 미션

### Response

```json
[
  {
    "missionId": 102,
    "title": "김말국 맛집 방문하기",
    "status": "completed",
    "completedAt": "2025-09-29"
  }
]
```

- **이유:**
    - 전체 미션 내역(진행 중/완료)을 조회하는 기능이라 `GET` 사용.
    - `status` query를 둔 이유는 클라이언트가 필요한 상태만 필터링해서 가져올 수 있도록 하기 위함이다.

---

## 5. 미션 성공 처리

### Endpoint

`POST /missions/{missionId}/complete`

### Path Variable

- `missionId`: 완료할 미션 ID

### Headers

- `Authorization: Bearer {JWT_TOKEN}`

### Response

```json
{
  "missionId": 102,
  "status": "completed",
  "completedAt": "2025-09-30T20:15:00"
}
```

- **이유:**
    - 단순 상태 변경이 아니라 "행위(Action)"이므로 `complete`라는 명시적인 action 엔드포인트로 설계했다.
    - `PATCH` 대신 `POST`를 사용한 이유는 행위 호출 성격이 강하고, "완료 처리"라는 이벤트를 발생시키기 때문이다.

---

## 6. 리뷰 작성 (마이 페이지 / 미션 리뷰)

### Endpoint

`POST /missions/{missionId}/reviews`

### Path Variable

- `missionId`: 리뷰할 미션 ID

### Headers

- `Authorization: Bearer {JWT_TOKEN}`

### Request Body

```json
{
  "rating": 5,
  "content": "김말국 맛있어요"
}
```

### Response

```json
{
  "reviewId": 301,
  "missionId": 102,
  "rating": 5,
  "content": "김말국 맛있어요",
  "createdAt": "2025-09-30T20:20:00"
}
```

- **이유:**
    - 리뷰는 새로운 리소스를 생성하는 작업이라 `POST`를 사용했다.
    - `/missions/{id}/reviews`로 설계한 이유는 리뷰가 미션에 종속된 하위 리소스이기 때문이다.
    - Request Body에 `rating`과 `content`만 포함시킨 이유는 리뷰 작성에 꼭 필요한 최소한의 정보이기 때문이다.