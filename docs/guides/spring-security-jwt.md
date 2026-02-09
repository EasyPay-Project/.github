# Spring Security + JWT 인증

## 핵심 개념

Spring Security는 **필터 체인(Filter Chain)** 기반으로 동작한다. HTTP 요청이 컨트롤러에 도달하기 전에 여러 필터를 순서대로 거치며, 각 필터에서 인증/인가/제한 등의 처리를 수행한다.

JWT(JSON Web Token)는 **Stateless 인증** 방식으로, 서버에 세션을 저장하지 않고 토큰 자체에 사용자 정보를 담는다. Access Token(단기)과 Refresh Token(장기)을 조합하여 보안과 편의성을 모두 확보한다.

## 프로젝트 적용

### 필터 체인 순서

```
HTTP 요청
  │
  ├─[1] RequestIdFilter (HIGHEST_PRECEDENCE)
  │     └─ X-Request-Id 헤더 → MDC 저장 (로깅용)
  │     └─ 없으면 UUID 8자리 자동 생성
  │
  ├─[2] JwtAuthenticationFilter (before UsernamePasswordAuthenticationFilter)
  │     └─ Authorization: Bearer {token} 또는 ?token={token} (SSE용)
  │     └─ 토큰 검증 → SecurityContext에 인증 정보 저장
  │     └─ Principal = userId (Long), Authority = "ROLE_" + role
  │
  ├─[3] ApiRateLimitFilter (after JwtAuthenticationFilter)
  │     └─ 인증 사용자: 100 req/min (api:user:{userId})
  │     └─ 미인증 사용자: 20 req/min (api:ip:{clientIp})
  │     └─ 초과 시: 429 Too Many Requests
  │
  └─[4] Controller
```

### JWT 토큰 구조

**Access Token** (30분):
```json
{
  "sub": "123",           // userId (subject)
  "email": "user@easypay.com",
  "name": "홍길동",
  "role": "USER",
  "iat": 1707580800,
  "exp": 1707582600       // 30분 후
}
```

**Refresh Token** (7일):
```json
{
  "sub": "123",           // userId만 포함
  "iat": 1707580800,
  "exp": 1708185600       // 7일 후
}
```

서명 알고리즘: **HMAC-SHA256** (대칭 키). `jwt.secret` 프로퍼티로 설정.

### 인증 흐름

```
[회원가입]
POST /api/v1/auth/signup
  → BCrypt(password) + BCrypt(pin) → User 저장
  → UserResponse 반환

[로그인]
POST /api/v1/auth/login
  → email/password 검증
  → Access Token 생성 (userId, email, name, role, 30분)
  → Refresh Token 생성 (userId, 7일)
  → Refresh Token Redis 저장 (key: refresh_token:{userId}, TTL: 7일)
  → TokenResponse 반환 {accessToken, refreshToken, expiresIn}

[토큰 갱신]
POST /api/v1/auth/refresh
  → Refresh Token 서명 검증
  → Redis에서 저장된 토큰과 비교 (탈취 방지)
  → 새 Access Token + 새 Refresh Token 발급
  → Redis에 새 Refresh Token 저장 (이전 토큰 자동 덮어쓰기)

[로그아웃]
POST /api/v1/auth/logout
  → Redis에서 Refresh Token 삭제 (즉시 폐기)
```

### SSE용 쿼리 파라미터 토큰

브라우저의 `EventSource` API는 커스텀 HTTP 헤더를 지원하지 않으므로, SSE 구독 시 JWT를 쿼리 파라미터로 전달한다.

```java
// JwtAuthenticationFilter.java - 토큰 추출 우선순위
private String resolveToken(HttpServletRequest request) {
    // 1순위: Authorization 헤더
    String bearer = request.getHeader("Authorization");
    if (StringUtils.hasText(bearer) && bearer.startsWith("Bearer ")) {
        return bearer.substring(7);
    }
    // 2순위: ?token= 쿼리 파라미터 (SSE EventSource용)
    return request.getParameter("token");
}
```

### permitAll 경로

인증 없이 접근 가능한 경로:
```java
// SecurityConfig.java
.requestMatchers("/api/v1/auth/signup", "/api/v1/auth/login", "/api/v1/auth/refresh").permitAll()
.requestMatchers("/swagger-ui/**", "/api-docs/**").permitAll()
.requestMatchers("/actuator/**").permitAll()
```

### CORS 설정

```java
// SecurityConfig.java
corsConfiguration.setAllowedOrigins(List.of(
    "http://localhost:3000",   // Grafana
    "http://localhost:3001",   // Next.js Dev
    "http://localhost:5173"    // Vite Dev
));
corsConfiguration.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
corsConfiguration.setExposedHeaders(List.of("Authorization", "Idempotency-Key"));
corsConfiguration.setMaxAge(3600L);  // preflight 캐시 1시간
```

## 코드 위치 및 구조

```
src/main/java/com/easypay/
├── common/config/
│   ├── SecurityConfig.java           # 필터 체인, CORS, permitAll, BCrypt
│   ├── JwtTokenProvider.java         # 토큰 생성/검증/파싱
│   ├── JwtAuthenticationFilter.java  # 토큰 추출 → SecurityContext 설정
│   ├── JwtProperties.java           # jwt.secret, 만료 시간 설정
│   ├── ApiRateLimitFilter.java       # API Rate Limit (Redis ZSET)
│   └── RequestIdFilter.java          # 요청 추적 ID (MDC)
├── user/
│   ├── service/AuthService.java      # 인증 비즈니스 로직
│   ├── controller/AuthController.java # 인증 API 엔드포인트
│   └── domain/User.java             # 사용자 엔티티
├── infra/redis/
│   └── RefreshTokenService.java      # Refresh Token Redis 저장
```

## 설정

```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET:your-256-bit-secret-key-here}
  access-token-expiration: 1800000    # 30분 (밀리초)
  refresh-token-expiration: 604800000 # 7일 (밀리초)
```

## 동작 흐름 (인증된 API 요청)

```
Client                    RequestIdFilter    JwtAuthFilter     RateLimitFilter    Controller
  │                           │                  │                  │                │
  │── GET /api/v1/accounts ──→│                  │                  │                │
  │  Authorization: Bearer eyJ...               │                  │                │
  │                           │                  │                  │                │
  │                     MDC.put("requestId",     │                  │                │
  │                      "a1b2c3d4")             │                  │                │
  │                           │──────────────→│  │                  │                │
  │                           │              resolveToken()        │                │
  │                           │              → "eyJ..."            │                │
  │                           │              validateToken()       │                │
  │                           │              → true                │                │
  │                           │              getUserId() → 123     │                │
  │                           │              getRole() → "USER"    │                │
  │                           │              SecurityContext.set(  │                │
  │                           │                userId=123,          │                │
  │                           │                ROLE_USER)           │                │
  │                           │                   │────────────→│  │                │
  │                           │                   │           isAllowed(             │
  │                           │                   │            "api:user:123",       │
  │                           │                   │            100, 1min)            │
  │                           │                   │           → true                 │
  │                           │                   │                │────────────→│   │
  │                           │                   │                │  controller     │
  │                           │                   │                │  logic          │
  │←── 200 OK ─────────────────────────────────────────────────────│                │
  │  X-Request-Id: a1b2c3d4   │                  │                  │                │
```

## 트러블슈팅 & 주의사항

### SecurityContext에 저장되는 Principal 타입
`authentication.getPrincipal()`의 반환 타입은 `Long`(userId)이다. Spring Security 기본과 다르게 `UserDetails`가 아니므로, 컨트롤러에서 `(Long) authentication.getPrincipal()`로 캐스팅해야 한다.

### Refresh Token Rotation
토큰 갱신 시 새 Refresh Token을 발급하고 Redis에 덮어쓴다. 이전 Refresh Token은 자동으로 무효화되어, 토큰 탈취 시 한 번만 사용할 수 있다.

### CORS와 Idempotency-Key 헤더
`Idempotency-Key`를 `exposedHeaders`에 추가해야 프론트엔드에서 응답 헤더를 읽을 수 있다. `allowedHeaders`에도 포함해야 preflight 요청이 통과한다.

### 테스트에서 Security Mock
`@WebMvcTest`에서는 `SecurityConfig`, `JwtTokenProvider`, `JwtProperties`, `RateLimiter`, `RefreshTokenService`를 모두 `@MockitoBean`으로 제공해야 한다.
