---
type: concept
created: 2026-04-27
updated: 2026-04-27
tags: [ai, agent, harness, long-running, ralph]
sources: 1
aliases: [Ralph, Ralph 循环, Ralph 方案, Ralph Wiggum 模式]
---

# Ralph 循环模式

> 外部 while 循环不断起干净的新 AI 会话,**前后会话靠文件系统接力**——用持久化文件换上下文窗口,让 AI 跑长任务而上下文永不爆炸。

## 一句话角色

让单个 LLM Agent 跑超长任务的最朴素 harness 模式。**不是模型能力提升,是工程结构兜底**——每次会话冷启动,任务状态外置成文件,循环跑到收敛或退出条件命中。

## 第一性原理

[[Wiki/Concepts/AIFoundations/Context-window|上下文窗口]] 是稀缺资源,且会被 [[Wiki/Concepts/AIFoundations/Context-window|Context Rot]] 拖垮。处理"做几小时几十步"的任务,单会话必然撑爆。

两条解法路线:

1. **[[Wiki/Concepts/AIFoundations/Multi-agent|多 Agent]]**:空间维度切——主 agent 派子 agent 并行/分工,每个 agent 自己的窗口
2. **Ralph**:时间维度切——一个 agent 跑短一段就重启会话,**状态外置到文件**,下一轮新会话从文件恢复

两者**正交不互斥**,实战常组合:Ralph 循环里每轮主 agent 自己也派子 agent。

## 结构骨架

```
while not done:
    # 1. 起干净新会话(全新 Claude Code / Codex 进程)
    # 2. 让它读 progress.md / state.json / TODO.md
    # 3. 做下一小段工作(改几个文件 / 跑测试 / 写代码)
    # 4. 把进度写回 progress.md(下一轮的入参)
    # 5. 退出条件:任务清单清零 / 测试全绿 / 达到最大轮次
```

外层循环可以是 bash / python / Makefile / GitHub Actions——载体不重要,**机制重要**。

## 核心组件

| 组件 | 作用 | 典型形式 |
|---|---|---|
| **任务清单文件** | 当前要做什么、做到哪 | `TODO.md` / `tasks.json` |
| **进度日志文件** | 已完成什么、踩了什么坑 | `progress.md` / `journal.md` |
| **经验库** | 项目约定 / 已踩坑的教训,每轮喂回 | `CLAUDE.md` / `LESSONS.md` |
| **退出条件检查** | 防止无限循环烧钱 | 测试通过 / max_iter / 用户介入 |

## 与多 Agent 的取舍

| 维度 | Ralph | Multi-Agent |
|---|---|---|
| **切的维度** | 时间(序列化) | 空间(并行/分工) |
| **上下文复用** | 完全冷启动,prompt cache 失效 | 子 agent 短命但同会话内可缓存 |
| **状态载体** | 文件(显式) | 主 agent 上下文(隐式) |
| **失败恢复** | 任意时间点中断/恢复(文件就在) | 子 agent 跑飞主 agent 兜底 |
| **典型场景** | 一个人跑一晚上 / 在 CI 跑大型重构 | 一次会话内的复杂任务 |

⚠️ **不是替代关系**——成熟方案常 Ralph 外循环 + 每轮内多 agent 协作。

## 视频里的"经验库"机制

Ralph 跑长后,AI 会重复犯相同错误(因为每轮新会话不记得上轮)。解法是把"已踩过的坑"持久化到经验库文件,每轮新会话开头喂回。这对应 Mitchell Hashimoto 对 [[Wiki/Concepts/AIFoundations/Harness-engineering|Harness]] 的核心定义:

> **每当发现 Agent 犯错,就花时间设计一个机制,确保它永远不会再犯同样的错。**

经验库就是这一定义的最朴素落地——文本文件 + 渐进式追加。

## ⚠️ 容易踩的坑

- **状态文件无 schema** → AI 自由格式写,几轮后自己读不懂自己的 progress.md。要预先定义结构(章节 / 字段 / 状态机标记)
- **退出条件设计不当** → 死循环烧 API。**永远要有 max_iter 兜底**
- **经验库无限膨胀** → 跑 100 轮后塞满 [[Wiki/Concepts/AIFoundations/Context-window|Context Window]],反而触发 Context Rot。需要定期人工或 AI 蒸馏(支柱四:熵管理)
- **没有人工 checkpoint** → 跑歪几小时无人发觉。关键节点应触发 HITL 审批(改 schema、删数据、对外发消息等)

## 名字来源

"Ralph" 据说取自《辛普森一家》Ralph Wiggum——那个执拗、单纯、反复做同一件事的小孩。社区里也有人写作 "Ralph Wiggum 模式"。具体首次提出者本笔记未考证(视频未明说,Claudian 不臆断)。

## 相关

- [[Wiki/Concepts/AIFoundations/Multi-agent|Multi-agent / Subagent 架构]] — 切上下文的另一条正交路径,常与 Ralph 组合
- [[Wiki/Concepts/AIFoundations/Harness-engineering|Harness Engineering]] — Ralph 是其"上下文管理 + 反馈循环 + 熵管理"三支柱的一个具名落地形态
- [[Wiki/Concepts/AIFoundations/Context-window|上下文窗口 & Context Rot]] — Ralph 存在的根本动因
- [[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]] — Ralph 包裹的对象

## 引用来源

- 原子 source:[[Wiki/Sources/AIFoundations/Ralph-multi-agent-video]] (raw: [[Raw/Notes/Ralph + 多智能体协同 - 费曼学徒冬瓜]])

## 开放问题

- **Ralph 与 Multi-agent 的最佳混合比例**:外层 Ralph 多少轮,内层并发多少子 agent,目前没有公开经验数据,只有"看着办"
- **经验库的蒸馏周期**:多少轮该做一次"压缩 / 重构"才不让经验库本身变成 Context Rot 源?
- **首次正式命名出处**:社区流传"Ralph Wiggum 模式"已久,本概念页未考证首次提出者(视频未给出溯源)
- **状态文件标准化**:有没有像 OpenAPI / JSON Schema 那样的"Ralph state file 标准"?目前看是各家自己拍脑袋
