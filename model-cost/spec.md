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

### 4.4 前端：问数页的模型下拉

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
