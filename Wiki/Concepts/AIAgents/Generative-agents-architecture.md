---
type: concept
created: 2026-04-25
updated: 2026-04-25
tags: [ai, agent, generative-agents, memory, reflection, planning, architecture]
sources: 2
aliases: [生成式智能体架构, Memory Stream, Reflection-Planning, Generative Agent Architecture]
---

# 生成式智能体架构(Memory + Reflection + Planning)

> Park et al. 2023 Smallville 论文提出的三件套 agent 架构——**Memory Stream(记忆流)+ Reflection(反思)+ Planning(规划)**——让单轮 LLM 能跨天演化出连贯个性、社交网络和涌现行为。这是当前 agent-based social simulation 最核心的架构模板。

## 一句话角色

把 LLM 的"无状态单轮响应"改造成"有持久记忆 / 会自我总结 / 会提前规划"的长期运行 agent 的最小可行架构。在此之前 agent 循环是"输入→LLM→输出→输入→LLM→..."的状态遗忘;这个架构让状态能**跨天累积**。

## 三件套

### 1. Memory Stream(记忆流)

> **问题**:200K context 装不下一个 agent 几天几十天的全部观察;全塞进去会 context rot。怎么"无限历史 + 有限 prompt"?

**方案**:所有观察以**自然语言 + 时间戳**追加到一条流水;检索时按**三维度打分** top-k 喂 prompt。

| 维度 | 含义 | 实现 |
|---|---|---|
| **recency** | 越新的记忆越重要 | 指数衰减:`score = λ^(t_now - t_memory)` |
| **importance** | 有些事(家人死亡)比另一些(吃早饭)值钱 | **让 LLM 自己打分 1-10 分**,入库时算好存着 |
| **relevance** | 当前问题相关的记忆更有用 | embedding 余弦(当前情境 query × 记忆 embedding) |

三分加权求和取 top-k → 注入 prompt。**这是 RAG 的一种,但按 recency+importance+relevance 三路加权,而不只是纯语义相似度**。

### 2. Reflection(反思)

> **问题**:原始观察是细粒度流水账("Klaus 在图书馆"、"Klaus 在读 gentrification 论文"、"Klaus 跟同学讨论社区议题")。agent 如何形成"Klaus 是个认真的学者"这种**人格级判断**?

**方案**:当最近记忆的 importance 累积超过阈值,触发一次 reflection:

1. LLM 扫描最近若干条记忆
2. 自己生成"几个最值得深究的问题"(meta-question)
3. 针对这些问题,从记忆流中检索相关证据
4. LLM 基于证据**生成高阶洞察**("Klaus 正在深入研究 gentrification")
5. 洞察本身也写回记忆流 → **可被更高阶的 reflection 继续提炼** → 形成层级

**关键性质**:反思不是后期加戏,是**pruning**——把 100 条"Klaus 在图书馆"压缩成 1 条"Klaus 认真"。长周期运行必需这层,否则 memory stream 爆炸同时 retrieval 噪声爆炸。

### 3. Planning(规划)

> **问题**:单纯反应式 agent 没有"一天 24 小时在做什么"的骨架,行为会随机乱跑。怎么让 NPC 有**长时间尺度**的连贯性?

**方案**:**粗到细**递归规划 + reactive 重排。

1. **一天计划**:早晨触发,LLM 按 agent 人设 + 昨天的反思生成粗粒度一天计划(5-10 条事件)
2. **小时细化**:粗计划细化到小时级
3. **分钟细化**:小时计划再细化到分钟级动作
4. **执行时遇事件**:比如另一个 agent 说"下午来喝咖啡",当前 agent 用 LLM 决定是否重排计划

**注意**:这不是传统 STRIPS / HTN 规划器——没有显式状态空间,**全靠 LLM prompt 生成 + 当前 agent 的记忆流上下文**。所以 plan 本身就是一条写入 memory stream 的自然语言记忆。

## 三件套之间的数据流

```
原始观察 ──→ Memory Stream(recency+importance+relevance 三路打分存储)
                │
      importance 累积过阈值
                ▼
           Reflection ──→ 高阶洞察 写回 Memory Stream
                                │
                    可被更高阶 Reflection 复用
                                │
           每天早晨触发
                ▼
           Planning ──→ 一天/小时/分钟级计划 写回 Memory Stream
                                │
                    执行时可 reactive 调整
                                │
           每个 tick
                ▼
        Agent 行动 ──→ 产生新观察 回到起点
```

## 关键性质

1. **所有东西都是自然语言**:观察、反思、计划全是字符串存 memory stream——人类可读、可调试、retrieval 统一
2. **LLM 自己打重要度**:importance 分数不靠人工标注,而是 LLM 自己读自己觉得多重要——这既是成本主力也是可行性关键
3. **反思是层级化的 pruning**:它是"无限历史的压缩器",不是附加特性
4. **Plan 写回 memory 构成 prompt 一部分**:执行时 agent "记得自己有什么计划",保持行动一致性

## 复现成本

- 每个 agent 每 tick 要若干次 LLM 调用:perceive(过滤观察)+ retrieve(三路打分)+ reflect(条件触发)+ plan(每日 + 细化)+ converse(对话生成)
- 25 agent × N tick × ~5 LLM 调用 / tick × 几天 → **论文级完整实验烧数千美元 GPT-3.5**
- 详见 [[Wiki/Entities/AIAgents/Stanford-smallville|Smallville]] 的复现要点

## 后续工作没直接继承这套架构

[[Wiki/Sources/AIAgents/Park-2024-1000-agents|Park 2024]] 走的是完全不同的 **self-report grounding** 路线——不追求 agent 自己"演化出"人格,而是把真人 2 小时访谈转录作为 persistent context。两种架构服务不同问题:

| | 2023 三件套 | 2024 self-report |
|---|---|---|
| 目标 | 可信的虚构人类行为 | 匹配特定真人 |
| 输入 | 人设 + 初始记忆 | 真人访谈 2 小时 |
| 关键机制 | Reflection + Planning | Retrieval-grounded prompt |
| 评测 | 主观 believability | 真人问卷 ground truth |

## 相关

- [[Wiki/Concepts/AIAgents/Agent-based-social-simulation]] — 本架构开辟的范式
- [[Wiki/Entities/AIAgents/Stanford-smallville]] — 本架构首次落地
- [[Wiki/Entities/AIAgents/A16z-ai-town]] — 简化版继承,memory/plan 保留、reflection 弱化
- [[Wiki/Concepts/AIFoundations/Context-window]] — memory stream 本质是对有限 context 的工程解
- [[Wiki/Concepts/AIFoundations/Multi-agent]] — 多 agent 架构在"长周期持续运行"场景的极端形态
- [[Wiki/Concepts/Methodology/Rag]] — memory stream 三路打分是 RAG 的特化

## 深入阅读

- 主题读本:[[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]] § 2
- Source:[[Wiki/Sources/AIAgents/Park-2023-generative-agents]]

## 开放问题

- `importance` 打分的鲁棒性:LLM 给"吃早饭"和"亲人去世"打的分稳定吗?论文附录有评测,本 wiki 未核验
- Reflection 阈值怎么选:太低→overfit,太高→记忆爆炸。Park 原版值 + 调参经验未登记
- 三件套的**成本能不能拆**:能否用小模型跑 perceive / retrieve,大模型只跑 plan / reflect?
