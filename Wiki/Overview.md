---
type: overview
created: 2026-04-17
updated: 2026-04-19
tags: [overview]
sources: 2
---

# Overview

> Wiki 的顶层综合视图。每次 ingest 有重大影响时,LLM 会在这里更新"当前的最佳理解"。

最后更新：2026-04-19

---

## 当前主题（3 个）

### 1. 知识库方法论（meta / bootstrap）

自举阶段的产出：基于 [[Raw/Notes/Karpathy Wiki 方法论]] 建的三层架构（Raw / Wiki / Schema）；详见 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]。

**核心主张**：持续整合 > 查询时检索。RAG 每次查询都在从零发现知识，LLM Wiki 则在 ingest 时就完成跨源整合、交叉引用、矛盾标注。

### 2. Niagara 源码学习（UE 4.26）

面向 C++ 零基础、UE 源码零基础的学员，通过 AI 辅助学习 Niagara 插件 749 个文件中的运行时部分（约 286 个文件）。

- 路线：[[Wiki/Syntheses/Niagara/Niagara-learning-path]] — 10 阶段路径
- Phase 0 已完成：心智模型建立（[[Wiki/Concepts/UE4/UE4-uobject-系统]]、[[Wiki/Concepts/UE4/UE4-资产与实例]]、[[Wiki/Concepts/Niagara/Niagara-vs-cascade]]、[[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]]）
- Phase 1-10 等待逐文件 ingest

### 3. AI 美术生成管线（LoRA / ComfyUI）

面向鸣潮美术向 TA 的落地方案，从 MidJourney + tag 库逐步迁移到 ComfyUI + 自训 LoRA。

- 核心源：[[Wiki/Sources/AIArt/Lora-deep-dive]]（2026-04，Eureka × Claude 撰写）
- 技术路径：[[Wiki/Concepts/AIArt/Lora|LoRA]] + [[Wiki/Entities/AIArt/Illustrious-XL|Illustrious/NoobAI]] 基座 + [[Wiki/Entities/AIArt/Kohya-ss|kohya_ss]] 训练 + [[Wiki/Entities/AIArt/ComfyUI|ComfyUI]] 部署
- 关键洞察：[[Wiki/Concepts/AIArt/Caption-strategy|Caption 策略反常识]]——想让 LoRA 永远带的不写，按需触发的写清楚
- 策略：复用飞书 tag 库前端，后端从 MJ 路由逐步切 ComfyUI
- **当前阶段：准备期**（评估/选型中，未开始 MVP 训练）
- 硬件：家里 RTX 5080、公司 RTX 5090（均 > 24GB VRAM，训/推都够）

---

## 跨主题联系（目前少）

- **方法论 → AI 美术**：这份 wiki 自己验证了 "LLM 可以管理跨领域知识库" 的假设——方法论、引擎源码、AI 管线三个毫不相关的主题共存
- **Niagara ↔ AI 美术**：都涉及"数据驱动的视觉产出"，但前者是 runtime 算法、后者是生产管线，目前无实质交集

---

## 主题知识图

```
methodology (meta, 自举)
    ├── Karpathy
    ├── LLM Wiki 方法论
    ├── RAG
    └── Memex

Niagara 源码学习 (UE 4.26)
    ├── UE4 基础
    │   ├── UObject 系统
    │   └── 资产与实例
    ├── Niagara 基础
    │   ├── vs Cascade
    │   └── CPU vs GPU 模拟
    └── Niagara 学习路径 (10 阶段, Phase 0 ✅)

AI 美术 (LoRA/ComfyUI)
    ├── 概念
    │   ├── LoRA
    │   ├── 基座模型选型
    │   ├── Caption 策略
    │   ├── Trigger Word
    │   └── Multi-LoRA 组合
    └── 实体
        ├── Illustrious XL / NoobAI XL
        ├── Flux.1
        ├── Kohya-ss
        └── ComfyUI
```

---

## 开放问题 / 待观察

### 关于 Wiki 本身

- **规模边界**：当前约 25+ 页，index.md 还够用；超过 200 页时怎么办？
- **Concept 页跨仓共享**：`stock` 仓 ingest 时，[[Wiki/Concepts/UE4/UE4-uobject-系统]] 等页能复用多少？
- **跨主题链接**：3 个主题几乎完全独立，是特性还是可以挖出深层联系？

### 关于 Niagara 路径

- **Phase 1 启动时机**：等待用户指令，下一个文件是 `NiagaraSystem.h`
- **Code root 本机可用性**：已登记 `stock`（D:\UE\...），`project-*` 本机未登记

### 关于 AI 美术

- **基座选型实时性**：每 3-6 月要重查一次当前 state
- **许可变动追溯**：Illustrious 条款历史变动是否影响已训 LoRA？
- ~~**鸣潮团队落地阶段**~~：已确认 = **准备期**（2026-04-19）

---

## 入口

- 所有页面目录：[[index]]
- 操作规程：[[CLAUDE]]
- 时间线日志：[[log]]
- **方法论原文**：[[Raw/Notes/Karpathy Wiki 方法论]]
- **AI 美术路线源**：[[Raw/Notes/Lora_Deep_Dive]]

---

## 下一步建议

- **Niagara Phase 1**：开始读 `NiagaraSystem.h` / `NiagaraEmitter.h` / `NiagaraScript.h`
- **AI 美术**：验证本机 kohya_ss 环境，跑第一个 MVP LoRA
- **lint**：运行 "帮我 lint 一下 wiki"，检查 3 个主题的内部一致性
