## 1. Spring Security 기본 설정
### 1.1 의존성 추가

build.gradle 파일에 보안 및 테스트를 위한 의존성을 추가합니다.

### 1.2 SecurityConfig 클래스 구현
보안 정책을 정의하기 위해 `@EnableWebSecurity` 어노테이션을 사용한 설정 클래스를 생성합니다.
- Filter Chain 설정: HTTP 요청에 대한 접근 권한을 정의합니다.
- PermitAll: Swagger UI, API 문서 등 공통적으로 허용할 URI를 allowUris 배열로 관리하여 가독성을 높였습니다.
- Role 기반 접근 제어: /admin/ 경로는 ADMIN 역할을 가진 사용자만 접근 가능하도록 제한합니다.

## 2. 회원가입 및 비밀번호 보안
### 2.1 BCrypt를 활용한 비밀번호 암호화
보안 강화를 위해 평문 비밀번호를 그대로 저장하지 않고, BCryptPasswordEncoder를 사용하여 단방향 해싱 및 솔팅처리를 수행합니다.

### 2.2 엔티티 및 DTO 확장
Member 엔티티에 email, password, role 필드를 추가합니다.

사용자 역할은 Role Enum(ROLE_ADMIN, ROLE_USER)으로 정의합니다.

## 3. JWT 기반 인증 구현 (실습 2)
### 3.1 JWT 설정 및 유틸리티 제작

application.yml에 시크릿 키와 만료 시간(4시간)을 설정하고, JwtUtil 클래스를 통해 토큰 생성, 검증, 정보 추출 로직을 구현합니다.

### 3.2 커스텀 필터 (JwtAuthFilter)

OncePerRequestFilter를 상속받아 모든 요청마다 JWT를 검사하는 필터를 구현합니다.

- 헤더에서 토큰을 추출하고 Bearer 접두사를 확인합니다.
- 토큰 유효성 검증 후, CustomUserDetailsService를 통해 사용자 정보를 조회합니다.
- 인증이 완료되면 SecurityContextHolder에 인증 객체를 저장합니다.

## 4. 예외 처리 통일
Spring Security 기본 에러 응답(403 Forbidden 등)을 프로젝트의 공통 응답 형식(ApiResponse)으로 통일하기 위해 AuthenticationEntryPoint를 커스텀 구현하였습니다.

- 인증 실패 시 GeneralErrorCode.UNAUTHORIZED를 반환하도록 설정하여 클라이언트가 일관된 에러 메시지를 받을 수 있게 조치했습니다.


## 5. 결과

DB에 암호화된 비밀번호와 ROLE_USER 권한이 정상 저장됨을 확인했습니다.

USER 권한 사용자가 관리자 페이지 접근 시 403 에러가 발생하며, ADMIN으로 변경 후 접근 시 정상 동작함을 확인했습니다.

로그인 시 발급된 accessToken을 Swagger의 Authorize 헤더에 담아 보낼 경우, 인증이 필요한 API 호출이 성공함을 확인했습니다.