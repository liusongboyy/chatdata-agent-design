# 建模助手：从数据源生成域草稿

- Status: Draft（待确认）
- Date: 2026-10-27
- Related: `docs/modeling-coverage.md`、`.scratch/self-built-chatbi/spec.md`、`docs/adr/0002`

## 1. 要解决什么

我们已经用评测证明了：**建模覆盖度是准确率最大的杠杆**——
保险域只补三条聚合登记（`COUNT_DISTINCT` / `AVG` / 双登记维度），
基线从 74.2% 升到 83.3%，**代码一行未改**。

但建模本身极贵。AdventureWorks 演示域：28 张表、172 个列定义、30 条关联，
是手工写的，花了大半天。**最大的杠杆挂着最高的使用成本。**

而且手工建模容易漏——覆盖度检查现在会报 22 条警告，
其中大部分是"忘了给某列配 AVG / COUNT_DISTINCT"这类机械遗漏。

## 2. 目标与非目标

**目标**：选几张表 → 生成一份**可直接编辑的域草稿**，把机械劳动压掉，
管理员只做业务判断。

**非目标**：
- **不自动建域**。草稿必须人工确认后才落库。
- **不生成业务别名**。只从注释/列名给出**候选**，标「待确认」。
  业务名（「保费」而不是「premium」）是业务知识，机器猜不出来。
- **不推断派生指标与条件指标**。口径必须人来定，这正是 `derived-metrics` 与
  `conditional-metrics` 两轮反复强调的。
- **不猜默认过滤**（`base_filter`）。它是业务口径，猜错会静默少数据。

## 3. 为什么是"草稿"而不是"直接写库"

推断必然有错：
- 把 `Status` 推断成可枚举维度是对的，把 `Description` 也当枚举就错了
- 把 `ProductID` 推断成指向 `product` 表的 JOIN，在有些库里 `ProductID`
  是业务编码而不是外键

**推断错的东西直接生效，就是这个项目最怕的那种失败**：不报错、只是数字不对。
所以流程只能是：**生成 → 人看 → 确认 → 落库**。

草稿里每一条推断都带 `reason` 字段，管理员能判断"这条为什么这么建议"。

## 4. 推断规则

### 4.1 列角色与聚合

输入：列名、类型、是否主键、注释、基数（去重计数，采样）。

| 判据 | 角色 | 建议聚合 | 置信度 |
|---|---|---|---|
| 主键 | 维度 | — | 高 |
| 整数/小数，名称含 `amount/price/cost/fee/total/qty/score/count/rate` 或中文「金额/数量/价格/费用/次数/率/额」 | 指标 **+ 同列维度** | `SUM` + 一条 `AVG` | 高 |
| 文本，基数 ≤ 12 | 维度 **+ 去重计数** | — + `COUNT_DISTINCT` | 中（基数决定） |
| `bit` / `boolean` | 维度 | — + `COUNT` 条件指标（`= 1`） | 中 |
| 日期/时间 | 维度（时间口径候选） | — | 高 |
| 其他文本 | 维度 | — | 高 |

**为什么金额列要同时补一条维度**：`docs/modeling-coverage.md` 里记的
「50 岁以上的客户」就是栽在这——只有 `AVG(age)`，没法写 `age > 50`。

**为什么文本列要补 `COUNT_DISTINCT`**：「一共涉及多少个省份」是最高频问题之一，
没有它模型只能逐项列出。

### 4.2 表间关联

**三档置信度，必须分开标**：

| 来源 | 判据 | 置信度 | 说明 |
|---|---|---|---|
| 外键 | JDBC `getImportedKeys` | **高** | 直接用 |
| 命名 | `XxxID` 且存在表 `xxx` 且有主键 `XxxID`（或 `id`） | 中 | 标注「按命名推断」 |
| 无 | 以上都不成立 | — | 不生成，留给人工 |

**为什么命名推断不能省**：本仓库的 AdventureWorks 库里**一条外键都没有**
（导入时删掉了），真实企业库也常如此。只靠外键，这个功能在主场景上直接失效。

命名推断的歧义要显式列出来：`XxxID` 可能匹配到多张表时，**列全部候选让人选**，
不挑一个。

JOIN 类型默认 `LEFT JOIN`：`INNER` 会让维表缺行时丢事实行，而那种丢失是静默的。

### 4.3 敏感列默认关闭

列名命中下面任一项时，`enabled=false` 并写明原因：

```
idcard / id_card / phone / mobile / email / password / pwd / bank / cardno
身份证 / 手机 / 电话 / 邮箱 / 密码 / 银行卡
```

**默认关闭而不是默认开启**：失败方向必须安全。漏关一个敏感列 = 数据泄露；
误关一个普通列 = 模型少一个字段，用户看得见（"这个字段问不出来"）。

### 4.4 基数估计

`SELECT COUNT(DISTINCT col) FROM (SELECT col FROM tbl LIMIT 5000) t`

- 采样 5000 行：基数判断只需要量级
- 只对文本列做：数值列的基数对角色判断没帮助
- 单列失败不阻断整表：超时/权限问题记 warning

## 5. 草稿结构

```json
{
  "domain": {"name": "销售分析域", "datasourceId": 140, "mode": "MODELED"},
  "tables": [{
    "name": "salesorderheader", "label": "销售订单", "mode": "TABLE",
    "columns": [{
      "col": "TotalDue", "type": "double", "agg": "SUM",
      "alias": "TotalDue", "aliasConfirmed": false,
      "enabled": true, "confidence": "HIGH",
      "reason": "数值列 + 名称含 total → SUM 指标；同列已补一条维度"
    }]
  }],
  "joins": [{
    "left": "salesorderdetail", "right": "product",
    "leftCol": "ProductID", "rightCol": "ProductID",
    "type": "LEFT JOIN", "confidence": "NAMING",
    "reason": "列名 ProductID 匹配表 product 的主键 ProductID"
  }],
  "coverage": [ /* ModelingCoverageCheck 的结果，直接附上 */ ],
  "warnings": [
    "表 vendor 有 0 行，创建后相关提问会返回空",
    "3 个列命中敏感列规则，已默认关闭：phone, email, idcard"
  ],
  "stats": {"tables": 5, "columns": 60, "joins": 7, "elapsedMs": 1200}
}
```

**`aliasConfirmed=false` 是关键**：管理台把它高亮出来，提示"这个名字是机器生成的，
请确认或改掉"。业务名不确认就不该上线。

## 6. 流程

```
管理台「从数据源生成域」
   │  选数据源 → 勾表（可全选 / 按前缀筛）
   ▼
后端 ModelingDraftService
   │  ① 读元数据（MetaReader，已有）
   │  ② 读外键（新增 getImportedKeys）
   │  ③ 逐列估计基数（采样）
   │  ④ 推断角色 / 聚合 / 关联 / 敏感列
   │  ⑤ 组装草稿 → 跑 ModelingCoverageCheck
   ▼
返回草稿（**不落库**）
   │
   ▼
管理台草稿编辑页
   │  逐表/逐列可改；待确认项高亮；覆盖度警告就地显示
   │  「确认」→ 走现有 createDomain / createTable / saveJoins
   ▼
落库
```

**不新增写接口**：确认后复用现有的域/表/关联创建接口，
避免出现"草稿路径"与"手工路径"两套写入逻辑。

## 7. 实现清单

| 位置 | 改动 |
|---|---|
| `MetaReader` | 新增 `listImportedKeys(datasourceId, table)` |
| `engine.modeling.ColumnInferrer` | 纯函数：列 + 基数 → 角色/聚合/置信度/理由。**可单测** |
| `engine.modeling.JoinInferrer` | 纯函数：表集合 + 外键 + 命名 → 关联候选。**可单测** |
| `engine.modeling.SensitiveColumns` | 敏感列名单（常量表） |
| `engine.modeling.ModelingDraftService` | 编排：读元数据 → 推断 → 覆盖度检查 |
| `AdminEngineController` | `POST /datasources/{id}/modeling-draft` |
| `AdminApiPermissions` | 权限规则（与 tables 同档） |
| 管理台 | 草稿编辑页（P1） |

**P0 只做后端 + API**，用 curl 验证；管理台 UI 放 P1。
理由：推断规则本身要先能验证准不准，UI 会掩盖规则的问题。

## 8. 验收

1. **AdventureWorks 五张表**（header / detail / product / customer / salesterritory）：
   生成的草稿里，金额列有 `SUM` + `AVG`、状态列有 `COUNT_DISTINCT`、
   日期列被识别、`ProductID`/`CustomerID`/`TerritoryID` 的关联被推断出来
2. **覆盖度警告数**明显低于手工配置：手工 28 表 22 条警告；草稿应把
   「缺 AVG / 缺 COUNT_DISTINCT / 缺双登记」这三类降到 0
3. **敏感列**：命中规则的列 `enabled=false` 且带原因
4. **元数据读不到时不崩**：数据源连不上 → 返回明确错误，不是 500 堆栈
5. **性能**：5 张表 < 3 秒（含基数采样）

## 9. 风险

| 风险 | 应对 |
|---|---|
| 推断错被当成对的用 | 每条带 `confidence` 与 `reason`；业务名必须人工确认；草稿不自动落库 |
| 命名推断产生大量噪音 | 只推 `XxxID` 且能对上主键的；对不上不推；多候选全列出让人选 |
| 大表采样拖慢 | 采样 5000 行 + 单列超时；基数只用于文本列 |
| 与手工建模分叉 | 不新增写接口，确认后走现有 createTable / saveJoins |
| 覆盖度检查的提示在草稿里重复 | 草稿直接内嵌检查结果，不另写一套判据 |

## 10. 不做

- 自动建域、自动生效
- 生成业务别名（只给候选，标待确认）
- 推断派生指标 / 条件指标 / 默认过滤
- 从 SQL 语句反推域
- 跨数据源关联
