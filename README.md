# Spring Authorization Server 演示（Spring Boot 4.0.0, Maven）

基于 Spring Authorization Server 7.0.0（随 Boot 4.0.0 引入）搭建的纯内存 OAuth2/OIDC 授权服务器示例，支持授权码（含 PKCE）与客户端凭证，签发 RS256 JWT，JWK 运行时生成。构建工具：Maven（自带 `mvnw`）。

## 技术选型与架构
- 框架与版本：Spring Boot 4.0.0、Spring Authorization Server 7.0.0、Spring Security 7.0.0、JDK 17。
- 构建工具：Maven Wrapper（无需预装 Maven）。
- 存储：全部内存（RegisteredClient、用户、JWK），重启即重置。
- 安全链：采用 Boot 4 的 Authorization Server starter 自动配置授权端点；用户认证使用内存用户与委派密码编码器，显式 Web 安全自定义较少。
- 密钥/JWT：启动时生成 RSA 2048 JWK，RS256 签发访问/ID Token；`AuthorizationServerConfig` 提供 `JWKSource` 与 `NimbusJwtDecoder`。
- 客户端：`demo-client / secret`，授权码（PKCE）+ 客户端模式 + 刷新，作用域 `openid profile api.read`，重定向 `http://127.0.0.1:8080/login/oauth2/code/demo-client`。
- 用户：`user / password`，角色 USER。
- 端点：授权 `/oauth2/authorize`，令牌 `/oauth2/token`，发现 `/oauth2/.well-known/openid-configuration`（注意默认需登录），JWKS `/oauth2/jwks`（默认需登录），健康 `/actuator/health`，根 `/`。

## 搭建步骤（本项目已完成）
1. 使用 Spring Initializr 生成 Maven 项目（Boot 4.0.0，依赖 web/security/authorization-server/actuator）。
2. 添加内存用户与密码编码器（`SecurityConfig`）。
3. 定义授权服务器配置：内存客户端、Token/JWK、Issuer、JwtDecoder（`AuthorizationServerConfig`）。
4. 提供简单健康入口（`HelloController`）。
5. 配置 `.gitignore` 忽略构建产物与 IDE 文件。
6. 验证：`./mvnw test` 通过；运行 `./mvnw spring-boot:run` 启动。

## 快速开始
```bash
./mvnw spring-boot:run
# 验证服务
curl http://localhost:8080/
```
默认账户：`user / password`。客户端：`demo-client / secret`。发行者：`http://localhost:8080`。

> 提示：在 Spring Security 7/Boot 4 默认配置下，`/.well-known/openid-configuration` 与 `/oauth2/jwks` 会被重定向到登录页。可根据需要在 SecurityFilterChain 中对这些路径 `permitAll` 后再运行测试。

## 端点（注意默认登录要求）
- OIDC Discovery: `GET /.well-known/openid-configuration`（默认会 302 至 /login）
- JWKS: `GET /.well-known/jwks.json`（默认会 302 至 /login）
- 授权端点: `/oauth2/authorize`
- 令牌端点: `/oauth2/token`
- 健康检查: `/actuator/health`
- 根: `/`

## 客户端模式示例
这里的“客户端”指 OAuth2 客户端凭证（client_id / client_secret），本示例内存预置了一个客户端：`demo-client / secret`，它被允许使用客户端模式（`client_credentials`）获取访问令牌。命令如下：
```bash
curl -u demo-client:secret \
  -d "grant_type=client_credentials&scope=api.read" \
  http://localhost:8080/oauth2/token
```
说明：
- `-u demo-client:secret` 通过 HTTP Basic 传递 client_id/client_secret。
- `grant_type=client_credentials` 表示客户端模式，不涉及用户登录，凭客户端身份获取访问令牌。
- `scope=api.read` 请求作用域，本客户端配置中允许的 scopes 包含 `api.read`。
返回的 JSON 中包含 `access_token`、`token_type`、`expires_in` 等，即表明授权服务器按 OAuth2 规范完成了客户端模式颁发。

## 如何为你自己的客户端接入（入门步骤）
1) 注册客户端（注册一次即可）  
   - 在 `AuthorizationServerConfig` 中新增一个 `RegisteredClient`，设置：  
     - `clientId` / `clientSecret`（生产环境请用加密存储）；  
     - 支持的 grant 类型（如授权码 + PKCE、client_credentials、refresh_token）；  
     - 允许的 `redirectUri`（授权码必填）；  
     - 允许的 `scope`（如 `openid`, `profile`, 业务 scope）。  
   - Demo 里用的是内存 `InMemoryRegisteredClientRepository`，生产可改为 JDBC/JPA。

2) （可选）暴露发现/JWKS 便于客户端自动配置  
   - 如果要让外部客户端自动发现配置和公钥，可在安全链中对 `/.well-known/openid-configuration`、`/.well-known/jwks.json` 放开匿名访问。

3) 客户端使用授权码 + PKCE（有用户交互）  
   - 构造 authorize URL：`/oauth2/authorize?response_type=code&client_id=...&redirect_uri=...&scope=...&code_challenge=...&code_challenge_method=S256`  
   - 用户在浏览器登录/同意后，携带 `code` 重定向回 `redirect_uri`。  
   - 客户端用 `code` + `code_verifier` 交换令牌：`POST /oauth2/token`，`grant_type=authorization_code`。  
   - 需要前端/后端应用能处理重定向、存储 `code_verifier` 并发起令牌请求。

4) 客户端使用 client_credentials（机器到机器，无用户交互）  
   - 直接 `POST /oauth2/token`，携带 HTTP Basic (`client_id:client_secret`)，参数 `grant_type=client_credentials&scope=...`。  
   - 返回 `access_token`（JWT），可带到受保护资源服务器的 `Authorization: Bearer ...` 头访问。

5) 校验令牌 / 资源访问  
   - 资源服务器（或本服务自身）启用 JWT 资源服务器支持，配置 issuer 或 JWKS URI，携带 `Authorization: Bearer <token>` 调用受保护接口（如本例 `/test1`）。

6) 生产注意事项  
   - 强制 HTTPS；持久化客户端/用户；开启同意页；合理的 Token TTL 与刷新策略；限流/审计/日志；密钥轮换；不要在源码中硬编码机密。

## 授权码 + PKCE 示例
```bash
# 生成 verifier + challenge
VER=$(openssl rand -base64 32 | tr -d '=+/')
CH=$(echo -n "$VER" | openssl dgst -sha256 -binary | openssl base64 | tr '+/' '-_' | tr -d '=')

# 浏览器访问授权（登录 user/password）
open "http://localhost:8080/oauth2/authorize?response_type=code&client_id=demo-client&redirect_uri=http://127.0.0.1:8080/login/oauth2/code/demo-client&scope=openid%20profile%20api.read&code_challenge=$CH&code_challenge_method=S256"
```
成功登录后，浏览器会重定向携带 `code`。用下述命令换取令牌（替换实际 `CODE`）：
```bash
curl -u demo-client:secret \
  -d "grant_type=authorization_code&code=CODE&redirect_uri=http://127.0.0.1:8080/login/oauth2/code/demo-client&code_verifier=$VER" \
  http://localhost:8080/oauth2/token
```

## 注意
- 纯内存存储；重启后客户端/用户/密钥都会重置。
- 默认发现/JWKS 需登录，若要对接外部客户端请修改 Security 配置放开匿名访问。

## 架构梳理
- 依赖：Spring Boot 4.0.0，Spring Authorization Server 1.4.x，Web/Security/Actuator；JDK 17；Maven Wrapper。
- 安全链：两条 `SecurityFilterChain`；一条专用于授权服务器端点（含 OIDC、JWT 资源服务器），一条默认链提供表单登录并保护其他请求。
- 客户端存储：`InMemoryRegisteredClientRepository`，内置 `demo-client`，支持授权码（PKCE）、客户端模式、刷新令牌，作用域 `openid profile api.read`。
- 用户存储：`InMemoryUserDetailsManager`，用户 `user/password`，`PasswordEncoder` 委派编码。
- 签名密钥：启动时生成 RSA JWK（2048，RS256），通过 `/.well-known/jwks.json` 暴露；发行者 `http://localhost:8080`。
- 端点：OIDC Discovery、JWKS、授权 `/oauth2/authorize`、令牌 `/oauth2/token`、健康检查 `/actuator/health`、根路径 `/` 文本。
