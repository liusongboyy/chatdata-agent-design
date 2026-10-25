# 派生指标：比值 / 差值 / 时长

- Status: Done（2026-10-25）
- Date: 2026-10-25
- Related: `.scratch/self-built-chatbi/spec.md` §8（DSL 是检查点）、`docs/adr/0002`（权限执行点）

## 1. 要解决什么

评测基线里有三类 HARD 问题答不了，因为它们需要**指标之间的运算**：

| 问法 | 需要的表达 | 当前 |
|---|---|---|
| 「已解决工单占比是多少」 | 已解决数 / 总数 | 答不了 |
| 「平均解决时长是多少小时」 | 两个时间列相减再平均 | 答不了 |
| 「赔付金额比申请金额少多少」 | 赔付 − 申请 | 答不了 |

现有 DSL 的 `metrics` 只能引用**已注册的物理列**，每列带一个固定聚合方式。
`SUM(a)/SUM(b)` 这类写法在 DSL 里没有位置。

## 2. 决定：派生指标定义在语义层，不由模型写算式

**模型只引用名字，不做算术。** 理由：

- **口径必须可治理。** 「赔付率」是分子分母都有业务定义的指标，不该让模型每次自己拼；
  拼错了不报错，只是数字不同——正是 PRD §12 说的静默语义错误。
- **「模型不写 SQL」这条不能被绕过。** 如果 DSL 允许随意算式，就等于把 SQL 表达式
  重新交给模型，权限与审计的检查点会失效。
- **顺带不需要改 DSL 结构。** `{"metrics":["赔付率"]}` 与引用普通指标完全一样。

派生指标在管理台随注册表一起配置，存在 `columns_json` 里。

### 2.1 配置形态

```json
{
  "col": "",
  "type": "decimal",
  "agg": "NONE",
  "alias": "赔付率",
  "enabled": true,
  "derived": { "kind": "ratio", "left": "赔付金额", "right": "申请理赔金额" }
}
```

三种 `kind`：

| kind | 字段 | 渲染 |
|---|---|---|
| `ratio` | `left` / `right` | `SUM(left) / NULLIF(SUM(right), 0)` |
| `diff` | `left` / `right` | `SUM(left) - SUM(right)` |
| `duration` | `start` / `end` / `unit` | `AVG(<timeDiff>(unit, start, end))` |

`left` / `right` / `start` / `end` 都是**域内已注册列的别名**，可以跨表——
渲染器按现有 JOIN 规划自动把它们所在的表拉进来。

`NULLIF(..., 0)` 是必须的：分母为 0 时返回 NULL 而不是报错，也不会得到
"除以零"这种让人以为系统坏了的失败。

### 2.2 🔴 脱敏必须传导

**这是本节最重要的一条。**

如果 `apply_amount` 配了列脱敏，而「赔付率」由它派生，那么**派生指标也必须脱敏**——
否则受限用户通过「赔付率」就能反推出被脱敏的金额。

规则：**任一输入列被脱敏 → 派生指标整体脱敏**（输出 `'***'`，不执行计算）。

同理，派生指标的输入列必须在权限判定时按普通列参与角色匹配。

### 2.3 时长需要方言

`TIMESTAMPDIFF(HOUR, a, b)` 只有 MySQL/Doris 有。PostgreSQL 用
`EXTRACT(EPOCH FROM (b - a))/3600`，Oracle 用 `(b - a) * 24`，SQL Server 用
`DATEDIFF(HOUR, a, b)`。

故在 `SqlDialect` 增加一个方法 `timeDiffToNumber(unit, startExpr, endExpr)`，
由各方言实现。**这是"只放真正有差异的项"原则的又一次应用**。

## 3. 实现清单

| 位置 | 改动 |
|---|---|
| `ColumnDef` | 增加可空 `derived` 字段；有 derived 时 `col` 可为空 |
| `DomainLoader` | 解析 `derived` 块 |
| `SemanticModel` | `resolveMetric` 能解析派生别名，返回带输入的引用 |
| `AskSqlRenderer` | 按 kind 渲染；JOIN 规划纳入输入的列所在表 |
| `SqlDialect` | 新增 `timeDiffToNumber` |
| 脱敏 | 派生指标的任一输入被脱敏 → 整体脱敏 |
| `PromptAssembler` | 派生指标与普通指标一起列给模型（只给名字） |
| 启动校验 | 派生指标的输入解析不了时在启动报 `BROKEN_DERIVED_METRIC`，不等到用户提问（判据复用 `resolveMetricRef`，与运行时一致） |
| 管理台 | 列定义里可配三种派生（P1，先支持 JSON 直接配） |

## 4. 验收

1. 「已解决工单占比」：`SUM(CASE...)` 不行，必须走 ratio 派生指标，结果与手算一致
2. 「平均解决时长（小时）」：与 gold SQL 的 `AVG(TIMESTAMPDIFF(HOUR,...))` 一致
3. **脱敏传导**：`分析师` 角色下，输入列脱敏时派生指标输出 `***`，而不是算出的数字
4. 分母为 0 时不报错（返回 NULL）
5. 跨表派生：分子分母在不同表上时，JOIN 自动补齐
6. 评测集：新增用例覆盖三类，且基线不下降

## 5. 不做

- 不支持任意表达式（`A*B+C`）。只做三种有明确业务含义的形态——
  开放任意算式等于把 SQL 交给模型。
- 不做派生**维度**。维度是分组的键，做运算没有业务含义。
