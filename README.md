# 设计过程

这个目录不是"正式文档"，是**每一轮开发的决策现场**。保留它的理由是：
多数设计文档只记结论，而这里连同**被否决的方案、当时的判断依据、后来发现判断错了的地方**
一起留下了。

读的时候注意几件事：

- **`Status: Done` 的不一定都对**。有几份 spec 写着 Done，但后来在更大规模的域上
  测出了新问题（比如分析编排在 28 表域上暴露的 join 方向问题）。
- **会有相互矛盾的文档**。比如 `self-built-chatbi` 里论证"要自研"，而更早的 ADR
  （已删）论证"应该集成"——两次判断的依据都记着，能看出反转的触发点。
- **不是教程**。里面的措辞是写给当时的自己看的，会有"实测踩到""这个方案否决了"这类
  口语化判断。

## 各目录

| 目录 | 内容 |
|---|---|
| `self-built-chatbi/` | 自研问数引擎的可行性论证与 P0 spike 结果 |
| `iam-platform/` | IAM 与功能权限平台的设计 |
| `policy-editor/` | 数据策略编辑器的设计（该路径后来下线） |
| `org-hierarchy-permission/` | 组织层级动态行权限 |
| `derived-metrics/` | 派生指标（比值/差值/时长） |
| `analytics-dsl/` | 排序 / Top-N、占比、同环比 |
| `conditional-metrics/` | 带条件的指标、自定义对比期 |
| `analysis-orchestration/` | 分析编排：拆解 → 执行 → 确定性洞察 → 结论 |
| `conversation-history/` | 会话历史 |
| `modeling-assistant/` | 建模助手（设计完成，未实现） |
| `ui-redesign/` | 三套界面方向的对比稿 |
| `roadmap/` | 本轮规划：为什么先做这三项、判断依据 |
| `modeling-coverage-ui/` | 把建模覆盖度检查显示出来：域详情页 + 健康度徽章 |

## 其中最有价值的几篇

- **`self-built-chatbi/spike-p0-results.md`**：23 次调用的对照实验，
  证明了"用 Java record 反射生成 schema 会让 filters 字段被系统性丢弃"。
  那个结论直接决定了手写 schema 的做法。
- **`analysis-orchestration/spec.md`**：写清了"为什么算和说必须分开"，
  以及自动路由为什么决定不做。
- **`modeling-assistant/spec.md`**：一份**没有实现**的设计。留着是因为里面
  关于"为什么不自动建域"的论证，比实现本身更有参考价值。
