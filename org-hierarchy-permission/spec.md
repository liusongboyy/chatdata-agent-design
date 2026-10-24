# 组织树层级的动态行权限

- Status: Draft
- Date: 2026-10-24
- Related: `docs/adr/0002-data-permission-enforcement-point.md`、`docs/adr/0010-module-owned-data-policies.md`、`.scratch/iam-platform/spec.md` §5.2

## 1. 场景

便利店订单交易数据落到**店铺**级。店铺在组织架构里是叶子节点，往上依次是
组 → 使用部门 → 子公司 → 大区 → 整个便利群。

需求：用户登录后只能看到**自己职权范围内**的订单——

| 用户 | 应看到 |
|---|---|
| 门店店长 | 本店 |
| 组主管 | 本组下所有门店 |
| 部门负责人 | 本部门下所有组与门店 |
| 大区经理 | 本大区下所有子公司、部门、组、门店 |
| 集团总部 | 全部 |

且这套规则**不能写死**：同一张表，不同角色看到不同范围，改配置即可调整。

## 2. 现状与缺口

### 已具备

- **动态行规则**：`rowRules` 支持 `{"dynamic": true, "sysVar": "orgPath"}`，
  `RowRuleCompiler` 从 `PermissionContext.dynamicVars()` 取值并拼进 WHERE。
- **物化路径**：`iam_org.path` 已经是 `/1/10/13/14/` 形态，天然适合层级比较。
- **fail-closed**：取不到动态变量时拒绝整个查询，不跳过规则。

### 三个缺口

**① 动态变量是单值，且只能做精确比较。**
`orgPath = '/1/10/'` 只会匹配到大区这一条，匹配不到它下面的任何门店。
要覆盖子树必须做**前缀匹配**，而现有运算符只有 `=`/`!=`/`>`/`<`/`IN`/`LIKE`，
`LIKE` 的通配符由代码在值两侧加 `%`（包含匹配），不能写成 `前缀%`。

**② 订单表没有组织字段。**
订单只有 `store_id`，组织路径在 `iam_org` 里。要按层级过滤，查询必须能拿到
"这一单属于哪条组织路径"。

**③ 一个用户可能属于多个组织。**
`iam_user_org` 支持多行（兼岗：同时管华东与华南），而令牌里 `orgPath` 是单值，
多归属时只能看到其中一个。

## 3. 决定

### D1 新增前缀匹配运算符 `starts_with`

在 `RowRuleCompiler.OPERATORS` 增加：

| 配置写法 | 生成的 SQL |
|---|---|
| `starts_with` / `前缀` / `starts` | `col LIKE ? ESCAPE '\'`，参数为 `值 + '%'` |

**与现有 `LIKE` 的区别必须保留**：`LIKE`（含"包含"）是值的**两侧**加 `%`，
用于模糊搜索；`starts_with` 只在**右侧**加 `%`，用于层级前缀。

两者不能合并成同一个运算符——"包含"用于搜索文本，"前缀"用于层级，
语义不同，混用会导致权限过滤变成模糊匹配（`LIKE '%/1/10/%'` 会匹配到
`/1/100/...` 这种不在同一子树下的路径，**是越权**）。

**通配符转义**沿用 `escapeLike`：值里的 `%`/`_` 必须转义，否则
组织路径里出现下划线时前缀范围会被放大。

### D2 令牌携带全部组织路径

`/iam/principal` 增加 `orgPaths`（数组），保留 `orgPath`（主组织，向后兼容）。

引擎侧 `PermissionContext.dynamicVars` 支持**多值**：值为 `List<String>` 时，
`starts_with` 编译成 `(col LIKE ? OR col LIKE ? ...)`。

单个动态变量的多个取值之间是 **OR**（并集），与多角色合并的语义一致。

### D3 订单侧通过 SQL 虚拟表拿到组织路径

不动业务表，在语义域里注册一个虚拟表：

```sql
SELECT o.*, org.path AS org_path
FROM convenience_order o
LEFT JOIN org ON org.id = o.store_id
```

理由：店铺表很小，join 代价低；而让 ETL 冗余 `org_path` 意味着店铺换组织时
要刷历史数据，那个一致性很难保证，且**错了就是越权或看不到数据**。

> 注意：`LEFT JOIN` 而不是 `INNER JOIN`。店铺在组织表里找不到时，
> `org_path` 为 NULL，前缀匹配不成立 → 那些单子对所有人不可见（fail-closed）。
> 用 INNER JOIN 会让行数悄悄变少，用户以为"没有数据"而不知道是配置问题。

## 4. 配置示例

```json
{
  "v": 1,
  "grantRoles": ["区域经理", "店长"],
  "rowRules": [
    {
      "id": 1, "role": "区域经理",
      "field": "org_path", "op": "starts_with",
      "value": null, "sysVar": "orgPath", "dynamic": true
    },
    {
      "id": 2, "role": "店长",
      "field": "org_path", "op": "=",
      "value": null, "sysVar": "orgPath", "dynamic": true
    }
  ]
}
```

- 区域经理：`org_path LIKE '/1/10/%'` —— 覆盖该大区下所有层级
- 店长：`org_path = '/1/10/13/14/'` —— 只有本店
- 兼岗用户 `orgPaths = ['/1/10/', '/1/12/']`：
  `(org_path LIKE '/1/10/%' OR org_path LIKE '/1/12/%')`

**为什么店长用 `=` 而不是 `starts_with`**：店长的组织就是门店本身，
用前缀匹配会把路径以本店为前缀的任何节点也算进来（虽然当前不存在子节点，
但组织树是会被改的）。这条差异由角色区分，不需要新机制。

## 5. 实现清单

| 位置 | 改动 |
|---|---|
| `RowRuleCompiler` | 运算符表加 `starts_with`；取值支持多值 → OR 组合 |
| `PermissionContext` | `dynamicVar` 支持 `List` 值 |
| `IamPrincipal` / `IamIntrospector` | 增加 `orgPaths` 字段并透传 |
| `PrincipalController` | 输出 `orgPaths`（查 `iam_user_org` 全部组织） |
| 前端行规则编辑器 | 运算符下拉加"前缀匹配"；动态变量候选加 `orgPaths` |

## 6. 验收

1. **层级正确**：大区经理看到本大区全部单子；组主管只看到本组；
   店长只看到本店。用同一张表、同一问题，只换账号。
2. **越权不成立**：前缀匹配不能跨子树。构造 `/1/10/` 与 `/1/100/` 两个组织，
   验证 `/1/10/%` 不会匹配 `/1/100/...`。
3. **多归属**：兼岗用户看到两个组织的并集。
4. **fail-closed**：组织路径为 NULL 的订单，对所有人不可见；
   令牌缺少 `orgPaths` 时整条查询拒绝，不是放行。
5. **不能写成 `LIKE`**：把运算符误配成"包含"，验证结果集**不同**
   （这条是防止将来有人"简化"成 LIKE）。

## 7. 未决

- **店长的 `=` 用哪个变量**：若用户主组织是大区而店铺在叶子，店长账号需要
  额外持有"我负责的门店"这个映射。当前 `iam_user_org` 直接绑组织节点，
  一个用户绑多个组织节点即可表达"我管这几家店"。是否需要专门的
  "负责门店"关系表，待确认后另议。
- **组织调整后的历史数据**：虚拟表 JOIN 是实时计算的，店铺换组织后历史单子
  会跟着变。若业务要求"按下单时的组织归属统计"，需要 ETL 冗余并带生效时间，
  这是另一个工程。
