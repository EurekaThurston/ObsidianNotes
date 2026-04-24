# 斯坦福 AI 小镇与生成式智能体 — 对话笔记

- **日期**:2026-04-25
- **参与者**:Eureka × Claudian
- **上下文**:Eureka 给了一篇知乎文章链接 [zhuanlan.zhihu.com/p/656007815](https://zhuanlan.zhihu.com/p/656007815)(a16z AI-Town 源码解读,接上一篇"斯坦福 AI 小镇源码解读"),询问:
  1. 文章讲什么
  2. 其中提到的"斯坦福筱桢"是谁,项目是什么,复现难度
  3. 这些项目都是几年前的,**目前进展怎么样,有没有新的相关项目**

> [!note]
> 这份对话是归纳版,不是逐字转录。原知乎文章(知乎 656007815)抓取时返回 403,基于 anchor 文字 + 同系列的相关文章检索还原上下文。

---

## 前置:术语辨识

Eureka 写作"斯坦福筱桢",实际是 **"斯坦福小镇"** 的视觉近形字误(两组汉字在某些 OCR / 输入法候选下会串)。议题本体是 **Stanford Smallville / Generative Agents**,作者 **Joon Sung Park**(朴俊成)。

## 1. 知乎 p/656007815 讲的是什么

那篇(根据上一篇、同系列标题、以及搜索结果)讲的是 **a16z-infra/ai-town 的源码解读**:

- 上一篇:斯坦福小镇本体源码(`joonspk-research/generative_agents`)
- 本篇:a16z 投资团队 + Convex CTO James Cowling 基于斯坦福小镇用 TypeScript + Convex 重写的开源可部署版本
- 对比维度:技术栈(Python + Phaser 前端 → TypeScript + Convex 后端 + PixiJS 前端)、存储方案、LLM 接入、部署门槛

## 2. 斯坦福小镇是什么

### 论文

- 《Generative Agents: Interactive Simulacra of Human Behavior》
- arXiv: [2304.03442](https://arxiv.org/abs/2304.03442)
- 会议:UIST 2023 Best Paper
- 机构:Stanford + Google
- 主作者:**Joon Sung Park**(斯坦福 HCI 博士,导师 Michael Bernstein),合作者含 Meredith Ringel Morris (Google)、Percy Liang、Michael Bernstein

### 设计

- 一个像素风小镇 **Smallville**,25 个 NPC 用 LLM(**GPT-3.5-turbo**,不是 GPT-4)驱动
- NPC 会"过日子":起床做饭上班、聊天、组织派对、形成观点、记住和反思前几天
- 意在证明:**只要 agent 架构合适,LLM 就能涌现出"可信的人类行为"**(believable human behavior)

### 架构三件套(真正的论文贡献)

1. **Memory Stream(记忆流)**
   - 存 agent 所有经历的原始观察,每条带时间戳
   - 检索时三维度打分:`recency`(时间衰减)+ `importance`(LLM 给这条记忆重要度打 1-10 分)+ `relevance`(当前情境 embedding 与记忆 embedding 的余弦)
2. **Reflection(反思)**
   - 触发:最近记忆 importance 累积超过阈值
   - 动作:让 LLM 从零散观察中提炼高阶洞察("Klaus 是个认真的学者")。反思也入记忆流,可被更高阶反思复用,形成层级
3. **Planning(规划)**
   - 粗→细:先一天计划 → 小时级 → 分钟级递归
   - 执行时遇到事件可被 reactive 打断重排

## 3. 复现难度

- **代码**:`joonspk-research/generative_agents` 已开源 Python 实现,能跑——所以"把 demo 跑起来"不是瓶颈
- **前端**:论文里 Phaser H5 游戏引擎 + 寻路 + 空间语义树(object tree / location tree)占工程量的一大半
- **Token 费**(真正的瓶颈):
  - 每个 agent 每 tick 都要调 LLM:感知过滤、记忆检索、对话生成、计划重排
  - 25 agent 并行,调用数量乘法爆炸
  - 复现者广泛报告:**25 agent 跑两天游戏时间烧几千美元 GPT-3.5 费用**
  - 国内流传数字:**模拟 100 秒游戏时间 ≈ 6 元 RMB**
- **改造**:想换地图 / 人设 / 换中文场景,工程量不小——主要是前端地图 + agent 空间感知那一套要重做

一句话:**代码不是问题,钱是问题。论文级完整实验不是个人能负担的**。

## 4. 后续进展(2024–2026)

### 4.1 Joon Sung Park 本人:从 25 人小镇 → 1000 个真实美国人

- 论文:《Generative Agent Simulations of 1,000 People》
- arXiv: [2411.10109](https://arxiv.org/abs/2411.10109)(2024-11)
- Stanford HAI 新闻:[AI Agents Simulate 1052 Individuals' Personalities with Impressive Accuracy](https://hai.stanford.edu/news/ai-agents-simulate-1052-individuals-personalities-with-impressive-accuracy)(2025-05)
- 斯坦福数字仓库:Park 的博士论文 [purl.stanford.edu/jm164ch6237](https://purl.stanford.edu/jm164ch6237)
- 方法:**真实找了 1052 个美国人做 2 小时深度访谈**,每人一份 agent + 把访谈转录塞给对应 agent 作上下文;样本按美国人口年龄/种族/性别/教育/政治光谱分层
- 结果:**agent 在 General Social Survey 上复现真人答案的 85%**——**跟真人两周后再答一遍的一致度相当**
- 意义转向:从 HCI demo → **"数字孪生社会学实验平台"**,可低成本预跑政策/内容推荐/公共健康干预的效应

### 4.2 a16z-infra/ai-town(知乎那篇的主角)

- repo: [github.com/a16z-infra/ai-town](https://github.com/a16z-infra/ai-town)
- 许可:MIT
- 技术栈:TypeScript + **Convex**(响应式后端/数据库)+ PixiJS 前端
- 默认 LLM:**llama3 + mxbai-embed-large**(本地 Ollama);也支持 Together.ai / OpenAI 兼容接口
- 定位:"斯坦福小镇的可部署生产级重写"——原版偏研究代码,a16z 这版"半小时 fork 能改成自己的版本"
- 2026-01 还有 podcast 讨论项目演进

### 4.3 同类 / 衍生

- **GenerativeAgentsCN**([x-glacier/GenerativeAgentsCN](https://github.com/x-glacier/GenerativeAgentsCN)):中文重构 + 本地 LLM 友好。国人复现首选起点
- **AgentSims**:国产"斯坦福小镇",更通用 agent 评测沙盘(不止生活模拟)
- **AI Town 衍生**:"猫猫小镇"之类民间二创,门槛被 a16z 版本降下来后遍地开花
- 学术:**LLM-driven agent-based simulation** 成了一个小方向,下游包括经济学实验 / 虚假信息扩散 / 游戏 NPC / 心理陪伴等

### 4.4 趋势

1. **方向分叉**:一支走"更多更真"的数字社会学(Park 2024 的 1000 人);一支走"更便宜更好玩"的游戏/陪伴(a16z AI Town 系)
2. **成本被开源本地模型部分解决**(llama3 / Qwen),但 reflection + planning + memory retrieval 的 token 成本仍是硬约束,工程优化方向:memory 压缩 / 异步 tick / 层级规划
3. **评测标准**:Park 2024 用真人问卷做 ground truth 是目前最严肃的答案

## 5. 对 Eureka 的实操建议

- 只看效果:跑 `a16z-infra/ai-town` + Ollama + llama3,本机就能起
- 研究架构:读 Park 2023 论文 + `joonspk-research/generative_agents` 代码,重点看 memory stream 打分 + retrieval,以及 reflection/planning prompt 模板(**精华都在 prompt 里**)
- 做二创/业务:a16z 版本作脚手架,替换人设/地图/事件系统,比从零搭省 2 个月

---

## 相关源

- [[Wiki/Sources/AIAgents/Generative-agents-discussion]] — 本对话的 source 摘要
- [[Wiki/Sources/AIAgents/Park-2023-generative-agents]] — arXiv 2304.03442 源摘要
- [[Wiki/Sources/AIAgents/Park-2024-1000-agents]] — arXiv 2411.10109 源摘要
