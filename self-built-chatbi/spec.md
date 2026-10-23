# PRD：自研问数与语义层，替换 JimuChatBI

Status: draft
Type: spec

## 1. 背景与动机

`docs/adr/0001` 决定**集成** JimuChatBI 承担语义层、NL2SQL、数据源适配与图表展现，本项目只自建治理层。该决策基于两个前提，其中第二个在本轮排查中已被证伪。

**前提一：自研同路径需 6–9 个月，NL2SQL 准确率无确定工期。** 仍然成立，本 PRD 不否认这一点。

**前提二（已证伪）：集成后仍可在需要时修改厂商行为。** 实测结论：

| 事项 | 实测结果 |
|---|---|
| 源码 | 三个 `-sources.jar` 均为**空壳**，只有 `README.md`，无一行 Java |
| 字节码 | **混淆**：`org/jeecg/modules/chat2bi/a/b.class`，方法名为 `a()`/`b()` |
| 页面模板 | **加密**：`META-INF/.templates/*` 为高熵密文，明文检索 `needLogin` 零命中 |
| 前端 JS | **明文可读**，且可被本项目 `static/` 覆盖（已实测验证） |

即：**后端是只读的黑盒**。能改的只有前端 JS，且要 fork 一份维护。

这直接推翻了一个隐含假设——"集成省下的工作量可以在治理层自由扩展"。实际是：任何厂商侧的行为缺陷（本轮就撞到两个：失效 token 导致页面永久卡死、连接测试无登录超时）都**无法修复**，只能绕过或提 issue。

**同时暴露的第三个问题：许可证未决。** `docs/adr/0001` 已记录：JimuChatBI 仓库声明 AGPL-3.0，但 POM 的 `<licenses>` 写的是"功能可以免费用"——不是许可证标识符。自研可彻底移除这项不确定性。

## 2. 目标与非目标

### 目标

用 `agentscope-java 2.0.4` + `Spring Boot 4.1` 自研一套**完整可用**的问数系统，覆盖 JimuChatBI 当前已实现的全部能力：

1. 管理数据源
2. 语义数据表
3. 语义建模
4. 助理管理
5. 问数链路（NL2SQL + 取数 + 图表 + 多轮追问）

### 非目标

- **不在本期决定如何切换。** 现有 JimuChatBI 集成保持原样不动，新模块独立建库建表并行存在。切换方式另出 ADR。
- **首版支持五种数据源，不追求一次做全。** MySQL / Doris / PostgreSQL / Oracle(12c+) / SQL Server。
  方言层按可扩展设计，但接口保持薄——后续新增库时再增厚（见 §8.1）。
- **不追求问数准确率一步到位。** 见 §10 分阶段：先跑通管道，再迭代准确率。

## 3. 术语

沿用 `GLOSSARY.md`。本 PRD 新增/明确：

**语义域（Domain）**：围绕一个数据源构建的语义模型单元，定义可用的表、表间关联、业务术语与权限规则。一次提问先路由到域，再在其中取数。_避免_：数据域、主题域

**注册表（Registered Table）**：域内一张可被问数的表。可以是物理表，也可以是 SQL 虚拟表。

**查询主题（Topic）**：域内一组预置字段组合，用于收敛 LLM 的取数范围。

**业务术语（Term）**：业务口径的显式定义，含近义词与三种应用方式（仅释义 / 时间口径 / 数据筛选）。

**问数 DSL**：LLM 产出的中间表示，由指标、维度、时间范围、筛选条件四项构成，先于 SQL 存在。**它是权限与审计的检查点。**

## 4. 用户与角色

沿用 `docs/adr/0004` 的结论：`chat2bi` 是**管理台角色**，不是问数角色。

| 角色 | 能做什么 |
|---|---|
| 平台管理员（`chat2bi`） | 全部管理面：数据源、语义数据表、语义建模、助理管理 |
| 业务用户 | 只能问数，且仅限被授权的域与助理 |

**ADR-0004 记录的教训必须继承**：不要让"能问数"隐含"能进管理台"。功能权限与数据权限在本期就要分开建模，不能重蹈覆辙。

## 5. 功能需求

### FR-1 管理数据源

| 编号 | 需求 | 优先级 |
|---|---|---|
| FR-1.1 | 数据源的增删改查：名称、类型、驱动类、JDBC URL、用户名、密码 | P0 |
| FR-1.2 | **连接测试**：新建时用表单值测，编辑时密码留空则沿用已存密码 | P0 |
| FR-1.3 | 密码加密存储，接口不回传明文 | P0 |
| FR-1.4 | 驱动可用性前置校验：所选类型的驱动不在 classpath 时，**明确报"未安装该数据库驱动"**，而不是抛裸类名 | P0 |
| FR-1.5 | 连接测试必须有超时（建议 10s），不得无限占用请求线程 | P0 |

> **FR-1.4 / FR-1.5 是对厂商缺陷的直接修正。** 实测：厂商下拉列出 14 种数据库，但 jar 内**只有 MySQL 驱动**；失败提示是 `连接失败：org.postgresql.Driver`（一个裸类名，用户无法理解）。且 chat2bi 的测试路径没有 `DriverManager.setLoginTimeout`，打黑洞地址实测 **30 秒**才返回。

### FR-2 语义数据表

| 编号 | 需求 | 优先级 |
|---|---|---|
| FR-2.1 | 从数据源同步表清单与列元数据（走 JDBC `DatabaseMetaData`） | P0 |
| FR-2.2 | 表注释双轨：`source_comment`（同步覆盖）+ `table_comment`（人工维护，同步不覆盖） | P0 |
| FR-2.3 | 列级：类型、注释（同上双轨）、是否主键、可空、样例值 | P0 |
| FR-2.4 | 表间关联关系维护（`column → ref_table.ref_column`，含类型） | P1 |
| FR-2.5 | **对 LLM 可见开关**（`is_enabled`）：关闭的表不进 prompt | P0 |

> FR-2.5 是成本与准确率的双重控制点：表越多，prompt 越长、选错表概率越高。

### FR-3 语义建模

| 编号 | 需求 | 优先级 |
|---|---|---|
| FR-3.1 | 语义域 CRUD：名称、描述、绑定数据源、启用状态、排序 | P0 |
| FR-3.2 | 域内注册表：物理表 / SQL 虚拟表两种模式；SQL 模式支持解析出字段 | P0 |
| FR-3.3 | 列定义：别名、聚合方式、是否启用 | P0 |
| FR-3.4 | 表关联（joins）：多表 join 定义，提供可视化（厂商用 ECharts 画图） | P1 |
| FR-3.5 | 查询主题 CRUD：字段组合 + 发布状态 | P1 |
| FR-3.6 | 业务术语 CRUD：近义词、释义、三种应用方式（none/time/filter） | P0 |
| FR-3.7 | 权限配置：授权角色/用户、敏感词、敏感类目、行规则、列规则、敏感度、是否公开 | P0 |
| FR-3.8 | **启动时校验**：每个启用中的域必须已设置有效授权，否则拒绝启动 | P0 |

> FR-3.8 对应 `AccessGrantStartupCheck` 的既有实现，必须保留——见 §7。

### FR-4 助理管理

| 编号 | 需求 | 优先级 |
|---|---|---|
| FR-4.1 | 助理 CRUD：名称、头像、描述、欢迎语 | P0 |
| FR-4.2 | 绑定数据域，支持**多域**（问数时智能路由） | P0 |
| FR-4.3 | 是否展示 SQL 开关 | P1 |
| FR-4.4 | 状态：上线 / 草稿 / 下线 | P0 |
| FR-4.5 | 推荐问题：手工维护 + AI 生成 | P1 |
| FR-4.6 | 访问权限：授权角色/用户 | P0 |

### FR-5 问数链路

| 编号 | 需求 | 优先级 |
|---|---|---|
| FR-5.1 | 自然语言提问 → 问数 DSL → SQL → 结果 | P0 |
| FR-5.2 | 多轮追问：携带上下文，支持"再按区域拆一下"这类指代 | P0 |
| FR-5.3 | **事件级 SSE 流式**：区分"思考中/正在查数/最终答案" | P0 |
| FR-5.4 | 图表：至少表格 + 柱/折/饼/数字卡 | P1 |
| FR-5.5 | 基于 DSL 重算（不经过 LLM），用于结果复核与权限回归测试 | P0 |
| FR-5.6 | 澄清追问：问题不明确时反问用户 | P1 |
| FR-5.7 | 全链路审计：问题、DSL、SQL、结果摘要、耗时 | P0 |

> FR-5.3 沿用 `docs/adr/0003` 的结论：事件级 SSE，不用 token 级。FR-5.5 是厂商 `ask/recompute` 的等价物——**它让权限能脱离 LLM 被确定性测试**，是 ADR-0002 得以验证的基础，必须保留。

## 6. 数据模型

新建库表，**前缀 `ask_`**，与厂商的 `chat2bi_*` 完全隔离（便于并行运行与回退）。

| 表 | 说明 | 对应厂商表 |
|---|---|---|
| `ask_datasource` | 数据源 | `jimu_report_data_source` |
| `ask_table_meta` | 表元数据（同步 + 人工） | `chat2bi_table_meta` |
| `ask_domain` | 语义域 | `chat2bi_domain` |
| `ask_domain_table` | 域内注册表 | `chat2bi_table` |
| `ask_topic` | 查询主题 | `chat2bi_topic` |
| `ask_term` | 业务术语 | `chat2bi_term` |
| `ask_assistant` | 助理 | `chat2bi_assistant` |
| `ask_conversation` | 会话 | `chat2bi_conversation` |
| `ask_message` | 消息 | `chat2bi_message` |
| `ask_audit_log` | 审计（已存在，复用） | — |

字段设计以厂商 schema 为**参考而非契约**——它的注释里已经暴露了几个我们应当修正的地方：

- `chat2bi_table_meta.db_source_type` 用字符串枚举（`db_source`/`api_source`/...），我们只做 JDBC，应简化为外键 + 类型
- `perm_json` 把所有权限塞进一个 JSON blob，**无法建索引、无法约束**。新设计建议拆成 `ask_domain_grant`（授权）、`ask_row_rule`、`ask_col_rule` 三张表
- 厂商所有表都有 `tenant_id`，但本项目单租户（见 `ChatbiTokenService.getTenantId()` 返回 null），新表**不加**该列

### 权限存储：保留 JSON blob（已决）

**决定：不拆表，沿用厂商的 `perm_json` JSON 字段形态。**

选它的收益：

- **厂商的语义可以直接照搬**，不需要在两种表示之间做映射——ADR-0002 的实测结论（`rowRules[].role`、`colRules[].field`、`grantRoles` 等）可以逐字段对应
- 加字段不用改表结构，语义层还在演进期，这点省事

**必须接受的代价与缓解措施：**

| 代价 | 缓解 |
|---|---|
| 无法加 DB 约束，脏数据进得去 | **写入时校验**：解析成 record 再落库，不合法直接拒绝。JSON schema 变更时同步改 record |
| 启动校验（FR-3.8）不能写成一句 SQL，必须解析 JSON | `AccessGrantStartupCheck` 改为解析后校验。**注意性能**：域数量大时逐个解析 JSON 是启动期成本，可接受 |
| 无法按角色建索引做"我有哪些域"的查询 | 域数量级小（人工维护），全表加载后在内存过滤即可。**若将来域数量上千需重新评估** |
| JSON 结构无版本号，将来变更难追踪 | **在 `perm_json` 里加 `v` 字段记录 schema 版本**，解析时按版本分支。这是不拆表方案下唯一必须补的东西 |

> 最后一条是我加的硬性要求：不拆表意味着 schema 演进只能靠约定，没有版本号的话，半年后没人知道某个 JSON 是哪个版本写的。

## 7. 权限模型

**这是本 PRD 里最不能出错的部分。** `docs/adr/0002` 用实测把厂商语义钉死了，新实现必须继承其**语义**，同时修正其**默认放行**的缺口。

### 必须继承的语义

| 层 | 闸门判据（厂商） | 失败表现 |
|---|---|---|
| 域 | `grantRoles` / `grantUsers` 有内容，或 `isPublic=true` | 返回 **0 行** |
| 助手 | `enabled=true` **且** 角色在 `grantRoles` 内 | 返回**明确错误** |

行规则 `rowRules`：`{id, role, field, op, value, sysVar, dynamic}`，按角色匹配后改写查询。
列规则 `colRules`：`{id, role, field, action}`，`action` 为脱敏。

### 必须修正的三个缺口

1. **默认放行 → 改为默认拒绝。** 厂商两层的默认值都是"放行"：授权为空 = 对所有已认证用户开放；`enabled` 缺失 = 对任何角色集放行。新实现必须**默认拒绝**，并且把"未配置授权"视为配置错误（启动即拒绝，见 FR-3.8）。

2. **被拒绝返回 HTTP 200 + `{"blocked":true}` → 改为明确的状态码。** 厂商把"被拒绝"伪装成成功响应，导致审计里"被拒绝"与"查了但没数据"无法区分。新实现用 `403` + 结构化错误体。

3. **列级脱敏按字段配置，同一口径的等价字段会漏。** ADR-0002 实测：`premium` 脱敏了，等价的 `premium_total` 没脱敏，受限用户换个指标就看到明文。厂商行为"正确"（规则本就只覆盖配了的字段），但**语义层的纪律问题要在产品上堵**：注册表层面应提示"该列与已脱敏列疑似等价"。

### 强制约束

- **角色名逐字符匹配，且不能经 HTTP 头传递**（RFC 7230 只允许 ASCII，中文角色名经头传输会乱码并**静默放行**）。新实现从令牌内取角色，不经过头部。
- **`getRoles` 返回空集不是失败关闭。** 任何"角色为空就放行"的分支都是漏洞。
- **权限改写发生在生成的 SQL 上**，不是 ORM 拦截器上。ADR-0002 已证伪 ORM 拦截方案。

## 8. 技术架构

### 技术栈

| 组件 | 版本 | 状态 |
|---|---|---|
| Spring Boot | 4.1.0 | 本项目 parent 已是，无冲突 |
| agentscope-java | 2.0.4 | 已核实：2026-10-08 发布，`agentscope-core-2.0.4.jar`（857KB）等在 Central 上真实存在 |
| Java | 21 | 当前环境 |
| MyBatis / MiniDao | 沿用现有 | — |
| MySQL | 8 | 当前环境 |

### 模块划分

```
chatdata-agent-ask-engine/          ← 新增
  ├── datasource/                   数据源 + 连接池 + 驱动校验
  ├── meta/                         元数据同步（JDBC DatabaseMetaData）
  ├── semantic/                     语义域 / 注册表 / 主题 / 术语
  ├── assistant/                    助理
  ├── permission/                   授权 + 行/列规则改写（独立于 ORM）
  ├── nlsql/                        DSL 生成 + SQL 渲染
  ├── agent/                        agentscope agent 装配与工具注册
  └── chat/                         会话、SSE、审计
```

**当前状态（2026-10-16）**：§8 列的各包均已落地，全量构建与测试通过
（引擎 273 例、ask 模块 92 例）。逐个包的实现散在上表的同名目录下，不再逐一登记。

> ⚠️ **此处曾写「只落地了三处，其余待 P1 填充」——那份状态早已过期，
> 而它误导过一次排查**（照着它去找"还没做的功能"，结果代码早就写完了）。
> 需要知道某块做到哪，**以代码和测试为准**，不要以本节的历史快照为准。

**实现与本节草案的一处重要分叉：模型不写 SQL。**

| | 本节草案 | 实际实现 |
|---|---|---|
| 取数 | 模型注册 SQL 工具、自己写 SQL、多轮 ReAct | 模型只产出**结构化 DSL**，SQL 由 `AskSqlRenderer` 渲染 |

这处分叉是有意的，也是「DSL 是检查点」得以成立的前提（见 §8.3）：

- 模型不写 SQL → **SQL 里不存在"越权字段"这种可能**，权限校验挂在 DSL 的字段上，
  可确定性验证
- 没有工具调用 → 没有中间推理轮次 → 不需要抑制中间文本（§8.3）
- 顺带修掉了 spike E2 暴露的「静默语义错误」：约束丢失发生在 DSL 上，能在渲染前拦住

代价是**放弃了多轮工具调用带来的取数灵活性**（如先看表结构再决定查哪张表）——
这正是 §9 里 P3「语义增强」要补的部分。

> `datasource/SqlTools` 因此**当前没有被任何链路引用**（只被它自己的测试引用）。
> 它是为上面那条被放弃的路径写的，保留是因为其只读白名单 / 行数上限 / 超时三重防护
> 在将来接工具时可直接复用。**注意：它不能直接暴露给最终用户**——
> 里面写明了权限改写必须插在 `runQuery` 之前。

### 8.1 多数据源方言策略

首版支持五种库。**不用 jOOQ**——OSS 版 14 个方言里没有 Oracle 与 SQL Server（实测枚举确认），
这两个在 jOOQ Professional/Enterprise（商业授权），已决定不引入。故自建方言策略。

**方言差异矩阵**（按会出错的严重程度排序）：

| 差异 | MySQL | Doris | PostgreSQL | Oracle 12c+ | SQL Server |
|---|---|---|---|---|---|
| **标识符大小写折叠** | 不折叠 | 不折叠 | **折小写** | **折大写** | 不折叠 |
| **日期格式化** | `DATE_FORMAT(d,'%Y-%m')` | 同 MySQL | `to_char(d,'YYYY-MM')` | `to_char(d,'YYYY-MM')` | `FORMAT(d,'yyyy-MM')` |
| **行数限制** | `LIMIT n` | `LIMIT n` | `LIMIT n` | `FETCH FIRST n ROWS ONLY` | `TOP n`（**前缀**） |
| 标识符引号 | 反引号 | 反引号 | `"` | `"` | `"` |
| 布尔类型 | `tinyint(1)` | 同 | `boolean` | **无** | `bit` |

**硬性设计规则：标识符一律加引号，且按元数据报告的原文照抄。**

不加引号时，同一份元数据（列名 `Region`）在三个库上会解析成三个不同名字——
PostgreSQL 折成 `region`、Oracle 折成 `REGION`、MySQL 保持 `Region`。
加引号后不折叠，而 `DatabaseMetaData` 报告的正是库里实际存储的名字，引用它一定对。

**一条重要简化：相对时间在 Java 侧解析成具体日期，不生成 `CURDATE()` 之类的方言函数。**

「上个月」→ 在 Java 里算出 `2026-09-01` / `2026-10-01`，SQL 里只出现 `>= ? AND < ?`——
这是**方言中立的**。方言接口因此不需要「当前日期」这一项，少一处差异就少一类 bug。

驱动（均已在本地 m2，离线可构建）：

| 库 | 驱动 | 许可 |
|---|---|---|
| MySQL / Doris | `mysql-connector-j` | GPL + FOSS 例外 |
| PostgreSQL | `postgresql` | BSD |
| Oracle | `ojdbc11` | **FUTC**（免费使用与再分发，非开源） |
| SQL Server | `mssql-jdbc` | MIT（本地仅 7.4.1，偏旧，需验证对新版服务端的兼容） |

### 前端

**新写 Vue 页面**（已决）。不复用厂商前端，也不走 `static/` 覆盖。

- 厂商前端 JS 是**明文可读**的，可作为交互与信息架构的参考（四块管理面的功能清单已从其 JS 中枚举出来，见 PRD 修订记录），但**模板加密，不可复制**。
- 现有 `chatdata-agent-ask` 里已有 `AskStreamController` 等自建端点，前端可与其直接对接，无需再适配厂商的响应形状。
**技术选型（已决）：Vue 3。**

- 厂商用的是 Vue 2.7 + iView 4.7（从其 `static/` 依赖可读出）。**Vue 2 已于 2023-12-31 停止维护**，新写没有理由再上 Vue 2。
- 代价：参考厂商交互时多一层"Options API → Composition API"的翻译成本。但其 JS 是明文的，**信息架构与交互流程仍可直接借鉴**（四块管理面的功能清单已从其中枚举出来）。
- 组件库、构建工具、状态管理在 P0 spike 时一并定（候选：Element Plus / Naive UI / Ant Design Vue + Vite）。

### 问数链路（草案）

```
question
  → 助理路由（domain_ids → 选域）
  → 权限前置校验（域授权 + 助手授权）—— 拒绝则 403，不进 LLM
  → 上下文装配（域内启用表 + 表/列注释 + 术语 + 主题）
  → LLM → 问数 DSL
  → DSL 校验（字段存在性、聚合合法性）
  → SQL 渲染
  → 权限改写（注入 rowRules 谓词、标记 colRules 脱敏列）
  → 执行 → 结果
  → 图表推荐 + 文字结论
  → 审计落库
```

**关键设计：DSL 是检查点。** 权限校验、字段存在性、审计都挂在 DSL 上，而不是挂在 SQL 上。这样"LLM 生成的 SQL 是否越权"这个问题变成"DSL 里的字段是否都在授权范围内"，可确定性验证。

> **⚠️ P0 spike 实测补充（见 [spike-p0-results.md](spike-p0-results.md)）：这个检查点必须校验语义完整性，不能只校验可解析性。**
>
> 实测中，问题「上个月**华东**的**有效**保单保费合计是多少」被模型翻译成：
> `{metrics:["保费"], dimensions:null, timeRange:{kind:"relative", start:"上个月第一天（…）锁定文本：上个月"}, filters:null}`
>
> **「华东」和「有效」两个核心约束全部丢失**，而对象解析成功、类型合法、无任何异常。若直接渲染 SQL，会退化成"全量保费合计"——**答案错误且全程静默**。
>
> 这与 `docs/adr/0002` 记录的「等价字段脱敏遗漏」是**同一类失败**：不报错，只是给出错误结果。因此 §8 的设计必须补三条：
>
> 1. **DSL 字段禁止 null。** `dimensions` / `filters` 用紧凑构造器兜底成空列表，`timeRange` 保持可空但语义明确。下游不写判空分支——判空分支本身是 bug 温床。
> 2. **字段取值必须受约束。** `timeRange.kind` 实测被编造成 `"relative"`（应为中文口径名），`start` 被塞进整句话（应为 `yyyy-MM-dd`）。需在 record 上做构造期校验，非法值直接拒绝而不是放行。
> 3. **必须加语义完整性校验步骤。** 例如要求模型同时输出「已覆盖的约束清单」，由代码校验问题中出现的实体值是否都落在 DSL 里；缺失则触发一次重问。**这是本设计唯一的防静默失败手段。**
> 4. **⚠️ schema 必须手写，不得由 record 反射生成。** P0.5 实测（见 [spike-p0-results.md](spike-p0-results.md)）：`call(msgs, Class<?>, ctx)` 由 record 生成的 schema 会让 **filters 被系统性丢弃，10/10 全败**；改用 `call(msgs, JsonNode, ctx)` 喂手写 schema 后 **13/13 全对**。两者走同一代码路径、同模型、同温度，唯一差别是 schema 内容。代价是 schema 与 Java 类型可能漂移，**需要一条测试断言两者一致**。
>
> **第 3、4 条是硬性的**：第 4 条决定「能不能用」，第 3 条决定「错了能不能发现」。
> 另：流式与结构化输出**可同时成立**（E5 实测），故链路用**单段**即可，不必拆两段。

### agentscope-java 2.0.4 能力核实（已调研）

结论：**agent 循环、工具调用、流式、会话持久化、结构化输出、人在回路全部开箱即用；唯一需要自研的是 NL2SQL 本体。**

| 能力 | API | 对本项目的意义 |
|---|---|---|
| Agent 核心 | `io.agentscope.core.ReActAgent`（builder）；`call(List<Msg>, RuntimeContext)` → `Mono<Msg>` | 问数主循环 |
| 工具注册 | `@Tool` / `@ToolParam` 注解 + `Toolkit.registerTool(Object)`；复杂场景继承 `ToolBase` | 数据源查询、表结构内省、SQL 执行、图表渲染各注册一个 tool |
| 流式 | `streamEvents(...)` → `Flux<AgentEvent>`，28 种事件类型（`THINKING_BLOCK_*` / `TOOL_CALL_*` / `TEXT_BLOCK_DELTA` / `REQUIRE_USER_CONFIRM`） | **直接满足 FR-5.3**：思考中 / 正在查数 / 答案增量 三类事件可直接映射 |
| 结构化输出 | `call(msgs, DslRecord.class, ctx)` + `Msg.getStructuredData(...)`；模型不支持时自动降级为合成工具 `generate_response` | **DSL 用 Jackson record 定义，不靠解析自由文本** —— 这是 §8「DSL 是检查点」得以成立的前提 |
| 会话记忆 | `Memory` **自 2.0.0 起废弃**；改用 `AgentStateStore`，按 `(userId, sessionId)` 自动存取。实现有 InMemory / JsonFile / Redis / **Mysql / Jdbc** | 多轮追问用 `MysqlAgentStateStore`（本项目已有 MySQL，Redis 模块目前是空的） |
| 人在回路 | 工具规则设 `PermissionBehavior.ASK` → `call()` 提前返回，调用方回填 `ConfirmResult` 再 call；或 `ToolBase.externalTool(true)` + `RequireExternalExecutionEvent` | **FR-5.6 澄清追问可直接落地** |
| 模型接入 | **OpenAI 兼容是唯一入口**：`OpenAIChatModel.builder().baseUrl().apiKey().modelName()` | 可直接复用现有 `jeecg.jmreport.ai.base-url`（火山方舟 ark），**无 ark 专用模块** |
| Spring Boot | `agentscope-spring-boot-starter` 的 `AgentscopeAutoConfiguration` | **默认 bean 不设 stateStore**，生产必须自建 bean |

**没有现成的 NL2SQL 工具。** 全库检索 `text2sql|nl2sql|schema introspect` 零命中；`agentscope-extensions-{jdbc,mysql,postgresql}` 是 **state 存储**，不是 Tool。最接近的蓝本是示例工程 `agentscope-examples/agents/agentscope-dataagent` 的 `DataAgentToolkit`，提供 4 个 `@Tool`：`list_data_sources`、`describe_table`、`run_sql_preview`（仅 SELECT/WITH、行上限 200、30s 超时）、`render_chart`。**SQL 由模型自己写，没有独立生成器。**

> **这条决定了工期。** agentscope 给的是 agent 骨架，不是问数能力。schema 采集与裁剪、prompt 工程、DSL 设计、SQL 安全校验与权限改写 —— 全部是我们自己的活，正是 ADR-0001 说的"6–9 个月"所在。`DataAgentToolkit` 可作为 `run_sql_preview` 安全策略的直接蓝本。

### 模型接入的具体配法

火山方舟（现有 `jeecg.jmreport.ai.base-url` 指向 `ark.cn-beijing.volces.com/api/coding/v3`）走 OpenAI 兼容栈：

```java
OpenAIChatModel.builder()
    .baseUrl("https://ark.cn-beijing.volces.com/api/coding/v3")   // 默认回退是 api.openai.com/v1
    .apiKey(...).modelName(...)
    .build();
```

两个必须注意的点：

- **provider 必须显式设成 `openai`。** `agentscope.model.provider` 未设置时，**默认 provider 是 DashScope**，会走错栈。
- **结构化输出开关要按模型调。** `OpenAIChatModel.Builder` 有 `nativeStructuredOutput(boolean)` 与 `nativeStructuredOutputWithTools(boolean)`；官方对 DeepSeek/Kimi 的建议是关掉后者（其 provider 实现即设 `nativeStructuredOutput(false)`）。**我们的模型是 `deepseek-v4-1-flash`，P0 spike 必须实测这两个开关的组合**——它决定 DSL 结构化输出走 native `response_format` 还是降级到合成的 `generate_response` 工具。

### 版本核实（已解决）

| 项 | 结论 |
|---|---|
| 2.0.4 是否存在 | **存在**。`agentscope-core/2.0.4/` 目录在 Central 上，jar 857KB。注意 `agentscope-bom` 只有 `.pom` 没有 `.jar`（BOM 的正常形态，不是缺件） |
| 发布时间 | **2026-10-08**（与 2.1.0-BETA1 同日） |
| 是否有更新版本 | 有，`2.1.0-BETA1`。**建议仍锁 2.0.4**——2.1.0 还在 BETA |
| 本地 clone 的可用性 | clone 停在 `2.0.3-SNAPSHOT`，**比目标版本旧**。调研出的 API 结论来自 2.0.3 代码，2.0.4 有新增，落地前需以 2.0.4 为准复核 |

> Maven Central 的 `maven-metadata.xml` 对这个项目**返回了陈旧副本**（版本列表停在 `2.0.0-RC4`、`<latest>` 却写 `2.1.0-BETA1`），但目录列表与 jar 都真实存在。**不要用 metadata 判断版本有无，直接探目录。**

### 2.0.4 带来的两个直接收益

1. **并发隔离 —— 解决了调研标记的线程安全疑问。** 本地 2.0.3 代码里，示例 `StreamingWebExample.java` 注释称 `ReActAgent` 非线程安全、建议每请求新建，而 `agent.md` 文档称其 stateless——材料自相矛盾。**2.0.4 的发布说明第一条就是"stronger isolation for concurrent agent runs, tools, and skills"**，并新增 `AgentRun` / `RunControl` 做 per-run 执行控制、请求级 tool 配置与 tool 可见性隔离（#3279、#3283）。结论：**按 2.0.4 用，并发模型比本地旧代码显示的更可靠**，但 P0 spike 仍需实测确认能否复用单例 agent。
2. **`FinalAnswerFilterMiddleware`（2.0.3 引入）—— 对 SSE 直接有用。** 它按 model call 缓冲文本事件，**当产出工具调用时抑制中间推理轮次的文本，只发最终答案**。问数链路里模型会先"思考"再查数再作答，若不抑制，中间推理文本会混进答案流。**建议 P2 直接启用**，与 FR-5.3 的三类事件配合。

### 待验证清单

**P0/P0.5 阶段的三项 —— 已由 spike 验证完毕**（结论见 [spike-p0-results.md](spike-p0-results.md)）：

| 项 | 结论 |
|---|---|
| 单例 vs 每请求新建 agent | **单例安全**。E6 实测 4 并发 0 串台 |
| `nativeStructuredOutput` / `nativeStructuredOutputWithTools` 组合 | **`WithTools` 必须为 false**。E3 实测 2×2 矩阵 |
| `MysqlAgentStateStore` 建表与并发写 | 可用，自动建 `agentscope` 库 + `agentscope_sessions` 表。E7 验证多轮上下文恢复 |

**仍未验证 —— 上线前必须补：**

1. **🔴 三种方言的日期格式串没跑过真库。** §8.1 里 PostgreSQL / Oracle 的
   `to_char(d,'YYYY-MM')` 与 SQL Server 的 `CONVERT(varchar(7), d, 120)` 目前是
   **按文档写的，不是验过的**。MySQL 那条可本地实测（`DATE_FORMAT(d,'%Y-%m')`），
   但 PG / Oracle / SQL Server 本地无实例。
   **风险形态**：格式串写错不会报错，只会得到**错的月份分组**——静默错误，与 §12 里
   「静默语义错误」同类。**必须各起一个实例实测**，不能靠 review 过关。
2. **SQL Server 驱动版本。** 本地 m2 只有 `mssql-jdbc 7.4.1.jre8`（2019 年），
   连新版 SQL Server 是否正常未验证。可能需要联网拉 12.x。
~~3. `FinalAnswerFilterMiddleware` 的实际抑制效果。~~
   → **已实测（2026-10-16），但结论是「本工程当前用不上」。** 见下方 §8.2。
~~4. `MysqlAgentStateStore` 的冲突策略。~~
   → **已确认：2.0.4 的默认策略是 `OVERWRITE`**（`ReActAgent` 构造器：
   `builder.conflictPolicy != null ? … : ConflictPolicy.OVERWRITE`），
   冲突时记一条 warn 日志后**直接覆盖**。多轮追问的产品实现已落地，见 §8.2。

---

### 8.2 多轮会话状态（2026-10-16 落地）

**多轮追问（FR-5.2）此前实际未实现。** agentscope 的 `ReActAgent` **默认不设
`AgentStateStore`**，此时每次调用都是全新会话——第 2 轮问「那华北呢」，
模型只能把它当成缺主语的新问题。E7 验的是 agentscope 的能力，不是产品链路。

已落地的部分：

| 项 | 落点 |
|---|---|
| 状态存储 | `AskSessionStores` + `MysqlAgentStateStore`，**复用业务库**，表名 `ask_agent_session`（`sql/12-ask-agent-session.sql`） |
| 配置 | `chatdata.engine.session.*`，见 `AskSessionProperties` |
| 上下文上限 | `TrimmingAgentStateStore`，默认保留最近 40 条消息，裁剪点收窄到用户消息 |
| 冲突策略 | 显式 `OVERWRITE`（agentscope 的默认值，但显式写出来，理由见 `AskSessionProperties`） |
| 前端会话 id | `AskView` 每次页面加载生成一个 id、「新会话」按钮换新、**换助理也换新** |

**不另建库是刻意的**：agentscope 默认要一个独立的 `agentscope` 库（E7 实测），
而生产账号通常只有单库权限，那样会直接起不来。库名留空时从 `DataSource` 的
`getCatalog()` 读，不在配置里重复写一遍库名。

**验证方式**：`DslGeneratorSessionRealDbTest`（真库 + 假模型，5 例）。用假模型是因为
要验的是「第 2 轮模型**看得见**第 1 轮」，而非「模型答得对不对」——后者要联网、
要花钱、还不稳定，且答错了也说明不了上下文有没有传过去。

### 8.3 `FinalAnswerFilterMiddleware`：实测有效，但对当前架构不适用

**实测结论（`FinalAnswerFilterBehaviourTest`，对照实验）**：它确实按文档工作。

| | 透出的文本流 |
|---|---|
| 不挂中间件 | `我先查一下数据` + `最终答案是 42` ← 中间推理混进答案 |
| 挂上中间件 | `最终答案是 42` ← 中间整轮被抑制 |

**但本工程的问数链路接不上它**，因此**决定不接**（不写空转代码）：

- 它的挂载点是 `ReActAgent.onReasoning` 返回的**事件流**，只有走 `streamEvents(...)`
  才有东西可拦。本工程走的是 `call(msgs, 手写schema, ctx)`——一次性拿结构化 DSL，
  从不消费 agent 事件。
- 链路上**没有注册任何工具**（`SqlTools` 目前只被它自己的测试引用，全仓无
  `Toolkit` 实例化），单次模型调用，**根本不存在"中间轮次"**，也就没有可泄漏的中间文本。
- 模型在本链路的输出是 **DSL JSON 而非散文**，把它流给用户是错的。

> **这条与 §8 的链路草案不一致，值得记下来。** 草案写的是「模型经工具自己写 SQL
> → 有中间轮次 → 需要抑制中间文本」，而实现走的是「结构化 DSL → 我们渲染 SQL」，
> 因此**从设计上就绕开了这个问题**——顺带也让「DSL 是检查点」真正成立。
> 待验证项 3 是按草案的前提写的，实现改道后它就不再适用。

**何时才需要它**：按 FR-5.3 把 agent 的思考文本也流给前端时。届时本文件的
对照组就是它的验收依据。

## 9. 分阶段计划

按 ADR-0001 的警告，**不排"一次做完"**。每阶段结束都有可验证产出。

| 阶段 | 内容 | 产出 | 估时 |
|---|---|---|---|
| **P0 技术验证** | ~~agentscope 2.0.4 跑通：定义 agent、注册 SQL 工具、结构化 DSL、SSE 流式~~ | ~~一个可运行的 spike~~ | **已完成 2026-10-15** |
| **P0.5 补验证** | ~~流式+结构化、单例并发、MysqlAgentStateStore、真实 SQL 工具~~ | ~~补齐后再进 P1~~ | **已完成 2026-10-15，另挖出 1 个实现陷阱** |
| **P1 管理面** | FR-1 / FR-2 / FR-3 / FR-4 全部 CRUD + 权限配置 + 启动校验 | 管理面可用（含前端） | 4–6 周 |
| **P2 问数最小版** | FR-5.1/5.3/5.5/5.7：单域、单表、聚合类问题 | 能回答"上个月华东的保费是多少" | 3–4 周 |
| **P3 语义增强** | FR-3.4/3.5/3.6 参与 prompt；多域路由；澄清追问 | 复杂问题可用 | 4–6 周 |
| **P4 准确率迭代** | 评测集 + 回归；指标口径对齐 | 准确率从基线往上推 | **不可排期** |

**P4 是不可排期的**——这是 ADR-0001 说"6–9 个月"的核心，也是本 PRD 与"3 个月铺完管道"的区别所在。

## 10. 风险

| 风险 | 等级 | 应对 |
|---|---|---|
| NL2SQL 准确率不达标 | **高** | **已有实证量级**：通用 LLM 在企业私有数仓（平均 101.5 表 / 869.4 列）上 EX 接近 0，GPT-5.2 agentic 也仅 10.8%（BEAVER 论文）。应对见 §12 评测方案；P2 只承诺聚合类问题 |
| 语义建模工作量大 | **高**（原判"中"，已上调） | 它不只是工程成本，**是准确率的主要杠杆**——语义层把 101 张表的搜索空间压到域内几张表，是通用模型从"接近 0"变得可用的前提。应对：元数据同步 + AI 生成表/列注释 + 术语库沉淀 |
| 评测集标注质量差 | **高** | BIRD Mini-Dev 实测 32% 标注错误（financial 域 49%）。应对：双人独立标注 + 争议仲裁 + gold SQL 审计（§12.6） |
| 方言矩阵维护成本 | 中 | 五种库 × 若干差异点。应对：接口保持薄（只放真正有差异的项）；标识符统一带引号消除大小写折叠；相对时间在 Java 侧解析，不生成方言函数（§8.1）。每种方言都要有测试 |
| Oracle 驱动许可（FUTC） | 低 | 免费使用与再分发，但非开源。若产品对外发布需法务确认——比 ADR-0001 的 AGPL 问题轻得多，但不该默认忽略 |
| SQL Server 驱动版本旧 | 中 | 本地仅 7.4.1.jre8（2019）。连新版 SQL Server 是否正常未验证，P1 需实测 |
| 权限实现出错导致数据泄露 | **高** | 默认拒绝；启动校验；用 FR-5.5 的 recompute 做确定性回归测试 |
| 前端从零写（**已决**：新写 Vue 页面） | 中 | 厂商前端 JS 是明文，可参考交互与信息架构，但不可复制模板；需按 P1 工期单列前端工作量 |
| 与现有 JimuChatBI 并行导致两套语义模型漂移 | 中 | 本期明确不切换；若要切，需先定迁移方案 |

## 11. 未决事项

1. **切换方式**：并行共存 → 渐进切换，还是直接替换？本期挂起，另出 ADR。（ADR-0006 已记录此项挂起）
2. ~~是否记录方向反转~~ → **已决**：`docs/adr/0006-self-build-ask-engine-supersedes-integration.md`（ADR-0001 已标记 superseded，ADR-0002 决定作废、结论保留）
3. ~~权限模型拆表 vs JSON blob~~ → **已决**：**不拆表**，保留 JSON blob；代价与缓解措施见 §6，其中「`perm_json` 必须加 `v` 版本字段」是硬性要求
4. ~~前端策略~~ → **已决**：**新写 Vue 3 页面**，不复用厂商前端，也不走 `static/` 覆盖
5. ~~评测集~~ → **已决**：见 §12 评测方案。**这是 P4 从"不可排期"变为"可测量"的关键**

## 12. 评测方案

### 12.1 为什么验收标准必须自建

**公开语料与企业私有数仓之间存在量级差异**，这不是"难一点"，是"换了一个问题"：

| | 平均表数 | 平均列数 | 单查询 schema token |
|---|---|---|---|
| BIRD（公开最难基准） | 6.8 | 72.5 | 37.3 |
| **企业私有数仓**（BEAVER 论文） | **101.5** | **869.4** | **316.7** |

后果（BEAVER 论文实测）：**通用 LLM 在企业私有数仓上 EX 接近 0**；其最新版论文中 GPT-5.2 走 agentic 路线也只有 **10.8%**。另一份企业 BI 基准 EntSQL（1,066 条中英双语）中 **96% 的样本需要领域知识**，最佳系统仅 **15.9%**。

> **这条数据是对 ADR-0001「准确率不可排期」的实证支持，也是对 §8「语义层」价值的实证支持。**
> 通用模型在裸 schema 上近乎不可用，**语义域 / 注册表 / 术语 / 主题这套收敛机制正是解法**——它把 101 张表的搜索空间压到域内的几张表。
> 反过来说：**语义建模做得不好，问数准确率不会有救**。这把 §10 里"语义建模工作量大"从工程成本问题升级为产品成败问题。

### 12.2 公开语料：只做基线，不做验收

| 用途 | 语料 | 说明 |
|---|---|---|
| 横向比较模型 / prompt 版本 | **BIRD Mini-Dev** | 500 题，**官方提供 MySQL 版**，与我们的技术栈直接匹配，可直接跑 |
| 验证中文链路 | **CSpider** | Spider 的中文版（10,181 问，schema 仍英文），用于暴露分词与跨语言映射问题 |

**明确不用**：Spider 1.0（英文、已饱和，test 榜首 MiniSeek 91.2）、WikiSQL（单表无 join，~93% 已饱和）、Spider 2.0（云数仓方言，与我们 MySQL 场景不匹配）、TableBench（表格 QA 非 SQL）。

**预期**：这两个语料上分数会**虚高**（schema 小且干净）。只用于横向比较，**不得作为上线判据**。

### 12.3 自建评测集（验收标准）

- **规模**：首版 **150–300 条**。（参照：ScienceBenchmark 每域 100 条、BEAVER 209 条起步、EntSQL 1,066 条）
- **分层**：按 SQL 复杂度 + 领域知识需求**双维度**，各档约 1/3
  1. 简单：单表聚合
  2. 中等：多表 join + 时间窗
  3. 困难：需业务口径（如"保费"的定义、"有效门店"的判定）
- **构造路径**：从 BI/取数平台的 SQL 日志挖高频查询 → LLM 反向生成中文问法 → 业务方逐条确认 → 人工写/审 gold SQL
- **中文特有坑必须单列成题**：同义词（"保费/保额"）、简称、时间口径歧义、需澄清的模糊问法

### 12.4 指标

| 指标 | 地位 | 说明 |
|---|---|---|
| **EX（执行结果集比对）** | **主指标** | 我们是 MySQL，天然可执行。预测 SQL 与 gold SQL 结果集相同即算对 |
| **无结果率** | 必测 | 真实场景最大杀手。SBBD 的企业实测中，加两层 RAG 后空结果率从 26.4% 降到 7.5% |
| **静默语义错误率** | **必测（P0 spike 后新增）** | 见下 |
| 分档准确率 | 必测 | 按 §12.3 的三档分别报告，**避免总分掩盖短板** |
| 人工抽检语义正确率 | 辅助 | |

**「静默语义错误」是 P0 spike 暴露的新类别，必须单列。**

定义：**DSL 结构合法、解析成功、无异常，但丢失或篡改了问题中的关键约束。** 实测样例——「上个月**华东**的**有效**保单保费合计」被翻译成不含任何 `filters` 的 DSL，SQL 退化为全量保费，**全程无任何报错**。

它为什么必须单列：

- **EX 会把它记为"错"，但不会告诉你错因**——与"SQL 写错"混在一起，无法定位到 DSL 装配环节
- **现有监控完全看不到**：无异常、无 5xx、审计里 DSL 结构合法
- **它与 ADR-0002 的「等价字段脱敏遗漏」同类**：不报错，只是给出错误结果。这是数据系统最危险的失败形态

**测法**：对每条评测问题，**逐条比对「问题中出现的约束」与「DSL 中体现的约束」**，缺失即计入。这个比对可以半自动做（问题里的实体值 vs DSL 各字段值），不必全靠人工。

**不用 ESM 作主指标**——它漏判等价改写（如列序不同）。ESM 仅在需要与 Spider 论文数字对齐时使用。

### 12.5 LLM-as-judge：引入，但只做辅助

**不可单独依赖，证据充分**：

| 研究 | 结论 |
|---|---|
| PolySQL | 无执行的 LLM judge κ=**0.38**（接近随机）；基于执行的方案 κ=0.72 |
| 生产审计论文 | 线上 gpt-4o-mini judge κ=**0.04**（争议富集集），**77.1% 误判** |
| FLEX (NAACL'25) | 直接让 LLM 判不如 EX；补全上下文 + 双准则后 κ 才到 87.04 |

**用法**：① 对 EX 判"错"的样本做二次判定，**识别 gold SQL 标注问题**；② 无法执行/歧义问题的兜底。
**前置条件**：先花 100–200 条人工金标测 κ，**κ < 0.5 不得用于任何决策**。

### 12.6 最大的坑：标注质量

**BIRD Mini-Dev 有 32% 的样本存在标注错误，financial 域高达 49%**（*Understanding the Effects of Noise in Text-to-SQL*、VLDB *Pervasive Annotation Errors*）。纠正 gold SQL 后，zero-shot 基线反而超过 SOTA prompting。

→ **自建集必须做 gold SQL 审计**，且**双人独立标注 + 争议仲裁**。否则评测集本身会成为噪声源，P4 的迭代会朝着错误方向优化。

### 12.7 工具

- **IBM text2sql-eval-toolkit**（Apache 2.0，推荐）：12+ 指标、多 gold、LLM-as-judge、错误分析、Web dashboard，支持 SQLite/PostgreSQL/MySQL/DB2，内置 BIRD Mini-Dev、Spider、Beaver
- **BIRD evaluator**（EX + VES）、**Spider 官方 evaluator**、**test-suite-sql-eval**
- **BenchPress**：从企业 SQL 日志快速构建评测集——**正好匹配 §12.3 的构造路径**

### 12.8 鲁棒性评测（建议同期做）

参考 **Dr.Spider**（17 种扰动）的思路，对同一批问题做**同义改写、加错别字、schema 改名**，测分数跌幅。**对中文问数尤其重要**——中文的口语化与同义表达远比英文发散。

## 13. 验收标准

**P1（管理面）**
- [ ] 每个管理面 CRUD 可用，且有集成测试
- [ ] 未安装驱动的数据库类型，测试连接返回明确提示（不是裸类名）
- [ ] 连接测试有超时，黑洞地址 10s 内返回
- [ ] 未配置授权的启用域导致启动失败（不是静默放行）

**P2（问数最小版）**
- [ ] 聚合类问题端到端可用，SSE 有"思考/查数/答案"三类事件
- [ ] 同一 DSL 经 recompute 重算，结果一致
- [ ] 无权限用户被拒绝时返回 403 且审计记为 `BLOCKED`
- [ ] 行规则生效：同一问题，`区域经理` 只看到华东
- [ ] 列规则生效：同一问题，`分析师` 看到 `***`

---

## Comments

（待补充）
