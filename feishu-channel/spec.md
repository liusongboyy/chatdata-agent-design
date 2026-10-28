# 飞书 Channel：把问数接入 IM 与后续多渠道抽象

- Status: Draft
- Date: 2026-10-28
- Related: `docs/adr/0002-data-permission-enforcement-point.md`、`docs/adr/0008-iam-oauth21-opaque-tokens.md`、`docs/adr/0010-module-owned-data-policies.md`、`.scratch/iam-platform/spec.md`、`.scratch/conversation-history/spec.md`

## 1. 场景

业务人员在飞书里 @ 机器人问一句「上周华东的订单金额是多少」，机器人直接回复表格与结论。
他们不该为此打开浏览器、记住另一个网址、再登录一次。

这是 channel 接入的第一个实现。**后续要接钉钉、企业微信**，所以本 spec 不只描述飞书，
还给出「什么该抽象、什么不该」的边界——但**第一阶段不建抽象层**，理由见 §9。

**传输选 Webhook**（不是长连接），理由与代价见 §3。

### 非目标

- 不做个人微信。个人微信走 iLink Bot API，形态完全不同（长轮询、bot 身份、群事件不保证送达），
  单独一份 spec。
- 不在 IM 里复刻 Web 管理台。配数据源、改语义域、审策略仍然只能在 Web 端。
- 不做流式逐字输出。见 §6.3。

## 2. 现状与缺口

### 已具备（可直接复用）

| 能力 | 位置 | 说明 |
|---|---|---|
| 外部身份列 | `iam_user.external_provider` / `external_id` | 建表时就留了，注释写着「如飞书 open_id」。**目前代码里零引用** |
| 唯一约束 | `uk_iam_user_external (tenant_id, external_provider, external_id)` | 一个人一个飞书号，天然防重 |
| 问数接口 | `POST /api/engine/engine`（SSE） | 产物是结构化 DSL/SQL/结果集，channel 只是另一个客户端 |
| 多轮会话 | `ask_agent_session` + `ask_conversation` | 前端 sessionId 直接复用，channel 需要一层映射 |
| 审计 | `ask_audit_log` | 按 `username` 记，只要身份映射对，审计自动成立 |
| 权限判定 | `@perm` + 行规则动态变量 | **完全在 ask 内部**，channel 拿不到也不需要碰 |
| 令牌校验 | `IamIntrospector` | ask 内省不透明令牌，与签发方式无关 |

### 缺口

| 缺口 | 影响 |
|---|---|
| 无 channel 模块 | 一切从零 |
| IAM 无法「代表用户」签发令牌 | channel 拿到的 `open_id` ask 不认识（§4 详解） |
| 飞书消息处理需在 **3 秒**内返回 | 问数要 5–15 秒，必须异步 |
| 无 `chat_id → 会话` 映射 | 「那上个月呢」会退化成新问题 |
| 无绑定流程 | 用户首次使用时不知道机器人是谁 |
| 无公网入口 | Webhook 必须有公网地址；本地开发依赖内网穿透（§3.1） |
| 飞书失败重推跨 6 小时 | 幂等窗口必须 ≥ 6 小时，用 Redis 默认 TTL 会重复回答（§3.4） |

## 3. 传输：Webhook

飞书事件订阅有两种方式。**选 Webhook**（已定），长连接留到后续。

| | Webhook（本期） | 长连接（后续可选） |
|---|---|---|
| 公网入口 | **必须有**（IPv4 + https） | 不需要，能出网即可 |
| 本地开发 | 需内网穿透 | 直接可用 |
| 验签 / 解密 | 自己做（签名 + AES-256-CBC） | SDK 内置 |
| 多实例 | ✅ 正常负载均衡 | ⚠️ 集群模式，随机投给一个实例 |
| 可观测 | ✅ 每个事件是一个 HTTP 请求，能被网关日志/APM 统一捕获 | ❌ SDK 内部行为，排查难 |
| 消息丢失 | ✅ 失败重推 4 次（15s/5min/1h/6h） | ❌ 断线期间静默丢失 |
| 基础设施复用 | ✅ TLS / 限流 / WAF / 灰度都由 ingress 承担 | ❌ 要单独处理 session affinity |

**选 Webhook 的真实理由不是「本地能不能开发」，而是线上形态**：多实例可正常负载均衡、
事件可被统一观测、复用现成的 ingress 能力，且失败有重推兜底。

### 3.1 公网入口（本地开发）

本地开发用**内网穿透**（ngrok / cpolar / frp）把 `chatdata-channel` 的 20000 端口映射出去。

⚠️ **这条会给开发带来实际摩擦，必须写进 README**：

- 免费版穿透的域名每次重启都变 → 每次都要去飞书后台改「请求地址」
- 飞书保存回调地址时会立即发一个 `challenge` 校验请求，**穿透未就绪时保存会失败**
- 顺序必须是：先起穿透 → 再起 channel → 最后去飞书后台保存地址

建议用带固定域名的穿透方案（cpolar 付费版 / 自建 frp），否则每次重启都要改配置。

### 3.2 请求地址校验（challenge）

飞书保存回调地址时会 POST：

```json
{ "challenge": "ajls384kdjxxxx", "token": "<Verification Token>", "type": "url_verification" }
```

要求 **1 秒内**原样返回 `{"challenge": "..."}`。

配置了 Encrypt Key 时，请求体是 `{"encrypt": "..."}`，需先解密再取 `challenge`。

> **1 秒**比 3 秒还紧。这个端点必须独立于业务处理路径，不能排在工作队列之后。

### 3.3 验签与解密

配置 Encrypt Key 后：

**签名**（`X-Lark-Signature`）：

```java
bytes b1 = (X-Lark-Request-Timestamp + X-Lark-Request-Nonce + encryptKey).getBytes(UTF_8);
bytes b  = concat(b1, rawRequestBody);      // raw body，不是解析后的 JSON
hex     s = sha256(b);
assert s.equals(X-Lark-Signature);          // 大小写不敏感比较
```

**解密**（AES-256-CBC）：

```java
key     = sha256(encryptKey)                        // 32 字节
raw     = base64Decode(encrypt);                    // base64(iv + ciphertext)
iv      = raw[0..16]
plain   = AES/CBC/PKCS7Padding.decrypt(key, iv, raw[16..])
```

**必须用 Encrypt Key。** 不配的话事件明文传输，Verification Token 也会明文出现——
而 body 里含用户提问原文与 open_id。

> ⚠️ **签名校验必须用原始 body 字节。** Spring 的 `@RequestBody` 已经把 body 反序列化成对象，
> 再 `toString()` 得到的字节与原文不同（空格、键序、Unicode 转义），签名为假失败。
> 实现上要用 `ContentCachingRequestWrapper` 或直接读 `InputStream` 一次缓存。
> **这是这类校验最常见的坑**，写测试钉住：同一条 body 重放到两次请求，两次都要通过。

### 3.4 幂等：TTL 必须覆盖 6 小时

飞书的重推间隔是 **15 秒 → 5 分钟 → 1 小时 → 6 小时，最多 4 次**。
最坏情况下，一条消息会在 **6 小时后**被重推。

所以去重窗口必须 **≥ 6 小时**（建议 24 小时，留余量）。用 10 分钟会让 6 小时后的重推
被当成新消息——用户收到一条 6 小时前问题的答案，且是重复的。

```sql
CREATE TABLE channel_inbound_dedup (
  channel    VARCHAR(32)  NOT NULL,
  message_id VARCHAR(128) NOT NULL,
  received_at DATETIME    NOT NULL,
  PRIMARY KEY (channel, message_id),
  KEY idx_dedup_received (received_at)
);
```

用**表**而不是 Redis：6 小时的 TTL 已经超出「缓存」的语义，而丢失去重记录会导致重复回答。
清理用定时任务（参照 `AskAuditRetentionJob`），保留 24 小时。

### 3.5 用哪个 id 去重

飞书两份文档给的答案不同，必须写清楚：

| 文档 | 建议 |
|---|---|
| 事件概述（推送机制） | 2.0 事件用 `event_id` |
| 接收消息 v2.0（事件本身） | **用 `message_id`，不要依赖 `event_id`** |

**取严格的那个：`message_id`。**

理由：`event_id` 标识「一次推送」，`message_id` 标识「一条消息」。
如果因为业务失败导致同一条消息被飞书重推、而 `event_id` 变了，用 `event_id` 去重会漏。
`message_id` 在重推时不变，是唯一稳定的键。

> 这也解释了为什么文档会不一致：`event_id` 适合「防重复推送」，
> 而我们要防的是「同一个问题被答两遍」——那是消息级的语义。

### 3.6 3 秒约束

飞书要求 **3 秒内返回 HTTP 200**（TCP 建连超时 2 秒），否则触发重推。

```
① Webhook 线程（< 3s，硬约束）
   ├─ 验签（用原始 body）
   ├─ 解密
   ├─ 若是 url_verification → 直接返回 challenge（< 1s）
   ├─ 提取 message_id / chat_id / open_id / text
   ├─ 幂等查重（INSERT ... ON DUPLICATE KEY，返回影响行数）
   │     └─ 已存在 → 直接 200，不处理
   ├─ 投递有界队列（满 → 200 + 异步回「繁忙」，不丢弃 HTTP 200）
   └─ return 200
```

**队列满也必须返回 200。** 返回非 200 会触发飞书重推 4 次，
而那些重推同样会失败——只会把队列压得更死。正确做法是吞下、异步告知用户。

## 4. 身份：两步换令牌，IAM 侧新增自定义 grant

### 4.1 问题

ask 的每个请求必须带 `Authorization: Bearer <IAM令牌>`，`AskSecurityConfig` 直接 `authenticated()`。
令牌内省出来的 `orgCodePaths` / `roles` 就是行规则的判据——**这是数据权限的唯一来源**（ADR-0002）。

飞书消息带的是 `open_id`。它证明「这个人是谁」，但 IAM 不认。

### 4.2 为什么不能用内置的 RFC 8693 Token Exchange

Spring Authorization Server 7.1.0 **内置了** `OAuth2TokenExchangeAuthenticationProvider`，
参数齐全（`subject_token` / `actor_token` / `audience` / `resource`）。
但它第 127 行做的是：

```java
.subjectToken 查 OAuth2Authorization
if (subjectAuthorization == null) throw new OAuth2AuthenticationException(INVALID_GRANT);
```

也就是 `subject_token` **必须是一个已存在于 `oauth2_authorization` 表里的令牌**。
飞书用户从没登录过 Web 端，库里没有他的令牌可换。

要绕过只能自己伪造一张 subject_token——那比直接写自定义 grant 更复杂，且留下一张不该存在的令牌。

**结论：注册自定义 grant type，不走内置 exchange。**

### 4.3 方案

```
飞书消息(open_id)
      │
      ▼
chatdata-channel ──① 查 iam_user (provider=feishu, external_id=open_id)
      │                    └─ 查不到 → 回「请先绑定」+ 绑定链接
      │
      ├─② 用自己的 client 凭据 POST /oauth2/token
      │     grant_type=urn:chatdata:params:oauth:grant-type:channel
      │     channel_user=<iam username>   ← 不是 user_id，见下
      │
      ▼
IAM 自定义 provider
      ├─ 校验 client 在 channel 白名单内（配置，默认空 = 全禁）
      ├─ 校验该 username 确实绑定了本 channel 的 external_id
      └─ 签发 access_token（REFERENCE 格式，与现有客户端一致）
      │
      ▼
chatdata-channel ──③ 带令牌调 ask，与浏览器完全同一条路径
```

**关键设计：提交 username 而不是 userId，且必须验反查。**
IAM 侧拿 `(channel=feishu, username)` 反查 `iam_user.external_id`，
确认这个 username **确实**绑了飞书，且绑定记录与请求来源一致。

这样即使 channel 的 client secret 泄露，攻击者也只能冒充「已绑定过飞书的人」，
而不是任意用户。若只校验白名单不反查，等于给了一把万能钥匙。

`channel_user` 参数用 username 而非 userId，因为审计链路（`ask_audit_log.username`）
与前端会话（`ask_conversation.username`）都以 username 为键，少一次映射。

### 4.4 令牌规格

| 项 | 值 | 理由 |
|---|---|---|
| 格式 | `REFERENCE`（不透明） | 与 `AuthorizationServerConfig` 现有全部客户端一致 |
| TTL | 5 分钟 | channel 拿到后立刻用来调 ask，不需要长命；短 TTL 削弱泄露影响面 |
| scope | `ask.read`、`ask.feedback` | 与 BFF 客户端同权，channel 不该有更高权限 |
| refresh | **不签发** | channel 每次现场换，不需要续期；省掉轮换逻辑 |
| 审计标识 | `oauth2_authorization.principal_name` = 真实用户 | 令牌来源可查 |

> ⚠️ **必须核实**：`OAuth2AccessTokenGenerator` 生成 REFERENCE 令牌时，
> `principal_name` 取自 `OAuth2TokenContext.getPrincipal().getName()`。
> 自定义 provider 要在 context 里放**目标用户**的 principal，否则审计里全是 `channel-bot`。

### 4.5 IAM 客户端注册

新增 `channel-bot` 客户端（与 `ask-resource-server` 并列）：

```java
RegisteredClient.withId(UUID.randomUUID().toString())
    .clientId("channel-bot")
    .clientSecret(encoder.encode(props.getClients().getChannelSecret()))
    .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
    .authorizationGrantType(new AuthorizationGrantType("urn:chatdata:params:oauth:grant-type:channel"))
    .scope("ask.read").scope("ask.feedback")
    .tokenSettings(referenceTokens())
    .build();
```

密钥走 `IAM_CHANNEL_SECRET` 环境变量，**不设默认值**——与 `IAM_BFF_SECRET` 同样的取舍。

## 5. 身份绑定：两种方式并存

用户 A 是管理员批量导入的，用户 B 想自助。

### 5.1 管理员侧（方式 a）

IAM 管理台「用户」页加两列输入：外部身份来源、外部 ID。
写入 `iam_user.external_provider` / `external_id`。

- 需要 `iam.users.write` 权限（已有）
- 支持批量：CSV 导入 `username,provider,external_id`
- 唯一约束冲突时给明确错误（「该飞书账号已绑定到 zhangsan」）

### 5.2 自助侧（方式 b）

用户在飞书给机器人发任意消息，若 `open_id` 未绑定：

```
机器人：你还没有绑定问数账号。
       [ 立即绑定 ]   ← 链接到 http://<gateway>/app/bind?ticket=<token>
```

点击后跳 Web 端，已登录（或先登录）则显示：

```
飞书账号「张三」(ou_xxx) 将绑定到当前账号 zhangsan。
[ 确认绑定 ]  [ 不是我，取消 ]
```

**ticket 的设计**（这是整条链路最容易写错的地方）：

| 性质 | 要求 |
|---|---|
| 存储 | Redis，key `channel:bind:<ticket>`，TTL 10 分钟 |
| 内容 | `{channel, externalId, displayName, createdAt}`，**不含用户身份** |
| 一次性 | 用掉即删（`getAndDelete`） |
| 长度 | 32 字节随机，base64url |
| 绑定方向 | ticket 只证明「某个飞书账号请求绑定」，绑定到**谁**由 Web 端当前登录用户决定 |

> **不能把 username 放进 ticket。** 那等于「谁拿到链接就能替别人绑定」。
> ticket 描述的是飞书侧的身份，用户侧身份只能来自 Web 端的登录会话。

绑定成功后 Web 端回显「已绑定，回飞书继续提问」。

### 5.3 解绑与冲突

- 用户可在 Web 端「我的账号」里解绑（`external_id` 置 NULL）
- 一个人只能绑一个飞书号（唯一约束保证）
- 一个飞书号只能绑一个人（同一约束的反向保证）
- 解绑后发消息 → 回到「请先绑定」分支

## 6. 消息处理

### 6.1 处理流程（3 秒硬约束）

Webhook 的验签、解密、幂等、入队细节见 §3.6。业务侧从这里接：

```
② 工作线程（无 3 秒限制，但仍受 ask 超时约束）
   ├─ 查/建会话映射（§6.6）
   ├─ 换 IAM 令牌（§4）
   ├─ 调 ask SSE，**只取 result / error 事件**
   ├─ 渲染飞书卡片（§6.4）
   └─ 发送回复（或 PATCH 占位消息，§6.2）
```

**幂等键用 `message_id`**，理由与 TTL 见 §3.4 / §3.5。

### 6.2 「正在查询」的两段式回复

用户发出问题后立刻看到反馈，否则 5–15 秒的沉默会被当成没发出去。

```
T+0s     发一张卡片：「已收到，正在查询…」（带一个转圈的表情）
T+8s     PATCH 同一张卡片，替换为结果
超时     替换为「查询超时，请重试」
```

限流：飞书消息更新有频率限制（约 5 次/秒/消息）。
**只更新两次**（占位 + 结果），不做进度条——问数的进度事件对用户没有信息量。

若更新失败（网络抖动），兜底为「另发一条新消息」，并在日志里记 reason。

### 6.3 不接流式

ask 的 SSE 是**事件级**的（ADR-0003）：`started` / `progress` / `heartbeat` / `result` / `error`。
channel **只消费 `result` 和 `error`**，其余丢弃。

理由：`progress` 的内容是「正在生成 DSL」「正在执行 SQL」，对业务人员是噪音；
而逐字流式在卡片里会引发大量 PATCH，撞限流。

实现上仍然是**读 SSE 流**，只是不转发。这样做的好处是拿到同一个终态结构，
不用新写一个同步接口——少一个接口就少一处与 ask 行为不一致的地方。

### 6.4 结果渲染

卡片结构：

```
┌─────────────────────────────────────┐
│ 📊 上周华东订单金额                   │
│                                     │
│ 合计 ¥1,284,300                     │  ← 聚合结论，最大字号
│ 较前一周 +12.3%                      │
│                                     │
│ ┌────────┬──────────┬──────────┐    │
│ │ 门店    │ 订单金额  │ 占比      │    │  ← 最多 10 行
│ ├────────┼──────────┼──────────┤    │
│ │ 上海门店1│ ¥482,100 │ 37.5%    │    │
│ │ ...    │          │          │    │
│ └────────┴──────────┴──────────┘    │
│ （共 47 行，仅展示前 10 行）           │
│                                     │
│ [ 查看完整结果 ]              [ 有用 ] │
│                                     │
│ ▸ 查看 SQL                           │  ← 折叠面板，默认收起
└─────────────────────────────────────┘
```

**列数上限 5，行数上限 10。** 飞书卡片在手机端约 20 字符宽，
列多会折行到不可读。超出的用「查看完整结果」兜底。

- **聚合行**（结果只有一行一列）：直接大字号展示数值
- **明细行**：表格
- **空结果**：明确说「没有符合条件的数据」，而不是空白卡片
- **被拒**（权限不足）：说清是权限问题，并列出当前角色

#### SQL 用折叠面板，不做服务端开关

用 `collapsible_panel` 组件，`expanded: false`：

```json
{
  "tag": "collapsible_panel",
  "expanded": false,
  "header": { "title": { "tag": "markdown", "content": "查看 SQL" } },
  "elements": [
    { "tag": "markdown", "content": "```sql\nSELECT ...\n```" }
  ]
}
```

**展开状态是客户端本地的**，不需要服务端回调，也不需要卡片按钮。

> 为什么不用「点按钮 → 回调 → PATCH 卡片」：那要额外处理 `card.action.trigger` 回调、
> 按钮去重、以及 PATCH 失败的重试。折叠面板零后端成本，且默认收起已满足
> 「默认关闭」的要求。**能不给服务端添路径就不添。**

前提：折叠面板需要飞书客户端 **V7.9+**。低版本会显示「请升级客户端」占位——
这不影响主信息（结论 + 表格）的呈现，可接受。

### 6.5 「查看完整结果」链接

带一次性 ticket 跳到 Web 端 `/app/ask?conv=<conversationId>`：

- ticket 存在 Redis，TTL 5 分钟，一次性
- Web 端凭 ticket 换取**该用户自己的**会话访问权（仍需登录）
- 不做「免登录查看」——那等于把数据权限开个洞

**为什么不让 channel 自己渲染图片**：那要引入无头浏览器，多一个进程、
多一份漏洞面、多一次截图失败的失败态。链路先跑通，需要再加。

### 6.6 多轮会话

`chat_id`（群）或 `open_id`（私聊）→ `ask_conversation.id` 的映射：

```sql
CREATE TABLE channel_session (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  channel       VARCHAR(32)  NOT NULL,   -- feishu
  chat_key      VARCHAR(128) NOT NULL,   -- 私聊=open_id；群聊=chat_id:open_id（按人隔离，见 §6.6）
  username      VARCHAR(64)  NOT NULL,   -- 归属人
  conversation_id VARCHAR(64) NOT NULL,  -- ask_conversation.id
  last_active_at DATETIME    NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_channel_chat (channel, chat_key),
  KEY idx_channel_last_active (last_active_at)
);
```

- 空闲超过 **60 分钟** → 下次提问开新会话（用户说「那上个月呢」时，
  一小时前的话题已经不在了，接上反而困惑）
- 用户显式说「新话题」「重置」→ 立即开新会话
- **群聊里切换发言人要开新会话**：张三在群里问完，李四接着问，
  不嫁接。判据是 `open_id` 变了

#### 群聊：直接在群里引用回复（已定）

回复用 `POST /im/v1/messages/:message_id/reply`，引用提问者那条消息：

```json
{ "msg_type": "interactive", "content": "<卡片 JSON>" }
```

`message_id` 取自事件体，不需要额外查询。回复会挂在原消息下方，形成引用关系。

> 限频：向同一群组发消息，**群内所有机器人共享 5 QPS**。
> 单条回复不受影响，但群里有多个机器人时要注意总量。

**⚠️ 群聊的数据可见性必须明确接受。**

张三有华东权限、李四只有华南。张三在群里问「上周订单金额」，
卡片发到群里后，**李四和群里任何人都能看到张三权限范围内的数据**。

这不是 bug，是「群里发言即公开」的固有语义——飞书/钉钉/企微都一样。
接受它需要满足以下**全部**条件：

1. **群里的人本来就该看到同样的数据**。例如「华东大区群」里所有人都是华东团队，
   那按张三的权限查出的结果，对群里其他人也不违规。
2. **不在跨权限的群里用问数**。如「全员群」「跨部门群」——那里一刀切地不该用。

**由此推出的硬约束：**

| 场景 | 行为 |
|---|---|
| 私聊 | 正常回复 |
| 群聊 @ 机器人 | 正常引用回复（就在群里） |
| 群聊但发言人无权限的域 | 拒绝，且**拒绝理由只说「无权限」**，不说具体是什么域，避免泄露配置 |
| 群聊结果含被脱敏列 | 照常展示脱敏值，群里看不到原值 |

> **本条已定，不再讨论「是否只私聊回复」。** 但要在 `docs/` 里留一条给部署者的提示：
> 若某个群是跨权限的，应把机器人移出该群，而不是靠代码兜底——
> 代码无法知道「这个群里的人该不该互相看到数据」。

##### 会话归属：群里按发言人隔离

群 `chat_id` 相同，但张三和李四的会话**必须分开**——否则李四的追问会接上张三的上下文，
而那条上下文里有张三权限下的数据。

判据：`chat_key = chat_id + ":" + open_id`。换人即换会话。

## 7. 模块设计

```
chatdata-channel/                        新模块，独立 Spring Boot 应用
├── application.yml
├── config/
│   ├── FeishuProperties                  appId / appSecret / encryptKey
│   ├── ChannelProperties                 IAM 地址、channel client 密钥、队列容量
│   └── FeishuClientConfig                Lark SDK Client 装配（发消息用）
├── inbound/
│   ├── InboundMessage                    统一入站消息：channel/chatKey/senderId/text/msgId
│   ├── FeishuEventController             POST /channel/feishu/event（验签→解密→幂等→入队）
│   ├── FeishuSignatureVerifier           签名校验（必须用原始 body 字节，见 §3.3）
│   ├── FeishuEventDecryptor              AES-256-CBC 解密
│   ├── InboundDedup                      幂等表（message_id，24 小时窗口，见 §3.4）
│   └── InboundQueue                      有界队列 + 工作线程池
├── identity/
│   ├── ChannelIdentityService            open_id → iam username，含绑定 ticket 签发
│   └── ChannelTokenService               自定义 grant 换 IAM 令牌，带本地缓存
├── ask/
│   ├── AskClient                         调 ask SSE，只取 result/error
│   └── AskResult                         简化的结果模型
├── render/
│   ├── FeishuCardRenderer                AskResult → 卡片 JSON
│   └── FeishuSender                      发消息 / PATCH 更新
├── session/
│   └── ChannelSessionService             chat_key → conversationId 映射
└── web/
    ├── BindController                    /channel/bind/confirm、/channel/bind/info
    └── ShareController                   /channel/share/<ticket> 换会话访问权
```

### 模块边界

| 模块 | 依赖 | 不依赖 |
|---|---|---|
| `chatdata-channel` | `chatdata-identity-client`（复用 `IamIntrospector`? 否——channel 不内省，只调 IAM） | `chatdata-query-engine`、`chatdata-query-service` 的任何代码 |
| `chatdata-channel` → ask | **只通过 HTTP** | — |

**不给 channel 依赖 engine**。它只需要「把问题变成结果」，结果的结构由 HTTP 契约定义。
一旦依赖 engine，channel 就会开始想要 `PermissionContext`、想要 DSL——那是错的。

### 部署形态

独立进程，独立端口（`20000`）。**不进网关**——网关是给浏览器的，channel 是给飞书的。
两边都能访问 ask，各走各的鉴权。

**暴露面**：只有 `POST /channel/feishu/event` 一个端点需要公网可达。
其余端点（绑定确认、分享换权）走网关，与 Web 端同一套会话鉴权。

因此反向代理要按路径分流：

```
公网 → /channel/feishu/event   → channel:20000   （无会话，靠飞书签名）
公网 → 其余一切                 → gateway:8080
```

> 不要把 channel 的整个端口暴露到公网。绑定确认接口如果公网可达且没有会话保护，
> 就成了「谁都能替别人完成绑定」的入口。

## 8. 配置

```yaml
channel:
  feishu:
    enabled: true
    app-id: ${FEISHU_APP_ID:}
    app-secret: ${FEISHU_APP_SECRET:}
    # 加密策略。**必填**——不配则事件明文传输，body 含用户提问原文与 open_id
    encrypt-key: ${FEISHU_ENCRYPT_KEY:}
    # 未配置 Encrypt Key 时才用得上；配置了就只做签名校验
    verification-token: ${FEISHU_VERIFICATION_TOKEN:}
    # 群聊：直接在群里引用回复（见 §6.6）。取值 in-group | private-only
    group-reply-mode: in-group

  iam:
    base-url: ${IAM_ISSUER:http://127.0.0.1:9000}
    client-id: channel-bot
    client-secret: ${IAM_CHANNEL_SECRET:}       # 无默认值

  ask:
    base-url: ${ASK_URI:http://127.0.0.1:8081}
    timeout-seconds: 120

  queue:
    capacity: 200                                # 有界；满了仍回 200，异步告知用户（§3.6）
    workers: 8

  dedup:
    # 必须 ≥ 6 小时：飞书最晚一条重推在 6 小时后（§3.4）
    retention-hours: 24

  session:
    idle-minutes: 60
```

**密钥一律无默认值。** 缺失时启动失败并说明缺哪个——
与 `PasswordCipher`、`seedClients` 的取舍一致（见 §4.5）。

## 9. 为什么不现在抽多 channel

后续要接钉钉、企业微信。但**现在抽接口大概率是错的**：

- 三个实现都没写，抽象出的接口是「想象中的共性」
- 飞书可 webhook、钉钉有 Stream 模式、企业微信只有 webhook——传输层差异比想象大
- 消息模型差异更大：飞书卡片 / 钉钉 ActionCard / 企微 markdown，无法统一成一套 DSL

所以第一阶段：

- 不建 `ChannelInbound` 接口。飞书直接落地为 `FeishuEventController`
- `InboundMessage` 只含飞书确实能提供的字段，不做「通用消息模型」
- 卡片渲染**直接写飞书 JSON**，不建中间层

**什么时候抽**：接第二个 channel 时，把两个实现的差异摆在一起看，
那时候抽出来的才是真的共性。这是「三次法则」的应用——第二次是复制，第三次才抽。

> 这条判断写进 spec 是为了**抵住「顺便抽象一下」的诱惑**。抽象的成本不在于写接口，
> 在于接口定错之后所有实现都要跟着改，而且改的时候看起来是「各实现自己有问题」。

## 10. 测试

### 单元

| 用例 | 断言 |
|---|---|
| 同一 `message_id` 投递两次 | 第二次不调 ask，不重复回复 |
| 未绑定用户 | 不发 ask 请求，返回绑定引导 |
| 绑定 ticket 用两次 | 第二次失败（一次性） |
| 群里换发言人 | 新开会话（`chat_key` 含 open_id），不继承上文 |
| 群里同一人连问 | 接上文 |
| 群里 @ 机器人 | 用 reply 接口引用原消息，不是发新消息 |
| 会话空闲 31 分钟 | 新开会话 |
| 结果 47 行 | 卡片只含 10 行 + 「共 47 行」 |
| 结果 0 行 | 明确文案，不是空卡片 |
| SQL 默认收起 | 卡片 JSON 里 `collapsible_panel.expanded == false` |
| ask 返回 403 | 卡片说明是权限问题，不暴露内部细节 |
| 令牌缓存过期 | 重新换令牌，不报错 |

### Webhook 入站（重点，易写错）

| 用例 | 断言 |
|---|---|
| `url_verification` 请求 | 1 秒内原样返回 challenge |
| 签名正确 + 加密事件 | 能解出明文并处理 |
| 签名错误 | 401，且**不处理**；记录来源 IP |
| **同一 body 重放两次** | 两次签名校验都通过（证明用的是原始字节，不是反序列化产物） |
| body 含中文/特殊字符 | 签名仍通过（UTF-8 字节一致） |
| 同一 `message_id` 24 小时内重推 | 第二次不调 ask |
| 队列已满 | 返回 200（不是 5xx），异步告知用户 |
| 未配置 Encrypt Key | **启动失败**（不允许明文跑） |

### 集成（需真实依赖）

| 用例 | 依赖 |
|---|---|
| 自定义 grant 换令牌 | IAM + MySQL |
| 换到的令牌能调 ask 并拿到 `orgCodePaths` | IAM + ask + MySQL + Redis |
| 绑定写入 `iam_user.external_id` | IAM + MySQL |
| 唯一约束冲突处理 | MySQL |

### 手工验证清单（飞书侧无法自动测）

- [ ] 穿透地址配置到飞书后台时 challenge 校验通过
- [ ] 私聊 @ 机器人得到卡片回复
- [ ] 「那上个月呢」接上上文
- [ ] 无权限用户看到权限提示而非空结果
- [ ] 「查看完整结果」跳转后看到同一份数据
- [ ] 解绑后再问，回到绑定引导

### 安全测试（必做）

| 用例 | 断言 |
|---|---|
| 用 `channel-bot` 为**未绑定**的用户换令牌 | 拒绝，`invalid_grant` |
| 用 `channel-bot` 为**绑定了别的飞书号**的用户换令牌 | 拒绝 |
| 用 `channel-bot` 为**任意**用户换令牌（无绑定关系） | 拒绝 |
| 伪造绑定 ticket | 拒绝 |
| 用过期的 ticket | 拒绝 |
| 用别人的 ticket 绑定到自己账号 | **拒绝**（ticket 不含用户身份，但需验证 ticket 与当前登录用户的关系） |

## 11. 实施顺序

按依赖排，每一步都能独立验证：

| # | 内容 | 可验证的产出 | 预估 |
|---|---|---|---|
| 1 | IAM 自定义 grant + `channel-bot` 客户端 | curl 能换到令牌，能调通 ask | 2–3 天 |
| 2 | 绑定：`external_id` 写入 + 管理台两列 | 管理台能手动绑 | 1 天 |
| 3 | 自助绑定：ticket + Web 页 | 飞书里点链接能绑 | 1–2 天 |
| 4 | `chatdata-channel` 骨架 + webhook 端点 + 验签解密 | 穿透后飞书保存回调地址成功；发消息日志能看到 | 2 天 |
| 5 | 换令牌 + 调 ask + 纯文本回复 | 端到端跑通（无卡片） | 2 天 |
| 6 | 卡片渲染 + 两段式更新 | 完整交互 | 2 天 |
| 7 | 会话映射 + 多轮 | 「那上个月呢」 | 1–2 天 |
| 8 | 「查看完整结果」链接 | 跳 Web 端 | 1 天 |
| 9 | 测试与安全用例 | 全绿 | 2 天 |

**第 5 步是最小可用切片**：能问、能答、身份对、权限生效。之后每一步都是体验优化。

## 12. 风险与未决

| 风险 | 应对 |
|---|---|
| 3 秒内处理不完（如 IAM 慢） | 队列 + 立即返回；换令牌失败时异步回错误卡片，不让飞书重推 |
| `principal_name` 记成 bot 而非用户 | §4.4 的核实项；写测试守护 |
| 群聊数据泄露 | 已在 §6.6 明确接受该语义并给出使用条件；会话按发言人隔离 |
| 令牌 TTL 5 分钟不够重试 | channel 侧缓存，过期前 30 秒主动刷新 |
| 签名校验用了解析后的 body | §3.3；必须用原始字节，写测试守护（同一 body 重放两次都通过） |
| 内网穿透地址变动 | 每次重启要改飞书后台地址；建议固定域名，或开发期改用长连接（后续） |
| 幂等窗口过短 | 必须 ≥ 6 小时（飞书最晚重推在 6 小时后）；用表不用 Redis，保留 24 小时 |
| channel 整个端口暴露公网 | 只放行 `/channel/feishu/event`，绑定接口必须经网关（§7 部署形态） |
| 飞书 API 限流 | 主动限流（每 chat 每秒 1 条），溢出时排队而非丢弃 |

### 已定（2026-10-28）

| 项 | 决定 |
|---|---|
| 传输 | Webhook，长连接留到后续 |
| 群聊 | **直接在群里引用回复**（`reply` 接口），数据可见性语义已在 §6.6 明确接受 |
| 会话空闲 | **60 分钟** |
| SQL 展示 | `collapsible_panel`，默认收起，所有人可展开；不需要服务端回调 |
| `channel_user` | 用 username（单租户够用；多租户时改 `(tenant, username)`） |
