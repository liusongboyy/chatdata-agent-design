# 数据策略编辑体验：模块自治 + 结构化配置

- Status: Done（P0/P1 全部落地；IAM 数据策略路径已于 2026-10-23 下线）
- Date: 2026-10-23
- Related: `docs/adr/0010-module-owned-data-policies.md`、`docs/adr/0002-data-permission-enforcement-point.md`、`docs/adr/0004-role-model.md`、`.scratch/iam-platform/spec.md`

## 1. 方向

数据策略**不再放 IAM**。按 ADR-0010：

- IAM 负责身份、组织、角色、功能权限与模块入口；
- ask 负责数据域/助理的资源授权与行/列策略；
- 引擎继续是执行点，策略缺失或解析失败一律 fail-closed。

本设计要解决的是：在 ask 管理台里，把原本"手写 `perm_json` / 角色与用户靠记忆填写"的体验，
换成**结构化、可选择、可校验**的配置界面；同时清理 IAM 与 ask 两处都能改权限的割裂。

## 2. 问题

### 2.1 两处都能改，只有一处生效

- 当前 `policy.source=iam` 时，引擎读 `iam_policy`，ask 管理台里改 `perm_json` 不生效。
- ask 管理台仍完整渲染权限表单，只在旁边放一句警告；IAM 的「数据策略」页是另一个入口。
- 管理员看到的是"两个地方都能配"，实际只有一处算数。

按 ADR-0010 切回模块自治后，事实来源会回到 `perm_json`；但在这之前必须完成迁移，
否则又会变成"两处都能改"。

### 2.2 手写 JSON 的典型故障

- 角色写错一个字：规则永不匹配。授权失效是"少给权限"；行/列规则失效是
  "限制没生效、看到全量/明文"，后者更危险。
- 用户标识写错：`grantUsers` 存的是 **username**，不是显示名，也不是用户表主键。
- 域和助理的文档结构不同，但 IAM 共用一个空白 JSON 编辑器：
  - 域：`v/isPublic/grantRoles/grantUsers/unrestrictedRoles/rowRules/colRules`
  - 助理：`enabled/grantRoles/grantUsers`
- 字段名、操作符、动态变量全靠记忆；后端保存时只保证"是 JSON 对象"。
- 列表只列出"已有策略"的资源，**没有策略的资源根本看不见**。而 fail-closed 下
  "没策略"= 谁都进不去，恰恰是最需要被发现的配置缺失。

## 3. 目标

1. **策略回到资源所在模块**：ask 管理台是数据域/助理权限的唯一编辑入口。
2. **不写 JSON 也能完成 95% 的配置**：角色、用户、字段、操作符、动作都从选项中选择；
   JSON 只作为高级逃生口。
3. **配置错误在保存前暴露**：不存在的角色、停用角色、不存在的用户、空字段、缺值规则，
   当场标红。
4. **能看见缺口**：ask 管理台能标出"已配置 / 未配置策略"的域与助理。
5. **IAM 保持独立**：不依赖 ask 的数据结构，不提供数据策略编辑能力。

## 4. 非目标

- 不把数据域/助理的资源定义搬进 IAM。
- 不在第一版支持组织级授权（`orgs`）。引擎当前的 `DomainPermission` /
  `AssistantPermission` 没有这个字段，先不造"配了但不生效"的入口。
- 不做跨模块的数据策略总览；若以后需要，做只读聚合，不把写入口搬回 IAM。
- 不做策略审批流、定时生效、历史版本回滚（审计与版本另见 §8，先做最小可用）。
- 不再让 ask 在问数热路径调用 IAM 读取数据策略。

## 5. 关键决策

### D1：数据策略归 ask，IAM 只提供身份目录

- `perm_json` 是 ask 数据策略的事实来源。
- 角色/用户只引用 IAM 的 `role.code` / `username`，不复制身份数据。
- IAM 不解析、不存储、不校验 DOMAIN/ASSISTANT 的策略文档。
- 迁移期用 `policy.source=local|iam` 开关兜底，默认切 `local`；迁移完成后下线 IAM 路径。

### D2：结构化表单 + 原始 JSON 作为高级模式

- 表单是主路径，按资源类型渲染不同字段。
- 「高级 / JSON」折叠区始终可打开；未知字段**原样保留**，避免表单序列化时丢字段。
- 结构化解析失败（历史脏数据）时自动落到 JSON 模式并提示，不能因此让已有策略无法查看/修复。

### D3：选项来源

| 选项 | 来源 | 说明 |
|---|---|---|
| 角色 | IAM 角色目录（ask 现有 `GET /api/engine/admin/roles` 已代理） | 存 `role.code`，显示 `name (code)`；停用角色标黄并提示"规则不会生效" |
| 用户 | IAM 用户目录（ask 后端新增只读查询或复用现有 IAM client） | 存 `username`，显示 `displayName (username)`；支持关键词搜索 |
| 数据域/助理 | ask 自己的表 | 不需要跨服务；直接展示名称、id、状态 |
| 字段 | ask `ask_domain` 下的表与列定义（`columnsJson`） | 只对 `DOMAIN`；读不到时退化为可自由输入的文本框 |
| 动态变量 | 固定候选 `orgCode` / `userCode` / `orgPath` | 与 `PermissionContext.dynamicVars` 对齐；未知值仍可手填 |

### D4：校验分层

- **前端（即时）**：必填、类型、角色/用户存在性、停用角色、规则完整性。
- **ask 后端（保存时，best-effort）**：角色码与用户存在性可由 IAM 目录校验；
  校验不可用时不阻断保存，但记录告警。字段语义不在保存时校验。
- **引擎（读取/启动时）**：字段是否存在、操作符是否支持、动态变量是否可取——
  只有引擎有物理列上下文。启动校验失败应拒绝启动或标记资源不可用，沿用现有行为。

### D5：能看见"未配置策略"

- ask 管理台的域/助理列表直接展示策略状态：`未配置 / vN / 解析失败`。
- "未配置"单独标记，因为 fail-closed 下它是高风险项。
- 资源定义与策略在同一个表单里，天然不会出现 IAM 那种"只列已有策略"的盲区。

### D6：不为了总览牺牲自治

- IAM 不再提供数据策略页；若未来要做治理总览，由独立只读页面调用各模块接口聚合。
- 模块策略的 schema、版本与审计由模块自己维护，IAM 不参与。

## 6. 页面设计

### 6.1 ask 管理台：数据域编辑

在现有 `DomainListView.vue` 的编辑弹窗里，把权限区升级为结构化表单：

```
┌ 编辑数据域 ─────────────────────────────────────────────┐
│ 基本信息：名称 / 数据源 / 状态 / 模式 …                │
│                                                        │
│ ┌ 访问授权 ──────────────────────────────────────────┐ │
│ │ ( ) 公开  (•) 指定角色/用户                        │ │
│ │ 角色  [区域经理 ×] [分析师 ×] +                    │ │
│ │ 用户  [张三 (zhangsan) ×] +                        │ │
│ │ 全量可见角色 [管理员 ×] +                          │ │
│ └────────────────────────────────────────────────────┘ │
│ ┌ 行规则 ────────────────────────────────────────────┐ │
│ │ 角色       字段      条件   值      动态变量       │ │
│ │ [区域经理] [region]  [=]   [华东]   [ ]      [删]  │ │
│ │ [+ 添加行规则]                                     │ │
│ └────────────────────────────────────────────────────┘ │
│ ┌ 列规则 ────────────────────────────────────────────┐ │
│ │ 角色      字段        动作                         │ │
│ │ [分析师] [premium]   [脱敏]                 [删]   │ │
│ │ [+ 添加列规则]                                     │ │
│ └────────────────────────────────────────────────────┘ │
│ ▸ 高级：查看 / 编辑 JSON                               │
│ [取消] [保存]                                          │
└────────────────────────────────────────────────────────┘
```

- 角色/用户选择器以 chip 形式展示，支持搜索、删除。
- 行规则角色、列规则角色同样用角色选择器；字段用 datalist（来自该域的表列定义），
  读不到时退回文本框。
- 动态规则勾选"动态"后隐藏值输入，改为选择 `sysVar`。
- 保存前汇总校验错误，定位到具体规则行。
- 高级 JSON 与表单双向同步：默认以表单编辑，展开 JSON 时以当前表单序列化为准；
  若原文档有未知字段，保留并提示"表单未覆盖的字段将在保存时原样保留"。

### 6.2 ask 管理台：助理编辑

助理权限结构简单，直接内嵌：

```
┌ 访问权限 ──────────────────────────────────────────────┐
│ [x] 启用访问控制                                      │
│ 角色 [区域经理 ×] +                                    │
│ 用户 [张三 (zhangsan) ×] +                            │
│ 提示：助理层只控制可见/可用，不控制行与列。            │
└────────────────────────────────────────────────────────┘
```

### 6.3 ask 管理台：策略状态

- 域/助理列表增加"策略"列：`未配置` / `v1` / `解析失败`。
- 未配置项在列表里高亮，编辑页顶部提示"该资源尚无策略，保存后将立即生效
  （或按启动校验规则拒绝）"。
- 解析失败时提供"修复"入口，直接打开 JSON 模式。

### 6.4 IAM 管理台

- 移除「数据策略」菜单与相关路由。
- 权限目录中删除/停用 `iam.policies`、`iam.policies.write`（迁移完成后）。
- 角色/用户页保留，继续作为 ask 管理台选择器的数据源。

## 7. 策略文档契约

### 7.1 数据域（与 `DomainPermission` 对齐）

```json
{
  "v": 1,
  "isPublic": false,
  "grantRoles": ["role_code"],
  "grantUsers": ["username"],
  "unrestrictedRoles": ["role_code"],
  "rowRules": [
    {"id": 1, "role": "role_code", "field": "region", "op": "=", "value": "华东", "dynamic": false}
  ],
  "colRules": [
    {"id": 1, "role": "role_code", "field": "premium", "action": "脱敏"}
  ]
}
```

动态规则用 `dynamic: true` + `sysVar: "orgCode"`，忽略 `value`。

### 7.2 助理（与 `AssistantPermission` 对齐）

```json
{
  "enabled": true,
  "grantRoles": ["role_code"],
  "grantUsers": ["username"]
}
```

### 7.3 与旧文档的差异

`.scratch/iam-platform/spec.md` 的示例使用嵌套 `grants`；**当前引擎实现是扁平字段**。
本设计以引擎实际解析的 `DomainPermission` / `AssistantPermission` 为准。
建议在 ask 内用一份 JSON Schema 固定契约，避免文档与实现继续漂移。

## 8. 后端改动

### P0（体验与方向落地）

1. ask 前端新增：
   - `PolicyEditor.vue`、`RoleMultiSelect.vue`、`UserMultiSelect.vue`
   - `RowRuleTable.vue`、`ColRuleTable.vue`
   - 纯函数 `policyCodec.ts`：解析/序列化/保留未知字段/结构校验。
2. `DomainListView.vue`、`AssistantListView.vue` 接入结构化编辑器，写入 `permJson`。
3. ask `api/admin.ts` 增加用户搜索调用；角色继续用现有 `listRoles()`。
4. ask 后端新增只读用户查询（代理 IAM 用户目录或复用 `IamRoleDirectoryClient` 的模式），
   仅返回 `username/displayName/status`，供选择器使用。
5. IAM 前端移除数据策略菜单/路由（可先隐藏，迁移完成后再删）。

### P1（迁移与校验）

1. 一次性迁移 `iam_policy` 的 DOMAIN/ASSISTANT 文档到 `perm_json`，做逐资源比对。
2. 默认 `policy.source=local`；保留 `iam` 一个发布周期用于回滚。
3. ask 保存时做角色/用户存在性校验（best-effort）；失败给出明确提示。
4. 策略变更审计：至少记录操作人、时间、资源、版本/摘要；可复用现有审计表或新增
   `ask_policy_audit`。
5. 下线 ask 的 IAM 策略客户端与 IAM 策略接口。

**P1 落地情况（2026-10-23）**

- 回迁脚本：`sql/iam/10-migrate-policies-back-to-ask.sql`（备份 → 覆盖 `perm_json` →
  比对差异 / 残留旧角色 / 资源数量）。默认 `policy.source=local` 已生效。
- 保存校验：`PolicyReferenceValidator` 接入域/助理四个写接口。规则：
  不存在的角色/用户 → 拒绝并列出全部缺失项；停用 → WARN 放行；IAM 不可用 → WARN 放行。
- 审计：`ask_policy_audit`（`sql/18-ask-policy-audit.sql`）记录
  `CREATE/UPDATE`、操作人、资源、时间与去空白 JSON 的 SHA-256；写失败只记 ERROR，不回滚保存。
  选型见 §12-3：先做摘要，不做前后快照。
- 批量用户校验走 `GET /iam/policy/users?usernames=a,b`，只返回真实存在的那些，
  不发模糊匹配。
- ~~剩余：删除 IAM 数据策略写接口/旧页面，以及 ask 的 IAM `PolicyProvider` 客户端；
  删除前需确认无模块引用。~~ → **已完成（2026-10-23）**：删除 IAM 的
  `AdminPolicyController`/`AdminPolicyService`/`PolicyMapper`/`EffectivePolicyController`/
  `PolicyVersionController` 与旧策略页；ask 侧删除 `IamPolicyClient`、`PolicyProvider`、
  `PolicyVersionRefresher`、`PolicyWriteGuard` 与 `policycheck` 双读工具；
  `chatdata.engine.policy.source` 开关一并移除。收尾脚本见
  `sql/iam/11-retire-iam-policy.sql`（改名保留数据，不 DROP）。

### P2（增强）

1. 字段下拉正式接入域表元数据；字段不存在时警告但不阻断保存（启动校验兜底）。
2. 策略摘要/预览：按选定角色模拟"可见行/脱敏列"。
3. 从任一资源复制策略；批量给多个资源追加同一角色。
4. 策略版本历史与回滚。

## 9. 权限与安全

- ask 数据域/助理策略编辑沿用现有管理面权限：
  `ask.admin.domains.write` / `ask.admin.assistants.write`。
- 角色/用户目录由 IAM 提供，ask 只读；不复制角色与用户数据。
- IAM 不再有 `iam.policies` / `iam.policies.write` 的数据策略含义；
  若保留权限码，只能用于"只读治理总览"。
- 前端不把角色/用户列表写入策略缓存；所有校验以保存时后端与启动校验为准。
- 删除资源时，其 `perm_json` 随资源一并删除，不再产生 IAM 孤儿策略。

## 10. 迁移与兼容

- 已有 `iam_policy.document` 原样迁移到 `perm_json`；结构化编辑器解析成功即转表单，
  失败则停留在 JSON 模式。
- 未知字段（例如未来新增、当前前端未识别的键）在保存时原样带回。
- `policy.source=local` 环境完全不受影响；`iam` 环境在迁移窗口内继续可读旧策略。
- 迁移完成后删除 IAM 策略表/接口前，先确认没有模块仍在使用。

## 11. 验收

1. 不打开 JSON 编辑器，能完成：数据域策略（公开/角色/用户 + 行规则 + 列规则）、
   助理策略（开关 + 角色/用户）。
2. 输入不存在的角色或用户，保存前有明确提示；后端 best-effort 校验也给出提示。
3. ask 管理台能看出哪些域/助理没有策略或策略解析失败。
4. IAM 不再能编辑数据域/助理策略；角色/用户仍可从 IAM 维护。
5. 未知字段在"表单 → 保存 → 再打开"后仍然存在。
6. 现有 ask/IAM 单测、两个前端 `typecheck/build` 通过。
7. 手工验证：迁移前后同一批资源的行/列结果逐条一致；策略缺失时仍 fail-closed。

## 12. 待确认

1. P0 是否包含"字段下拉"？它依赖 ask 元数据加载，但不再跨服务，成本较低。
2. 组织级授权（`orgs`）是否排在下一期？需要先扩展引擎的 `DomainPermission` 与动态变量。
3. ~~策略审计的最小粒度~~ → 已定：先记"谁在何时改了哪个资源 + 摘要"，
   不存前后全文；需要还原时对照备份/迁移脚本。
4. IAM 的 `iam_policy` 表与 `iam.policies*` 权限码是立即删除，还是保留一个发布周期？
