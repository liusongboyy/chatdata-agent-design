# Agent Mesh：把问数暴露成可被发现的 A2A agent

- Status: Draft
- Date: 2026-10-30
- Related: `docs/adr/0002-data-permission-enforcement-point.md`、`docs/adr/0008-iam-oauth21-opaque-tokens.md`、`docs/adr/0011-channel-owns-integration-iam-owns-identity.md`、`.scratch/feishu-channel/spec.md`

## 1. 场景

公司里其他 agent（报表生成、经营分析、自动周报…）需要「查一下数」时，不应该各自接一套
数据源、各自写 SQL、各自实现权限——那会变成 N 个权限判定点。

问数系统把能力暴露成一个 **A2A agent**：注册到 Nacos，其他 agent 通过 A2A 协议发现并调用它。
调用时说明「代表谁问」，问数系统用那个人的身份去查——权限与 Web 端完全一致。

同时它要能被 **agentscope service** 的管理台看到、被编排。

### 非目标

- 不替代现有的浏览器/IM 入口。A2A 是**第三个入口**，与它们并列。
- 不暴露写操作。本期只有查询。
- 不做 agent 之间的自主协商（多 agent 协作编排）。那是调用方的事。
- **不改系统提示词的来源**。提示词现在是程序动态拼的（表结构、字段别名、范例），
  挪到 Nacos 需要先把「模板」与「运行时上下文」拆干净，那是独立的一次改动。

## 2. 现状与缺口

### 已具备

| 能力 | 位置 | 说明 |
|---|---|---|
| 问数 HTTP 接口 | `POST /api/ask/engine`（SSE） | mesh 直接复用，不新增接口 |
| 代表用户换令牌 | IAM 自定义 grant `urn:chatdata:...:channel` | channel 已验证；mesh 注册自己的客户端即可复用 |
| 能力发现 | `GET /api/ask/assistants`、`/capabilities` | 可作为 AgentCard 的技能来源 |
| agentscope 2.0.4 全套 | 本地/中央仓库 | A2A starter、a2a-server、nacos-a2a 全部 2.0.4 可用 |
| Nacos 注册实现 | `NacosAgentRegistry`（现成） | 不用自己写注册逻辑 |

### 缺口

| 缺口 | 影响 |
|---|---|
| 无 A2A 端点 | 其他 agent 发现不了、也调不通 |
| 无「代理用户」的授权机制 | 谁能被代理没有判据（§4 详解，本 spec 最重的一节） |
| A2A 客户端无强认证 | A2A 协议靠传输层认证，而我们只有「可信网络」这个模糊前提 |
| 技能描述要手写 | 数据域与 skill 的映射关系没建立 |

## 3. 已核实的技术事实

写方案前核过源码，这几条决定实现形态，**不要再凭印象改**：

| 事实 | 来源 | 影响 |
|---|---|---|
| agentscope 2.0.4 的 A2A 全套齐备 | 中央仓库 POM 探测（`maven-metadata.xml` 的 `<latest>` 会撒谎，要直接探 POM 是否存在） | 版本不用动 |
| `AgentRunner` 只有 4 个方法 | `javap` | 适配层很薄：`getAgentName` / `getAgentDescription` / `streamEvents` / `stop` |
| **`AgentRequestOptions` 自带 `userId`** | `javap` | 「代表谁」有官方位置，**不用自造请求头** |
| `AgentRegistry` 只有 2 个方法 | `javap` | `registryName` / `register(AgentCard, transports)` |
| `NacosAgentRegistry` 是现成实现 | 2.0.4 jar | 不写注册逻辑，只装配 |
| `NacosA2aRegistryProperties` 有 `enabledRegisterEndpoint`、`transportProperties`（含 host/port/path/protocol） | `javap` | 注册内容由这些属性决定 |
| A2A starter 对 `agentscope-core` 是 `provided`+`optional` | 2.0.4 POM | 不会强拉版本，与主工程一致 |

## 4. 身份与信任（本 spec 最重的一节）

### 4.1 问题

A2A 请求来自**其他 agent**，但它要读的是**某个人的**数据：
同一个「上周订单金额」，张三看华东、李四只能看华南。

所以调用方必须说明「代表谁」。**而这句话是调用方说的——无条件相信就是后门。**

### 4.2 「代表谁」放在哪

用 A2A 的 `AgentRequestOptions.userId`（已核实存在）。语义约定：

```
userId = "user:zhangsan"     // 前缀标识这是"代理某人"，不是 agent 自己的身份
userId = null 或空           // 不是代理请求 → 一律拒绝
```

**为什么加前缀**：将来可能出现「agent 以自己的身份查公共数据」的场景，
那时 `userId = "agent:report-bot"`。现在只实现 `user:` 分支并拒绝其他形态——
少一种形态就少一条要保护的路径。

### 4.3 三道限制

| # | 限制 | 不做的后果 | 认证强度 |
|---|---|---|---|
| 1 | 客户端白名单（`X-Agent-Client`） | 内网任何服务都能调 | 软——头可伪造 |
| 2 | 目标用户存在且启用 | 报错不明确 | — |
| 3 | **用户在授权表里授权了该 agent** | 说出管理员用户名就能以他身份查数 | **硬**——需要用户主动同意 |

第 3 道是核心，也是唯一真正的边界。它让「谁能被代理」成为**用户自己的决定**，
而不是调用方的。

### 4.4 第 3 道怎么表达：用户自己的授权表

**不是权限码，是一张用户可以自己填的授权表。**

飞书的绑定之所以安全，是因为**用户本人参与了**：他扫码、登录、点确认。
直接照搬「绑定」这个词不行——agent 不是用户主动加的东西。
但那个**形态**是对的：让用户显式同意一次，之后一直有效。

```sql
CREATE TABLE `iam_agent_grant` (
  `username`     VARCHAR(64) NOT NULL COMMENT '被代理的用户',
  `agent_client` VARCHAR(64) NOT NULL COMMENT '允许哪个 agent 代理',
  `granted_at`   DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  -- 软撤销：留记录才能回答"他什么时候取消的"
  `revoked_at`   DATETIME    NULL,
  PRIMARY KEY (`username`, `agent_client`),
  KEY `idx_grant_client` (`agent_client`)
) COMMENT = '用户授权某 agent 代表自己查数';
```

判据变成**两条同时成立**：

```sql
SELECT 1 FROM iam_agent_grant
WHERE username = ? AND agent_client = ? AND revoked_at IS NULL
```

#### 为什么比权限码好

| | 权限码 `agent.proxy` | 授权表 |
|---|---|---|
| 谁授权 | 管理员改角色 | **用户自己** |
| 粒度 | 有/没有 | **按 agent** |
| 撤销 | 改角色 | 用户自己点关闭 |
| 审计 | 弱 | **有授受记录** |

按 agent 授权是关键：用户可以只允许「周报助手」，不允许「随便什么 agent」。

#### Web 端「我的账号」页

```
允许 AI 助手代表我查数

  [x] 周报助手              2026-10-30 授权    [撤销]
  [ ] 经营分析助手                            [授权]

  开启后，被授权的 AI 助手可以以你的身份查询。
  它们看到的数据范围与你本人完全相同——不会更多。
```

#### 用户能自己授权的风险

用户可能不看清楚就点「授权」，然后某个 agent 就能代表他查数。

这不算越权（agent 看到的仍是他本人的数据范围），但仍是**用户主动扩大了暴露面**。
所以：

- 授权页必须写明「它能看到的数据与你本人相同」
- 每个 agent 的名称与描述要来自**可信来源**（Nacos 注册的 AgentCard），
  不能让 agent 自报一个「安全的名字」骗用户点授权

> 第二条是这套方案唯一的薄弱点：agent 名字由 agent 自己声明。
> 缓解办法是 agent 注册时由管理员审核（Nacos 侧的流程，不在本期）。

### 4.5 第 1 道：客户端白名单

**A2A 协议本身不认证调用方**（它假定传输层已认证）。本期用白名单做软校验：

- 配置 `mesh.identity.allowed-clients`（如 `report-bot,ops-agent`）
- A2A 请求头 `X-Agent-Client` 带上自己的身份
- 不在白名单 → 拒绝
- **白名单里的 client 值，必须与授权表里的 `agent_client` 一致**

**必须写清楚：这不是安全边界。** 伪造这个头是零成本的。
真正的边界是第 3 道（用户的授权表）。

**要真正认证调用方，必须靠传输层**——mTLS 或 API 网关的客户端凭证。
列为待办（§11），不在本期实现。

> 这条写进 spec 是为了**抵住「我们在内网所以没事」的自我安慰**。
> 内网横向移动是最常见的攻击路径，而它恰恰是"可信网络"假设失效的地方。

#### `userId` 的位置：`message.metadata`，不是 `params.metadata`

A2A 调用方最容易写错的一处。starter 的 `AgentScopeAgentExecutor.getUserId(Message)` 读的是
**`params.message.metadata["userId"]`**：

```json
{
  "jsonrpc": "2.0", "id": "1", "method": "message/send",
  "params": {
    "message": {
      "kind": "message", "role": "user", "messageId": "...",
      "metadata": { "userId": "user:report-bot:admin" },
      "parts": [{"kind": "text", "text": "各门店的订单金额合计"}]
    }
  }
}
```

写在 `params.metadata`（少一层）的话，`getUserId` 拿到空串，mesh 回一句
「无法确定代表谁」——**HTTP 200、JSON-RPC 也没 error**，只是内容是一句拒绝。
排查看 mesh 日志里的 `A2A 调用：... userId=` 是不是空的。

另外 `X-Agent-Client` 这种 HTTP 头没用：starter 在异步线程池上调用 `streamEvents`，
请求线程的 ThreadLocal 传不过去。客户端标识只能编进 `userId`（`user:<client>:<user>`）。

### 4.6 完整链路

```
调用方 agent
   │  A2A: task + userId="user:zhangsan"
   │       header X-Agent-Client: report-bot
   ▼
mesh 端点
   │  ① report-bot 在白名单里？（软校验）
   │  ② 解析 userId 前缀
   ▼
IAM
   │  ③ 用户存在、启用
   │  ④ iam_agent_grant 里有 (zhangsan, report-bot) 且未撤销 ← 核心限制
   │  ⑤ 签发代表该用户的不透明令牌
   ▼
ask
   │  ⑥ 内省令牌 → 角色/组织 → 行级规则 → 列脱敏
   ▼
结果 → A2A artifact
```

**ask 一行不用改。** 它看到的仍是真实用户，权限判定逻辑完全复用。

#### 换令牌的 grant 要扩展

现有的 `urn:chatdata:...:channel` grant 只校验
`(external_provider, username, external_id)`——那是**渠道绑定**的形态。
mesh 要校验的是 `(username, agent_client)`，判据不同。

两个选择：

| 方案 | 说明 |
|---|---|
| A. 复用 channel grant，`channel=agent-mesh` | 但它的判据是 `external_id`，与授权表对不上 |
| **B. 新增一个 grant 变体** | `urn:chatdata:...:agent-proxy`，判据是授权表 |

**选 B。** 两种准入的**语义不同**：渠道绑定是"这个飞书号是谁"，
agent 代理是"这个人同意哪个 agent 代表他"。塞进同一个 grant 会让
判据变得含糊，而出问题时要能一眼看出走的是哪条路。

代价是 IAM 多一个 `AuthenticationProvider`——但那本来就该分开。

## 5. 技能建模

### 5.1 按数据域拆 skill

AgentCard 的 `skills` 不是一个大技能「能问所有数据」，而是**一个数据域一个 skill**：

```json
{
  "name": "问数助手",
  "description": "把自然语言问题翻译成语义层查询并执行",
  "skills": [
    {
      "id": "domain-2300",
      "name": "门店分析",
      "description": "回答门店销售额、库存、报废、打卡等经营问题。支持按组织、时间、商品维度聚合。",
      "tags": ["retail", "store", "sales"],
      "examples": ["上周各门店的订单金额", "库存周转天数是多少", "报废金额最高的门店"]
    }
  ]
}
```

**为什么拆开**：其他 agent 要判断「该不该用这个 agent」「用它查哪类数据」。
一个大 skill 说「能问所有数据」，调用方只能盲试；按域拆开，它能按 `tags` 选。

### 5.2 静态 vs 动态

| 方案 | 优点 | 缺点 |
|---|---|---|
| 配置文件写死 | 简单、可审 | 加数据域要改配置重启 |
| 运行时从库读 | 自动跟随 | AgentCard 每次请求都查库，且**未授权的域会泄露存在性** |

**本期选静态配置**，理由是第二个缺点：AgentCard 是**对外公开**的，
从库里读会把"系统里有哪些数据域"暴露给所有能发现它的 agent——
那本身就是信息泄露。

将来加域时改配置重启，是可接受的运维成本。

### 5.3 本期只暴露「门店分析」

按你的决定，只暴露 `domain-2300`。理由写在这里备查：先验证整条链路，
skill 的数量不影响架构；等第二个域要暴露时再补。

## 6. 模块设计

```
chatdata-agent-mesh/                    新模块，独立 Spring Boot 应用
├── config/
│   └── MeshProperties                  IAM/ask 地址、客户端白名单、代理开关
├── a2a/
│   ├── AskAgentRunner                  实现 AgentRunner：4 个方法
│   └── AgentCardFactory                从配置构建 AgentCard（技能在此）
├── identity/
│   ├── ProxyTokenService               换代表用户的令牌（缓存、提前刷新）
│   └── ProxyTargetResolver             解析 userId 前缀 → 校验用户与授权
├── ask/
│   └── AskClient                       调 ask（与 channel 的 AskClient 同构，不共享）
└── AgentMeshApplication
```

### 模块边界

| 依赖 | 不依赖 |
|---|---|
| 通过 HTTP 调 ask | `chatdata-query-engine`、`chatdata-query-service` 的任何代码 |
| 通过 HTTP 调 IAM | `chatdata-identity-client`（不走内省——mesh 不是资源服务器） |
| 直连 IAM 库查 `iam_agent_grant` 授权表 | — |

**和 channel 一样不判断权限。** 它只做「谁要问什么 → 拿他的令牌 → 翻译结果」。

### 为什么是独立进程

与 channel 同理：ask 是资源服务器（内省用户令牌），A2A 来的是 agent 调用
（服务代理用户）。混进去会破坏现有鉴权模型。独立进程还让「A2A 端点挂了」
不影响 Web 与 IM。

### 不共享 channel 的 AskClient / ProxyTokenService

两处代码几乎一样，但**不抽公共模块**：

- 信任边界不同（飞书推送的人 vs 公司内 agent）
- 失败话术不同（给业务人员看 vs 给其他 agent 的开发者看）

重复几十行，代价远小于把两个信任边界混在一起。

## 7. A2A 适配层

### 7.1 AgentRunner 实现

```java
public class AskAgentRunner implements AgentRunner {
    public String getAgentName() { return "chatdata-ask"; }
    public String getAgentDescription() { return "语义层问数：把业务问题翻译成受权限约束的查询"; }
    public Flux<AgentEvent> streamEvents(List<Msg> messages, AgentRequestOptions options) { ... }
    public void stop(String taskId) { ... }
}
```

`streamEvents` 的处理：

```
① 取最后一条 Msg 的文本作为问题
② 解析 options.userId → 目标用户（不合法 → 发 error 事件，Flux.error）
③ 换该用户的令牌
④ 调 ask（SSE），只取 result/error
⑤ 转成 AgentEvent 序列发出去
```

### 7.2 结果怎么变成 artifact

A2A 的产物是 artifact。问数结果是表格，用 **DataPart**（结构化 JSON）而不是 TextPart：

```json
{
  "type": "data",
  "data": {
    "columns": ["所属组织", "交易金额"],
    "rows": [["上海门店1", 482100]],
    "rowCount": 47,
    "sql": "SELECT ...",
    "mode": "MODELED"
  }
}
```

**同时给一段 TextPart 摘要**（「上海门店1 交易金额 48.2 万，共 47 行」）——
有些调用方只读文本。

> 不给全量 rows（最多 1000 行）塞 artifact：A2A 的消息会有大小限制，
> 而调用方通常只需要聚合结果。真要看全量，调 ask 的接口。

### 7.3 会话与多轮

`options.sessionId` 存在 → 透传给 ask 作为 `sessionId`，复用现有会话机制。

**但 A2A 的 task 是有生命周期的**（submitted/working/completed）。两者语义要对齐：

| A2A task 状态 | ask 侧 |
|---|---|
| submitted → working | SSE 开始 |
| completed | result 事件 |
| failed | error 事件 |

不引入新状态，按 A2A 的标准映射即可。

## 8. Nacos 注册

### 8.1 注册内容

`NacosAgentRegistry` 的 `register(AgentCard, transports)`：

- **AgentCard**：见 §5，从配置构建
- **transports**：`NacosA2aRegistryTransportProperties`（host/port/path/protocol）

关键配置：

```yaml
mesh:
  a2a:
    # AgentCard 里的 url 必须能被其他 agent 访问到
    # 本地开发是 127.0.0.1，部署后要改成内网可达的地址
    advertised-host: ${MESH_ADVERTISED_HOST:127.0.0.1}
    advertised-port: ${MESH_PORT:20001}
```

**这是最容易配错的一处**：注册到 Nacos 的地址如果写 `127.0.0.1`，
其他机器上的 agent 拿到这个地址后连的是它自己。表现为「注册成功但调用失败」。

### 8.2 Nacos 3.x —— 已实测通过（2026-10-30）

**结论：`nacos-client 3.2.1-2026.03.30` + `Nacos 3.2.3` + `agentscope 2.0.4` 全部兼容。**

验证方式与结果：

| 步骤 | 结果 |
|---|---|
| `NamingService.getServerStatus()` | `UP` |
| `NacosAgentRegistry.builder(props).build()` | `registryName = Nacos` |
| `registry.register(card, transports)` | 成功 |
| `NacosAgentCardResolver.getAgentCard("chatdata-ask")` | 读回 name/description/url/skills 全对 |

#### 端口：3.x 与 2.x 不一样

| 容器内 | 用途 | 本地映射 |
|---|---|---|
| 8080 | **新控制台**（3.x 新增） | 18080 |
| 8848 | 老 HTTP API（v1 已移除，`/nacos/v1/...` 返回 404） | 18848 |
| 9848 | gRPC = 8848 + 1000 | 19848 |
| 9849 | gRPC（集群） | 19849 |

**`SERVER_ADDR` 要写 `127.0.0.1:18848`**（映射后的主端口）。
客户端会自动用「主端口 + 1000」连 gRPC，所以 19848 必须映射到 19848，
否则报「连接超时」——而那个错误看不出是端口问题。

> 3.x 的 `/nacos/v1/console/health/readiness` 已不存在。
> 想看服务端状态去 18080 的控制台，不要按 2.x 的习惯拼 v1 路径。

#### 三个 API 细节（凭印象会写错）

| 项 | 实际 | 说明 |
|---|---|---|
| 创建注册器 | `NacosAgentRegistry.builder(Properties)` | `NacosFactory.createAiService()` **不存在**——不要自己造 AiService |
| `TransportProperties` 签名 | `(transport, host, Integer port, path, supportTls, Map<String,Object> extra)` | 协议（http）放在最后的 Map 里，不是独立参数 |
| AgentCard 构造 | `new AgentCard.Builder()...build()` | 字段名：`preferredTransport` / `capabilities(AgentCapabilities)` / `skills` |

本地起 Nacos 3.x：

```bash
docker run -d --name nacos \
  -p 18080:8080 -p 18848:8848 -p 19848:9848 \
  -e MODE=standalone nacos/nacos-server:v3.2.3
```

### 8.3 命名空间与分组

建议独立的命名空间（如 `agent-mesh`），与业务配置分开。
理由：其他 agent 的 Nacos 客户端可能按命名空间批量拉取，混在一起会拉到不相关的。

## 9. 测试

### 单元

| 用例 | 断言 |
|---|---|
| `userId` 缺省 | 拒绝，不发 ask 请求 |
| `userId = "agent:xxx"` | 拒绝（本期只支持 `user:`） |
| `userId = "user:不存在的人"` | 拒绝 |
| 用户在授权表里没有授权该 agent | **拒绝**（核心限制） |
| 授权过但已撤销（`revoked_at` 非空） | 拒绝 |
| 用户授权了 A agent，B agent 来调 | 拒绝（粒度是按 agent） |
| 目标用户已停用 | 拒绝 |
| 客户端不在白名单 | 拒绝 |
| 结果超过 100 行 | artifact 里只放聚合摘要，不塞全量 |
| 空结果 | 明确说「没有符合条件的数据」 |

### 集成（需真实依赖）

| 用例 | 依赖 |
|---|---|
| 换到代理令牌并调通 ask | IAM + ask + MySQL + Redis |
| AgentCard 端点返回正确技能 | — |
| 注册到 Nacos 后能查到 | Nacos |

### 安全测试（必做，与功能同等重要）

| 用例 | 断言 |
|---|---|
| 伪造 `X-Agent-Client` 为白名单内的值，但用户没授权该 agent | **拒绝**（证明真正的边界不是白名单） |
| 代理管理员查数据 | 能查到（管理员本来就有权限），但**审计里记的是管理员本人** |
| 代理只有华东权限的人查华南 | 返回 0 行（行级规则照常生效） |
| 同一 token 复用给不同用户 | 不发生（缓存键是用户名） |

## 10. 实施顺序

| # | 内容 | 可验证的产出 |
|---|---|---|
| 1 | 建 `iam_agent_grant` 表 + IAM 新增 `agent-proxy` grant | 手工插一条授权，能换到令牌 |
| 2 | mesh 模块骨架 + A2A 端点 + AgentCard（固定返回） | `curl /a2a` 能拿到 AgentCard |
| 3 | ProxyTokenService + ProxyTargetResolver | 单测覆盖全部拒绝路径 |
| 4 | AskAgentRunner 接 ask | A2A 调用能拿到真实数据 |
| 5 | Nacos 注册 | Nacos 控制台能看到 agent |
| 6 | 安全测试全绿 | — |

**第 4 步是最小可用切片**：能通过 A2A 查到一个真实数字，且权限正确。

## 11. 风险与未决

| 风险 | 应对 |
|---|---|
| **白名单只是软校验** | 明确它不是安全边界；真正的边界是用户的授权表。上 mTLS 或网关认证列入后续 |
| **用户不看清楚就点授权** | 授权页写明「数据范围与你本人相同」；agent 名称来自 Nacos 注册的 AgentCard，不让 agent 自报 |
| ~~Nacos 3.x 与 nacos-client 版本兼容性~~ | **已验证通过**（2026-10-30，见 §8.2） |
| 注册地址写成 127.0.0.1 | `advertised-host` 必须可配，且部署时检查 |
| agent 名字由 agent 自己声明，可能骗用户点授权 | 缓解靠 Nacos 侧的管理员审核流程（不在本期）；记录为已知薄弱点 |
| AgentCard 静态配置与库里数据域不同步 | 可接受。加域时改配置重启 |
| 大结果塞进 artifact | 只给摘要 + 聚合，全量让调用方自己查 |

### 待确认

1. ~~Nacos 3.x 的版本具体是哪个~~ → **3.2.3**，已实测（§8.2）。
2. **授权表先给谁一条记录**？建议先手工插一条 `(admin, report-bot)` 验证链路。
3. **agentscope service 的管理台**需要 AgentCard 里哪些字段才能正确展示？本 spec 按 A2A 标准写，可能需要按管理台要求补。

## 12. 与现有 spec 的关系

- **ADR-0011**（渠道）：本 spec 复用它的「服务代表用户」模式，但**不共享实现**——信任边界不同
- **feishu-channel/spec.md**：`ProxyTokenService` 与 `ChannelTokenService` 同构，是刻意的重复
- **ADR-0002**（权限执行点）：本 spec **不新增权限判定点**。授权表管的是"能否被代理"的准入，
  不是"能看哪些数据"——后者仍只在 ask 里判

## 13. aistio：接入 agentscope service 管理台（2026-10-31 补）

### 13.1 为什么要单独一条通道

Nacos + A2A 解决「别的 agent 怎么发现并调用我」。它不解决「人在管理台上看到我、对我发一句话、
让我被编排成工作流的一步」。后者是 agentscope service 的 aistio 控制面提供的，两条通道互不替代：

| 通道 | 作用 | 关掉的影响 |
|---|---|---|
| Nacos + A2A | 其他 agent 发现并调用 | 互相发现不了；本进程照常服务已有调用方 |
| aistio | 管理台可见、可对话、可编排 | 管理台看不到；A2A 照常 |

所以 `mesh.aistio.enabled=false` 不影响 `mesh.a2a.*`，反之亦然。

### 13.2 为什么不用 `AgentScopeAdapter`

`agentscope-extensions-aistio` 自带的适配器要求 `target instanceof Agent`（真的 `ReActAgent`）。
mesh 是**转发器**：收请求 → 换令牌 → 调 ask → 返回。没有 agent 循环、没有模型、没有工具。

硬造一个假 `ReActAgent` 会让 mesh 多一层无意义的模型调用——管理台问一句，我们先生成一句再转发，
既慢又会让「模型额度用完」这种错误凭空出现。

所以实现 SDK 的 `FrameworkAdapter`（只有 4 个必实现方法），声明 `frameworkName = "chatdata-mesh"`，
在管理台上标清「我不是 agentscope 原生 agent」。

### 13.3 只声明真做到的能力

`capabilities()` 只返回 `conversation-inbound`（能处理一句话并回答）。

`extractContext()` 返回空 `Mono` —— 不声明 `context-query`。管理台查上下文会得到
`no context snapshot recorded for this session`，那是**准确**的：mesh 确实没有累积上下文对象，
每轮都是「拿问题去问一次」。强行拼一个假快照会让管理台展示不存在的东西，比不展示更糟。

SDK 还会自动加 `context-reporting` / `session-reporting`（它自己的契约端点实现的），
这两个不是我们声明的。

### 13.4 两个端口，不是一个（踩过的坑）

aistio 控制面（`aistiod`，Go 写的）有**两个独立监听**：

| 用途 | 容器内 | 本地映射 |
|---|---|---|
| REST API + 控制台 SPA | 8081 | 18081 |
| ASDP gRPC（数据面长连接） | **15010** | 15010 |

最初我把 `control-plane` 和 `control-plane-http` 写成同一个地址（18081），结果是：

1. HTTP 自注册成功 → 管理台 agent 列表里能看到 `chatdata-ask`
2. gRPC `Connect` 从没连上 → `/api/v1/dataplanes` 是空的
3. 管理台发对话 → `503 {"error":"asdp: instance not connected","code":"conversation_unavailable"}`

**排查顺序**（下次直接照这个查）：

```bash
# ① gRPC 有没有建立长连接 —— 控制面日志里应有 handshake accepted / data plane connected
docker logs agentscope-service-control-1 --since 5m | grep -i "asdp\|handshake"

# ② 数据面是否注册上了 —— 空数组 = gRPC 没连上
curl -s -H "Authorization: Bearer $JWT" \
  "http://127.0.0.1:18081/api/v1/dataplanes?tenant=default&namespace=default"
```

配置约束：

- `control-plane`（gRPC）**只能填 `host:port`**，带 `http://` 会让 gRPC 的 `NameResolver` 直接报错
- compose 必须显式映射 `15010:15010`。原 compose 只映射了 `8081`，所以这条最容易漏

### 13.5 回调地址不能用 localhost

控制面要回调 agent 的契约端点（查上下文/历史、推配置）。这些端口注册给控制面的 `baseUrl`
如果留空，**SDK 会回落到 `http://localhost:<contract-port>`**——control 在容器里，那指向容器自己。

症状：Dashboard 里有 agent、对话能跑、但 Sessions 页永远是空的、`/sessions/:id/context` 里
的 live fallback 永远打不到。`registeredAt` 正常、`lastSeenAt` 在刷新，唯独回调没到。

所以 `AistioRegistration` **必须显式** `.publicBaseUrl(cfg.getPublicBaseUrl())`，
默认 `http://host.docker.internal:<contract-port>`。`AistioConfig` 的字段注释写了"empty derives"，
但它 derive 出来的是 localhost，不是「探测本机 IP」——这一点和直觉不符，需要记住。

### 13.6 管理台对话「代表谁」

A2A 有 `AgentRequestOptions.userId` 说明代表谁；aistio 的 `runConversationTurn(sessionId, content)`
**没有用户身份**。所以用 `mesh.aistio.default-agent-client` + `default-user` 约定一个默认被代理人。

关键：仍然走完整的「换令牌 → 调 ask」链路，**不绕过权限**。否则管理台就成了能看全部数据的后门，
而我们建授权表正是为了防这个。默认被代理人也必须在 `iam_agent_grant` 里有授权记录。

### 13.7 已验证（2026-10-31）

```
控制面 agent 列表：chatdata-ask | active | ["context-reporting","conversation-inbound","session-reporting"]
数据面：chatdata-ask | healthy | framework=chatdata-mesh | baseUrl=http://host.docker.internal:18090
实例：MacBook-Pro-4.local | generation=3 | health=healthy

一轮对话（POST /api/v1/chats/{id}/turns）：
  1 user.message      各门店的订单金额合计
  2 turn.started
  3 turn.accepted
  4 turn.started
  5 assistant.message 共 4 行（列：门店名称、订单金额）…
  6 turn.completed
```

数据本身受被代理人权限约束（行级规则照常生效）。

### 13.8 本地环境的前置改动（不在本 repo 内）

`agentscope service` 的 `docker-compose.yml` 需要加一行端口映射：

```yaml
control:
  ports:
    - "18081:8081"
    - "15010:15010"   # ASDP gRPC；不映射 = 只能注册、不能对话
```

`docker/Dockerfile.control` 在国内网络下要改三处（只影响构建期下载，产物不变）：
npm registry → npmmirror、GOPROXY → goproxy.cn、基础镜像 `gcr.io/distroless` → `alpine`。
另外 `npm ci` 在慢网络下会跑 8 分钟后以 `Exit handler never called` 自崩，改用 `npm install`。
