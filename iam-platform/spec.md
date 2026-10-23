# 用户与组织架构平台（IAM）设计

> Status: draft
> 日期：2026-10-20
> 相关：ADR-0002（数据权限执行点）、ADR-0004（角色模型）、ADR-0005（临时密码认证，本设计将其取代）

## 0. 结论先行

1. **新建 `chatdata-agent-iam`**：独立部署的认证与权限中心，拥有用户、组织、角色、权限、数据策略、渠道身份绑定。它有自己的库，不 join 业务库。
2. **网关成为唯一入口**：`chatdata-agent-gateway` 从空壳变成 Spring Cloud Gateway，负责路由、令牌内省/会话校验、限流、CORS、渠道回调入口与静态前端。
3. **IAM 是权限的事实来源，引擎仍是行级/列级权限的执行点**。这一条是硬约束（ADR-0002）：策略可以集中存储，但把谓词拼进 SQL 必须留在 `AskSqlRenderer`，不能挪到 IAM 或网关。
4. **认证与授权走 OAuth 2.1 + Spring Security，但令牌是"不透明引用令牌"而不是 JWT**：授权记录落 MySQL（Spring Authorization Server 的 `oauth2_authorization`），会话与内省缓存落 Redis；资源服务器通过 RFC 7662 内省校验。**不再自研 HMAC 令牌，也不签发 JWT access token**。
5. **Web 采用 BFF 模式**：浏览器只拿 HttpOnly 会话 Cookie（Spring Session + Redis），令牌由网关持有并附加到上游请求；浏览器里没有 access token，XSS 拿不到。
6. **渠道（微信/飞书/钉钉）做成独立 worker + 统一 SPI**，不在网关里写厂商 SDK。渠道用户通过绑定表映射到 IAM 用户。
7. **分四阶段迁移**，每一阶段都可回滚；不搞一次性大爆炸。

## 1. 目标与非目标

### 目标

- `chatdata-agent-ask` 不再拥有用户表、密码、角色表与登录逻辑。
- 一套身份同时服务：Web、开放 API、微信/飞书/钉钉渠道。
- 组织架构是一等公民：用户属于组织（可多属），角色可授到人也可授到组织，数据范围沿组织树继承。
- 权限有统一的管理面与审计：谁在什么时候把什么权限给了谁。
- 支持未来的 SSO/OIDC 接入，而不推翻现有模型。

### 非目标（本期不做）

- 完整的 HR 主数据（入转调离、编制、薪酬）。IAM 只做"身份与授权"，组织信息可后续从 HR 同步。
- 把问数引擎改造成多租户。IAM 可以先多租户，ask 仍是单租户（见 §11）。
- 自研消息队列。渠道异步任务先用数据库表 + 轮询/延迟队列，量上来再换。

## 2. 现状盘点（代码事实，不是设想）

| 能力 | 现在在哪 | 关键事实 |
|---|---|---|
| 网关 | `chatdata-agent-gateway` | **空壳**：只有 pom 与 target，没有路由代码 |
| 登录 | `chatdata-agent-ask/.../auth` | 自研 HMAC 令牌（`LoginTokenService`），不是 JWT；验证码在内存 Map |
| 用户 | `chatdata_user` 表 | 临时方案，ADR-0005 明确"SSO 落地后整体删除" |
| 组织 | `chatdata_user.org_code` 单列 | 只支持"一人一组织"，且只是给动态行规则取值的字符串 |
| 角色 | `sys_role` / `sys_user_role` + `RoleService` | 角色码是字符串，必须与 `perm_json` 里的 `rowRules[].role` 逐字符一致 |
| 数据权限 | `ask_domain.perm_json` / `ask_assistant.perm_json` | 行规则、列脱敏、授权名单都在域配置里；引擎在渲染 SQL 时注入 |
| 权限判定 | `PermissionGuard` / `AskPermissions` | `PermissionContext{userId, roles, dynamicVars}`；角色从服务端查，不经 HTTP 头 |
| 兼容端点 | `JeecgCompatAuthController` | `/sys/login` 等，为厂商界面契约而存在；厂商依赖已移除，但这套契约被自研前端沿用 |

**结论**：IAM 不是"从零加一个模块"，而是把上面这四块（用户、组织、角色、登录）**搬出去并升级**，同时把 `perm_json` 的授权部分收敛进来。迁移必须逐块替换，不能一次改完。

## 3. 目标架构

```
                    ┌────────────────────────────────────────────┐
   Web (Vue)  ─────▶│                                            │
   开放 API   ─────▶│   chatdata-agent-gateway                   │
   微信/飞书/钉钉 ──▶│   · 路由 / TLS / CORS / 限流 / 请求 ID       │
                    │   · 不透明令牌内省 / BFF 会话（尽早拒绝）    │
                    │   · 渠道回调入口（签名由 adapter 校验）        │
                    └───────┬───────────────┬────────────────────┘
                            │               │
                  ┌─────────▼──────┐   ┌────▼─────────────────────┐
                  │ chatdata-      │   │ chatdata-agent-ask        │
                  │ agent-iam      │   │ · 问数编排 / SSE / 审计    │
                  │ · 用户/组织     │   │ · 通过 iam-client 取主体   │
                  │ · 角色/授权     │   │   与有效数据策略           │
                  │ · 数据策略      │   └────┬─────────────────────┘
                  │ · 渠道绑定      │        │
                  │ · OAuth 2.1     │   ┌────▼─────────────────────┐
                  └─────────┬──────┘   │ ask-engine                │
                            │          │ · 行级/列级权限的**执行点**  │
                  ┌─────────▼──────┐   └──────────────────────────┘
                  │ iam 库          │
                  │ (独立 schema)   │   ┌──────────────────────────┐
                  └────────────────┘   │ channel workers           │
                                       │ feishu / wecom / dingtalk │
                                       └──────────────────────────┘
```

### 关键原则

- **网关不承载业务逻辑**。它只做跨切面的事；任何"这个人能不能看这个域"的判断都不在网关。
- **服务之间不传明文身份头**。`X-User: 张三` 这类头是事故来源（ADR-0002 已记录中文角色经 HTTP 头乱码导致静默放行）。服务直接内省令牌。
- **IAM 不成为每一次问数的同步阻塞点**。策略走 SDK 缓存，缓存失效靠版本号，不靠 TTL 猜。

## 4. 模块划分

| Maven 模块 | 类型 | 职责 |
|---|---|---|
| `chatdata-agent-iam` | 可执行 Spring Boot 应用 | 认证、用户、组织、角色、授权、数据策略、渠道绑定、审计 |
| `chatdata-agent-iam-web` | Vue 3 + Vite 前端 | 登录页、用户/组织/角色管理。构建产物放进 IAM 的静态资源目录，与 `chatdata-agent-web → chatdata-agent-ask` 一致 |
| `chatdata-agent-iam-client` | 库 | 给 ask/channel worker 用：令牌内省与缓存、`Principal`、策略查询 + 缓存 |
| `chatdata-agent-gateway` | 可执行应用（Spring Cloud Gateway） | 唯一入口、路由、限流、静态前端、渠道 webhook 转发 |
| `chatdata-agent-channel-spi` | 库 | 渠道统一接口：验签、解析、回复、身份解析 |
| `chatdata-agent-channel-feishu` / `-wecom` / `-dingtalk` | 可执行 worker | 各自厂商 SDK 的隔离层，独立伸缩 |
| `chatdata-agent-ask` | 现有应用 | 去掉 `auth/**` 与用户/角色表依赖，改用 `iam-client` |
| `chatdata-agent-ask-engine` | 现有库 | 保持权限执行点；`PermissionContext` 由宿主用 IAM 主体构造 |

> 为什么不把 IAM 做成 ask 的一个模块：用户组织与渠道身份是**跨产品**能力；放在 ask 里会让下一个产品再抄一遍登录与角色。独立部署也便于单独扩容与审计。

## 5. 领域模型

### 5.1 核心概念

| 概念 | 定义 | 与现有术语的关系 |
|---|---|---|
| 租户（tenant） | 部署内的隔离边界。当前单租户，但所有表带 `tenant_id` | 不是"多租户部署形态"，是**租户级隔离**的载体 |
| 组织（org） | 树形结构：集团/公司/部门/团队。有 `path`（物化路径） | 现有 `org_code` 的升级 |
| 用户（user） | 登录主体。可属于多个组织，有一个主组织 | `chatdata_user` |
| 角色（role） | 权限集合。分三类：管理台、数据、功能（沿用 ADR-0004） | `sys_role` |
| 授权（grant） | 把角色/用户/组织授到某个资源（数据域、助手、功能） | 现有 `grantRoles`/`grantUsers`/`isPublic` |
| 数据策略（data policy） | 行级/列级规则的集合，绑定到资源与主体 | 现有 `perm_json` 的 `rowRules`/`colRules` |
| 渠道身份（channel identity） | 微信/飞书/钉钉的用户标识到 IAM 用户的映射 | 新增 |

### 5.2 组织树

- `iam_org.path` 用物化路径，如 `/1/12/37/`。`ORG_AND_CHILDREN` 判定变成 `path LIKE '/1/12/%'`，不需要递归查询。
- 组织移动时批量更新子树 `path`；组织数量在千级以内，可以接受。
- 组织有 `code`（稳定、可读），但**授权与策略内部一律用 id**，code 只用于展示与外部同步映射。

### 5.3 角色与授权

- `iam_user_role` 支持三种赋值：租户级（`org_id = null`）、组织级（`org_id = X`，可 `include_children`）。
- 有效角色 = 直接授予 + 所在组织（含祖先）授予。计算后放进 `Principal.roles`。
- **`role.code` 一经使用不可重命名**（ADR-0004 的教训）。新角色只能新增；废弃角色置 `enabled=0`，保留 code 以免历史策略失效。
- 功能权限（`iam_permission`：`resource:action`）用于管理面按钮与开放 API scope；数据权限不走这里。

### 5.4 数据策略

> **部分被取代（2026-10-23）**：数据策略的存储与编辑已回归各业务模块，
> IAM 只保留身份、组织、角色与功能权限。见
> [ADR-0010](../../docs/adr/0010-module-owned-data-policies.md)。
> 本节以下内容仅作为历史设计保留。

策略文档是**不透明 JSON**，由 IAM 存储、引擎解释：

```json
{
  "v": 1,
  "resourceType": "DOMAIN",
  "resourceId": "1372",
  "grants": {
    "isPublic": false,
    "roles": ["chat2bi", "区域经理"],
    "users": ["admin"],
    "orgs": ["12"],
    "unrestrictedRoles": ["分析师"]
  },
  "rowRules": [
    {"role": "区域经理", "field": "region", "op": "=", "value": "华东"},
    {"role": "客户经理", "field": "org_code", "op": "=", "sysVar": "orgCode"}
  ],
  "colRules": [
    {"role": "分析师", "field": "premium", "action": "脱敏"}
  ]
}
```

**为什么 IAM 存策略、引擎解释策略**：

- 存：权限的事实来源必须只有一处，否则管理台改了一处、另一处没改，失败形态是静默放行。
- 解释：行规则引用的是物理列（`region`）与 SQL 语义，只有引擎知道怎么把它变成 `WHERE`。ADR-0002 已否决"在网关/中间层解析 SQL 后注入谓词"。

有效策略接口（IAM → 引擎）：

```
POST /iam/policy/effective
{ "userId": "u_123", "resourceType": "DOMAIN", "resourceId": "1372" }

200 {
  "allowed": true,
  "version": 42,
  "grants": {...},
  "rowRules": [...],
  "colRules": [...],
  "attrs": { "orgCode": "华东", "orgPath": "/1/12/", "userId": "u_123" }
}
```

引擎把它映射成现有的 `DomainPermission`，`PermissionGuard` 与 `AskSqlRenderer` **不需要改判据**。这是把改动压到最小的关键。

### 5.5 存储分工（MySQL + Redis）

| 存储 | 放什么 | 不放什么 |
|---|---|---|
| MySQL（IAM 独立库） | 用户、组织、角色、授权、数据策略、渠道绑定、OAuth 客户端、`oauth2_authorization`、审计 | 浏览器会话、高频缓存 |
| Redis | BFF 会话（Spring Session）、内省结果缓存、限流计数、吊销广播频道 | **不作唯一事实来源**——Redis 丢了能重建，MySQL 丢了才是事故 |
| ask 业务库 | 语义配置、问数审计、会话、反馈 | 用户、密码、角色表 |

一句话：**登录信息与授权记录以 MySQL 为准，Redis 只做会话与加速**。这既满足"不用 JWT、状态可查"，也避免把 Redis 变成不可恢复的单点。

## 6. 认证与授权协议：OAuth 2.1 + Spring Security

### 6.1 为什么不再自研令牌

现状是 ADR-0005 的临时方案：`base64url(username).expiry.HMAC`。它有三个硬伤：无法细粒度授权（没有 scope）、无法单独吊销、第三方接入只能复制密钥。OAuth 2.1 把这三点都解决了，而且 Spring Security 已有成熟实现，自己维护一套只会持续欠债。

OAuth 2.1 相对 2.0 的关键约束（都必须遵守）：

- **取消 password / implicit grant**：登录必须是授权码流程（或客户端凭证），用户名密码只出现在授权服务器的登录页。
- **PKCE 强制**：所有授权码流程都要 PKCE，不再区分公共/机密客户端。
- **刷新令牌轮换**：公共客户端必须轮换；本项目所有客户端都轮换。
- **redirect_uri 精确匹配**：不做前缀/通配匹配。
- **Bearer token 只放 Authorization 头**：不接受 query 参数。

### 6.2 三个角色

| 角色 | 由谁承担 | 职责 |
|---|---|---|
| Authorization Server | IAM（Spring Authorization Server） | 登录页、授权码、令牌签发、刷新、撤销、JWKS、同意页 |
| Resource Server | 网关、ask、IAM 管理面 | 内省不透明令牌、把 `roles`/`scope` 映射成 authorities、401/403 |
| Client | Web（经 BFF）、开放 API、渠道 worker | 走对应 grant 拿令牌 |

### 6.3 客户端与授权类型矩阵

| 客户端 | grant | PKCE | 令牌存放 | 说明 |
|---|---|---|---|---|
| 自研 Web | `authorization_code`（**经网关 BFF**） | 必须 | 令牌只在网关，浏览器只有 HttpOnly 会话 Cookie | 推荐；见 §6.6 |
| 第三方 SPA（未来） | `authorization_code` | 必须 | 内存 + refresh 轮换 | 公共客户端，风险高于 BFF |
| 开放 API（服务端） | `client_credentials` | 不适用 | 服务端 | scope 限定 |
| 渠道 worker | `client_credentials` + **RFC 8693 token exchange** | 不适用 | worker 内存 | 用服务身份换用户身份 |
| CLI / 设备（可选） | `device_code` | 不适用 | 本地安全存储 | 后置 |

**不使用**：password grant（OAuth 2.1 已移除）、implicit、JWT bearer grant。

### 6.4 端点

| 端点 | 用途 |
|---|---|
| `/.well-known/oauth-authorization-server` | 授权服务器元数据 |
| `/.well-known/openid-configuration` | OIDC 元数据（启用 OIDC 时） |
| `/oauth2/authorize` | 授权码（PKCE） |
| `/oauth2/token` | 换令牌 / 刷新 |
| `/oauth2/jwks` | 公钥（仅启用 OIDC 时，用于 `id_token`；access token 是 opaque，不用它） |
| `/oauth2/revoke` | 撤销 refresh/access |
| `/oauth2/introspect` | 内省（仅服务端，需客户端凭证） |
| `/userinfo` | OIDC 用户信息 |
| `/connect/logout` | RP-Initiated Logout |

`/sys/**` 兼容层保留在 IAM：它内部走同一套 `OAuth2AuthorizationService`，只是把结果包装成厂商契约，供现有前端过渡。

### 6.5 令牌：不透明引用令牌（Opaque / Reference Token）

**不使用 JWT 作为 access token。** 令牌本身只是一串随机引用，所有信息在服务端：

| 数据 | 存放 | 说明 |
|---|---|---|
| 授权记录（code、access/refresh token、客户端、scope、主体） | MySQL `oauth2_authorization` | Spring Authorization Server 的 JDBC 实现；**权威来源** |
| 浏览器会话（BFF） | Redis（Spring Session） | `SESSION` Cookie 只带会话 id |
| 内省结果缓存 | Redis | key 用令牌的 SHA-256，TTL ≤ 30 秒 |
| 吊销广播 | Redis Pub/Sub | 撤销后通知各资源服务器清缓存 |
| 用户/组织/角色/策略 | MySQL | IAM 的业务表 |

令牌设置（Spring Authorization Server）：

```java
OAuth2TokenSettings.builder()
    .accessTokenFormat(OAuth2TokenFormat.REFERENCE)   // 关键：不透明引用令牌
    .accessTokenTimeToLive(Duration.ofMinutes(30))
    .refreshTokenTimeToLive(Duration.ofDays(14))
    .reuseRefreshTokens(false)                        // 每次刷新轮换
    .build();
```

内省响应（`POST /oauth2/introspect`）携带资源服务器需要的全部信息：

```json
{
  "active": true,
  "sub": "u_123",
  "client_id": "chatdata-web-bff",
  "scope": "ask.read ask.feedback",
  "exp": 1760000000,
  "tenant": "t_default",
  "username": "zhangsan",
  "name": "张三",
  "org_id": "12",
  "org_path": "/1/12/",
  "roles": ["区域经理"],
  "channel": "web",
  "ver": 7
}
```

要点：

- **令牌里不携带任何业务信息**：角色、组织、scope 都在 IAM 侧，由内省返回；权限变更/角色调整立即生效（配合缓存失效）。
- `ver` = `iam_user.token_version`，用于"改密码/踢下线"时批量失效；单条令牌吊销直接删除 `oauth2_authorization` 记录 + 广播清缓存。
- Refresh Token 不透明、轮换、重放检测；其存储由 Spring 的 `JdbcOAuth2AuthorizationService` 负责，**令牌值在库里，按敏感数据保护**（不要假设它被哈希）。
- **OIDC 的 id_token 是 JWT**：如果启用 OIDC（SSO 场景），`id_token` 按规范必须是签名的 JWT。它只用于"证明这次认证"，不作为调用 API 的凭据；access token 仍然是不透明的。若不启用 OIDC，则整条链路没有 JWT。
- 为什么不把令牌直接放 Redis 而不落库：`oauth2_authorization` 是 Spring 官方实现，支持码/令牌/同意记录的完整生命周期；Redis 只做会话与缓存，宕机后授权记录仍可从库恢复。**存储分工是刻意的，不是二选一。**

### 6.6 Web：BFF 模式（推荐）

```
浏览器 ──(会话 Cookie)──▶ 网关(BFF) ──(Bearer access token)──▶ ask / IAM
```

- 浏览器访问 `/bff/login` → 网关以机密客户端身份走授权码 + PKCE → 回调后**令牌只留在网关**（Spring Session + Redis 存会话与 OAuth2AuthorizedClient），浏览器只拿 `HttpOnly; Secure; SameSite=Lax` 的 `SESSION` Cookie。
- 网关在转发时注入 `Authorization: Bearer ...`，并在 access token 快过期时用 refresh 静默续期。
- 好处：浏览器无令牌（XSS 偷不到）、无 CORS、无 token 存储讨论；退出登录 = 删会话 + 撤销 refresh。
- 代价：网关需要会话存储（Redis）与 OAuth2 Client 依赖；多实例必须共享会话。

> 如果坚持纯 SPA（不用 BFF），就必须接受"令牌在浏览器内存 + refresh 轮换 + 严格 CSP"这一套，且 XSS 的后果更严重。本项目已经在用 HttpOnly Cookie，BFF 是更自然的延续。

### 6.6.1 IAM 自身的登录（前后端分离）

IAM 与 `chatdata-agent-ask` 一样采用前后端分离：`chatdata-agent-iam-web`（Vue 3 + Vite）构建到 IAM 的 `static/app`，由 IAM 自己提供静态资源。**不使用服务端模板（Thymeleaf）。**

授权码流程中的"未登录"处理：

1. 浏览器访问 `/oauth2/authorize?...`；
2. 未认证时，授权服务器的 `AuthenticationEntryPoint` 重定向到 SPA 的登录路由 `/app/login?returnTo=<原始 authorize URL>`；
3. SPA 调 `POST /iam/auth/login`（用户名 + 密码 + 验证码 + CSRF token）；后端用 `AuthenticationManager` 认证，成功后通过 `SecurityContextRepository` 把 SecurityContext 写进 Redis 会话（`SESSION` Cookie），**不在这里签发 access token**；
4. SPA 读取 `returnTo`，`window.location.assign` 回 `/oauth2/authorize`；此时会话已建立，授权服务器继续签发授权码；
5. 授权码回到 BFF 的 `/bff/callback`，由网关换取不透明令牌。

配套端点：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/iam/csrf` | 取 CSRF token（SPA 启动时调用，写 Cookie/返回 JSON） |
| POST | `/iam/auth/login` | 建立会话，返回当前主体；失败 401 + 统一提示 |
| POST | `/iam/auth/logout` | 销毁会话（并撤销该会话关联的 refresh，如果由 BFF 管理） |
| GET | `/iam/me` | 当前主体 |

要点：

- **登录接口不签发令牌**。令牌的唯一签发点是 `/oauth2/token`；否则会出现两条令牌路径，审计与吊销都会分叉。
- CSRF：`/iam/auth/login` 必须带 CSRF token。SPA 从 `/iam/csrf` 取，或由服务端下发 `XSRF-TOKEN` Cookie 由前端回填 `X-XSRF-TOKEN`。
- 验证码先于密码校验（ADR-0005 的结论），实现与旧 ask 一致：Redis 存储、一次性。
- 登录页静态资源由 IAM 提供；开发时 Vite 代理 `/iam`、`/oauth2`、`/captcha` 到 IAM。

### 6.7 服务与渠道

- **开放 API**：`client_credentials`，scope 最小化；每个 client 有独立 secret 与限流配额。
- **渠道 worker**：先用 `client_credentials` 证明"我是可信 worker"，再用 **RFC 8693 Token Exchange** 把 `(channel, external_id)` 换成该绑定用户的 access token（`subject_token` = worker token，参数带渠道身份）。
  - Spring Authorization Server 新版本提供 token exchange 支持；若使用的版本没有，则实现自定义 `AuthenticationProvider` + grant type，**语义保持 RFC 8693**，不要另造一套。
  - 换出的用户令牌带 `channel` 属性与较短 TTL（如 5 分钟），并限制 scope（通常只有 `ask.read`）。
- **服务间**：不新造内部信任通道；worker 调 ask 也用用户令牌。这样审计里能追到"是哪个渠道的哪个用户"。

### 6.8 资源服务器配置（ask / 网关 / IAM 管理面）

资源服务器**不验签，改走内省**。Spring Security 7 / Boot 4：

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: http://iam:9000/oauth2/introspect
          client-id: ask-resource-server
          client-secret: ${ASK_RS_SECRET}
```

```java
@Bean
SecurityFilterChain api(HttpSecurity http, OpaqueTokenIntrospector introspector) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(a -> a
            .requestMatchers("/api/engine/admin/**").hasAuthority("SCOPE_iam.admin")
            .anyRequest().hasAuthority("SCOPE_ask.read"))
        .oauth2ResourceServer(o -> o.opaqueToken(op -> op.introspector(introspector)));
    return http.build();
}
```

自定义内省器做三件事（`RedisOpaqueTokenIntrospector`）：

1. **先查 Redis 缓存**（key = `sha256(token)`，TTL 30s），未命中再调 IAM；
2. 把 `roles` 转成 `ROLE_*`，`scope` 转成 `SCOPE_*`；
3. 订阅 `iam:revoke` 频道，收到消息立即删除对应缓存——吊销延迟从 30 秒降到广播延迟。

要点：

- **每个服务自己内省**，不信任网关注入的身份头（网关内省只为尽早拒绝）。
- 内省请求带资源服务器自己的 client 凭证；IAM 对 `introspect` 限流并审计。
- 内省失败（IAM 不可用）→ **fail-closed**：401/503，不回落成放行。
- 网关是 reactive：用 `ReactiveOpaqueTokenIntrospector` + `SecurityWebFilterChain`。
- 不建议让各服务直连 IAM 的 Redis 读令牌：那是把 IAM 的存储 schema 变成公共契约，改一次要改所有服务。缓存 + 内省是更稳的边界。

### 6.9 scope 与数据权限的边界（重要）

不要把行级/列级权限做成 scope。原因：

- scope 是**静态、粗粒度、客户端级**的；数据权限是**动态、细粒度、主体+资源级**的（同一角色在不同域看到不同行）。
- 把 `domain:1372:region:华东` 塞进 scope 会让令牌爆炸，且策略一变就得重新授权。
- OAuth 负责"能不能调用这个 API"，策略层负责"调用后能看哪些行/列"。

因此：`scope=ask.read` 只说明"可以问数"；具体能问哪个域、看哪些行，由 §7 的有效策略判定。

### 6.10 Spring Security 组件清单

| 依赖 / 组件 | 用途 |
|---|---|
| `spring-boot-starter-oauth2-authorization-server` | 授权服务器 |
| `spring-boot-starter-oauth2-resource-server` | OAuth2 资源服务器（配 `opaquetoken` 内省） |
| `spring-boot-starter-security` | 过滤链、方法级安全 |
| `spring-session-data-redis` | BFF 会话、内省缓存、吊销广播 |
| `JdbcRegisteredClientRepository` | 客户端注册（官方表 `oauth2_registered_client`） |
| `JdbcOAuth2AuthorizationService` | 授权码/不透明令牌/刷新令牌持久化（支持撤销与轮换） |
| `JdbcOAuth2AuthorizationConsentService` | 同意记录 |
| `OAuth2TokenCustomizer<OAuth2TokenContext>` | 内省时补充 tenant/roles/org 等属性 |
| `OpaqueTokenIntrospector` | 资源服务器侧解析不透明令牌（带 Redis 缓存） |

Spring Authorization Server 自带的三张表（`oauth2_registered_client`、`oauth2_authorization`、`oauth2_authorization_consent`）用官方 DDL，不自造。

### 6.11 登录页与暴力破解

- 登录页是 IAM 的 Vue 前端（`/app/login`），通过 JSON 登录接口建立会话；**没有服务端模板**。
- 后端保留图形验证码；验证码先于密码校验（ADR-0005 的结论）。
- 登录失败计数、账号锁定、按 IP + 账号双维度限流；失败提示统一为"账号或密码错误"。
- 登录成功后重建会话（session fixation 防护），写审计。
- 授权服务器的 `SecurityFilterChain` 与资源服务器链用 `securityMatcher` 分开，顺序明确。

### 6.12 SSO 现状与预留

**已确认：公司在用飞书，但暂时没有可用的自建应用权限。**

因此：

- **P0–P3 以本地账号为主**（密码 + 验证码 + 锁定策略），不等待飞书权限；AD​R-0005 的临时密码方案退役，但"本地密码"作为 IAM 的一等功能保留。
- 数据模型先留好外部身份位：`iam_user.external_id` / `external_provider`，`iam_org.external_id`；飞书权限到位后直接同步，不改表。
- 飞书权限到位后的接入顺序：
  1. 飞书自建应用（企业内应用）→ OAuth 2.1 授权码 + PKCE 免登；
  2. 通讯录 API 同步部门/用户（`external_id` 映射），**只读**，不反向写飞书；
  3. 组织与角色：通讯录只提供组织树，**角色仍在 IAM 内分配**，不做"飞书组自动变角色"（自动映射是提权风险）。
- 若后续还有其它 IdP（LDAP/OIDC），走同一套 OIDC 客户端适配，不新增第二套身份体系。
- 本地密码保留为 break-glass，可由配置关闭；启用飞书 SSO 后，密码登录审计级别提高。

## 7. 授权判定

### 7.1 判定输入

`Principal`（来自令牌内省）+ 资源（域/助手/功能）+ 请求动作。

### 7.2 判定顺序（与现有引擎一致）

1. 租户匹配（令牌 tenant == 资源 tenant）
2. 功能/管理面角色（如 `chat2bi`）
3. 资源授权（grant：public / role / user / org）
4. 数据策略（行规则、列规则）
5. 任何一步"取不到"→ **拒绝**，不是跳过（ADR-0002：厂商跳过规则放行是事故）

### 7.3 组织与动态变量

- `Principal.attrs.orgCode` = 主组织 code；`orgPath` 物化路径；多组织时提供 `orgCodes[]`。
- 现有动态行规则只支持单值。P0 只填主组织；多组织与 `in` 操作符放到 P2，避免旧规则语义漂移。

## 8. 与问数引擎的集成

### 8.1 改动点（尽量小）

| 位置 | 现在 | 目标 |
|---|---|---|
| `AskEngineStreamController` | `IdentityResolver` 解析 HMAC 令牌 → username | `IamIdentityResolver` 内省不透明令牌 → `Principal` |
| `RoleService.findRoleCodesByUsername` | 查 `sys_user_role` | `Principal.roles()` |
| `PermissionContext` | `of(userId, roles)`，动态变量为空 | `of(principal)`，带 `attrs` |
| `DomainLoader` / `perm_json` | 从域配置读 `DomainPermission` | 从 `iam-client` 取有效策略；无策略则拒绝 |
| `AskPermissions.require` | 判据不变 | 判据不变，输入换成有效策略 |

### 8.2 缓存与失效

- SDK 缓存 key：`(userId, resourceType, resourceId)`；TTL 30–60 秒 + `version`。
- IAM 修改策略时递增 `policy_version`；SDK 响应带版本，发现版本变化立即失效。
- **缓存失败时 fail-closed**：拿不到策略 → 拒绝并给出"权限服务暂不可用"，不回落成"放行"。
- 评测（`eval` profile）需要绕过 HTTP：SDK 提供 `PolicyProvider` 接口，评测用夹具实现，不让评测依赖 IAM 在线。

### 8.3 直连模式

直连模式没有行/列权限（ADR-0007），只做表级授权。它的授权名单仍由 IAM 的 `grants` 决定；策略里的 `rowRules`/`colRules` 在该模式下不生效——这一点必须在界面上继续明示，不能因为"IAM 里配了规则"就让人以为生效。

## 9. 网关设计

### 9.1 路由

| 路径 | 目标 | 鉴权 |
|---|---|---|
| `/.well-known/**`、`/oauth2/**`、`/login`、`/connect/logout` | IAM 授权服务器 | 放行（授权服务器自己的过滤链） |
| `/bff/**` | 网关自身（BFF） | 会话 Cookie；负责授权码流程与令牌续期 |
| `/iam/admin/**` | IAM | 内省 + `SCOPE_iam.admin` |
| `/api/ask/**`、`/api/engine/**` | ask | 内省（`/api/ask/engine` 支持 SSE；BFF 注入 Bearer） |
| `/sys/**` | IAM 兼容层 | 放行 + 端点自校验（保留前端契约） |
| `/app/**` | 静态前端 | 放行 |
| `/channel/{provider}/**` | 渠道 worker | provider 签名校验 |
| `/actuator/health` | 各服务 | 内网放行 |

### 9.2 过滤器链

1. `RequestIdFilter`：生成/透传 `X-Request-Id`，贯穿日志与审计
2. `BffSessionFilter`：读写 Spring Session（Redis）；未登录访问受保护页面 → `/bff/login`
3. `OpaqueTokenFilter`：内省（Redis 缓存）、过期、`ver`；失败 401（授权服务器端点白名单）
4. `TokenRelayFilter`：从 BFF 会话取出 access token 注入上游 `Authorization`；
   快过期时用 refresh 静默续期（单飞，避免并发刷新互相踢掉）
5. `RateLimitFilter`：Redis 令牌桶，按 `(tenant, user|ip, route)`；登录接口单独更严
6. `CorsFilter`：白名单 Origin，不允许 `*` + credentials
7. `BodySizeFilter` + 超时：SSE 路由关闭响应缓冲，读超时放宽
8. `AuditFilter`：管理面写操作记审计（谁、什么、结果）

### 9.3 信任模型

- 网关**剥离**客户端自带的 `X-User`、`X-Roles` 等头。
- 下游服务**自己内省令牌**，不依赖网关注入的身份；网关的内省只是提前拒绝，不替代服务端校验。
- 服务间调用（channel worker → ask）用用户 access token，不新造内部信任通道。

## 10. 渠道

### 10.1 SPI

```java
public interface ChannelAdapter {
    String provider();                                  // feishu / wecom / dingtalk
    VerifiedEvent verifyAndParse(HttpRequest req);      // 验签 + 解析 + 去重键
    ExternalIdentity identityOf(VerifiedEvent event);   // open_id / userid / union_id
    Reply reply(VerifiedEvent event, AskReply answer);  // 文本/卡片/图片
}
```

### 10.2 入站流程

```
渠道回调 → 网关 → worker
  1. 验签（时间戳窗口 + 重放缓存）
  2. 事件去重（event_id 唯一键）
  3. 渠道身份 → IAM 用户（绑定表）；未绑定 → 引导绑定
  4. 换取用户 access token（服务凭证 + external_id）
  5. 调 ask（非流式或消费 SSE）
  6. 回复；长任务先 ack 再异步回填
```

### 10.3 绑定

- `iam_channel_identity(tenant_id, provider, external_id, union_id, user_id, status)`。
- 未绑定用户：回复一次性绑定链接（带签名、短 TTL）→ Web 登录 → 确认绑定。
- 也可配置"按手机号/邮箱匹配"，但必须显式开启并记录审计（自动匹配是账号劫持的入口）。

### 10.4 异步与时限

- 渠道回复有时限（微信 5 秒内 ack、客服消息 48 小时窗口；飞书/钉钉类似）。问数一次 2–10 秒，必须：
  - 5 秒内回"正在查询"；
  - 结果落 `channel_message_job`，worker 轮询/延迟投递后再发。
- 去重与幂等：`(provider, event_id)` 唯一键；重复投递直接返回上次结果。

### 10.5 渠道路由与产品形态

- 同一个 ask 服务，多渠道只是不同的入口与回复格式。
- 渠道会话与 Web 会话**不共享** `sessionId`（上下文口径不同）；渠道会话按 `(provider, external_id, thread)` 维护。

## 11. 多租户

**已拍板：短期只有 1 个租户。**

- 所有 IAM 表从第一天带 `tenant_id`，默认值 `t_default`；令牌/内省结果带 `tenant`。
- **不做租户切换 UI、不做租户级管理员**。单租户下这些是纯负担。
- 网关与 ask 启动时读取部署配置的 `chatdata.tenant`；令牌里的 `tenant` 必须与之一致，否则 403。这样即使 IAM 以后多租户，单个 ask 部署也不会串租户。
- **ask 暂不做多租户**：现有 `ask_domain`、`ask_assistant` 等表没有 `tenant_id`，加列与数据迁移是独立工程，排在 P4 之后。
- 第二租户真正落地时要补的：租户管理页、租户级角色、跨租户隔离测试、ask 表加列。现在只保证"加第二租户时不改 IAM 表结构"。

## 12. API 清单（首版）

### OAuth 2.1 / OIDC（授权服务器）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/.well-known/oauth-authorization-server` | 元数据 |
| GET | `/.well-known/openid-configuration` | OIDC 元数据 |
| GET/POST | `/oauth2/authorize` | 授权码 + PKCE（登录页由 IAM 提供） |
| POST | `/oauth2/token` | 授权码/刷新/客户端凭证/token exchange |
| GET | `/oauth2/jwks` | 公钥（仅 OIDC `id_token`） |
| POST | `/oauth2/revoke` | 撤销 |
| POST | `/oauth2/introspect` | 内省（服务端） |
| GET | `/userinfo` | OIDC 用户信息 |
| GET/POST | `/connect/logout` | 登出 |

### 会话与主体

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/bff/login` | 网关发起授权码流程 |
| GET | `/bff/callback` | 授权码回调（换令牌，写会话） |
| POST | `/bff/logout` | 删会话 + 撤销 refresh |
| GET | `/iam/me` | 当前主体 + 组织 + 角色（管理面/前端用） |

### 用户/组织/角色

| 方法 | 路径 | 说明 |
|---|---|---|
| GET/POST/PUT/DELETE | `/iam/admin/users` | 用户 CRUD、启用/禁用、重置密码 |
| GET/POST/PUT/DELETE | `/iam/admin/orgs` | 组织树 CRUD、移动 |
| GET/POST/PUT/DELETE | `/iam/admin/roles` | 角色 CRUD（code 不可改） |
| PUT | `/iam/admin/users/{id}/roles` | 角色赋值（租户级/组织级） |
| PUT | `/iam/admin/orgs/{id}/roles` | 组织角色 |
| GET/PUT | `/iam/admin/policies/{resourceType}/{resourceId}` | 数据策略读写 |
| POST | `/iam/policy/effective` | 有效策略（内部，需服务凭证） |
| GET | `/iam/admin/audit` | 审计查询 |
| GET/POST/PUT | `/iam/admin/clients` | OAuth 客户端注册（scope、redirect_uri、grant） |

### 渠道

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/iam/channel/bind/{token}` | 绑定确认 |
| POST | `/iam/channel/{provider}/callback` | 渠道事件（由 worker 校验签名） |

## 13. 迁移路径

### P0：IAM 独立可跑，ask 不动（约 3–4 周）

- 建库与表；导入现有 `chatdata_user` / `sys_role` / `sys_user_role` / `org_code`。
- 引入 Spring Authorization Server：授权码 + PKCE、客户端凭证、刷新轮换、撤销、官方三张表；**access token 使用 `OAuth2TokenFormat.REFERENCE`（不透明）**。
- 自定义登录页（密码 + 验证码）、`OAuth2TokenCustomizer` 补充 tenant/roles/org 属性（内省时返回）。
- **`chatdata-agent-iam-web`**：Vue 3 + Vite 的登录页与管理台，构建到 IAM 的 `static/app`；登录走 `POST /iam/auth/login` 建会话，再回 `/oauth2/authorize`。**不用 Thymeleaf。**
- 会话与内省缓存接 Redis；吊销走"删授权记录 + Pub/Sub 广播"。
- IAM 实现 `/sys/**` 兼容端点（内部走同一套授权服务），现有前端零改动可继续登录。
- 管理 API 与最小管理页（用户/组织/角色/客户端）。
- **验收**：`curl` 走完 authorization_code + PKCE 与 client_credentials 两条流程；opaque 令牌可通过 `/oauth2/introspect` 解析；吊销后内省立即 `active=false`；refresh 轮换生效；现有前端仍能登录；ask 仍用旧令牌，双跑。

### P1：网关接管入口，ask 换认证（约 2–3 周）

- 网关路由、`ReactiveOpaqueTokenIntrospector`、BFF 会话（Redis）与 token relay；静态前端由网关提供。
- ask 引入 `oauth2-resource-server`：`opaqueToken` + 自定义 introspection 映射 `roles`/`scope` 到 authorities。
- `iam-client`；ask 删除 `auth/**`、`chatdata_user` 依赖、`RoleService` 查库逻辑。
- 角色从内省结果取；`PermissionContext` 带 attrs；未登录/无 scope 返回 401/403，不落到通用 500。
- **验收**：ask 代码中不再有用户表与登录逻辑；浏览器里看不到 access token；令牌吊销在 TTL 内生效；权限矩阵测试通过；SSE 经 BFF 不断流。

### P2：授权与数据策略收敛（约 2–3 周）

- `perm_json` 的 grants/rowRules/colRules 导入 IAM 的 policy 文档；引擎改为读有效策略。
- 管理台：策略编辑、角色赋值、组织移动；策略变更版本号。
- 双读校验：迁移期同时计算"旧配置"与"IAM 策略"，不一致告警；校验通过后切换。
- **验收**：同一问题的行/列结果与迁移前逐字节一致；策略缓存失效 < 1 分钟；无策略时拒绝。

### P3：渠道（飞书优先，约 3–4 周/渠道）

- `channel-spi` + `channel-feishu`；绑定流程；异步任务表。
- worker 用 `client_credentials` + token exchange 换用户令牌；令牌带 `channel` 属性与短 TTL。
- 验收：飞书私聊提问 → 5 秒内 ack → 结果回复；未绑定用户被引导绑定；重复事件不重复回复；审计能追到渠道用户。

### P4：飞书 SSO 与多租户硬化

- 飞书应用权限到位后：飞书 OAuth 免登（授权码 + PKCE）+ 通讯录只读同步（部门/用户 → `external_id`）。
- 角色仍由 IAM 分配；不做"飞书组自动变角色"。
- 多租户（第二租户）另立项目：租户管理、租户级角色、ask 表加 `tenant_id`。

### 回滚

- 每个阶段都有开关：`auth.mode=legacy|oauth`、`policy.source=local|iam`。
- P2 的双读校验结果保留 7 天；出现不一致立即切回 `local`，不影响问数。

## 14. 安全与运维

- **密码**：BCrypt cost ≥ 10（或 Argon2id）；登录失败计数与锁定；验证码一次性。
- **令牌**：不透明令牌至少 256 位随机；只有 IAM 能签发与解析；内省端点必须认证并限流。
- **存储**：`oauth2_authorization` 属敏感数据，限制 DB 账号权限与备份访问；Redis 里只放会话与 `sha256(token)` 缓存，**不把明文令牌写进日志/缓存 key**。
- **吊销**：删除授权记录 + Redis Pub/Sub 广播；资源服务器收到广播立即清缓存。
- **Cookie**：`HttpOnly`、`Secure`、`SameSite=Lax`；写操作校验 `Origin`。
- **审计**：登录、刷新、吊销、角色/策略变更、渠道绑定、管理面写操作。
- **PII**：渠道 external_id 与手机号属个人信息；最小化存储、访问受角色限制、按保留期清理。
- **限流**：登录 5 次/分钟/IP；开放 API 按 client；渠道按用户。
- **可观测**：登录成功率、401/403 按路由、策略缓存命中率、IAM 延迟 P99、渠道 ack 时延。

## 15. 测试策略

- **权限矩阵**：用户 × 组织 × 角色 × 域 → 期望的行/列结果；必须覆盖"动态变量取不到 → 拒绝"。
- **令牌**：伪造/过期、内省不可用、吊销后缓存及时失效、`ver` 不匹配、refresh 重放。
- **策略迁移**：迁移前后同一批问题的 SQL/结果逐条比对（复用评测 harness 的 EX 比对）。
- **网关**：白名单、SSE 不被缓冲、限流、超大 body、CORS。
- **渠道**：签名错误、时间戳过期、重放、未绑定、长任务。

## 16. 风险与待决策

| 风险/决策 | 说明 | 建议 |
|---|---|---|
| 数据策略放 IAM 还是引擎 | 放 IAM 是单一事实来源，但引擎多一次依赖 | 放 IAM；引擎保留执行点；缓存 + fail-closed |
| 令牌形态：opaque vs JWT | 用户明确要求不用 JWT，改用 DB/Redis 登录态 | **opaque reference token**；introspection + Redis 缓存；OIDC 的 id_token 例外 |
| 内省性能 | 每个请求一次内省会成为瓶颈 | Redis 缓存 30s + 吊销广播；IAM 水平扩容；内省端点单独限流 |
| Redis 可用性 | 会话/缓存宕机会影响登录与鉴权 | Redis 主从 + 哨兵/集群；缓存 miss 回落内省；**不回落到放行** |
| ask 多租户 | 现有表无 `tenant_id` | IAM 先多租户；ask 单租户并校验租户一致 |
| 角色 code 字符串耦合 | 策略按 code 引用 | 新策略用 role id 存储、code 仅展示；旧策略迁移时做映射表 |
| 渠道优先级 | 飞书/企微/钉钉 | 先飞书（API 与事件模型最清晰），其次企微 |
| 是否保留 `/sys/**` | 现有前端契约 | P1 保留兼容层；前端切到 `/iam/**` 后删除 |
| 网关技术栈 | 空壳模块 | Spring Cloud Gateway（与现有 Spring Boot 4 对齐） |
| Web 接入方式 | BFF vs 纯 SPA | **BFF**：浏览器无令牌；纯 SPA 只在必须支持第三方前端时才做 |
| token exchange 版本 | Spring Authorization Server 不同版本支持度不同 | 先确认所选版本是否内置 RFC 8693；没有就实现自定义 grant，语义不变 |
| Spring 版本对齐 | 项目已是 Boot 4.1 / Security 7 | 固定 Authorization Server 版本并写进父 pom；升级前跑令牌兼容测试 |
| 组织来源 | 手工维护 vs HR 同步 | 先手工 + 预留 `external_id`；HR 同步后置 |

## 17. 已拍板的三个问题

| 问题 | 结论 | 对设计的影响 |
|---|---|---|
| 租户模型 | **短期只有 1 个租户** | IAM 表带 `tenant_id`、默认 `t_default`；不做租户切换 UI；网关/ask 校验令牌 tenant 与部署配置一致；ask 多租户后置 |
| SSO 现状 | **有飞书，但暂无应用权限** | P0–P3 本地账号为主；预留 `external_id` 与 OIDC 适配；飞书权限到位后做免登 + 通讯录同步，角色仍由 IAM 分配 |
| 数据策略归属 | **策略存 IAM、执行在引擎** | IAM 提供有效策略接口；`PermissionGuard`/`AskSqlRenderer` 判据不变；缓存 + fail-closed；迁移期双读校验 |
| IAM 前端形态 | **前后端分离（Vue 3 + Vite）** | 新增 `chatdata-agent-iam-web`；登录走 JSON 接口建会话再回授权码流程；不用 Thymeleaf |

## 18. P0 实施进度（2026-10-20）

### 已完成并实测通过

- `chatdata-agent-iam` 模块：Spring Boot 4.1 + Spring Security 7 + Spring Authorization Server 7.1。
- 库表：`sql/iam/01-schema.sql`（IAM 表 + 授权服务器官方三张表）、`02-seed-dev.sql`、`03-migrate-from-ask.sql`。
- 授权服务器：授权码 + PKCE、客户端凭证、刷新轮换、撤销、内省；**access token 为 `reference`（不透明）**，已在 `oauth2_registered_client.token_settings` 中确认。
- 会话与 CSRF：Spring Session + Redis；`GET /iam/auth/csrf` 下发 `XSRF-TOKEN`。
- 登录：`POST /iam/auth/login`（密码 + 验证码 + CSRF）建立会话；验证码存 Redis、一次性。
- 主体：`GET /iam/me`；认证对象用框架 `User`，额外属性按用户名现查。
- 端到端实测（脚本 `/tmp/iam-flow.py`）：
  登录 200 → 授权码 302 → 换令牌 200（opaque，`expires_in=1799`）→ 内省 `active=true` →
  刷新轮换成功、**旧 refresh 重放 400 invalid_grant** → 客户端凭证 200。

### 两个踩过的坑（已修，写进代码注释）

1. **自定义 `UserDetails` 会让令牌端点 500**：授权服务器把 Authentication 序列化进
   `oauth2_authorization.attributes`，其 Jackson 白名单不认自定义类型，报
   `PolymorphicTypeValidator denied resolution`。改用框架 `User` 作 principal，
   userId/组织按用户名现查（也正好符合"角色与组织不进令牌"的设计）。
2. **验证码 details 会被一起序列化**：`CaptchaWebAuthenticationDetails` 没有反序列化入口，
   且验证码本就不该落库。认证成功后清空 `Authentication.details`。

### 管理 API 与 iam-web（同日完成）

- **管理 API**：`/iam/admin/users`（列表/详情/新建/更新/重置密码/分配角色）、
  `/iam/admin/orgs`（列表/新建/更新）、`/iam/admin/roles`（列表/新建/更新）。
  鉴权在过滤链上（`/iam/admin/**` 需 `ROLE_chat2bi`），异常经 `@RestControllerAdvice` 映射为 400/500。
- **`chatdata-agent-iam-web`**：Vue 3 + Vite，登录页 + 用户/组织/角色三页；
  构建到 IAM 的 `static/app`，`WebAppForwardController` 处理深链接。
- 实测（脚本 + 浏览器）：
  - 管理 API：建用户（绑组织 + 授角色）、不存在的角色码被拒、重名被拒、覆盖式改角色、
    建组织（path 自动算成 `/1/12/`）、建角色与重名拒绝、重置密码——全部符合预期。
  - SPA：登录页/用户/组织/角色四页渲染正常；从 `/oauth2/authorize` 进入会被送到
    `/app/login?returnTo=<authorize URL>`，登录后浏览器继续授权码流程并**签发了授权码**
    （`oauth2_authorization` 出现新记录）。回调地址指向尚未实现的网关，所以浏览器停在错误页——预期。
- 单元测试 12 个：角色码校验、密码长度、重名提示、密码哈希、组织 path 计算、父组织校验等。

### 又一个踩过的坑（已修）

**returnTo 一度是 `/login` 而不是原始 authorize 地址**：到达 `/login` 控制器时 URI 已被
授权服务器的 302 改写成 `/login`，原始地址只能从 Spring Security 的 `SavedRequest`（会话）取。
不修的话表现为"登录成功后又跳回登录页"。

### P1 阶段一：网关已立起来（2026-10-21）

- `chatdata-agent-gateway`：Spring Cloud Gateway 5.0.2（Boot 4 对应版本；
  根 pom 补上了此前只定义属性、未导入的 Spring Cloud BOM）。
- **路由**：`/iam/**`、`/oauth2/**`、`/.well-known/**`、`/login`、`/captcha/**` → IAM；
  `/app/**`、`/sys/**`、`/api/ask/**`、`/api/engine/**` → ask；`/bff/**` → 网关自身。
- **BFF**：`/bff/login` 发起授权码 + PKCE（已确认 redirect 带 `code_challenge`/`S256`），
  `/bff/callback` 换令牌并把 `OAuth2AuthorizedClient` 存进 Redis 会话，
  路由上的内置 `TokenRelay` 过滤器把 access token 注入上游 `Authorization`。
- **主体来源**：IAM 新增 `/iam/principal`（不透明令牌 → 主体信息），
  既作网关的 `user-info-uri`，也是后续资源服务器的"富内省"接口。
- **端口重排**：8080 归网关，ask 退到 8081，IAM 保持 9000。
- **会话 Cookie 名分离**：IAM 用 `IAM_SESSION`，网关用 `SESSION`——
  Cookie 不区分端口，同名会互相覆盖。
- **IAM 的 SPA 迁到 `/iam/app/`**：`/app/**` 属于 ask 的问数页，两个前端不能撞路径。

实测：

```
/iam/auth/csrf  → 200（经网关代理到 IAM）
/iam/app/       → 200（IAM 的 SPA）
/app/           → 200（ask 的问数页）
/bff/login      → 302 → /oauth2/authorization/iam → IAM /oauth2/authorize（含 PKCE）
浏览器走完 BFF 登录后 /bff/me → {"authenticated":true,"username":"admin",
  "attributes":{"roles":["chat2bi"],"scope":["ask.read","ask.feedback"],…},"tokenReady":true}
```

### P1 阶段二：ask 切成资源服务器（2026-10-21）

- **新增 `chatdata-agent-iam-client`**：`IamIntrospector` 用 IAM 的 `/iam/principal`
  把不透明令牌换成主体（含角色与组织），Redis 缓存 30 秒（key 是令牌的 SHA-256，
  不落明文）；**拿不到就拒绝**，绝不回落成放行。缓存编码用分隔符而不是 JSON——
  Boot 4 的 classpath 是 Jackson 3，而这个库要被 servlet 与 reactive 服务共用，
  少引一个序列化器就少一处版本摩擦。
- **ask 变成资源服务器**：`/api/**` 需登录，管理面需 `ROLE_chat2bi`/`ROLE_admin`；
  角色来自令牌内省结果，不再查 `sys_user_role`。
- **删掉 ask 的本地认证**：`ask/auth/**`（含 `/sys/**` 兼容端点、验证码、HMAC 令牌）、
  `IdentityResolver`、`chatdata_user` 的 mapper 与 `chatdata.auth.*` 配置。
  新增 `CurrentUser`：身份与角色从 SecurityContext 取。
- **网关**：`/api/**` 要求已登录；打开 CSRF（`XSRF-TOKEN` Cookie → `X-XSRF-TOKEN` 头，
  并加了一个过滤器让 Cookie 在第一个 GET 就下发）；删掉 `/sys/**` 路由。
- **前端**：登录页不再有表单，直接跳 `/bff/login`；会话状态来自 `/bff/me`；
  写请求带 CSRF 头。

实测（浏览器，全程走网关）：

```
打开 /app/ → SPA 守卫 → /app/login → 自动跳 /bff/login → IAM 登录页
登录后回到 /app/ask；助理列表来自 /api/ask/assistants（令牌中继 + 内省）
提问「华东的保费合计是多少」→ 627,604.31（含理解回显、图表切换、SQL、反馈区）
未登录访问 /api/** → 401（不是 302）；访问 /app/ask → 200（静态页）
```

### 阶段二的三个坑（都已修）

1. **`Mono<Void>` 被 `switchIfEmpty` 当成空流**：入口点分流写成
   `flatMap(401).switchIfEmpty(重定向)`，`commence` 完成时是空的，
   于是 401 刚写好就被重定向覆盖成 302。改成先归一成布尔再分支。
2. **iam-client 引不到 Jackson 2**：Boot 4 用 Jackson 3（`tools.jackson`），
   改用分隔符编码缓存，顺带少一个依赖。
3. **Cookie 不区分端口**：IAM 与网关同主机，`SESSION` 同名会互相覆盖 → IAM 用 `IAM_SESSION`。

### 阶段二之后仍未完成

- **吊销广播**：内省缓存靠 30 秒 TTL 过期，IAM 侧还没有 Pub/Sub 通知（spec §6.5 已列）。
- ~~**`sys_role` 仍在**：ask 管理台的"角色点选"仍读本地 `sys_role`~~ → **已收敛到 IAM**，
  见「P2 阶段三」。
- **`chatdata_user` 表仍在库中**：已无人读，P2 一并清理。

### 阶段一的两个已知取舍

- `/api/**` 与 `/sys/**` 在网关**直接放行**：ask 仍用自己的 HMAC 令牌。
  阶段二把 ask 切成资源服务器时，这里改成 `authenticated()` 并同时打开 CSRF。
- 网关的 CSRF 当前关闭，原因同上（SPA 还没带 CSRF token）。

### 尚未完成（下一步）

- ~~`/iam/principal`（资源服务器用的富内省接口）~~ → **已在 P1 阶段一完成**（网关的
  `user-info-uri` 与资源服务器的富内省都走它）。**吊销广播仍未做**，见上一节。
- ~~`/sys/**` 兼容层~~ → **不再需要**：P1 阶段二把 ask 切成资源服务器时，
  `/sys/**` 路由与兼容端点已一并删除，前端也切到了 `/iam/**`。
- 集成测试（登录 → 授权码 → 换令牌 的自动化用例，目前是脚本 + 手工验证）。

### 本地运行

```bash
mysql -uroot -proot -e "CREATE DATABASE IF NOT EXISTS chatdata_iam DEFAULT CHARSET utf8mb4"
mysql -uroot -proot chatdata_iam < sql/iam/01-schema.sql
mysql -uroot -proot chatdata_iam < sql/iam/02-seed-dev.sql
cd chatdata-agent-iam && . ~/.chatdata-agent.env && mvn spring-boot:run   # 端口 9000
```

环境变量：`IAM_BFF_SECRET`、`IAM_RESOURCE_SECRET`（已加入 `~/.chatdata-agent.env`），
`IAM_ISSUER` 默认 `http://127.0.0.1:9000`。

### P2 阶段一：策略版本化失效 + 双读校验 + 助理策略接线（2026-10-21）

P2 的三条验收此前只有一条半成立：**「失效 < 1 分钟」不达标**（TTL 正好 60 秒）、
**「逐字节一致」从未验证过**（迁移只有三行 `COUNT(*)` 核对），
另发现 **IAM 里的 ASSISTANT 策略是死数据**（引擎从不读，管理员在 IAM 改助理授权毫无反应）。

**① 策略版本化失效——用后台批量轮询，不是请求路径上的逐资源探测**

初版设计是「每次请求先探一下版本」，被否掉：探测的 404 与「IAM 里没有策略」不可区分
（路由未部署时会把好缓存删掉并永久拒绝）；Redis 变成鉴权的硬依赖（缓存抖动 = 全量问数被拒）；
`version` 非单调（删表重导入会重置为 1，配合「探测成功就续 TTL」会让陈旧文档永不失效）；
`/api/ask/assistants` 每次加载触发 N 次探测。

改为：`IamPolicyClient.effective()` 的请求路径**一行不改**，另起
`PolicyVersionRefresher`（`@Scheduled`，默认 15 秒）批量拉版本号、只驱逐版本变了的条目。

| 落点 | 说明 |
|---|---|
| `GET /iam/policy/versions?resourceType=` | 只返回 `resourceId + version`，不带文档。轮询能又勤又便宜 |
| `IamServiceClientAuth` + `PolicyClientAuthInterceptor` | 鉴权从 `EffectivePolicyController` 的私有方法抽成**拦截器**，挂在整个 `/iam/policy/**` 前缀上。抽成普通组件的话，以后每个新 handler 都得记得手动调用，而这个前缀在过滤链上是 `permitAll`，没有兜底 |
| `chatdata.iam.policy-probe-seconds` | 默认 15。**它就是策略的实际失效延迟**；构造期校验 `1..60` 且不得大于 `policy-cache-seconds`，否则启动失败（设成 300 会让验收静默失效） |
| 缓存值格式 | **未变**（`version␾document`），滚动发布期间新旧实例可互读 |

失效延迟 ≤ 轮询间隔（15s）；轮询任务挂掉时退化为 TTL 兜底（60s），正确性不依赖它存活。

**② 双读校验——比的是编译产物，不是 JSON**

`PolicyDriftChecker` 对同一探测主体分别用两份权限跑 `PermissionGuard.check`，
比较产出的 `RowScope`（`Restricted` 里就是最终拼进 WHERE 的 `SqlPredicate` 原文与参数）
与脱敏列集合。不比 JSON 文本，因为 `DomainPermission` 的自动 `equals` **不是语义相等**：
它对 `rowRules`/`colRules` 的列表顺序、`v`、规则 `id`、`ColRule.action` 字面值逐字比较，
而这些行为上无关；反过来算子别名（`=`/`==`、`包含`/`like`）会被编译器归一。

探测主体有三处刻意设计，缺一个校验器就会**空转通过**：

1. **一个持有全部角色的「并集主体」**——`RowRuleCompiler` 按列表顺序用 OR 合并，
   单角色主体永不触发合并，于是**规则顺序变了也看不出来**
2. **为每个 `sysVar` 都供值**——缺变量会让两侧同时判 `RULE_NOT_EVALUABLE`，比对通过却从未编译过该规则
3. **一个刻意缺变量的主体**——覆盖 §15 要求的「动态变量取不到 → 拒绝」

助理分支**单独比** `AssistantPermission.grantsAccess`：`PermissionGuard.check` 的助理分支
在生产是死代码（两个调用点都传 `null`），只比它等于什么都没验证。
存在性不对称（本地有/IAM 无、IAM 有/本地无）是**硬失败**，不是报告行。

用法（有差异非零退出，可当 CI 门禁）：

```bash
mvn -q -pl chatdata-agent-ask-engine -am install -DskipTests
mvn -pl chatdata-agent-ask spring-boot:run -Dspring-boot.run.profiles=policycheck
```

**③ 助理策略接线**

`PolicyProvider` 泛化为 `policyDocument(ResourceType, String resourceId)`，域与助理走同一条覆盖路径；
`AskServiceFactory.forAssistant` 与 `listUsableAssistants` **都**经 `loadAssistantWithIamPolicy` 加载
（两处各写一遍的话迟早只改一处，表现是"列表里看得见、点进去被拒"）。
`DomainLoader`/`AssistantLoader` 支持覆盖感知加载：有覆盖时**完全不解析** `perm_json`，
否则本地 JSON 坏掉时加载直接抛，IAM 里那份好策略永远没机会生效。

**同时关掉了由 ③ 引入的新静默失效**：ask 管理台的权限编辑写的是本地 `perm_json`，
③ 之后会被 IAM 静默覆盖。现在 `policy.source=iam` 时写入返回 **409** 并指路去 IAM 控制台
（`PolicyWriteGuard`，只拦"权限真的变了"的请求，改名/改描述照常放行），
管理台界面上也提前给了提示（`GET /api/engine/admin/policy-source`）。
判定信号复用「有没有 `PolicyProvider` bean」——与决定"IAM 是不是权威"的是同一个信号。

**已知限制（本次不做）**

- ~~**`EngineStartupCheck` / `RoleReferenceValidator` 校验的不是生效策略。**~~ → **已在
  「P2 阶段三」修掉**：校验改为按 `PolicyProvider` 取生效文档，角色清单来自 IAM 角色目录。
- 双读校验比的是**当前**本地 vs **当前** IAM（持续发现漂移），不是历史迁移审计。
- **`policy.source=iam` 时 `IAM_RESOURCE_SECRET` 必填**，缺了启动就失败。此前这条被掩盖了
  （没有 bean 急切依赖策略客户端）；`PolicyVersionRefresher` 让它显现出来。
  这是刻意的：缺密钥时"每个问题都被拒绝"比启动失败难排查得多。
- 更强的端到端证据（对固定 DSL 逐条比渲染 SQL 与结果字节）未做。当前结论建立在
  「渲染器是『谓词 + Domain』的纯函数」这一假设上，**该假设写在 `PolicyDriftChecker` 的注释里**；
  一旦渲染器出现依赖权限的分支，本校验器会静默失效，届时必须补 DSL 回放。
- 仍未做：`chatdata_user` 清理、吊销广播（Pub/Sub）、多组织 / `in` 操作符、
  `/iam/policy/**` 限流与 `basicAllowed` 每个未认证请求一次 BCrypt 的开销。

### P2 阶段二：功能权限 + 组织树下钻 + 内置超管（2026-10-22）

设计决定见 `docs/adr/0009-functional-permissions-and-builtin-super-role.md`。落地要点：

| 落点 | 说明 |
|---|---|
| `iam_role_permission(role_id, code)` | 关联表而不是 JSON 列：取"某用户的全部权限"是一条 join，还能在 code 上建索引回答"谁有这项权限" |
| `PermissionCatalog`（IAM，约 27 个码） | 唯一的一份定义：配置界面数据源、保存校验基准、`@PreAuthorize` 字符串的出处 |
| `EffectivePermissionResolver`（5 秒缓存） | **界面展示与后端强制同源**。最初两者各算一份，结果是"界面有按钮、点了全 403"，连超管也一样 |
| IAM 侧 `@PreAuthorize("@perm.has(...)")` | 每个控制器**类级 `@PreAuthorize("false")`** + 逐方法覆盖。漏挂一个方法是"用不了"，不是"对全员开放" |
| ask 侧 `AdminApiPermissions` 规则表 | 39 个端点集中在同一个类里，逐方法注解既啰嗦又易漏。改为一张表 + 类级一个注解，并由**覆盖测试**断言每个映射都有规则 |
| `super_admin` 内置超管 | 代码展开全量、界面只读、不可停用；`chat2bi` 降为普通角色（预置 24 项） |
| 提权防护 | 带 `roles` 建用户额外要 `iam.users.roles`；授超管需自己已持有；内置角色不可停用；最后一个超管用户不可停用 |
| `ask.ui.charts` / `ask.ui.feedback` | 前端隐藏图表 tab 与反馈按钮。**无 `ask.ui.charts` 时强制回退表格**——只藏切换器的话，后端推荐饼图时照样画出来 |

**升级顺序（顺序错了会锁死）**

1. 跑 `sql/iam/08-bootstrap.sql`：建 `super_admin` 并把现有 `chat2bi` 持有者补授上去
2. 再上新版 IAM 与 ask

反过来（先上新版再跑脚本）会有一段时间**所有人都进不了管理台**：`chat2bi` 不再等于超管，
而 `super_admin` 还没授出去。

**全新部署的初始化**

```bash
# 1) 建库建表 + 种子（08 会把 super_admin 建好）
mysql -uroot -p chatdata_iam < sql/iam/01-schema.sql
mysql -uroot -p chatdata_iam < sql/iam/07-role-permission.sql
mysql -uroot -p chatdata_iam < sql/iam/08-bootstrap.sql

# 2) 建第一个管理员账号（密码走环境变量，不写进脚本）
BOOTSTRAP_ADMIN_USERNAME=admin \
BOOTSTRAP_ADMIN_PASSWORD='一次性强密码' \
mvn -pl chatdata-agent-iam spring-boot:run -Dspring-boot.run.profiles=bootstrap-admin
```

运行器是幂等的：用户已存在时**保留其密码不动**（要改密码需显式
`BOOTSTRAP_ADMIN_RESET_PASSWORD=true`），角色关系只加不减。

**从旧库升级**（已有 `sys_role` / ask 数据）：依次跑 `03-migrate-from-ask.sql`、`06-import-policies.sql`。
`06` 会把旧角色码 `管理员` 替换成 `super_admin`；**已经迁过的库**再跑
`09-fix-stale-role-refs.sql` 补一次（幂等）。

**本次修掉的三个实现 bug**（都是"看起来对、跑起来不对"那类）

1. `IamApplication` 的 `@MapperScan` 是白名单式列包的，漏了 `...iam.permission`
   → 启动报 `required a bean of type 'RolePermissionMapper'`
2. `/api/ask/me` 多套了一层 `Result` 信封，而前端按裸对象解析
   → `permissions` 读成 `undefined`，表现为"权限全丢、管理台入口消失"。
   ask 侧 `/api/ask/**` 的约定是裸负载，`Result` 是 `/api/engine/admin/**` 的约定
3. 两个前端的**动态 `redirect` 在路由守卫之前执行**，那时身份还没解析、权限是空的
   → 一律落到兜底页。改成静态重定向 + 让守卫按权限跳转

**仍未做**

- 权限码在两处重复（ask 不依赖 IAM 模块），由 `PermissionCatalogTest.askCodesExist` 兜住漂移；
  真正的修法是把目录挪进共享模块
- ~~`EngineStartupCheck` / `RoleReferenceValidator` 校验的仍不是生效策略（见阶段一的说明）~~ →
  已在「P2 阶段三」修掉

### P2 阶段三：角色目录 + 启动校验切生效策略（2026-10-22）

**目标**：把 `sys_role` 从 ask 的读路径上摘掉，并让启动校验看**运行时真正生效的那份策略**。

| 落点 | 说明 |
|---|---|
| `GET /iam/policy/roles`（IAM，新增） | 服务间只读角色目录：`{code,name,category,enabled}`。含停用角色——「不存在」与「存在但停用」要能分开报 |
| `IamRoleDirectoryClient`（iam-client，新增） | 失败**抛出**而不是返回空：空在策略语义里是"没有策略"，两者含义完全不同；降级由调用方决定 |
| `ServiceTokenSupplier`（iam-client，新增） | client_credentials 的令牌获取从 `IamPolicyClient` 里抽出来，供策略与角色目录共用；提前 30 秒刷新 |
| ask 管理台 `/api/engine/admin/roles` | 改读 `IamRoleDirectoryClient`；IAM 不可用 → **503**，不返回空列表（空列表会让管理员以为"角色被删光了"） |
| `EngineRoleDirectoryConfig` | IAM 不可用 → `Optional.empty()` + WARN；启动校验据此跳过角色引用检查 |
| `EngineStartupCheck` | 注入 `ObjectProvider<PolicyProvider>`：**能取到就按生效策略校验**，取不到才回落本地 `perm_json`（本地模式/测试） |

**三个必须一起做的理由**

1. **只切角色不切策略会制造假报错**：本地 `perm_json` 是迁移留下的旧拷贝，里面的旧角色
   在 IAM 里已不存在 → `DANGLING_*`（域上的 `RULE_ROLE` 还是 fatal）会拦住一次本来正确的启动。
   这正是 `EngineStartupCheck` 注释里警告的"配置改不了所以起不来"死锁。
2. **IAM 不可用时跳过而不是拦**：拦启动会把一次 IAM 抖动变成崩溃循环；静默跳过又会让
   "校验通过"变成空话。折中是记一条 `POLICY_SOURCE_UNAVAILABLE`（WARNING）并跳过该资源。
3. **"没有策略" ≠ "取不到策略"**：前者按运行时语义 fail-closed（`NO_GRANT`，ERROR），
   后者跳过。若把两者都当空处理，在 IAM 里删掉策略会变成一次静默放行。

**不归一化角色码**（不 trim、不改大小写）：运行时是逐字符匹配，这里做归一化会造出
"校验通过、运行时匹配不上"这种最坏组合。有测试钉住（`IamRoleDirectoryClientTest`、
`PolicyClientAuthInterceptorTest`、`EngineStartupCheckPolicySourceTest`）。

**测试**

- `IamRoleDirectoryClientTest`：解析不改码、401 抛出、缺密钥本地失败（`MockRestServiceServer`）
- `PolicyClientAuthInterceptorTest`：把新 handler 放进同一套夹具，验证它真的走 `/iam/policy/**` 闸门
- `RoleDirectoryWiringTest`（ask）：bean 接线、码不归一化、IAM 不可用 → 空
- `EngineStartupCheckPolicySourceTest`（ask-engine）：本地旧角色不再误报、生效策略里的坏角色照样报、
  不可用 → 仅一条 WARNING、无策略 → `NO_GRANT`

**清理（已做）**

- 删除 `chatdata-agent-common-security` 模块（`RoleService` / `SysRoleMapper` /
  `SysUserRoleMapper` 与其 Testcontainers 测试）及根/聚合 pom 里的声明；连带删除
  `sql/01-role.sql`、`sql/02-role-seed.sql`（仅那个模块的集成测试在用）。
- `EngineStartupCheckRealDbTest` 的种子角色改从 `chatdata_iam.iam_role` 读——此前读
  `sys_role`，而后者比 IAM 多一个 `管理员`，"零违规"是靠一份更宽松的角色清单换来的。
- 前端 `RoleHint.vue` / `AssistantListView.vue` / `DomainListView.vue` / `api/admin.ts`
  与 `docs/permission-config.md` 的 `sys_role` 说法统一改为 IAM 角色目录。
- `RoleHint.vue` 修掉一个由 503 引出的坑：角色列表为空时它会把**所有**角色报成"不存在"。
  现在只在拿到目录时才判"不存在"，否则提示"暂时无法校验"。
- 数据补丁 `sql/iam/09-fix-stale-role-refs.sql`：把 `管理员` → `super_admin`（IAM 策略文档 +
  `ask_domain` / `ask_assistant` 的本地副本）。此前每次启动 5 条 WARNING 就是它。

### P2 收尾：IAM 数据策略路径下线（2026-10-23）

按 `docs/adr/0010`，数据策略的事实来源回到业务模块。IAM 侧不再理解
`DOMAIN` / `ASSISTANT` 两套文档结构，也不再跨服务读取 ask 的资源元数据。

| 落点 | 变化 |
|---|---|
| 写接口 | 删除 `/iam/admin/policies`（列表/详情/保存） |
| 服务间接口 | 删除 `/iam/policy/effective` 与 `/iam/policy/versions` |
| 读接口 | **保留** `/iam/policy/roles`、`/iam/policy/users`——IAM 仍是身份目录的唯一来源 |
| 权限码 | 删除 `iam.policies`、`iam.policies.write`；`08-bootstrap.sql` 不再预置 |
| 前端 | 删除 IAM 的「数据策略」页与路由；`PolicyListView.vue` 与相关 API 函数一并移除 |
| ask | 删除 `IamPolicyClient`、`PolicyProvider`、`PolicyVersionRefresher`、`PolicyWriteGuard`、`policycheck` 双读工具；`chatdata.engine.policy.source` 开关移除 |
| 数据 | `sql/iam/11-retire-iam-policy.sql` 把 `iam_policy` 改名为 `iam_policy_retired_20261023` 保留，不 DROP |
| 历史脚本 | 05/06/09/10 保留作迁移记录，文件头标记「已退役」，全新部署不再执行 |

**顺序**：先确认 10 的复核查询为 0 行，再跑 11。IAM 路径已不再被读取，
所以顺序反了不会泄露数据，但会少一份可回溯的策略副本。

**没有改的东西**：`EngineStartupCheck` 的角色引用校验、`PolicyReferenceValidator`
（保存时校验角色/用户存在性）与 `ask_policy_audit` 都继续有效——它们依赖的是
IAM 的**角色/用户目录**，不是数据策略存储。

---

## Comments

（待补充）
