# 问数会话历史（Conversation History）

- Status: Draft
- Date: 2026-10-23
- Related: `docs/adr/`, `.scratch/self-built-chatbi/spec.md` (FR-5.x)、`sql/12-ask-agent-session.sql`

## 1. 目标

在问数页面增加「历史会话」列表：

1. 按最后活跃时间倒序展示当前登录用户的会话。
2. 支持分页（`page/size`，默认 20，最大 100）。
3. 支持关键词搜索（会话标题 + 消息正文）。
4. 点击历史会话可回放该会话的全部问答（问题、理解、表格/图表、SQL、失败原因）。
5. 数据严格按登录用户隔离，客户端不得指定归属人。

同时保持 agentscope 的多用户多会话能力：`ask_agent_session` 仍按
`(userId, sessionId)` 存上下文，会话 id 继续复用 `sessionId`。

## 2. 非目标

- 不做跨用户共享、导出、收藏。
- 不做会话重命名（v1 只读标题 = 首问截断）。
- 不把历史消息重新灌回模型上下文（回放只影响界面；继续追问仍以 agentscope
  自己的状态为准）。
- 不为功能上线前的旧会话补历史记录（见 §8）。

## 3. 现状与约束

- 现有 `ask_audit_log`（`sql/04-audit.sql`）与 `AskAuditRecord`：
  **没有 sessionId，也不保存答案**，无法支撑会话列表。
- `ask_agent_session`（`sql/12-ask-agent-session.sql`）是 agentscope 的状态存储契约：
  `session_id/state_key/item_index/state_data/updated_at`，一条会话一行；
  `MysqlAgentStateStore` 的 slotId 为 `userId + ":" + sessionId`，用户隔离在库层成立。
  但 `state_data` 是 agentscope 内部 JSON，不适合做搜索/分页/展示。
- 前端 `AskView.vue` 的 `sessionId` 是页面内随机生成、刻意不持久化；
  「新会话」只是换一个 id。该语义保留：**sessionId 即会话 id**。
- 所有 `/api/ask/**` 已要求登录，身份来自 `CurrentUser.username()`。

## 4. 数据模型

新增 `sql/17-ask-conversation.sql`。

### 4.1 `ask_conversation`

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | VARCHAR(64) PK | 即 `sessionId`，前端 UUID |
| `username` | VARCHAR(64) NOT NULL | 归属人，来自令牌 |
| `assistant_id` | VARCHAR(32) NULL | 会话绑定的助理（首问确定） |
| `title` | VARCHAR(255) NOT NULL | 首问截断（60 字） |
| `preview` | VARCHAR(512) NULL | 最后一条消息摘要 |
| `message_count` | INT NOT NULL DEFAULT 0 | 也是 seq 分配依据 |
| `deleted` | TINYINT NOT NULL DEFAULT 0 | 软删除 |
| `created_at` / `updated_at` | DATETIME NOT NULL | `updated_at` 用于倒排 |

索引：`(username, deleted, updated_at)`、`(username, deleted, created_at)`。

### 4.2 `ask_message`

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | BIGINT AUTO_INCREMENT PK | |
| `conversation_id` | VARCHAR(64) NOT NULL | |
| `seq` | INT NOT NULL | 会话内递增，唯一 |
| `role` | VARCHAR(16) NOT NULL | `USER` / `ASSISTANT` |
| `kind` | VARCHAR(16) NOT NULL | `question` / `recompute` |
| `content` | LONGTEXT NULL | 用户问题 / 助理文本摘要 |
| `result_json` | LONGTEXT NULL | 助理：完整结果事件 JSON，供回放 |
| `status` | VARCHAR(16) NOT NULL | `SUCCESS` / `FAILED` / `STREAMING` |
| `failure_code` | VARCHAR(32) NULL | |
| `failure_message` | VARCHAR(512) NULL | |
| `created_at` | DATETIME NOT NULL | |

唯一键：`(conversation_id, seq)`；索引 `(conversation_id, seq)`。

`result_json` 直接存 `result` 事件 payload（与前端 `AskResultEvent` 同构），
含 rows/columns/chart/dsl/sql/understanding；取数本身已有行数上限，v1 不再额外截断。

## 5. 后端设计

### 5.1 包与接线点

- 新增 `com.liusongboyy.chatdata.ask.conversation`
  - `ConversationController`（HTTP）
  - `ConversationService`（事务内 seq 分配、归属校验、搜索转义）
  - `mapper/ConversationMapper`（`@MapperScan` 增加该子包）
- `AskEngineStreamController` 在 `run()` / `recompute()` 中接线：
  1. 解析 `username`；
  2. `ensureConversation(sessionId, username, assistantId, question)`：
     - 不存在 → 以客户端 `sessionId` 建行（title=首问截断）；
     - 已存在且 `username` 不符 → 403；
     - 已存在且 `assistant_id` 与请求不一致 → 409；
     - `sessionId` 为空 → 服务端生成 UUID 并回传。
  3. 写 `USER` 消息（`kind=question|recompute`）；
  4. 调引擎；
  5. `result` 分支写 `ASSISTANT/SUCCESS/result_json`；
     `error` 分支写 `ASSISTANT/FAILED/failure_*`；客户端中断不影响写入。
- 会话 id 通过 `result` / `error` 事件的 `conversationId` 字段回传，前端据此
  把本地 `sessionId` 落定。`assistants`、`recompute` 请求体保持 `sessionId` 字段。

### 5.2 接口（裸 JSON，沿用 `/api/ask/**` 契约）

- `GET /api/ask/conversations?page=1&size=20&keyword=&assistantId=`
  → `{ items: [...], total, page, size }`
- `GET /api/ask/conversations/{id}` → `{ ...summary, messages: [...] }`，消息按 seq 升序
- `DELETE /api/ask/conversations/{id}` → 204（软删除；同时尽力清 agentscope 状态）

所有查询强制追加 `username = 当前用户`。详情/删除不存在或不属于当前用户一律 404
（不泄露存在性）。`page<1` 归 1，`size` 夹在 1..100；`keyword` 去空白、上限 100 字、
LIKE 中 `%`/`_`/`!` 转义。

### 5.3 一致性

- `appendMessage` 在单个事务内 `SELECT ... FOR UPDATE` 锁会话行，`seq = message_count + 1`，
  插入消息、更新 `message_count/preview/updated_at`。避免并发重算产生重复 seq。
- 写消息失败不应阻断问数主流程：服务层记录日志并吞掉异常（best-effort），
  但会话创建失败（如越权）必须阻断。

## 6. 前端设计

- `src/api/ask.ts` 增加：
  `ConversationSummary/ConversationPage/ConversationMessage/ConversationDetail` 与
  `listConversations/fetchConversation/deleteConversation`。
- `AskView.vue` 布局调整为「历史侧栏 + 主列」：
  - 侧栏：搜索框、倒序会话列表、上一页/下一页、当前会话高亮、删除；
  - 主列：原助理条、消息流、输入区；
  - 移动端侧栏默认收起，由报头「历史」按钮切换。
- 点击会话：`sessionId = detail.id`、`assistantId = detail.assistantId`，
  按消息序列重建 `turns`：
  - `USER` → 新建 turn（`kind`、`question`）；
  - 后续 `ASSISTANT` → 填充 `result`（解析 `result_json`）或 `failure`。
  回放轮次可直接重算/反馈，与实时轮次一致。
- 每次问答结束（result/error）刷新第一页列表，使新会话/排序即时可见。
- 搜索 300ms 防抖；切会话时 abort 正在进行的流。

## 7. 测试计划

- `ConversationServiceTest`：
  - 归属校验（他人会话 404/403）、assistant 不一致 409；
  - 分页夹取、关键词 LIKE 转义；
  - 标题截断、seq 递增、软删除后列表不可见。
- 手工联调：新会话 → 连续追问 → 刷新页面 → 历史列表出现并按时间倒排 →
  搜索命中标题/正文 → 分页 → 回放结果与 SQL → 删除。
- `npm run typecheck` + `npm run build`（chatdata-agent-web）。

## 8. 已知取舍 / 后续

- 上线前已存在的 agentscope 会话没有对应 `ask_conversation` 行，不会出现在列表；
  如果需要，可另做一次性回填（解析 `ask_agent_session.state_data`），不作为 v1 范围。
- 软删除后 agentscope 状态可能残留；v1 删除时 best-effort 清理，后续可加保留任务。
- 标题不做 LLM 摘要；首问较长时只截断。
- 会话继续追问依赖 agentscope 状态；若状态因保留策略被清理，历史仍可回放，
  但模型不再记得更早上下文（界面不额外提示）。
