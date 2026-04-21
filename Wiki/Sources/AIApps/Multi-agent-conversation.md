---
type: source
created: 2026-04-21
updated: 2026-04-21
tags: [ai, agent, multi-agent, subagent, context-engineering]
source_type: note
---

# Multi-agent 对话 — Claudian 解释自己怎么"多 agent"工作

- **原文**:[[Raw/Notes/Multi-agent 对话]]
- **作者**:Eureka × Claudian(对话式 Q&A)
- **日期**:2026-04-21
- **类型**:note(对话记录,归纳非逐字)

## 摘要

Eureka 提出两个直觉性问题:(1) 多 agent / 规划-执行链条到底怎么工作?上下文有限,切 agent 的意义是什么?(2) 流程是不是"用户派活给主 agent,主 agent 再新开会话派活给子 agent"?Claudian 以自身(带 `Agent` 工具的 Claudian 实例)为例给出完整回答。核心论点:**多 agent 的第一性原理不是"分工并行",而是"上下文隔离"**——正因上下文有限,才必须切多 agent,让子 agent 的脏活(大量文件读取/搜索/中间思考)关在一次性上下文里,只把压缩过的结论吐回主 agent,从而把"有效上下文"从单窗口 200K 放大到 N × 200K。附带四个次级收益(角色专用 system prompt / 模型分工 / 独立审视视角 / 失败隔离),以及长链条的衰减规律(扁平扇出 > 深度嵌套)。

## 关键要点

- **反直觉核心**:"上下文有限,切多 agent 才有意义"——反过来说才对。单 agent 200K 窗口会被中间噪音迅速污染(Context Rot);多 agent 让每个上下文保持干净,代价是 agent 间通信必须压缩
- **上下文的三个病**:Context Rot(注意力衰减)+ Prompt Cache 5 分钟 TTL 失效成本 + 指令漂移(system prompt 被子任务噪音稀释)
- **包工头隐喻**:主 agent 不亲自读文件,派"一次性实习生"读,实习生累死自己、交报告后被"解雇"(释放上下文)
- **"新开会话"字面准确**:每个子 agent = 一次全新的 Claude 对话,不知道用户、不知道主 agent 的 system prompt、只看派活那一刻的 prompt、干完吐一条消息、然后销毁
- **派 prompt 要像对刚入职同事**:背景、目标、已知信息、期望格式全写清楚,不能说"你懂的"
- **要求压缩汇报**(如"200 字以内"):不是省钱,是防子 agent 把原始信息倾倒回主 agent 上下文,白隔离
- **并行 vs 串行**:独立任务在同一条消息里发起多个 Agent 调用并行;依赖任务必须串行
- **四个次级收益**:
  1. 角色专用 system prompt + 限定工具集(规划 agent 不给 Write,执行 agent 不给 WebSearch)
  2. 模型分工(Opus 规划,Haiku 苦力)——单 agent 切模型就断对话
  3. 独立审视视角(审查 agent 未被主 agent 推理锚定)
  4. 失败隔离(子 agent 跑飞,主 agent 丢 trace 即可,不被污染)
- **长链条衰减**:5 级嵌套几乎不存在;现实常见是扁平扇出(1 主 → 3-5 子并行)或两级(planner → executor → reviewer);Anthropic 观察 agent 数量与 token 线性,但质量非线性
- **操作系统类比**:subagent = 子进程,主 agent 通过 IPC(文字消息)通信,隔离和通信受限都是特性不是 bug
- **Claudian 的实际用法举例**:"在 vault 里查所有提到注意力机制的笔记"→ 派 Explore agent,要求返回文件路径+一句话摘要清单、禁止返回原文,主 agent 上下文只长一张表

## 涉及实体 / 概念

- [[Wiki/Concepts/AIApps/Multi-agent|Multi-agent / Subagent 架构]](本次新建)
- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]]
- [[Wiki/Concepts/AIApps/Context-window|上下文窗口 & Context Rot]]
- [[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]]
- [[Wiki/Entities/Claudian|Claudian]](自证案例)

## 与既有 wiki 的关系

- **扩展**:[[Wiki/Concepts/AIApps/Ai-agent]] 原有"Subagent"一节只有 4 行概述;本 source 把多 agent 抽成独立概念页,补足第一性原理(上下文隔离)、四个次级收益、长链条衰减规律、派活 prompt 工程细节
- **印证**:[[Wiki/Concepts/AIApps/Context-window]] 关于 "Context Rot 是 subagent 分工的动机";本 source 把这个动机从"原因之一"升级为"核心第一性原理"
- **新增链接**:[[Wiki/Entities/Claudian]] 作为自证案例出现(Claudian 拥有 Agent 工具,能派 Explore/general-purpose/Plan 子 agent)
- **方法论印证**:与 [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution|三段论演进]] 中 Harness "上下文管理"支柱直接相关——多 agent 就是上下文管理的最强工具
- **配套读本**:[[Readers/AIApps/为什么上下文有限反而必须切多 Agent]] — 本 source 对应的线性叙事版,含自检问题
