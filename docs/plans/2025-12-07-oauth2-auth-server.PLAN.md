# OAuth2 Authorization Server Demo Plan (Spring Boot 4.0.0 + Spring Authorization Server 1.x)

- [x] Scaffold Spring Boot project with Authorization Server deps [pomodoro::1/1] @{pom.xml}
  - Spring Boot 4.0.0, Java 17+, deps: `spring-boot-starter-web`, `spring-boot-starter-security`, `org.springframework.security:spring-security-oauth2-authorization-server`
- [x] Configure Authorization Server security chains [pomodoro::2/2] @{src/main/java/.../config/SecurityConfig.java}
  - Default filter chain with表单登录 + 受保护请求；授权服务器链由 starter 自动配置
- [x] Configure in-memory `RegisteredClientRepository` and provider settings [pomodoro::2/2] @{src/main/java/.../config/AuthorizationServerConfig.java}
  - Client with `authorization_code` + `client_credentials`, PKCE enabled, redirect to localhost; scopes `openid`, `profile`, `api.read`
- [x] Configure JWK source / keys [pomodoro::1/1] @{src/main/java/.../config/Jwks.java}
  - Generate RSA key pair in-memory; expose JWK set endpoint
- [x] Basic landing/status endpoint for smoke test [pomodoro::0.5/0.5] @{src/main/java/.../HelloController.java}
- [x] README usage doc in Chinese [pomodoro::0.5/0.5] @{README.md}
- [x] 切换构建工具为 Maven + wrapper [pomodoro::1/1] @{pom.xml, mvnw, .mvn/**}
- [ ] Verify flows manually [pomodoro::2/0] @{curl/httpie scripts}
  - Auth code: open `/oauth2/authorize?response_type=code&client_id=demo-client&redirect_uri=...&scope=openid` -> exchange code for token
  - Client credentials: `POST /oauth2/token grant_type=client_credentials` returns access token
  - JWKS reachable at `/.well-known/jwks.json`

## Test Cases (to execute after implementation)
1) Server boots: `./mvnw spring-boot:run` succeeds; `GET /actuator/health` returns `UP`.
2) Discovery docs: `GET /.well-known/openid-configuration` returns JSON with issuer and endpoints.
3) Auth code flow: login form authenticates default user → redirect with `code` → `POST /oauth2/token` returns `access_token`, `id_token`.
4) Client credentials flow: valid client/secret returns `access_token`; invalid secret fails with 401.
5) JWKS: `GET /.well-known/jwks.json` returns RSA key with `kid` matching issued tokens.

## Execution Log
- `./mvnw test` (passes; Maven wrapper downloads Maven + deps)

## Notes / Dependencies
- Java 17+, Maven wrapper provided; in-memory storage only; restart resets clients/keys/users.
