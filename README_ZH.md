# Awesome Jev ⚡

[![Awesome](https://awesome.re/badge.svg)](https://awesome.re)
[![License: CC0-1.0](https://img.shields.io/badge/License-CC0%201.0-lightgrey.svg)](http://creativecommons.org/publicdomain/zero/1.0/)

> 一份关于 **Jev** 的精选资源清单——[TypeSafe AI](https://www.typesafe.ai) 于 **2026 年 9 月 15 日** 发布的首个 **"System One" 决策模型**。
>
> *"Decisions, not strings."* —— Jev 不生成文本。输入一个 `state` 和一组预定义的 `questions`，它返回**类型化、经过校准的概率化决策**：选择、评分与是非判断。

**目录**

- [Jev 是什么？](#jev-是什么)
- [工作原理](#工作原理)
- [规格与定价](#规格与定价)
- [官方资源](#官方资源)
- [SDK 与集成](#sdk-与集成)
- [学术论文](#学术论文)
- [基准测试与独立评测](#基准测试与独立评测)
- [典型用例](#典型用例)
- [开源复现与本地替代方案](#开源复现与本地替代方案)
- [社区资源](#社区资源)
- [媒体报道](#媒体报道)
- [注意事项与运维风险](#注意事项与运维风险)
- [贡献指南](#贡献指南)

---

## Jev 是什么？

Jev 是 TypeSafe AI 称之为 **System One 模型**（借自卡尼曼《思考，快与慢》）的全新模型类别：一个快速、非自回归、并行化的**决策引擎**，专为 Agent 循环内部高频、低价值的判断（路由、分诊、评分、验证）而设计——在这些场景调用完整的生成式 LLM 又慢又贵。

| Jev **是** | Jev **不是** |
|---|---|
| "系统一"模型（快速直觉） | 大语言模型（LLM） |
| Agent 循环中的分类器 / 路由器 | 文本生成器 |
| 输出概率的类型化决策引擎 | 通用推理模型 |
| 快速、并行、单次前向 | 对话式模型 |

- **公司**：TypeSafe AI（旧金山，2024 年成立）——由 **Diogo Almeida**（前 OpenAI 核心研究科学家，RLHF / InstructGPT / ChatGPT 的共同发明人）、Erik Gafni 和 Sasha Sheng 创立。经过约两年隐身开发，携 **DCVC 领投的 4000 万美元种子轮** 浮出水面（Forbes 报道估值约 2 亿美元）。
- **发布**：2026 年 9 月 15 日——发布帖登顶 Hacker News；上线 Vercel AI Gateway 24 小时内，近 13% 的付费团队已使用过它，创下该平台史上最快的新模型采用纪录。
- **命名**：致敬经济学家 **威廉·斯坦利·杰文斯（William Stanley Jevons）** 与**杰文斯悖论**——把决策做得足够便宜，决策需求反而会爆炸式增长。
- **定位**：一次"前沿智能函数调用"——*非结构化 state 进，类型化概率决策出*。

> 📌 **名词辨析**：Jev 模型与**日本脑炎病毒（JEV）** 无关——同缩写，不同领域。另外提醒：尽管本仓库名含 "for-sequence-modeling"，Jev 是**决策模型**而非序列生成架构——它的输出单位是"选项 + 置信度"，不是 token 序列。

## 工作原理

已公开的技术栈由三部分组成：

1. **全新的 Transformer 架构** —— 非自回归；尚未发布架构论文，参数量未公开。
2. **并行采样器（parallel sampler）** —— 一次请求中的所有问题在**单次并行前向**中全部回答（总计 70–500 毫秒，常低于 100 毫秒），无需像自回归模型那样逐 token 解码。这是速度的来源。
3. **RLCD（Reinforcement Learning for Calibrated Decisions，校准决策强化学习）** —— 训练方法：RLHF 奖励"人类更喜欢的答案"，RLVR 奖励"通过可验证检查的答案"，而 RLCD 优化的是**置信度追踪真实准确率**（0.9 的置信度应当约九成正确）。训练**仅使用合成数据**。

### 三种问题原语

整个 API 只有三种问题类型——这是设计理念，而非功能限制：

| 原语 | 返回内容 | 备注 |
|---|---|---|
| **Choice（选择）** | 所选选项 + 完整概率分布 + 置信度 | 最多 **255 个选项**；可显式添加 `other` 选项让模型表达"以上都不是" |
| **Score（评分）** | 在你用文字描述的 2–10 级量表上的连续分数 | 可落在级别之间（如 `1.4`） |
| **Noul（是否）** | 对一个是/否命题返回单个 0–1 概率 | 名字来自伯努利（Bernoulli）分布；在 Vercel AI SDK 中拼写为 `boolean` |

请求只有三个参数：`state`、`model`、`questions`。**没有** temperature、max_tokens、思考预算——官方文档也没有流式（SSE）记载。

### "零幻觉"的准确含义

Schema 匹配由构造保证：Jev 在物理上无法输出你声明的选项集之外的任何内容，类型错误不可能发生。但它**仍然可能自信地选错**——这个保证覆盖的是输出空间，而不是选择本身的正确性。设计阈值时请务必考虑这一点。

## 规格与定价

| 规格 | 数值 |
|---|---|
| 延迟 | 70–500 毫秒（常低于 100 毫秒）；官方称比可比 LLM 快 40–200 倍 |
| 输入价格 | **$0.042 / 百万 token**（$42 / 十亿）——比 GPT-5 Nano（$0.05/M）还便宜 |
| 输出价格 | **$0**（输出免费） |
| 成本对比 | 官方称在特定工作流上比 LLM 便宜 40–400 倍 |
| 上下文 | 原生 64k（约 32k state + 最长问题）；网关通道为 32k |
| 限流 | 250,000 token/s、1,200 请求/分钟（会动态调整，不另行通知） |
| 模态 | 仅文本 + 结构化数据——**不支持图像/音频/视频输入** |
| 当前版本 | `jev-1.13.0`（别名 `jev-latest`） |
| 免费层 | 无 |

**成本实测感**：一次典型决策约 400 token ≈ **$0.000017**——1 美元约可做 6 万次决策。

## 官方资源

- [TypeSafe AI 官网](https://www.typesafe.ai)
- [官方文档](https://docs.typesafe.ai) —— models、state、原语（Choice / Score / Noul）、confidence、API 参考
- [控制台](https://console.typesafe.ai) —— API 密钥管理
- [发布博客](https://www.typesafe.ai/blog) —— "Decisions, not strings"（2026 年 9 月）

## SDK 与集成

- [`@typesafe-ai/sdk`](https://www.npmjs.com/package/@typesafe-ai/sdk) —— 官方 Node/TypeScript SDK
- [`typesafe-sdk`](https://pypi.org/project/typesafe-sdk/) —— 官方 Python SDK
- **Vercel AI Gateway 与 AI SDK** —— 通过 `experimental_evaluate` 提供原生支持；发布当日即接入。注意网关的特殊行为，见[注意事项](#注意事项与运维风险)。
- **OpenRouter** —— 统一路由接入 `typesafe-ai/jev`
- **LangChain** —— [`langchain-typesafe`](https://python.langchain.com) 包：模型路由中间件 + `AutoModeMiddleware`（用 Jev 标记高风险工具调用）
- **Langfuse** —— 通过 OpenInference instrumentor 做追踪

### 最小示例（Python）

```python
from typesafe_sdk import Jev

jev = Jev(api_key="...")
result = jev.evaluate(
    model="jev-1.13.0",
    state='客户留言："我的退款三周前就承诺了，现在还没到账。"',
    questions=[
        {"id": "department", "type": "choice",
         "options": ["billing", "technical_support", "sales", "other"]},
        {"id": "severity", "type": "score"},
        {"id": "refund_due", "type": "noul"},
    ],
)
```

### 最小示例（Vercel AI SDK / TypeScript）

```typescript
import { experimental_evaluate as evaluate } from 'ai';

const result = await evaluate({
  model: 'typesafe-ai/jev',
  state, // string | JSON object | string array
  questions: [
    { id: 'department', type: 'choice', options: ['billing', 'technical_support', 'sales', 'other'] },
    { id: 'severity', type: 'score' },
    { id: 'refund_due', type: 'boolean' }, // Noul 在此拼写为 "boolean"
  ],
  providerOptions: { gateway: { zeroDataRetention: true } },
});
```

## 学术论文

2026 年 9 月出现的基于 Jev 的早期学术工作：

- **REFLEX: Jev for Efficient Selective Control in LLM Agents**（arXiv:2609.26532）—— 将 Jev 作为 Agent 的决策层：高置信度步骤直接执行，低置信度步骤升级给强 LLM。在 100 任务冻结基准上取得 95% 成功率，强模型调用减少 72.7%；在 Qwen3.8-Max、Kimi K3、DeepSeek-V4-Pro 三种回退模型上均保持 66%–72% 的调用节省。
- **Jev-Mem: System-One-Controlled Agentic Memory for Efficient AI Agents**（arXiv:2609.23986）—— 由 System-One 控制器管理记忆构建与自适应检索（route → retrieve → assess → expand → reassess），LLM 仅负责答案合成。
- **Fast Intent-Driven Service Orchestration with Jev for 6G Edge Networks**（arXiv:2609.23136）—— 在 6G 边缘网络编排中，Jev 的低延迟带来更高的按时完成率（97.0%，对比 DeepSeek 93.5% / Gemini 88.7%）。

## 基准测试与独立评测

官方宣称（特定工作流）：比前沿大模型**最快提速 193.6 倍**、**成本最低降至约 1/444.6**。独立 / 社区数字（均为作者自报，谨慎对待）：

- **国际象棋谜题**（@krishras23）：100 道评分谜题，Jev 对 gpt-6 astra，4 并发——Jev 解出 25 道，用时 8 秒，花费 $0.03；astra 解出 68 道，用时 33 分钟，花费 $11。11 道一步杀全对；需要 3 步的 30 道中 Jev 仅 3/30，且在 astra 解出而 Jev 未解的 44 道里**有 40 道第一步就走错**。结论：Jev 适合做快速首过，不适合深推理。
- **俄罗斯方块**（@ashutoshftw）：200 块、仅合法动作——Jev 9200 分 / 75 行 / 每步 300ms；得分与 Claude Haiku 4.5、Gemini 3.5 Flash-Lite 相当，单步速度快约 5 倍。
- **语义断路器**（@ashutoshftw）：20 个"故意卡死"的 Agent 场景——模型调用 115→80、token 151.7K→104.9K，且 20 个健康运行场景 0 误判 HALT。
- **客服判断实测**（虎嗅）：50 道客服分诊题准确率约 64%——快且便宜，但判断质量未超同级模型。
- **置信度闸门实验**（llt22/jev-lab）：唯一稳定失败的模式是**规则互相冲突**（3 轮全选错规则），而这些失败样本的 `confidence` 极低（0.04–0.16）——置信度作为升级闸门的机制有初步证据支持。
- **1.13 "jaggedness" 文档**（官方）：目前不擅长数字、日期和对抗性内容；数值类判断建议拆成多个窄 Noul + 代码取 argmax。

## 典型用例

共同模式：系统需要在循环内做出**快速、廉价的判断**，而不是生成文本。

- **分诊与路由** —— 工单路由、紧急程度判断、部门分类
- **模型路由** —— 简单请求走便宜模型，难请求走前沿模型
- **护栏** —— 越狱检测、输入门禁、工具调用风险标记（LangChain `AutoModeMiddleware`）
- **验证** —— 在用户看到之前检查 LLM 输出
- **欺诈与内容审核** —— 对用户内容的标记 / 评分决策
- **实时控制** —— 旗舰演示：Doom 每秒 10 次决策；另有 Minecraft 机器人、驾驶模拟、无人机避障
- **记忆管理** —— Jev-Mem：Agent 记忆的存储 / 检索 / 路由决策
- **咨询型否决** —— 如 Orus Agent 把 Jev 放在下单执行*之前*（批准 / 拒绝 / 再观察），执行权仍在外部系统

## 开源复现与本地替代方案

- **OpenJev**（APUS 麒麟合盛 AI 实验室，2026 年 9 月 19 日）—— 针对 Jev 的独立开源跨平台复现；MIT 协议；支持 macOS / Linux / Windows，GPU 服务器或无 GPU 的 Mac/PC 均可运行；封装为开箱即用的 Agent Skill（`fast-browser-use`），支持纯本地离线执行。
- **jeff** / **poorjev** —— 社区本地（非 API）方案，适合数据不能出机的场景。
- [cobanov/awesome-jev](https://github.com/cobanov/awesome-jev) —— 社区维护的 awesome 列表（star 最高的索引；注意生态索引间的 star 数可能不一致，请直接核对仓库）。

## 社区资源

- [Raunaksplanet/jev-research-sept-2026](https://github.com/Raunaksplanet/jev-research-sept-2026) —— 全面的调研报告：规格、所有接入路线（原生 / OpenRouter / Vercel）、集成、注意事项、安全分析
- [llt22/jev-lab](https://github.com/llt22/jev-lab) —— 可复跑的实验，探测 Jev "置信度作闸门"的行为，原始数据入库
- [Simon Willison: "Jev introduces a new shape of LLM — System One, aka Decision Models"](https://simonwillison.net/2026/Sep/21/jev/) —— 有影响力的独立评论；认为 "decision model" 是更好的名字
- [AI Profit Boardroom: Jev Architecture](https://aiprofitboardroom.com/blog/jev-architecture/) —— 架构蓝图（语音浏览器、试衣镜），"智能活在英文描述里"的范式
- [Eigent AI：什么是 Jev](https://www.eigent.ai/zh-CN/blog/typesafe-ai-jev-system-one-models) —— 详细的中文解读
- [DEV Community 指南](https://dev.to) —— 发布周以来流传的入门指南

## 媒体报道

- [TechCrunch](https://techcrunch.com)（2026-09-18）—— 发布报道
- [BusinessWire](https://www.businesswire.com)（2026-09-16）—— 4000 万美元 DCVC 融资
- [Forbes](https://www.forbes.com) —— 创始人专访与估值
- [36氪](https://m.36kr.com/p/3988164509711361) ——《一个"不说话"的 AI 刷屏，Jev 真是新范式吗？》
- [虎嗅](https://www.huxiu.com/article/4892583.html) —— 上手实测
- [新华网](http://www.news.cn/tech/20260921/8f1c9bd6a9254e629383f1ac51e0d27d/c.html) —— APUS 开源复现 OpenJev
- [开源中国](https://www.oschina.net/news/502609/typesafe-ai-system-one-models-and-jev)
- [TechSpot](https://www.techspot.com)（2026-09-20）

## 注意事项与运维风险

1. **锁定版本** —— `jev-latest` 会随新发布漂移；调阈值时请固定 `jev-1.13.0`，并记录每次响应的 `model` 字段以便归因。
2. **Vercel 网关不支持版本锁定** —— 带版本号的模型 ID（如 `typesafe-ai/jev-1.13.0`）在网关上返回 404，响应只回显别名。
3. **confidence 字段位置因通道而异** —— 在 Vercel 上位于 `providerMetadata.typesafe.confidence`，按问题 ID 键控；从原生 API 复制的代码会静默丢失置信度。
4. **Noul 的拼写** —— 在 Vercel AI SDK 中拼写为 `boolean`。
5. **上下文限制** —— 原生 64k，但网关通道约 32k state；官方文档内部存在 32k 与 64k 的矛盾表述，容量规划取保守值。
6. **Choice 上限 255** —— 更大的选项集需要两段式设计（先评分缩小候选，再做最终选择）。
7. **仅结构化输入** —— 不支持图像输入；代码必须先把世界翻译成文本 / 结构化数据。
8. **能力弱点** —— 数字、日期、对抗性内容，以及**规则互相冲突**（目前最稳定的失效模式）。
9. **安全** —— 作为闸门使用时，攻击者控制的内容会流入 Jev：需测试 **state 投毒**（类提示注入攻击）与**置信度阈值绕过**（精心构造落在决策边界错误一侧的输入）。把 Jev 的判断当作闸门的输入之一，而非闸门本身；敏感数据使用 ZDR。
10. **"零幻觉" ≠ 永远正确** —— 它表示输出永远是合法枚举值。高置信度的错误选择仍有可能发生。
11. **生态索引漂移** —— 第三方目录把 Jev 错标为 "Text Generation"；npm 存在同名仿冒包；`npx skills` 是第三方 CLI。一切以官方仓库为准。

## 贡献指南

欢迎贡献！请遵循 awesome-list 惯例：

1. 提交前请确认资源与 **Jev / System One 决策模型** 直接相关；
2. 条目格式：`- **标题**（作者/来源，日期）—— 一句话说明。[链接]`
3. 优先收录：官方来源、可复跑的评测、被广泛使用的集成；
4. 修复失效链接 / 错别字请直接提 issue。

### 许可

本清单内容遵循 [CC0 1.0](http://creativecommons.org/publicdomain/zero/1.0/) 公有领域协议，可自由使用。

---

*Last updated: 2026-09-24*
