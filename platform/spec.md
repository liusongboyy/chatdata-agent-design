# 平台化：可插拔渠道 + 可嵌入问数

- Status: Draft
- Date: 2026-11-06
- Related: `chatdata-channel`、`AgentProxyGrantType`、`ChannelGrantType`、
  `docs/adr/0007-raw-mode-table-level-authorization.md`

## 1. 为什么要做这两件事

现在的形态是「一个产品」：一套部署、一个飞书接入、一个网页。

而平台的定义不是功能多，是**别人能在上面加东西**。当前没有任何「别人能加」的接口：

| 平台要素 | 现状 |
|---|---|
| 语义层 | ✅ 有，但配置绑在库里 |
| 接入渠道 | ⚠️ 飞书硬编码（9 个文件直接引用 `Feishu*`） |
| 嵌入第三方应用 | ❌ 无（令牌只能从网关 BFF 出） |

这两件事做完，系统从「你部署一套自己用」变成「别人能接进来、能嵌进去」。

## 2. 方向 B：可插拔渠道

### 2.1 现状的问题

`InboundMessage` 的注释里写着一句话，值得原文引用：

> 只含飞书能提供的字段，**不做「通用消息模型」**。
> 抽成跨渠道通用形态的成本在于：接口定错之后所有实现都要跟着改，
> 而那时看起来是「各实现自己有问题」。等钉钉/企微真接进来再抽。

这个判断是对的——**过早抽象比不抽象更糟**。而现在正是那个时刻：要接第二个渠道了。

当前耦合点（按文件）：

| 文件 | 飞书引用 | 性质 |
|---|---|---|
| `FeishuEventController` | 7 | webhook 入口，飞书特有 |
| `SubscriptionRunner` | 6 | 推送执行，**应该渠道无关** |
| `FeishuCrypto` | 5 | 验签，飞书特有 |
| `MessageHandler` | 5 | **核心逻辑，不该认识飞书** |
| `FeishuSender` | 4 | 发送，飞书特有 |
| `CardActionHandler` | 3 | 卡片回调，飞书特有 |
| `FeishuCardRenderer` | 1 | 渲染，飞书特有 |

**最关键的是 `MessageHandler`**：它做的是「收消息 → 查身份 → 调问数 → 回卡片」，
这套逻辑与渠道无关，却直接依赖 `FeishuCardRenderer` 与 `FeishuSender`。

### 2.2 抽象边界：**只抽三个接口**

不做「通用消息模型」。只抽渠道真正需要替换的三个动作：

```java
/** 出站：把一次问数结果送到用户面前。 */
public interface ChannelSender {
    /** 发一条新消息，返回渠道侧的消息 id（用于原地更新）。 */
    String send(String chatKey, String replyToMessageId, RenderedCard card);

    /** 原地更新。渠道不支持时抛 UnsupportedOperationException，调用方降级为再发一条。 */
    void update(String messageId, RenderedCard card);

    /** 渠道标识，写进去重表与会话映射。 */
    String channel();
}

/** 渲染：把问数结果变成渠道能显示的形态。 */
public interface ChannelRenderer {
    /** 进度卡片（"正在查询…"）。 */
    RenderedCard progress(String text);

    /** 结果卡片。 */
    RenderedCard result(AskResult result, AnswerRef answer);

    /** 错误卡片。 */
    RenderedCard failure(String code, String message);

    String channel();
}

/**
 * 入站：把一次 webhook 解析成统一的消息。
 *
 * **这是唯一需要「通用模型」的地方**，因为 MessageHandler 要吃它。
 * 字段保持最小：只放 MessageHandler 真正用到的。
 */
public record InboundMessage(
        String channel,
        String messageId,      // 渠道侧消息 id，去重键
        String chatKey,        // 会话键
        String chatId,         // 群 id，私聊为 null
        String senderId,       // 发送者标识，用于查绑定
        String text,
        boolean fromGroup
) {}
```

**为什么只抽三个**：

- `FeishuCrypto`（验签）、`FeishuEventController`（webhook 端点）**不抽**——
  每个渠道的验签和端点格式都不同，抽象它们等于抽象 HTTP
- 卡片结构不抽——每个渠道的卡片格式不同，`RenderedCard` 就是一个不透明对象，
  由各渠道的 renderer 生成、sender 消费

### 2.3 `MessageHandler` 的改造

```java
// 现在
public MessageHandler(..., FeishuCardRenderer renderer, FeishuSender sender, ...)

// 之后
public MessageHandler(..., ChannelRegistry channels, ...)
```

`ChannelRegistry` 按 `channel` 找对应的 `ChannelSender` / `ChannelRenderer`。
`SubscriptionRunner` 同理——它现在直接调 `FeishuSender`，改成按订阅的 `channel` 字段查。

### 2.4 迁移策略：**飞书先作为唯一实现**

不改行为，只把飞书那套包进 `FeishuChannelSender` / `FeishuChannelRenderer`。
验收标准：**改造前后，飞书渠道的端到端测试全部通过，行为逐字节相同**。

第二个渠道（钉钉）作为验证：如果接钉钉时发现还得改 `MessageHandler`，说明抽象错了。

### 2.5 不做的

- **不做渠道插件加载**（SPI / jar 热插）。渠道是编译期集成的，不是运行时加载的
- **不做「渠道市场」**。那是生态，现在没有
- **不抽验签**。每个渠道自己实现 webhook 端点

## 3. 方向 C：可嵌入问数

### 3.1 要解决什么

现在想用问数，必须**打开问数网页**。而用户真正在看数据的地方是业务系统。

嵌入后：

```html
<script src="https://chatdata.example.com/embed/v1/chatdata-embed.js"></script>

<chatdata-ask
  endpoint="https://chatdata.example.com"
  assistant="insurance"
  token="<短时令牌>"
  placeholder="问点数据…">
</chatdata-ask>
```

### 3.2 最难的不是组件，是**令牌**

现在的令牌只有两种来源：

| 来源 | 谁能拿 | 问题 |
|---|---|---|
| 网关 BFF 会话 | 浏览器用户 | 只对第一方页面有效（Cookie 同源） |
| `channel` grant | 渠道服务 | 需要 `app_secret`，不能给第三方 |
| `agent-proxy` grant | 其他 agent | 需要 `iam_agent_grant` 授权记录 |

第三方页面嵌入，需要**第三种**：宿主系统（比如 CRM）替它的用户拿一张令牌，
交给嵌入组件用。

#### 设计：`embed` grant type

```
urn:chatdata:params:oauth:grant-type:embed
```

参数：

| 参数 | 说明 |
|---|---|
| `client_id` | 宿主系统的应用标识，**必须在白名单里** |
| `client_secret` | 宿主系统的密钥 |
| `user_ref` | 宿主系统里这个用户的标识（如 CRM 的 user id） |
| `assistant` | **限定**只能问这个助理 |

**三条硬约束**：

1. **令牌绑定助理**。宿主系统只能替用户拿「问这个助理」的令牌，
   不能问别的域。这是最小权限——CRM 不该能问保险域的任意问题
2. **令牌短时**。默认 5 分钟。嵌入组件用它换一次会话，之后靠会话
3. **必须能映射到本系统的用户**。`user_ref` 通过 `iam_user.external_provider='embed'`
   + `external_id=user_ref` 找到本系统的账号。**找不到就拒绝**——
   不能让第三方凭空造一个身份

#### 为什么不用现有的 `channel` grant

`channel` grant 判的是「这个渠道账号绑定了哪个本系统用户」，它的语义是
**一个持久绑定**（飞书 open_id ↔ 账号）。

而 embed 是**一次性的代表**：CRM 说「这个请求是张三发的」，不需要张三
事先绑定过。语义不同，塞进同一个 grant 会让判据含糊——正是
`AgentProxyGrantType` 注释里说的那个问题。

### 3.3 组件形态：Web Component

```js
// chatdata-embed.js，约 15KB，无框架依赖
class ChatdataAsk extends HTMLElement {
  // Shadow DOM 隔离样式
  // 内部用 iframe 而不是直接渲染：
  //   ① 宿主页面的 CSS 不可能污染问数界面
  //   ② 令牌不暴露给宿主页面的 JS（只在 iframe 的 URL fragment 里传一次）
}
```

**为什么用 iframe**：

直接渲染进宿主 DOM 有三个问题：
- 样式冲突（宿主页面的全局 CSS 会打进来）
- 令牌暴露（组件在宿主的 JS 环境里跑，宿主能读它的变量）
- 升级困难（宿主页面里跑的旧版本没法强制更新）

iframe 全解决，代价是首屏多一次加载。可以接受。

### 3.4 安全模型

| 威胁 | 对策 |
|---|---|
| 宿主页面伪造用户 | 宿主用 `client_secret` 换令牌，`user_ref` 由宿主决定——**信任边界在宿主**，这是它的责任 |
| 令牌泄露 | 5 分钟过期 + 绑定助理 + 绑定用户 |
| 宿主问不该问的 | 令牌里写死 `assistant`，服务端校验 |
| 点击劫持 | iframe 加 `sandbox`，且不允许宿主嵌入到任意页面（`Content-Security-Policy: frame-ancestors` 白名单） |

**最后一条必须做**：`frame-ancestors` 只允许白名单里的宿主域名。
不做的话，任何人可以嵌你的问数页面做钓鱼。

### 3.5 不做的

- **不做 SDK 的多语言版本**。只做 Web Component，其他语言调 HTTP API
- **不做「让第三方定义助理」**。宿主只能用已有的助理
- **不做跨租户**。所有宿主共享一套用户体系

## 4. 实现顺序

| 阶段 | 内容 | 验收 |
|---|---|---|
| 1 | 抽 `ChannelSender` / `ChannelRenderer` / `ChannelRegistry`，飞书包进去 | 飞书端到端测试全过，行为不变 |
| 2 | 接钉钉（或企微）验证抽象 | 新渠道不改 `MessageHandler` |
| 3 | `embed` grant type + 用户映射 | 能用 curl 换到令牌，越权被拒 |
| 4 | `chatdata-embed.js` + iframe 页面 | 在一个假宿主页面里能问数 |
| 5 | `frame-ancestors` 白名单 | 非白名单域名嵌入被浏览器拒绝 |

**先做 1，因为它风险最低且立刻改善架构。** 3–5 可以并行。

## 5. 风险

| 风险 | 应对 |
|---|---|
| **抽象错了要返工** | 只抽三个接口；接第二个渠道时如果发现还得改 `MessageHandler`，说明错了，那时返工代价还小 |
| **embed 成了数据泄露通道** | 令牌绑助理 + 5 分钟 + 用户映射必须存在；`frame-ancestors` 白名单 |
| **第三方拿到令牌后滥用** | 审计里记 `client_id`，能追溯是哪个宿主系统在问 |
| **iframe 性能** | 首屏多一次加载；用 `preconnect` 缓解 |

## 6. 待定（写实现前必须回答）

1. **宿主系统怎么注册？** 现在 `oauth2_registered_client` 只有 4 条。embed 客户端需要
   一个管理界面吗？还是先手工 SQL？
2. **`user_ref` 映射不到怎么办？** 拒绝，还是自动建一个「只读、只限这个助理」的账号？
   ——**我倾向拒绝**，自动建号会让账号体系被第三方污染
3. **iframe 的页面是复用问数页还是单独一个精简版？** 复用的好处是不用维护两套；
   坏处是问数页有很多宿主用不上的东西（历史、模型选择）
