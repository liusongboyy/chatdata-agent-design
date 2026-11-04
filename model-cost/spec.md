# 模型选择与成本可见性

- Status: Draft
- Date: 2026-11-04
- Related: `AskEngineModelProperties`、`ChatUsage`（agentscope 2.0.4）、
  `ask_audit_log`、`.scratch/health-dashboard/spec.md`

## 1. 要解决什么

现在**只有一个全局模型**，且**完全不记录消耗**：

```yaml
chatdata.engine.model:
  base-url: ${LLM_BASE_URL}
  model-name: ${LLM_MODEL}        # 一个字符串，改它要重启
  max-tokens: 16384
```

两个问题：

| 问题 | 后果 |
|---|---|
| 只能全局配一个模型 | 「简单问题用便宜模型、复杂问题用贵的」做不到；换模型要改配置重启 |
| 不知道花了多少 | 企业落地必问「一个月多少钱」，现在答不出来 |

`ask_audit_log` 里有 `duration_ms` 但没有 token；agentscope 的
`Msg.usage` 有完整 `ChatUsage`（见 §3），但我们从来没读它。

**目标**：能按域/助理选模型，能按模型统计输入/输出/缓存 token 与费用。

## 2. 目标与非目标

**目标**
1. 模型可配置多条，按**域**（或助理）绑定
2. 每次提问记录 `ChatUsage` 六个维度 + 归属模型
3. 成本看板：按模型 / 按域 / 按时间看 token 与费用

**非目标**
- **不做自动选模型**（按问题难度路由）。那需要准确率反馈信号，
  而现在连「哪个模型答得准」都还没度量。先有数据，再谈路由。
- **不做预算告警/限额**。这一期只解决"看得见"。
- **不改模型调用链路**。只加采集，不动生成逻辑。

## 3. 已核实的技术事实

`agentscope-core-2.0.4` 的 `io.agentscope.core.model.ChatUsage`：

```java
int getInputTokens();          // 输入 token
int getOutputTokens();         // 输出 token
int getCachedTokens();         // 缓存命中（省钱的那部分）
int getCacheCreationTokens();  // 写缓存（有的 provider 单独计费）
int getReasoningTokens();      // 推理 token（deepseek-r1 这类会有）
int getToolUsePromptTokens();  // 工具调用消耗
int getTotalTokens();
double getTime();
```

它挂在 **`Msg.usage`** 上（`ReActAgent.collectAggregatedUsage(AgentState)`
从 context 里的消息聚合）。所以**不需要改 HTTP 层**——
只要在拿到回复消息时读 `usage` 即可。

## 4. 设计

### 4.1 模型配置：从「一个」到「一组」

```yaml
chatdata.engine:
  model:
    # 默认模型。没绑定特定模型的域走它
    default: fast

    models:
      fast:
        base-url: https://ark.cn-beijing.volces.com/api/coding/v3
        api-key: ${LLM_API_KEY}
        model-name: deepseek-v4-1-flash-261017
        display-name: DeepSeek V4 Flash
        # 计费单价（元 / 百万 token）。**必填**，否则算不出钱
        price:
          input-per-mtok: 0.5
          cached-input-per-mtok: 0.05
          output-per-mtok: 2.0

      strong:
        base-url: https://ark.cn-beijing.volces.com/api/coding/v3
        api-key: ${LLM_API_KEY}
        model-name: deepseek-v4-1-pro-261017
        display-name: DeepSeek V4 Pro
        price:
          input-per-mtok: 2.0
          cached-input-per-mtok: 0.2
          output-per-mtok: 8.0
```

**为什么单价放配置而不是硬编码**：各家价格会变，而且同一模型在不同
渠道（官方 / 云厂商 / 代理）价格不同。硬编码进代码意味着每次调价都要发版。

**`price` 缺失时怎么办**：能算 token，不能算钱。看板上那条显示「未配单价」
而不是显示 0——**0 元比"不知道"危险得多**，它会让成本看起来不存在。

### 4.2 绑定：按域

域表加一列 `model_key`：

```sql
ALTER TABLE ask_domain ADD COLUMN model_key VARCHAR(32) NULL
  COMMENT '绑定的模型键；空表示用默认模型';
```

**为什么绑在域上而不是助理上**：助理是「一组域的入口」，
一个助理可能跨多个域（`hasMultipleDomains`）。模型选择跟着数据规模走
——保险域数据量大、门店域小，按域绑才对得上。

管理台的域编辑页加一个下拉。**不填就用默认**，不强制选。

### 4.3 采集：每次提问记一条

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

**允许 NULL**：老数据没有这些值，而且失败请求可能拿不到 usage。
NULL 表示"没采到"，与 0 不同——0 表示"确实没消耗"。

**只记在审计里，不建成本快照表**：现在数据量小（30 天 6600 次），
实时聚合够用。等量级上来再加预聚合——与健康度看板同一取舍
（见 `health-dashboard/spec.md`）。

### 4.4 成本看板

挂在健康度页第三个标签「成本」。

```
概览（最近 30 天）
  ￥12.47      总计
  1,204        提问数
  ￥0.0104     单次均价
  38.2%        缓存命中率  ← cachedTokens / (inputTokens + cachedTokens)
```

```
按模型
模型                提问    输入      缓存命中   输出      费用
DeepSeek V4 Flash  1,102   892K      341K(38%)  156K     ￥8.21
DeepSeek V4 Pro      102   104K       28K(27%)   31K     ￥4.26
```

```
按域
域           提问    费用     单次均价
保险分析域    412    ￥6.83   ￥0.0166
门店分析域    204    ￥1.02   ￥0.0050
```

**缓存命中率单独一列**：它是省钱最直接的手段。命中率高说明
prompt 前缀稳定（语义模型没频繁变），那是个可以追求的健康指标。

## 5. 实现

### 5.1 后端

| 位置 | 改动 |
|---|---|
| `AskEngineModelProperties` | `model-name` 单值 → `default` + `models` 映射 |
| `AskEngineModelFactory` | 按 key 建模型实例，**按 key 缓存**（每次新建会重复建 HTTP 连接池） |
| `AskServiceFactory` | 取域时带上 `modelKey`，传给引擎 |
| `AskService` / 审计 | 拿到 `Msg.usage` 后写进审计 |
| `sql/24-model-cost.sql` | 两个 ALTER（域 + 审计） |
| `HealthService` | 新增 `cost(days)` 聚合 |
| `HealthController` | `GET /health/cost` |

### 5.2 计价

新建 `ModelPricing`（纯函数，可单测）：

```
费用 = inputTokens     × inputPerMTok     / 1e6
     + cachedTokens    × cachedInputPerMTok / 1e6
     + outputTokens    × outputPerMTok    / 1e6
```

**三个易错的点，已用真实调用实测确认**（2026-11-04，火山方舟）：

实测方法：同一个长 system prompt 连续调两次，看第二次的 `cached_tokens`。

```
第 1 次:  prompt=1032  cached=0      completion=42  total=1074
第 2 次:  prompt=1032  cached=1024   completion=49  total=1081
```

**① `prompt_tokens` 已包含 `cached_tokens`**

两次的 `prompt_tokens` 都是 1032，而缓存第二次命中了 1024。
如果 `cached_tokens` 是额外加的，第二次应该变成 2056。

所以**不能相加**，正确写法：

```
未缓存部分 = inputTokens - cachedTokens
费用 = 未缓存部分 × inputPerMTok / 1e6
     + cachedTokens × cachedInputPerMTok / 1e6
     + outputTokens × outputPerMTok / 1e6
```

写错成 `inputTokens × 全价 + cachedTokens × 缓存价` 会**高估一倍**。

**② `reasoning_tokens` 已包含在 `completion_tokens` 里**

第一次实测：`completion=29`，其中 `reasoning=27` —— 不是 56。
再单独加会重复计费。

**③ `total_tokens = prompt_tokens + completion_tokens`**

两次都对得上（1032+42=1074、1032+49=1081）。
但**不要用它算钱**——它把缓存命中的部分也按全价算了。

**这些不是查文档得出的，是实测。** 各家语义不同，换 provider 时必须重跑
这两个测试（见验收 §7）。

### 5.3 前端

健康度页加第三个标签「成本」，与「概览」「质量分析」同一层级。

## 6. 验收

1. **多模型可配**：配 `fast` / `strong` 两个，域分别绑定，提问后审计里
   记录的 `model_key` 与域绑定一致
2. **token 采集正确**：拿一次真实调用，`input/output/cached` 与
   方舟控制台看到的数字一致
3. **费用口径正确**：用一次已知 token 的调用手算，与看板一致
4. **未配单价的模型**：显示「未配单价」而不是 0
5. **老数据兼容**：加列前 6600 条审计记录，`model_key` 与 token 为 NULL，
   成本看板跳过它们而不是算成 0
6. **失败请求**：没有 usage 时记 NULL，不记 0

## 7. 风险

| 风险 | 应对 |
|---|---|
| **`inputTokens` 是否含 `cachedTokens` 各家不同** | 用方舟实测核对；实现里写成可配置的开关，不靠猜 |
| 单价配置过期，成本算错 | 单价带「生效日期」字段；看板显示「单价更新于 X」 |
| 模型实例按 key 缓存，改配置不生效 | 与数据源同一取舍：配置变更后清缓存（`evict`） |
| 成本算得过于精细反而误导 | 明确标注「估算，以云厂商账单为准」 |

## 8. 不做

- 自动选模型（缺准确率反馈信号）
- 预算告警 / 限额
- 跨租户成本分摊（现在没有多租户）
- 缓存命中率的优化建议（先有数据）
