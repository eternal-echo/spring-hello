# OAuth2 Authorization Server Demo (Spring Boot 4.0.0 + Spring Authorization Server 1.4.x)

## Abstract
In-memory OAuth2/OIDC authorization server exposing authorization code (with PKCE) and client credentials grants, issuing JWT access/ID tokens signed by an ephemeral RSA JWK set for quick demo usage.

## Methodology
- Dependencies: Spring Boot 4.0.0, Spring Authorization Server 1.4.x (via `spring-boot-starter-oauth2-authorization-server`), Web, Security, Actuator; Java 17+; Maven（使用 `mvnw` wrapper 自动下载 Maven）。
- Security: two filter chains—`AuthorizationServerSecurityFilterChain` for OAuth2 endpoints and a default chain enabling form login for user auth; CSRF disabled on token endpoints only.
- Clients: `InMemoryRegisteredClientRepository` defining `demo-client` with `authorization_code` (PKCE) + `client_credentials`, redirect to `http://127.0.0.1:8080/login/oauth2/code/demo-client`, scopes `openid profile api.read`, secret encoded via delegated `PasswordEncoder`.
- Keys: runtime-generated RSA key pair exposed via JWK source; issuer set to `http://localhost:8080`.
- Users: `InMemoryUserDetailsManager` with demo user `user`/`password` and roles USER.
- Endpoints: OIDC discovery `/.well-known/openid-configuration`, JWKS `/.well-known/jwks.json`, authorize `/oauth2/authorize`, token `/oauth2/token`.
- Observability: health at `/actuator/health`, simple landing `/` returns static text for smoke test.

## Algorithm / State
Token issuance follows OIDC standard. JWT signing uses RS256 with random key id per boot. Access token lifetime: $T_{access} = 5 \\text{ min}$; ID token lifetime: $T_{id} = 5 \\text{ min}$; refresh token lifetime: $T_{refresh} = 8 \\text{ h}$ (configurable).

## Implementation Details (target paths)
- `pom.xml`: Spring Boot parent 4.0.0，依赖 web/security/authorization-server/actuator，测试依赖 spring-boot-starter-test + spring-security-test。
- `config/SecurityConfig.java`: define two filter chains, authentication provider, password encoder.
- `config/AuthorizationServerConfig.java`: configure `RegisteredClientRepository`, `AuthorizationServerSettings`, JWK source, `JwtDecoder`.
- `config/Jwks.java`: utility generating RSA JWK with `kid`.
- `HelloController.java`: `GET /` returns health text.
- Maven wrapper (`mvnw`, `.mvn/wrapper/maven-wrapper.properties`) 自动下载 Maven。

## Pseudocode (flow ≤15 lines)
```
onApplicationStart():
  jwkSource = generateRsaJwk()
  registeredClients = inMemory([demoClient])
  authServerChain = authorizationServerHttp(jwt(jwkSource))
  defaultChain = httpFormLogin()
  userStore = inMemoryUser(user/password)
  runServer()
```
