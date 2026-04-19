---
type: concept
created: 2026-04-20
updated: 2026-04-20
tags: [ai-coding, paradigm, karpathy, methodology]
sources: 1
aliases: [Vibe Coding, 氛围编程, vibe-coding]
---

# Vibe Coding (氛围编程)

> **Andrej Karpathy 2025 年 2 月在推文中首次命名**的 AI 辅助编程范式——"用自然语言描述,让 AI 去写"。原话:"There's a new kind of coding I call 'vibe coding'"。随后被 Y Combinator 等机构推广,成为 AI 编程演进的第一阶段标志。

## 一句话角色

Vibe Coding 是 AI 编程范式演进 **Vibe → Spec → Harness** 三阶段的起点。核心特征:**人给模糊意图、AI 自由发挥、人验收结果**——极大降低编程门槛(非技术人员能跑小项目、MVP、内部工具),但不可靠、难维护、难规模化是结构性短板。

## 核心事实 / 字段速查

| 维度 | 说明 |
|---|---|
| **命名** | Karpathy, 2025-02 推文(见 [[Wiki/Entities/Methodology/Karpathy]]) |
| **推广** | Y Combinator 把它作为"新一代创业范式"大力布道 |
| **典型场景** | 一次性脚本 / MVP / 个人项目 / 非技术人员做内部工具 |
| **工作方式** | 自然语言 prompt → AI 生成代码 → 跑 → 不对就再 prompt |
| **反模式** | 生产级代码 / 多人协作 / 需要长期维护的系统 |

## 三阶段演进中的位置

**引自 [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]] 第九章**:

> **Vibe Coding**:"随便跑!" → 乱
> **Spec Coding**:"按图纸跑!" → 好一点但还是难管
> **Harness Engineering**:"在专业赛道上跑,有护栏、裁判、计时器。" → 可靠、可维护、能规模化

每个阶段是对上一阶段痛点的响应:

1. **Vibe**(2025)——门槛低,但结果不可靠、代码不可维护、无质量保障
2. **Spec**(2025 中)——先写 Spec(类 PRD + 技术设计)再让 AI 按 Spec 实现,可控性变好但开发体验变重、Spec 本身容易过时
3. **Harness**(2026)——承认 AI 会犯错,在它外面搭"牧场":CI / test / lint / review / context 文件 / subagent。**把编码"随便跑"的自由保留,但加护栏兜底**

📖 完整脉络见 [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution]]。

## ⚠️ 常见误读

- **Vibe Coding 不是"AI 帮我写代码"的同义词**——那只是"用 AI 辅助编程";Vibe Coding 特指"**不写 spec、不做 review、口头意图到跑起来**"这种极简工作流。用 Cursor/Copilot 配合 code review 走的是 Spec / Harness,不是 Vibe
- **Vibe Coding 不等于"非程序员也能写代码"**——它确实降低了门槛,但非程序员跑出的代码用于生产,风险会在几周内暴露。Karpathy 本人也强调这是"**个人/一次性项目**"的范式
- **Vibe Coding 不是被"淘汰"的**——三阶段不是替代关系,是**场景分化**。写个一次性脚本、做个周末黑客松 demo,Vibe Coding 仍是最高效路径;上生产的多人协作项目才必须走 Harness

## 对非开发角色的适用

Vibe Coding 的思路在非编程场景也能借鉴,本质是"**先以最低摩擦跑起来,再在失败时加约束**":

- 美术: 先用 MidJourney 随便抽卡(Vibe)→ 发现需要风格锁定时上 LoRA(Spec)→ 落地生产需要多 LoRA 组合 + 版本控制 + 审核(Harness,见 [[Wiki/Concepts/AIArt/Multi-lora-composition]])
- 策划: 先让 AI 自由帮生成文案(Vibe)→ 发现语气不稳定时写提示词模板(Spec)→ 接入团队术语库 + 事实校验 + 统一风控(Harness)
- 管理: 面对新业务先"放一组人随便跑"(Vibe),出方向后定 Spec,规模化时建 Harness

## 相关

- [[Wiki/Entities/Methodology/Karpathy]] — Vibe Coding 命名者
- [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution]] — 三阶段演进的完整叙事
- [[Wiki/Concepts/AIApps/Harness-engineering]] — 演进的第三阶段
- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]] — Harness 阶段的主要执行体
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] — Karpathy 另一个"加护栏"的方法论

## 深入阅读

- 主题读本:[[Wiki/Readers/AIApps/AI-primer-v2-读本]](第九章)
- 扩展综合:[[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution]]

## 引用来源

- [[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]]) — 第 9.1 节

## 开放问题 / 待深入

- Vibe Coding 推文的原文链接与配图,需回到 Karpathy 的 X/Twitter 原帖追溯(v2 源是间接转述)
- YC 是如何"大力推广"的——有哪篇 YC blog / YC 演讲是关键节点,本 wiki 暂未追
- **Spec Coding** 尚无独立页;如被多次引用可单独建 `Wiki/Concepts/Methodology/Spec-coding.md`
