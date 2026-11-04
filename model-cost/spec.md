# 模型选择与成本可见性

- Status: Draft
- Date: 2026-11-04
- Related: `AskEngineModelProperties`、`ChatUsage`（agentscope 2.0.4）、
  `DomainPermission`、`ask_audit_log`

## 1. 要解决什么

现在**只有一个全局模型**，且**完全不记录消耗**：

```yaml
chatdata.engine.model:
  base-url: ${LLM_BASE_URL}
  model-name: ${LLM_MODEL}        # 一个字符串，改它要重启
```

三个问题：

| 问题 | 后果 |
|---|---|
| 只能全局配一个 | 总经理和普通员工用同一个模型，成本与能力无法区分 |
| 换供应商要改配置重启 | 换供应商（方舟 → OpenAI → 百炼）是一次发布 |
| 不知道花了多少 | 企业落地必问「一个月多少钱」，现在答不出来 |

**目标**：管理员配一组模型 → 用户按角色选 → 每次消耗可查可算。

## 2. 目标与非目标

**目标**
1. 管理员可配**多个模型**，可来自**不同供应商**
2. 每个模型声明**哪些角色可以用**
3. 问数页有模型下拉，**只列当前用户有权用的**
4. 每次提问记录 `ChatUsage` 六个维度 + 归属模型
5. 成本看板：按模型 / 按域 / 按用户看 token 与费用

**非目标**
- **不做自动选模型**（按问题难度路由）。那需要"哪个模型答得准"的反馈信号，
  而现在连这个都还没度量。先有数据，再谈路由。
- **不做预算告警/限额**。这一期只解决"看得见、选得了"。
- **不改模型调用链路**。只加采集与选择，不动生成逻辑。

## 3. 已核实的技术事实

### 3.1 agentscope 能拿到 usage

`agentscope-core-2.0.4` 的 `io.agentscope.core.model.ChatUsage`：

```java
int getInputTokens();          // 输入 token
int getOutputTokens();         // 输出 token
int getCachedTokens();         // 缓存命中（省钱的那部分）
int getCacheCreationTokens();  // 写缓存
int getReasoningTokens();      // 推理 token
int getToolUsePromptTokens();  // 工具调用消耗
```

它挂在 **`Msg.usage`** 上（`ReActAgent.collectAggregatedUsage(AgentState)`
从 context 里的消息聚合）。**不需要改 HTTP 层**。

### 3.2 token 语义已实测（2026-11-04，火山方舟）

同一个长 system prompt 连调两次：

```
第 1 次:  prompt=1032  cached=0      completion=42  total=1074
第 2 次:  prompt=1032  cached=1024   completion=49  total=1081
```

| 结论 | 影响 |
|---|---|
| **`prompt_tokens` 已包含 `cached_tokens`** | 不能相加。**写错会高估一倍** |
| **`reasoning_tokens` 已含在 `completion_tokens` 里** | 单独加会重复计费 |
| `total_tokens` = 两者之和 | 但它把缓存按全价算了，不能用来算钱 |

**换供应商必须重跑这个测试**——各家语义不同。

## 4. 设计

### 4.1 模型配置：从「一个」到「一组」，带角色白名单

```yaml
chatdata.engine:
  model:
    # 用户没选、域也没绑定时用哪个
    default: fast

    models:
      fast:
        # ── 供应商 ──
        # 换供应商只改这三行。base-url 是关键：
        # 方舟 / OpenAI / 百炼 / DeepSeek 官方 都是 OpenAI 兼容协议
        base-url: https://ark.cn-beijing.volces.com/api/coding/v3
        api-key: ${ARK_API_KEY}
        model-name: deepseek-v4-1-flash-261017

        display-name: DeepSeek V4 Flash（快速）
        description: 日常问数，响应快、成本低

        # ── 谁能用 ──
        # 空 = 所有能问数的角色都能用。
        # **默认空是刻意的**：新加的模型不该默认对所有人不可见，
        # 那会让管理员以为配置没生效。
        allowed-roles: []

        # ── 单价（元 / 百万 token）──
        price:
          input-per-mtok: 0.5
          cached-input-per-mtok: 0.05
          output-per-mtok: 2.0

      strong:
        base-url: https://ark.cn-beijing.volces.com/api/coding/v3
        api-key: ${ARK_API_KEY}
        model-name: deepseek-v4-1-pro-261017
        display-name: DeepSeek V4 Pro（精准）
        description: 复杂分析，推理更强、成本较高
        # **只给这几个角色**。总经理能选，普通员工看不到这一项
        allowed-roles: [manager, admin]
        price:
          input-per-mtok: 2.0
          cached-input-per-mtok: 0.2
          output-per-mtok: 8.0

      gpt:
        # 换成另一家供应商：只改 base-url 与 api-key
        base-url: https://api.openai.com/v1
        api-key: ${OPENAI_API_KEY}
        model-name: gpt-4o
        display-name: GPT-4o
        allowed-roles: [admin]
        price:
          input-per-mtok: 18.0
          cached-input-per-mtok: 9.0
          output-per-mtok: 72.0
```

**为什么不硬编码供应商枚举**：全部走 OpenAI 兼容协议
（方舟、OpenAI、百炼、DeepSeek 官方都支持），一个 `base-url` 就够。
加一个供应商 = 加一段配置，不改代码、不发版。

**`allowed-roles` 为空 = 所有人可用**，不是"所有人不可用"。
后者会让新加的模型默认隐身，管理员配完以为没生效。

### 4.2 三层模型解析：用户选 > 域绑定 > 全局默认

```
用户在下拉里选了     → 用它（但要校验角色权限与域绑定都允许）
否则域绑了模型       → 用域绑的
否则                 → 用 default
```

**校验顺序有讲究**——三道都要过：

1. **角色白名单**：`allowed-roles` 含用户的任一角色
2. **域绑定**（如果域绑了 `model-key`）：只能选绑定的那个或它允许的集合
3. **模型存在且已配置价格**

任一条不过 → 请求被拒，**不是静默降级到默认模型**。
理由：用户以为自己在用强模型，实际用了弱模型，那会让他对结果产生错误判断。

### 4.3 域的模型约束

域表加 `model_key`：

```sql
ALTER TABLE ask_domain ADD COLUMN model_key VARCHAR(32) NULL
  COMMENT '域绑定的模型键；空表示不限制（用户在角色允许范围内自选）';
```

**两种语义要分清**：

| `model_key` | 含义 |
|---|---|
| NULL | 这个域**不限制**，用户在角色允许范围内自选 |
| 非空 | 这个域**只能用这一个**，下拉里锁死 |

**什么时候该锁死**：某个域的数据量大到弱模型跑不动（会超 context 或
答得极差），那时锁强模型。多数域应该留 NULL。

### 4.4 前端设计

#### 4.4.1 问数页：模型下拉放在助理右边

现有 `assistant-bar` 是一整行（助理下拉 + 落点域 + 欢迎语）。模型下拉接在助理后面：

```
┌──────────────────────────────────────────────────────────────────┐
│ 助理 [🛡️ 保险问数助手 ▾]  模型 [快速 ▾]   正在问 保险分析域        │
└──────────────────────────────────────────────────────────────────┘
```

**为什么不放进设置弹窗**：模型是**每次提问都可能变**的选择
（这次要快、下次要准），藏起来用户就不会用。而助理也是同样的性质，
两者并排最自然。

**为什么不做成图标按钮**：模型名本身就是信息（"快速"/"精准"），
图标化之后用户要靠记忆。

#### 4.4.2 下拉内容

```
▾ 模型
┌───────────────────────────────────────────────┐
│ ● DeepSeek V4 Flash（快速）                    │
│   日常问数，响应快、成本低                      │
│ ─────────────────────────────────────────────│
│ ○ DeepSeek V4 Pro（精准）              ¥¥      │
│   复杂分析，推理更强、成本较高                  │
└───────────────────────────────────────────────┘
```

三个设计点：

1. **`¥` 标记贵模型**：用户不知道单价，但"这个更贵"是选择依据。
   `¥` = 中等，`¥¥` = 贵。**不显示具体金额**——那会让用户分心算账，
   而他真正该想的是"这个问题难不难"。
2. **第二行写用途不写参数**：「复杂分析用」比「2.0元/百万token」有用。
3. **当前选中的打点**，不是加背景色——背景色在深色主题下对比度不好控。

#### 4.4.3 三种边界状态

| 状态 | 表现 |
|---|---|
| 只有一个可用模型 | **不渲染下拉**，显示静态标签（与助理同一取舍：只有一个选项的下拉是噪音） |
| 一个都没有 | 显示「你的账号没有可用模型，请联系管理员」，且**禁用提问输入框** |
| 域锁定了模型 | 下拉置灰，旁边显示「此数据域限定模型」 |

**「一个都没有」必须禁用输入框**——否则用户打了半天字、点发送才报错，
那是白费功夫。

#### 4.4.4 域编辑页：绑定模型

在域的编辑弹窗里加一个下拉：

```
模型绑定
  ○ 不限制（用户在角色允许范围内自选）     ← 默认
  ○ 限定为 [DeepSeek V4 Flash ▾]
```

**默认「不限制」**，且有说明——多数域不该锁死模型，
只有"弱模型跑不动这个域"时才锁。

#### 4.4.5 管理端：模型管理页

新增 `/admin/models`，与「数据源」「语义域」并列在管理台导航。

**为什么不放进 application.yml 让运维改**：模型是业务决策
（哪个角色能用哪个、单价多少），不是部署参数。改一次要重启一次
是不能接受的——而单价变化、新模型上线都很频繁。

##### 列表页

```
┌────────────────────────────────────────────────────────────────────┐
│ 模型                                           [+ 新增模型]        │
│ 问数使用的模型。可来自不同厂商——只要兼容 OpenAI 协议。               │
├────────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ DeepSeek V4 Flash（快速）         [默认]        [内置]        │  │
│ │ 火山方舟 · deepseek-v4-1-flash-261017                         │  │
│ │                                                                │  │
│ │ 单价  输入 ¥0.5 / 缓存 ¥0.05 / 输出 ¥2.0   每百万 token        │  │
│ │ 可用角色  全部                                                  │  │
│ │ 近 30 天  1,102 次 · ¥8.21                                     │  │
│ │                                                                │  │
│ │ [测试] [编辑] [停用]                                            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ DeepSeek V4 Pro（精准）                          [内置]        │  │
│ │ 火山方舟 · deepseek-v4-1-pro-261017                           │  │
│ │ 单价  输入 ¥2.0 / 缓存 ¥0.2 / 输出 ¥8.0                       │  │
│ │ 可用角色  manager, admin                                       │  │
│ │ 近 30 天  102 次 · ¥4.26                                       │  │
│ │ [测试] [编辑] [停用]                                            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ GPT-4o                                           [已停用]     │  │
│ │ OpenAI · gpt-4o                                                │  │
│ │ 可用角色  admin                                                │  │
│ │ 近 30 天  0 次  ← **显式显示 0，不省略**                       │  │
│ └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

四个设计点：

1. **「近 30 天」显示使用量与费用** —— 管理员改配置前先知道它现在值不值。
   0 次要显式显示，省略会让管理员以为配置没生效。
2. **`[默认]` 标记** —— 全局默认只有一个，改了要把旧的摘掉。
   这个不能靠配置手写，容易两个都标默认。
3. **`[内置]` vs 可从 UI 删** —— 见 §5.3 的取舍。
4. **停用而不是删除** —— 历史审计里引用了这个模型键，
   删掉会让成本回看时查不到名字。

##### 新增/编辑弹窗

```
┌─────────────────────────────────────────────────────────────┐
│ 新增模型                                                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 厂商  [火山方舟 ▾]                                          │
│       ├ 火山方舟                                            │
│       ├ OpenAI                                              │
│       ├ 阿里百炼                                            │
│       ├ DeepSeek 官方                                       │
│       └ 自定义…                                             │
│                                                             │
│  选了厂商后 base-url 自动填好，可改                           │
│                                                             │
│ 接口地址  [https://ark.cn-beijing.volces.com/api/...       ]│
│ 密钥      [••••••••••••] [显示]   ← 保存后不回显，只显示"已配置"│
│ 模型名    [deepseek-v4-1-flash-261017                     ]│
│ 显示名    [DeepSeek V4 Flash（快速）                       ]│
│ 用途说明  [日常问数，响应快、成本低                          ]│
│                                                             │
│ ── 可用角色 ──                                              │
│  ☑ 全部（不限制）                                           │
│  或  ☐ manager  ☐ admin  ☐ analyst  ☐ 区域经理              │
│                                                             │
│ ── 单价（元 / 百万 token）──                                 │
│ 输入      [0.5    ]   必须大于 0                            │
│ 缓存输入  [0.05   ]   缓存命中的部分单独计价                  │
│ 输出      [2.0    ]                                        │
│ 单价更新于 [2026-11-04]                                     │
│                                                             │
│ ── 其他 ──                                                  │
│  ☐ 设为全局默认                                             │
│  ☑ 启用                                                     │
│                                                             │
│                          [取消]  [测试连接]  [保存]          │
└─────────────────────────────────────────────────────────────┘
```

**几个关键设计**：

| 设计 | 为什么 |
|---|---|
| **厂商下拉自动填 base-url** | 手打 URL 极易错（少个 `/v1`、多了空格），而错了的表现是 404 或奇怪的解析错误 |
| **密钥不回显** | 与数据源密码同一取舍。只显示「已配置 / 未配置」 |
| **「全部」是个显式勾选** | 不能让"不选任何角色"隐式等于"所有人"——那是默认放行，本项目的禁忌 |
| **单价必须 > 0** | 0 会让成本算成 0，而 0 元比"不知道"危险 |
| **单价更新于** | 价格会变，看板上要能看出这个成本是按哪天价格算的 |
| **测试连接** | 与数据源同一个理由：配完当场能验，不用等到用户提问失败 |

##### 厂商预设

```js
const VENDORS = [
  { key: 'ark',     name: '火山方舟',   baseUrl: 'https://ark.cn-beijing.volces.com/api/coding/v3' },
  { key: 'openai',  name: 'OpenAI',     baseUrl: 'https://api.openai.com/v1' },
  { key: 'bailian', name: '阿里百炼',   baseUrl: 'https://dashscope.aliyuncs.com/compatible-mode/v1' },
  { key: 'deepseek',name: 'DeepSeek 官方', baseUrl: 'https://api.deepseek.com/v1' },
  { key: 'custom',  name: '自定义…',    baseUrl: '' },
]
```

**不做厂商枚举的硬编码校验**——`base-url` 是唯一判据。
自定义选项让管理员接任何兼容 OpenAI 协议的服务（vLLM、Ollama、代理）。

#### 4.4.6 配置存哪里：从 yml 搬到数据库

**这是 UI 化的前提。** 配置在 yml 里就没法从页面改——改文件要重启。

```sql
CREATE TABLE ask_model (
  id            BIGINT       NOT NULL AUTO_INCREMENT,
  model_key     VARCHAR(32)  NOT NULL COMMENT '稳定标识，审计里存它',
  display_name  VARCHAR(64)  NOT NULL,
  description   VARCHAR(200) NULL     COMMENT '用途说明，写给人看',

  -- 供应商
  base_url      VARCHAR(500) NOT NULL,
  -- **密文**。与数据源密码同一套 PasswordCipher
  api_key_enc   VARCHAR(500) NULL,
  model_name    VARCHAR(128) NOT NULL COMMENT '供应商侧的模型名',

  -- 单价（元 / 百万 token）。**允许 NULL** 表示未配单价——
  -- 那时能算 token 不能算钱，看板显示"未配单价"而不是 0
  price_input   DECIMAL(12,6) NULL,
  price_cached  DECIMAL(12,6) NULL,
  price_output  DECIMAL(12,6) NULL,
  priced_at     DATETIME      NULL COMMENT '这套单价是哪天生效的',

  -- 权限
  -- JSON 数组，空数组表示不限制。**不能存 NULL 表示"不限制"**——
  -- NULL 与"还没配"无法区分，而这两者的含义完全相反
  allowed_roles JSON          NOT NULL,

  is_default    TINYINT(1)    NOT NULL DEFAULT 0,
  enabled       TINYINT(1)    NOT NULL DEFAULT 1,
  created_by    VARCHAR(64)   NULL,
  created_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_model_key (model_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='问数使用的模型';
```

**四条约束必须做在数据库层**：

| 约束 | 为什么 |
|---|---|
| `model_key` 唯一 | 审计里存的是它，重复了成本统计会串 |
| `allowed_roles` 非 NULL | NULL 与"空数组"含义相反，混用会造成默认放行 |
| `is_default` 全局只有一个 | 靠应用层保证容易漏（并发改两个都成默认） |
| `api_key_enc` 密文存储 | 与数据源密码同一套 `PasswordCipher`，复用已有实现 |

**迁移期的兼容**：`@ConfigurationProperties(prefix="chatdata.engine.model")`
保留，作为**首次启动的种子**——表里没数据时用它初始化一条。
之后以表为准。这样老部署升级后行为不变。

#### 4.4.7 成本页：健康度第三个标签

与「概览」「质量分析」同一层级，用同一套表格样式。

#### 4.4.6 切换模型的影响要可见

切换后**历史问答不受影响**（每次提问记录了自己用的模型），
但要在界面上说清楚：

- 切换是**从下一问开始生效**，已经问过的不变
- 历史问答的答案卡片上标出**当时用的模型**（小字，hover 看详情）

**为什么要标**：用户可能记得"上次那个答案不太准"，如果知道
那次用的是弱模型，就理解了；不标的话他会以为系统不稳定。



```
┌─────────────────────────────────────────────┐
│  助理：[保险问数助手 ▾]   模型：[快速 ▾]      │
│                                              │
│  ▾ 模型                                      │
│  ┌────────────────────────────────────┐     │
│  │ ● DeepSeek V4 Flash（快速）         │     │
│  │   日常问数，响应快、成本低            │     │
│  │ ○ DeepSeek V4 Pro（精准）  ¥¥      │     │
│  │   复杂分析，推理更强、成本较高         │     │
│  └────────────────────────────────────┘     │
└─────────────────────────────────────────────┘
```

三个设计点：

1. **`¥` 标记贵模型**：用户不知道单价，但"这个更贵"是选择依据
2. **描述写用途不写参数**：「复杂分析用」比「2.0元/百万token」有用
3. **域锁定时下拉置灰**，并显示「此数据域限定模型」

**下拉只列有权用的**——不能列出但选不了。后者是"看得见吃不着"，
比不显示更让人困惑。

### 4.5 采集：每次提问记一条

`ask_audit_log` 加六列 + 模型键：

```sql
ALTER TABLE ask_audit_log
  ADD COLUMN model_key VARCHAR(32) NULL,
  ADD COLUMN input_tokens INT NULL,
  ADD COLUMN output_tokens INT NULL,
  ADD COLUMN cached_tokens INT NULL,
  ADD COLUMN cache_creation_tokens INT NULL,
  ADD COLUMN reasoning_tokens INT NULL;
```

**允许 NULL**：老数据没有；失败请求可能拿不到 usage。
NULL 表示"没采到"，与 0（"确实没消耗"）不同。

### 4.6 成本看板

健康度页第三个标签「成本」。

```
概览（最近 30 天）
  ￥12.47      总计
  1,204        提问数
  ￥0.0104     单次均价
  38.2%        缓存命中率  ← cachedTokens / inputTokens

按模型
模型                提问    输入     缓存命中    输出     费用
DeepSeek V4 Flash  1,102   892K    341K(38%)  156K   ￥8.21
DeepSeek V4 Pro      102   104K     28K(27%)   31K   ￥4.26
GPT-4o                 0      —          —       —      —
                        ↑ 配了没人用。要显示，不显示会让管理员以为配置没生效
```

```
按用户（管理员看）
用户      提问    费用     常用模型
admin      412   ￥6.83   Flash
zhangsan   204   ￥1.02   Flash
```

**「按用户」只在有 `ask.admin.domains` 权限时显示**——
普通用户不该看到别人花多少钱。

## 5. 计价

```
未缓存输入 = inputTokens - cachedTokens
费用 = 未缓存输入 × inputPerMTok / 1e6
     + cachedTokens × cachedInputPerMTok / 1e6
     + outputTokens × outputPerMTok / 1e6
```

`ModelPricing` 做成纯函数，可单测。

**未配单价时**：能算 token，不能算钱。显示「未配单价」而不是 0——
0 元比"不知道"危险得多，它会让成本看起来不存在。

## 6. 实现清单

| 位置 | 改动 |
|---|---|
| `AskEngineModelProperties` | 单值 → `default` + `models` 映射 + `allowedRoles` + `price` |
| `AskEngineModelFactory` | 按 key 建实例，**按 key 缓存**（每次新建会重复建连接池） |
| `ModelAccessService`（新） | 纯函数：用户角色 + 模型白名单 + 域绑定 → 可选模型集合 |
| `AskServiceFactory` | 取域时带 `modelKey`，解析出实际模型 |
| `AskService` / 审计 | 读 `Msg.usage` 写进审计 |
| `GET /api/ask/models` | 当前用户可选的模型列表（问数页下拉用） |
| `sql/24-model-cost.sql` | 两个 ALTER |
| `HealthService.cost()` | 聚合 |
| `GET /api/engine/admin/health/cost` | 成本看板 |
| 问数页 + 域编辑页 | 下拉 + 域绑定 |

## 7. 验收

1. **角色隔离**：`zhangsan`（普通员工）调 `/api/ask/models` 只看到 `fast`；
   `admin` 看到 `fast` + `strong` + `gpt`
2. **越权拒绝**：`zhangsan` 请求体里硬写 `modelKey=gpt` → **403**，
   不是静默降级
3. **域锁定**：绑了 `strong` 的域，下拉置灰且只能用 `strong`
4. **token 采集准确**：一次真实调用，`input/output/cached` 与方舟控制台一致
5. **费用口径准确**：用已知 token 的调用手算，与看板一致
6. **未配单价**：显示「未配单价」而不是 0
7. **配了没人用**：模型列表里显示 0 次，不能省略
8. **老数据兼容**：加列前 6600 条审计，token 为 NULL，成本看板跳过不算 0

## 8. 风险

| 风险 | 应对 |
|---|---|
| **`inputTokens` 语义各家不同** | 已实测方舟；换供应商必须重跑 §3.2 的两次调用测试 |
| **角色白名单配错，某些人一个模型都选不了** | 启动时校验：每个能问数的角色至少有一个可用模型，否则警告 |
| 单价过期，成本算错 | 单价带生效日期，看板显示「单价更新于 X」 |
| 模型实例按 key 缓存，改配置不生效 | 与数据源同一取舍：配置变更后清缓存 |
| 用户不知道选哪个 | 默认模型 + 描述写用途；不给用户"最优解"的负担 |

## 9. 不做

- 自动选模型（缺准确率反馈信号）
- 预算告警 / 限额
- 跨租户成本分摊（现在没有多租户）
- 按 token 数配额（那是限额，属下一期）

---

## 10. 实现记录（2026-11-04）

### 10.1 usage 采集的关键改造

原设计写的是"读 `Msg.usage` 写进审计"，实现时发现两处会让它悄悄少记：

**① `DslSource.generate` 只返回 `AskDsl`，`Msg` 被丢弃。**
usage 只在那次调用的 `Msg` 上存在过一次，事后补采不可能。改成返回
`Generated<T>(value, usage)`，`Generated` 也用在 `AnalysisPlanSource` 与
`NarrativeSource` 上——三条产出路径都会调模型。

**② `agent.call()` 的返回消息只带**最后一轮**的 usage。**
ReAct 循环每轮都计费，`call()` 只保留最终消息上的那一份。改用 `streamEvents`，
对每个 `ModelCallEndEvent` 的 usage 累加（`UsageCollector.run`）。
`call()` 内部就是 `streamEvents` 过滤 `AgentResultEvent`，所以这不是换条链路，
只是多拿一个副产物。

**③ 用 `Msg.getChatUsage()` 而不是 `getUsage()`。**
结构化输出走"合成 generate_response 工具"那条路径时，框架把多轮用量聚合后塞进
**metadata**（`_chat_usage`），`usage` 字段上只有最后一轮。`getChatUsage()` 两条都覆盖。

### 10.2 一次提问调多次模型

约束缺失会重问、渲染失败会再重问。`AskService` 里加了**请求内**的 `UsageMeter`
（局部变量，不是字段——字段在并发下会把 A 的用量记到 B 头上，且是静默的）。
只记最后一次会系统性低估，而低估恰好落在"这个域老是重问"这种最该被看见的场景上。

### 10.3 模型越权在发起之前就拒

`AskServiceFactory.forDomain(domainId, ctx, modelKey)` 用 `ModelAccess.resolve`
判定，与 `GET /api/ask/models` 是同一个函数。拒绝抛 `AskDeniedException`，
reason 新增 `MODEL_NOT_ALLOWED`——与数据权限分开，因为处理方式不同：
模型问题换个模型，数据权限要找管理员。

控制器把 `MODEL_NOT_ALLOWED` 作为单独的 SSE 错误码下发，前端可据此提示"换一个模型"。

### 10.4 金额的"不知道"不写成 0

`Cost` 在模型没配单价时返回 `null`；`CostAggregate.Row` 分开数
`asks` 与 `pricedAsks`；前端对 `partial` / `unpriced` 分别显示
「覆盖 x/y 次」与「未配单价」。

理由：0 元是"免费"，而没配单价是"不知道"。把后者显示成 0，
管理员会得出"成本很低"的结论——而这个结论是假的。

### 10.5 累加精度：微元，不是展示精度

`Cost` 产出的是 6 位小数的金额（单次调用常见 0.00216）。成本看板的累加器最初
按**展示精度 4 位**累加，结果第一条记录就抛 `ArithmeticException: Rounding necessary`。

改成按**微元（1e-6）整数**累加、输出时再舍到 4 位：既不会在第一步就炸，
也不会有浮点累积误差。

这个 bug **单测一开始没抓到**——`CostTest` 用手算的金额直接比，走不到累加器。
是 `CostServiceTest` 用真实量级的数据喂进去才暴露的。

### 10.6 测试

| 文件 | 覆盖 |
|---|---|
| `UsageCollectorTest` | 六字段原样搬运、缺失→UNKNOWN 而非 ZERO、累加语义 |
| `UsageRealAgentTest` | **真实 `ReActAgent`** + 假模型：usage 确实传得回来（框架改用法时这条会红） |
| `EngineAuditMapperRealDbTest` | **真库**：INSERT 列序与占位符对得上；未采集落成 NULL 不是 0 |
| `CostTest` | cached 是 input 子集（防高估一倍）、未配价→null、脏数据不倒扣 |
| `CostServiceTest` | 三种"没有"（未配价 / 没采到 / 零提问）都不写成 0；均价分母是 pricedAsks |
| `AskServiceTest.UsageCapture` | 一次提问记 modelKey+usage、重问累加、失败也记、UNKNOWN→null |

其中后两条真库/真 agent 的测试是**刻意的**：手写的 fake mapper 不解析 SQL，
SQL 列序错了它照样"通过"；而 `UsageCollector` 只验从 `Msg` 取数，
验不了"框架到底把 usage 放在哪"。
