## 1. @RestControllerAdvice

- 전역 예외/응답 바인딩을 담당하는 컴포넌트
- `@ControllerAdvice + @ResponseBody`의 합성 애너테이션

- **주요 용도**
    - `@ExceptionHandler`: 전역 예외 처리
    - `@InitBinder`: 바인딩/검증기 설정
    - `@ModelAttribute`: 공통 모델 속성 주입 (REST에선 드뭄)

### 예시
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ApiError> handleIllegalArg(IllegalArgumentException e) {
        return ResponseEntity
            .badRequest()
            .body(new ApiError("INVALID_ARGUMENT", e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .findFirst().orElse("Validation error");
        return ResponseEntity.badRequest().body(new ApiError("VALIDATION_ERROR", msg));
    }

    public record ApiError(String code, String message) {}
}
```

- **계층화**: 도메인별 Advice 분리 가능 (@Order로 우선순위)

- **패키지 범위 제한**: @RestControllerAdvice("com.example.api")

- **로그 일관화**: 예외 처리부에서 공통 로거 사용

---

## 2. Lombok

- 보일러플레이트(게터/세터/생성자/빌더/로그) 제거를 위한 애너테이션 도구

- **자주 쓰는 애너테이션**

    - `@Getter`, `@Setter`

    - `@RequiredArgsConstructor`, `@AllArgsConstructor`, `@NoArgsConstructor`

    - `@Builder` (불변 DTO/명세적 생성 시 유용)

    - `@Value` (모든 필드 private final, 불변 객체)

    - `@Data` (주의: equals/hashCode/toString까지 생성 — 엔티티엔 지양)

    - `@Slf4j` (로그ger 주입)

### 장점

- 코드량 감소, 가독성↑, 빌더로 생성 안정성↑

### 주의사항

- 팀/빌드 환경 의존 (IDE/플러그인 필요)

- JPA 엔티티에 `@Data` 지양(지연로딩 연쇄 호출, equals/hashCode 문제)

- `@Builder` + JPA 엔티티는 생성자 남발/불변성 혼동 주의 → 보통 DTO에만 사용 권장

- 레코드(Java 16+)가 가능한 경우 Lombok 의존도 줄이기 검토

---

## 3. DTO 형식: public static class VS record 비교
### 1) public static class DTO (정적 중첩 클래스 DTO)
```java
public class PostApi {
    @Getter @Setter
    public static class CreateRequest {
        private String title;
        private String content;
    }

    @Getter
    public static class CreateResponse {
        private final Long id;
        private final String title;

        @Builder
        public CreateResponse(Long id, String title) {
            this.id = id;
            this.title = title;
        }
    }
}
```
- **장점**

    - 구버전 JDK/프레임워크와 호환성 최고

    - 상태 변경이 필요한 DTO에 유리(세터/검증 로직 삽입 용이)

    - 중첩 구조로 API 스펙 그룹핑 용이 (PostApi.CreateRequest)

- **단점**

    - 보일러플레이트 발생(Lombok에 의존적)

    - 불변/동등성 보장이 약함(개발자 규율 필요)

    - 생성자/세터 혼용 시 객체 일관성 저하 가능

- **쓰임새**

    - 검증/정규화(Setter에서 트리밍 등)가 필요한 입력 DTO

    - 점진적 마이그레이션 환경(JDK 11/레코드 미도입 조직)

### 2) record DTO (불변값 객체)
```java
public record CreatePostRequest(
    @NotBlank String title,
    @NotBlank String content
) {}

public record CreatePostResponse(Long id, String title) {}
```

- **장점**

    - 불변 + 명세적(필드=상태, 생성자=단 하나)

    - equals/hashCode/toString 자동, 의도 명확

    - 불변이라 스레드 세이프, 유지보수 용이

    - Jackson 2.12+ 기본 지원(생성자 바인딩)

- **단점**

    - 필드 변경 불가 → 점진적 바인딩/수정엔 부적합

    - 커스텀 무인자 생성자/세터 없음

    - 일부 라이브러리/구버전 Jackson에서 추가 설정 필요할 수 있음

- **쓰임새**

    - 응답 DTO(읽기 전용), 쿼리 결과 매핑, 값 객체(Value Object)


### 요약

- **요청(Request) DTO**

    - 간단 + 불변 원하면 → `record`

    - 입력 후 정규화/치환/세팅 로직 필요 → `public static class` + 명시 생성자/세터 또는 정적 팩토리

- **응답(Response) DTO**

    - 기본은 `record` 권장(불변/표현력/가독성)

    - 레거시/MapStruct 커스텀/라이브러리 제약 → `public static class` 유지

- **팀/환경**

    - JDK 16+ 표준화 되어 있으면 → `record` 중심, Lombok 최소화

    - 구버전/혼합 환경 → `public static class` + Lombok로 생산성 확보

---

## 마무리

전역 예외 처리는 @RestControllerAdvice로 일관된 에러 응답을 제공한다.

Lombok은 보일러플레이트를 줄이되, 엔티티/핵심 도메인에는 남용 금지한다.

DTO는 응답용 → record, 요청용 → 단순하면 record / 로직 필요하면 public static class로 구분한다.

---