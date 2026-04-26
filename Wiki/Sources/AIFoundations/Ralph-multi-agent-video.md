---
type: source
created: 2026-04-27
updated: 2026-04-27
tags: [ai, agent, multi-agent, harness, ralph, claude-code]
source_type: note
---

# Ralph + 多智能体协同 — 费曼学徒冬瓜(2026-04-26 视频)

- **原文**:[[Raw/Notes/Ralph + 多智能体协同 - 费曼学徒冬瓜]]
- **作者**:费曼学徒冬瓜(bilibili UP 主)
- **日期**:2026-04-26
- **URL**:https://www.bilibili.com/video/BV1t9oZBDENp/
- **类型**:note(视频元数据 + Claudian 解读;非逐字稿)

## 摘要

讲让 AI 长时间高品质自主工作的两种工程方案,实操平台是 Claude Code。一是 **Ralph**:外部 while 循环不断起干净新会话,前后状态靠文件系统衔接,本质是"用持久化文件换上下文窗口"。二是**多智能体协同**(视频推荐):主 Agent 只协调不干活,子 Agent 按角色(计划 / 代码 / 测试)分工,配合任务拆分、每角色单独的 system prompt、以及把"踩过的坑"沉淀进**经验库**做迭代优化。

## 关键要点

- **Ralph = while 循环 + 文件系统接力**:每轮新会话冷启动,prompt cache 失效是固有代价,换来上下文绝对干净
- **多 Agent = 角色分工 + 上下文隔离**:主 agent 拆任务派活,不亲自读代码;子 agent 各司其职,用单一职责限定其工具集和输出格式
- **典型角色**:主(协调)/ 计划(只产 plan)/ 代码(只按 plan 改)/ 测试(只跑测试 + 报告)。开发-测试分工是质量闭环的关键
- **三件套配套**:任务拆分 / 提示词设计 / 经验库优化。**经验库**是把"已踩过的坑"持久化喂回 AI,防同错重犯——这条与 Mitchell Hashimoto 对 Harness 的核心定义同构
- **落地工具**:Claude Code 命令行版(原生 subagent 机制)

## 涉及实体 / 概念

- [[Wiki/Concepts/AIFoundations/Ralph-pattern|Ralph 循环模式]] (新建)
- [[Wiki/Concepts/AIFoundations/Multi-agent|Multi-agent / Subagent 架构]]
- [[Wiki/Concepts/AIFoundations/Harness-engineering|Harness Engineering]]
- [[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]]
- [[Wiki/Concepts/AIFoundations/Context-window|上下文窗口 & Context Rot]]

## 与既有 wiki 的关系

- **印证**:[[Wiki/Concepts/AIFoundations/Multi-agent]] 第一性原理"上下文隔离" + 角色专用 system prompt + 限定工具集 + 不同模型分工 + 失败隔离;视频提供了具体角色模板(计划/代码/测试)做实例化
- **印证**:[[Wiki/Concepts/AIFoundations/Harness-engineering]] 四大支柱——任务拆分(支柱一)、角色 prompt(支柱二)、测试-反馈闭环(支柱三)、经验库(支柱四 + Hashimoto 定义)
- **新增**:[[Wiki/Concepts/AIFoundations/Ralph-pattern|Ralph 循环模式]]——作为"长任务 harness 模式"的具名解法,与多 agent 是平行(而非互斥)的两条路
- **未矛盾**

## ⚠️ 抓取局限

WebFetch 只拿到视频 metadata(标题 + 简介),无字幕全文。视频里大概率包含但本 source 未捕获:Ralph 的状态文件具体结构 / while 循环退出条件 / 视频展示的具体提示词模板 / 经验库的文件格式与喂回机制。后续若用户整理逐字稿,可升级本 source。
