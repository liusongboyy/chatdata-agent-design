# 分析编排：一次提问，多次取数，确定性洞察

- Status: Done（2026-10-26）
- Date: 2026-10-26
- Related: `docs/adr/0002`（权限执行点）、`.scratch/self-built-chatbi/spec.md` §8、`.scratch/analytics-dsl/`、`.scratch/conditional-metrics/`

## 1. 要解决什么

现在一条问题 = 一个 DSL = 一条 SQL。下面这类问题**结构上答不了**：

| 问法 | 需要什么 |
|---|---|
| 「为什么 2025 年保费比 2024 年下降了？」 | 总量对比 + 按险种拆解 + 找出贡献最大的下降项 |
| 「各区域对增长的贡献分别是多少？」 | 分区域增量 + 增量占总体增量的比例 |
| 「华东的异常点在哪几个月？」 | 按月序列 + 偏离基线的识别 |
| 「保费下滑主要是哪个渠道造成的？」 | 分渠道变化 + 排序 + 归因结论 |

它们的共同点：**不是一条 SQL 能算完的，而是一组取数 + 一层计算 + 一句解释**。
上一轮做的 sort/占比/同环比/条件指标是"一条查询能表达更多"，
这一轮要解决的是"一次提问能跑多条查询"。

## 2. 非目标

- **不是让模型自由编排、自由写 SQL。** 每个子步骤仍然是一个受约束的 DSL，
  仍然过同一套权限、审计、只读、行数上限。这一点不让步。
- **不是 BI 笔记本**：没有自由拖拽、没有跨域查询、没有用户自定义公式。
- **不是长时运行的自主 agent**：一次提问内的编排，有硬上限，结束即返回。
- **不改现有单查询链路的行为**。193 条评测基线必须原样通过。

## 3. 总体结构

```
问题
 │
 ├─ 路由：单查询（现状）还是分析计划？（§5.1）
 │
 ├─ ① 计划：模型产出 AnalysisPlan（N 个步骤，每步一个 DSL）
 │      模型只做**分解**，不做算术
 │
 ├─ ② 执行：逐步 渲染(权限) → 执行 → 收集
 │      每步与今天的单查询走**同一条** render/execute 路径
 │
 ├─ ③ 计算：确定性洞察（Java）
 │      环比/贡献度/占比变化/离群点，全部是可复核的算术
 │
 └─ ④ 叙述：模型只根据 ③ 算出来的数字写结论
        不允许出现 findings 之外的新数字
```

**为什么把"算"和"说"分开**：让模型边算边说，算错了不报错，只是数字不对。
这与前面几轮反复处理的是同一类失败——静默语义错误。

## 4. 数据模型

```java
/** 分析计划。steps 有硬上限（≤4）。 */
record AnalysisPlan(String intent, List<PlanStep> steps) {}

/**
 * 一个步骤。
 * @param id      稳定标识（s1/s2…），叙述里引用它
 * @param label   人话标签（「2025 年保费总额」），界面上作为小标题
 * @param purpose 这一步想回答什么，给叙述阶段做上下文
 * @param dsl     与单查询**完全一样**的 DSL
 */
record PlanStep(String id, String label, String purpose, AskDsl dsl) {}

/** 一个步骤的执行结果或失败。status 只有 OK / FAILED 两种。 */
record StepResult(String id, String label, String purpose, AskDsl dsl,
                  RenderedSql sql, QueryResult rows, Set<String> maskedColumns,
                  Status status, String error) {}

/**
 * 一条由 Java 算出来的事实。
 * @param kind     TREND / CONTRIBUTION / SHARE_CHANGE / OUTLIER / COMPARE
 * @param stepId   来自哪一步
 * @param text     人话（「产品 A 贡献了 62% 的降幅」）
 * @param evidence 计算依据的原始数字，供叙述引用与审计复核
 */
record Finding(Kind kind, String stepId, String text, Map<String, Object> evidence) {}

/** 放在 AskResult 里，单查询时为 null。 */
record AnalysisResult(String intent, List<StepResult> steps,
                      List<Finding> findings, String narrative,
                      int queryCount, boolean truncated) {}
```

`AskResult` 增加一个可空字段 `analysis`。**不新开一条返回链路**——
前端、审计、历史记录都复用现有的 `AskResult`，单查询时只是这个字段为空。

## 5. 执行

### 5.1 入口：先显式，后自动

第一版**不做自动路由**，入口有两个：

1. 界面上的「深入分析」开关（用户自己决定要不要多花几次查询）
2. 问题里出现分析意图词（「为什么」「原因」「贡献」「拆解」「主要是什么造成的」）

理由：自动路由会改变现有 193 条基线的行为，任何准确率波动都分不清是
"编排变强了"还是"路由选错了"。等编排本身稳定、并且有一套分析评测集之后，
再打开自动路由（§12 Phase 2）。

### 5.2 计划阶段

- 复用现有的手写 JSON Schema 思路（`AskDslSchema` 的教训：反射生成会丢字段），
  新增 `AnalysisPlanSchema`。每个 step 的 `dsl` 复用 `AskDslSchema` 的字段定义。
- 计划阶段的 prompt 里给出域内指标/维度（同 `PromptAssembler`），并要求：
  每步一个明确目的、步骤之间不要重复取数、总步数 ≤4、**不要在某一步里塞多个问题**。
- 校验：每一步都过 `DslParser`；解析失败的步骤**整条计划作废**并回退到单查询
  （不做"跳过坏步骤"——半份计划得出的结论比单查询更可疑）。

### 5.3 执行阶段

- 每个步骤：`SemanticModel.resolve*` → `AskSqlRenderer.render` → `SqlExecutor.execute`。
  与 `AskService` 里今天那条路径**是同一份代码**，不复制。
- 顺序执行（不并发）。原因：同一数据源上并发多条查询会把连接池和慢查询放大，
  而分析问题的步骤之间有依赖（后一步可能要看前一步的结果），
  并发只对第一步之后的"并行取数"有意义，留到确有需要时再加。
- 单步失败**不中断整体**：标记 `FAILED`，继续跑后续步骤（它们不依赖失败步骤）。
- 总行数上限、单步行数上限、总耗时上限见 §11。

### 5.4 叙述阶段

- 输入：意图 + 每步的**摘要**（列名、行数、前 5 行、findings）。
  **不把全量结果喂给模型**——成本不可控，而且模型看几十行数字只会瞎猜。
- 输出：≤3 句结论，必须引用 findings 里的数字与步骤编号；
  有失败步骤时必须明说"这一步没查出来"。
- 叙述**不影响**表格与 findings 的展示：即使模型输出为空，界面仍然完整。

## 6. 确定性洞察（Java 侧）

只做能写清判据、能复核的几类；每类都要有单测钉住数字。

| 类型 | 输入形态 | 算法 |
|---|---|---|
| 趋势 | 时间维度 + 指标，按日/月/年 | 相邻期变化、首末变化、最大变化点 |
| 贡献度 | 分类维度 + 指标，两期（compare） | 各类增量 / 总增量；按增量绝对值排序，给出 Top 3 |
| 占比变化 | 分类维度 + shareOf，两期 | 各类占比差值，给出变化最大的分类 |
| 离群点 | 时间/分类序列 | 偏离均值 2σ 以上的点（样本 < 8 不算） |
| 对比 | 同比/环比结果 | 增长率、正负、是否由某类主导 |

**规则**：

- 每条 Finding 必须带 `evidence`（参与计算的原始数字），审计与界面都能核对。
- **不基于被脱敏的列计算**：脱敏列的值是 `***`，在它上面算出来的洞察是垃圾，
  还会泄露"某个受限数字的大致量级"。若步骤的唯一指标被脱敏，该步不产出 findings。
- 数字一律用 `BigDecimal`，避免浮点误差让"贡献 62%"变成"贡献 62.00000001%"。

## 7. 权限

- **请求级判定一次**（域可见 + 行规则 + 脱敏列），与今天 `AskPermissions.require` 相同。
- 每个步骤的 SQL 都过同一个 `AskSqlRenderer`，因此行级谓词、脱敏、只读、
  行数上限**自动**对每一步生效——这是"步骤必须是 DSL"最重要的理由。
- `Finding` 的计算在**脱敏之后**的数据上做；被脱敏的列不参与（§6）。
- 失败步骤的错误信息不透露字段/表结构（沿用现有错误脱敏）。

## 8. 审计

一次分析 = 一条主记录 + N 条步骤记录。

```sql
ALTER TABLE ask_audit_log
  ADD COLUMN parent_id BIGINT NULL COMMENT '父记录 id；步骤行为空表示这是一次完整提问',
  ADD COLUMN step_no   INT    NULL COMMENT '步骤序号，从 1 开始',
  ADD COLUMN plan_json TEXT   NULL COMMENT '分析计划原文（仅主记录）',
  ADD COLUMN narrative TEXT   NULL COMMENT '合成结论（仅主记录）';
```

- 主记录：question / plan_json / narrative / 总耗时 / 总行数 / outcome。
- 步骤记录：dsl_json / sql_text / row_count / duration_ms / outcome，`parent_id` 指向主记录。
- 被拒绝的请求仍然只有一条主记录（没有计划），与今天一致。
- 查询侧只查 `parent_id IS NULL` 就能得到"提问列表"，向后兼容。

## 9. 会话与多轮

- 多轮追问（「那华东呢」）需要上一轮的计划上下文。第一版只把
  **上一轮的计划摘要 + findings** 存进 agent session state（按 `(userId, sessionId)`），
  **不存原始行**——行数可能上千，塞进状态存储会把上下文撑爆。
- 追问时计划阶段能看到"上一轮查了什么"，从而只改一个维度，而不是重跑全部。

## 10. 前端

- 新增阶段：`PLANNING("正在拆解问题")`、`SYNTHESIZING("正在总结")`；
  `EXECUTING` 文案带上进度（「正在查询数据（2/4）」）。
- 结果区自上而下：
  1. **结论**（narrative，1–3 句）
  2. **发现列表**（findings，每条带步骤标签）
  3. **各步骤**：可折叠，每步一张小表 + 自己的 SQL（沿用现有的表格/图表组件）
  4. 页脚：`共查询 N 次 · 耗时 X 秒`
- 「我理解成了」按步骤展示（每步一份），不合并——合并会掩盖"某一步理解错了"。
- 失败步骤在列表里显式标红并写明原因，**不隐藏**。

## 11. 上限与降级

| 项 | 上限 | 超限行为 |
|---|---|---|
| 步骤数 | 4 | 计划阶段就拒绝，回退单查询 |
| 单步行数 | 1000（沿用） | 标记 `truncated`，叙述里说明 |
| 总耗时 | 30s（可配） | 停止后续步骤，已完成的照常展示 |
| 总行数 | 5000 | 同上 |

降级顺序：计划解析失败 → 单查询；单查询也失败 → 今天的错误路径。
**任何降级都必须在结果里说明降级了**，不能悄悄返回一个更窄的答案。

## 12. 分阶段

**Phase 1（本轮）**
1. `AnalysisPlan` / `PlanStep` / `StepResult` / `Finding` / `AnalysisResult` 模型
2. `AnalysisPlanner`（prompt + schema + 校验 + 回退）
3. `AnalysisExecutor`（复用现有 render/execute）
4. `InsightCalculator`（趋势/贡献/占比变化/离群，含单测）
5. `NarrativeSynthesizer`（prompt + 数字白名单）
6. `AskService` 增加 `analyze(...)`，`AskResult.analysis`
7. SSE 两个新阶段 + 前端结果区
8. 审计父子表 + 迁移脚本
9. 分析评测集（6–8 条）+ 权限回归

**Phase 2（下一轮）**
- 自动路由（按意图词 + 评测门禁）
- 步骤间依赖（后一步可引用前一步的 Top 分类）
- 图表：每步独立推荐，结论区不画图

## 13. 验收

**功能**
1. 「为什么 2025 年保费比 2024 年下降」能给出：总量对比 + 险种拆解 + 结论，结论里的数字与表格一致
2. 「各区域对增长的贡献」的贡献度之和与总增量一致（允许四舍五入误差）
3. 某一步失败时：其余步骤正常展示，结论明确说明哪一步没查出来
4. 计划解析失败 → 自动回退单查询，界面标明"已回退为单次查询"

**权限**
5. 同一分析计划在受限角色下执行，每步 SQL 都带行级谓词；被脱敏列的步骤不产出 findings
6. 审计里主记录 + N 条步骤记录可还原每一次查询

**评测**
7. 新增 `eval/analysis-fixture.json`（6–8 条）。评分不只看"结果对不对"，
   还要看 **findings 是否命中**（如"贡献最大的分类"是否等于手工计算值）与
   **叙述是否引用了正确数字**（用数值抽取比对，不用 LLM 打分）
8. 现有 193 条单查询基线的通过率波动在 ±3pp 以内（编排不影响单查询链路）

## 14. 风险

| 风险 | 应对 |
|---|---|
| 延迟与成本翻倍（N 次查询 + 2 次模型调用） | 显式入口 + 步数上限 + 界面上写清"查了几次" |
| 模型过度拆解（同一件事拆成 4 步） | 计划 prompt 要求每步一个目的；步骤数上限；重复步骤检测（同一 DSL 哈希） |
| 叙述编造数字 | 只给 findings 里的数字；评测里做数值抽取比对；失败即扣分 |
| 半份计划比单查询更误导 | 计划解析失败整条回退，不做部分执行 |
| 审计体量 N+1 倍 | 步骤行只存 SQL/DSL/行数，不存数据 |

## 15. 已确认的决定（2026-10-26）

1. **入口**：第一版 = 界面「深入分析」开关 + 意图词（为什么/原因/贡献/拆解/主要造成）。
   自动路由留到 Phase 2，且必须先有分析评测集做门禁。
2. **叙述合成**：模型写结论，但只允许引用 findings 里的数字（数字白名单）；
   评测用数值抽取比对，不做 LLM 打分。
3. **审计**：`ask_audit_log` 增加 `parent_id / step_no / plan_json / narrative` 四列，
   一次分析 = 1 条主记录 + N 条步骤记录。

## 16. 实现顺序

| # | 内容 | 依赖 |
|---|---|---|
| 1 | 计划/结果/发现的数据模型 | — |
| 2 | `InsightCalculator`（确定性计算 + 单测） | 1 |
| 3 | 计划 schema / 解析 / 校验 | 1 |
| 4 | `AnalysisExecutor`（复用 render/execute） | 1 |
| 5 | `AskService.analyze` + `AskResult.analysis` | 2/3/4 |
| 6 | 叙述合成（prompt + 数字白名单校验） | 2/5 |
| 7 | 审计父子表 + 迁移脚本 | 5 |
| 8 | SSE 阶段 + 前端结果区 | 5 |
| 9 | 分析评测集 + 权限回归 | 全部 |

进度：**1–9 全部完成**。

- 数据模型、`InsightCalculator`、计划 schema/解析：21 条单测
- `AnalysisExecutor`（复用 render/execute，单步失败不中断、行数/耗时上限）
- `AnalysisService`（编排主流程，含降级、叙述回落、审计）
- `NarrativeGuard`（数字白名单）：7 条单测
- 审计：`sql/19-ask-audit-analysis.sql` 加 4 列 + `idx_batch`
- SSE：`PLANNING` / `SYNTHESIZING` 两个新阶段；前端「深入分析」开关与结果区
- 引擎测试 578 通过

实现时改了一处设计：审计关联用 `batch_id` 而不是 `parent_id`。
原因是要拿父记录自增 id 就得"插入后回读主键"，而审计记录是不可变 record +
MyBatis 注解 mapper，绕这一圈没有收益；批次号同样能唯一还原一次分析。
