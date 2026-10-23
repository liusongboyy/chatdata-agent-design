# P0 spike 结果：agentscope-java 2.0.4 + Spring Boot 4.1 可行性验证

Status: done
Type: research
日期：2026-10-15
工程：`ask-engine-spike`（一次性验证工程，与 `chatbi-spike` 同级的独立工程，**不并入产品**）

---

## 结论速览

| 实验 | 问题 | 结果 |
|---|---|---|
| **E1** | 能否连上火山方舟、跑通最小 agent | **✓ 通过** |
| **E2** | 能否拿到强类型 DSL | **⚠️ 机制通过，语义不合格** |
| **E3** | 两个结构化输出开关的实际行为 | **✓ 结论明确** |
| **E4** | 流式事件能否支撑三态 UI | **✓ 通过** |

**技术栈可行**（E1/E3/E4 全部达成）。**但 E2 暴露了一个比技术栈更麻烦的问题**，见下文。

---

## E1 最小 agent — 通过

- 依赖解析干净：agentscope 三模块均 2.0.4，Spring Framework 7.0.8，reactor 3.8.6，无冲突
- 连火山方舟成功，**耗时 1710ms**
- 产出的 SQL 正确：

```sql
SELECT SUM(premium) AS total_premium FROM ins_policy
WHERE region = '华东' AND status = '有效'
  AND DATE_FORMAT(effective_date, '%Y-%m') = DATE_FORMAT(CURDATE() - INTERVAL 1 MONTH, '%Y-%m');
```

**API 修正**：`OpenAIChatModel` 在 `io.agentscope.extensions.model.openai`，**不是** `io.agentscope.core.model.openai`（前期调研的包路径有误）。

---

## E2 结构化 DSL — ⚠️ 机制通过，语义不合格

### 机制是好的

`agent.call(msgs, AskDsl.class, ctx)` → `msg.hasStructuredData() == true`，`msg.getStructuredData(AskDsl.class)` 返回非 null 的 Java 对象。**结构化输出这条链路成立。**

### 但模型返回的 DSL 是错的

问题：「上个月**华东**的**有效**保单保费合计是多少」

模型实际产出：

```json
{
  "metrics": ["保费"],
  "dimensions": null,
  "timeRange": {
    "kind": "relative",
    "start": "上个月第一天（相对当前自然月的上个月）锁定文本：上个月",
    "end": null
  },
  "filters": null
}
```

四个问题，按严重程度排：

1. **🔴 `filters` 完全丢失。** 问题里的「华东」和「有效」是**核心约束**，一个都没进 DSL。
   实际后果：SQL 会退化成「全量保费合计」，**答案错误，且不报错**。
2. **🟠 `dimensions` / `filters` 返回 `null` 而非空数组。** prompt 里明确要求「没有则返回空数组」，模型没遵守。
   后果：下游代码每个字段都要判空，且判空分支本身就是 bug 温床。
3. **🟠 `timeRange.kind` 编造了 `"relative"`。** 要求是中文口径名（如「上个月」）。
4. **🟠 `timeRange.start` 塞进了整句话** `"上个月第一天（相对当前自然月的上个月）锁定文本：上个月"`，而该字段约定是 `yyyy-MM-dd`。

### 为什么这条最重要

**失败是静默的。** 对象解析成功、字段类型合法、没有任何异常——但语义是错的。

这与 `docs/adr/0002` 记录的「等价字段脱敏遗漏」是**同一类失败**：不报错，只是给出错误结果。对数据系统而言这是最危险的形态，因为：
- 监控看不到（无异常、无 5xx）
- 审计看不到（DSL 结构合法）
- 只有对着业务数字才能发现

**结论：「DSL 是检查点」这个设计成立，但检查点必须校验语义完整性，不能只校验可解析性。** 具体措施见 PRD 修订。

---

## E3 结构化输出开关 — 结论明确

2×2 矩阵，每格都注册了工具：

| `nativeStructuredOutput` | `nativeStructuredOutputWithTools` | 结果 | 耗时 |
|---|---|---|---|
| true | **true** | ✗ 异常 | 5742ms |
| true | **false** | **✓ 成功** | 3520ms |
| false | **true** | ✗ 异常 | 5056ms |
| false | **false** | **✓ 成功** | 3575ms |

**决定性开关是 `nativeStructuredOutputWithTools`，必须为 `false`。** `nativeStructuredOutput` 本身对结果无影响。

失败时的日志：
```
WARN io.agentscope.core.ReActAgent : Failed to parse native structured output as JSON:
     Failed to deserialize JSON to java.lang.Object
Caused: No structured output in message metadata. Key '_structured_output' not found.
```

**这实证确认了官方对 DeepSeek 系的建议**（其 DeepSeek provider 实现即设 `nativeStructuredOutput(false)`），且证明该建议**同样适用于「OpenAI 兼容栈 + 火山方舟网关」这条路径**——而这条路径的默认值是否安全，此前无从推断。

**我们当前 `SpikeModels.defaultModel()` 用的正是 `(true, false)`，配置正确。**

> ⚠️ 需要注意：`nativeStructuredOutputWithTools=true` 时，网关**确实接受**了请求（否则会是 400），但返回内容无法解析。**这是一个静默的兼容性陷阱**——如果将来有人为了"更好的结构化输出"把它打开，问数链路会直接崩，且错误信息指向 JSON 解析，与真实原因（网关不支持）相距甚远。

---

## E4 流式事件 — 通过

152 个事件，总耗时 4236ms。事件类型分布：

| 类型 | 次数 | UI 状态 |
|---|---|---|
| `AGENT_START` / `AGENT_END` / `AGENT_RESULT` | 1 / 1 / 1 | 会话生命周期 |
| `MODEL_CALL_START` / `MODEL_CALL_END` | 3 / 3 | 模型调用边界 |
| `THINKING_BLOCK_START/DELTA/END` | 3 / 44 / 3 | **思考中** |
| `TOOL_CALL_START/DELTA/END` | 2 / 22 / 2 | **正在查数** |
| `TOOL_RESULT_*` | 2 / 2 / 2 | 查数返回 |
| `TEXT_BLOCK_START/DELTA/END` | 2 / 57 / 2 | **答案输出** |

**FR-5.3 三态可达性：思考中 ✓ / 正在查数 ✓ / 答案输出 ✓**

注意 `MODEL_CALL_*` 出现 3 次——说明这是**多轮 ReAct 循环**（思考→调工具→再思考→作答），事件序列真实反映推理轮次。前端据此可以显示"第几轮"。

**`AgentEventType` 在 2.0.4 有 31 个值**（调研说 28，2.0.4 有新增）。其中对澄清追问直接有用的是 `REQUIRE_USER_CONFIRM` / `USER_CONFIRM_RESULT`（FR-5.6）。

---

## 未验证项（本轮未做，需后续补）

1. **流式 + 结构化输出能否同时用。** `ReActAgent` 有 `stream(msgs, StreamOptions, Class<?>, ctx)` 重载，但 E4 未验证它。**这决定问数链路能否既流式又拿到 DSL**——如果不能，就要改成「流式拿文本 + 非流式拿 DSL」两段式，或先拿 DSL 再流式渲染。
2. **单例 vs 每请求新建 agent。** 2.0.4 宣称加强了并发隔离，但未实测。
3. **`MysqlAgentStateStore` 的建表方式与并发写表现**（多轮追问要用）。
4. **工具注册在真实链路下的表现**（E3/E4 只注册了玩具工具，没做真实的 SQL 执行工具）。

---

## 对 PRD 的影响

见 `.scratch/self-built-chatbi/spec.md` 的修订记录。要点：

- §8 的 DSL 设计需增加**非空约束**与**语义完整性校验**
- §12 的评测方案需增加一类**「静默语义错误」的专项测试**

---

# P0.5 补验证（2026-10-15）

四项按计划做完，另外**挖出一个必须修的实现陷阱**（E9–E15）。

## E5 流式 + 结构化输出 — 通过，且是架构级好消息

214 个事件流完后，末尾的 `AGENT_RESULT` 事件（`isLast=true`）**携带结构化 DSL**。

→ **架构可用单段**：边流式推进 UI，边在末尾取 DSL 做权限校验与审计。
**不需要**改成「先非流式拿 DSL、再流式渲染答案」的两段式——省掉一次模型调用。

## E6 并发 — 通过，单例安全

| 模式 | 成功 | 串台 |
|---|---|---|
| 单例 agent，4 session 并发 | 4/4 | **0** |
| 每请求新建，4 session 并发 | 4/4 | **0** |

**单例可复用**，2.0.4 的并发隔离改进属实。这推翻了「官方示例注释说非线程安全、建议每请求新建」的保守判断。

（注：本实验用的是**不同** sessionId。同一 session 的并发调用未测，但真实场景下不同用户天然是不同 session。）

## E7 MySQL 状态存储 — 通过，但有一个部署约束

**它默认要一个独立的 `agentscope` 库**，不复用应用库。不显式允许创建会直接抛：

```
IllegalStateException: Database does not exist: agentscope.
Use MysqlAgentStateStore(dataSource, true) to auto-create.
```

用 `new MysqlAgentStateStore(ds, true)` 后自动建出 `agentscope` 库 + `agentscope_sessions` 表。

多轮追问验证通过：第 1 轮「上个月华东的保费合计」，第 2 轮「**那华北呢**」（刻意省略指标名）——
第 2 轮 DSL 仍带上了「保费」，**上下文恢复成立**。

## E8 真实 SQL 工具 — 通过

换成真表 `ins_policy`（300 行真实数据），问「华东地区处于生效状态的保单，保费合计是多少」。

- 工具被调用 ✓
- 取到真实数据（18 行分组结果）✓
- **答案 369,140.20 与金标准一致** ✓（还正确报了 43 张保单）

**注意**：E8 里模型**直接写 SQL**，过滤条件（`region='华东' AND status='生效'`）写得完全正确。
这个对照后来成了定位 DSL 问题的关键线索。

---

# 挖出的实现陷阱：`call(msgs, Class<?>, ctx)` 不可用

## 现象

E8 直接写 SQL 时过滤条件完全正确，但走 DSL 时 **`filters` 字段被系统性丢弃**：

问题「上个月**华东**的**有效**保单保费合计是多少」→ 模型产出：

```json
{ "metrics": ["保费"], "timeRange": { "start": "上个月" } }
```

**「华东」和「有效」两个核心约束凭空消失**，且解析成功、无异常。若直接渲染 SQL，
会退化成「全量保费合计」——**答案错误且全程静默**。

## 排查过程（四个假设，三个被证伪）

| # | 假设 | 实验 | 结论 |
|---|---|---|---|
| 1 | prompt 不够明确，加范例即可 | E9（原样 / 加范例 / 加范例+自查） | **证伪**，三种写法全败 |
| 2 | schema 缺 `required`，模型因此省略字段 | E10、E12、E14 | **证伪**，无 required 的变体也成功 |
| 3 | 嵌套 `timeRange` 干扰了 `filters` | E12（无/嵌套/扁平三种 timeRange） | **证伪**，加回嵌套 timeRange 照样正确 |
| 4 | temperature 抖动 / maxTokens 截断 | E13 前后各跑一次 | **证伪**，temperature=0、maxTokens=16384 后仍失败 |

E13 还一度推翻了我自己的中间结论（我以为根因是「Class 路径 vs JsonNode 路径」），
因为那一轮两条路径同时失败——**说明当时还有变量没受控**。

## 真正的根因

把全部 23 次调用按 **schema 来源**重新分类，分离得干干净净：

| schema 来源 | 实验 | 结果 |
|---|---|---|
| **Java record 生成** | E2、E9×3、E13×6 | **✗ 10/10 全丢 filters** |
| **手写 JSON Schema** | E10×4、E12×3、E14×6 | **✓ 13/13 全对** |

**关键对照**：E13 与 E14 走**同一条代码路径**（`call(msgs, JsonNode, ctx)`）、
同模型、同 temperature=0、同 maxTokens=16384，**唯一差别是 schema 内容**。
E13 喂的是 `JsonSchemaUtils.generateSchemaFromClass(AskDsl.class)` 的结果，E14 喂的是手写的。

## 结论与对策

**不要用 `call(msgs, Class<?>, ctx)`。** 产品实现必须**手写 JSON Schema**，走
`call(msgs, JsonNode, ctx)`。

E11 已确认生成的 schema **结构上是合法的**（`filters` 定义完整、类型正确），
所以问题不在结构，而在**内容的某个特征**上——最可能是：

- 生成器给**每个**字段都加了 `description`，其中包含「不要臆造」「不要默认填」这类**否定式指令**
- 字段顺序被排成字母序（dimensions → filters → metrics → timeRange），
  而 `filters` 恰好排在**第二个**，且失败时的输出形态像是**从某个位置起被截断**

**具体是哪个特征导致，本轮未定位到**，但结论可用：换用手写 schema 即可稳定工作。

## 对 PRD 的硬性要求（新增）

1. **DSL schema 必须手写并纳入版本管理**，不得由 record 反射生成。
   代价是 schema 与 Java 类型可能漂移——**需要一条测试断言两者一致**。
2. **DSL 校验 + 重试循环是必需组件，不是可选项。**
   即便换用手写 schema 稳定了，模型输出的完整性也不能假设。
   校验至少要能检出「问题中出现的实体值未出现在 DSL 任何字段里」。
3. **审计需记录重试次数**。重试率高说明 schema 或 prompt 在退化，
   这是 P4 迭代时的重要信号。

