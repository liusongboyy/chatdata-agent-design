# chat-data-agent 的设计过程

**这不是正式文档，是决策现场。**

对应仓库：[chatdata-agent](https://github.com/liusongboyy/chatdata-agent)

---

多数设计文档只记结论。这里连同**被否决的方案、当时的判断依据、后来发现判断错了的地方**
一起留下了。

## 为什么单独一个仓库

主仓库是给人看"这东西是什么"的——README、截图、快速开始。
而这个目录是给人看"**为什么是这样**"的，两者读者不同、读法也不同。

混在一起的结果是：第一次来的人 clone 下来看到一堆内部 spec，不知道该读哪个；
真想深究的人又得在几百个源文件里翻。

## 读的时候注意

- **`Status: Done` 的不一定都对。** 有几份 spec 写着 Done，但后来在更大规模的域上
  测出了新问题（比如分析编排在 28 表域上暴露的 join 方向问题）。
- **会有相互矛盾的文档。** 比如 `self-built-chatbi` 里论证"要自研"，而更早的 ADR
  论证"应该集成"——两次判断的依据都记着，能看出反转的触发点。
- **不是教程。** 措辞是写给当时的自己看的，会有"实测踩到""这个方案否决了"这类口语化判断。

---

## 各目录

| 目录 | 内容 |
|---|---|
| `self-built-chatbi/` | 自研问数引擎的可行性论证与 P0 spike 结果 |
| `feishu-channel/` | 飞书渠道：webhook、身份绑定、卡片渲染、分页与反馈 |
| `agent-mesh/` | A2A agent：身份代理、Nacos 注册、aistio 接入管理台 |
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
| `health-dashboard/` | 全局健康度：失败原因排行、域成功率、没人用的域 |
| `quality-analysis/` | 问数质量：按问题结构看哪类问题在拖后腿 |
| `failure-diagnosis/` | 失败诊断：不只是「能修」，还要说清「为什么修不了」 |
| `model-cost/` | 模型选择与成本：多模型按域绑定，token 与费用可见 |

## 其中最有价值的几篇

- **`self-built-chatbi/spike-p0-results.md`**：23 次调用的对照实验，
  证明了"用 Java record 反射生成 schema 会让 filters 字段被系统性丢弃"。
  那个结论直接决定了手写 schema 的做法。
- **`analysis-orchestration/spec.md`**：写清了"为什么算和说必须分开"，
  以及自动路由为什么决定不做。
- **`modeling-assistant/spec.md`**：一份**没有实现**的设计。留着是因为里面
  关于"为什么不自动建域"的论证，比实现本身更有参考价值。
