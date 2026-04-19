# Index — Wiki 目录

> 本文件是整个 wiki 的内容目录。LLM 每次 ingest 都会更新。
> 查询时先读这里,再深入相关页面。

最后更新:2026-04-19

---

## Overview
- [[Wiki/Overview]] — 当前主题:知识库方法论(自举)

## Entities(实体)
*人、组织、地点、产品、项目。*

### 方法论相关
- [[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]] — AI 研究者;LLM Wiki 方法论提出者 (来源:1)

### AI 美术（工具与基座）
- [[Wiki/Entities/AIArt/Illustrious-XL|Illustrious XL]] — SDXL 架构二次元基座,2026 初二次元 SOTA 之一 (来源:1)
- [[Wiki/Entities/AIArt/NoobAI-XL|NoobAI XL]] — Illustrious 的社区继续训练版,许可更友好 (来源:1)
- [[Wiki/Entities/AIArt/Flux|Flux.1]] — BFL 的 12B 写实 SOTA,Dev 版非商用 (来源:1)
- [[Wiki/Entities/AIArt/Kohya-ss|Kohya-ss]] — 主流 LoRA 训练工具,Windows 首选 (来源:1)
- [[Wiki/Entities/AIArt/ComfyUI|ComfyUI]] — 节点式推理平台,生产管线首选 (来源:1)

## Concepts(概念)
*想法、理论、方法、术语。*

### 方法论
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] — 让 LLM 增量构建并维护持久化 wiki 的范式 (来源:1)
- [[Wiki/Concepts/Methodology/Rag|RAG]] — 查询时检索文档片段的主流 LLM+文档范式 (来源:1)
- [[Wiki/Concepts/Methodology/Memex|Memex]] — Vannevar Bush 1945 提出的个人关联式知识存储愿景 (来源:1)

### UE4 基础
- [[Wiki/Concepts/UE4/UE4-uobject-系统|UE4 UObject 系统]] — UCLASS/USTRUCT/UPROPERTY 宏、GC、反射、序列化
- [[Wiki/Concepts/UE4/UE4-资产与实例|UE4 资产与实例]] — Asset（磁盘，只读模板）vs Instance（运行时，有状态）二元模型

### Niagara 基础
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade|Niagara vs Cascade]] — 设计哲学对比：黑盒模块 vs 完全可编程数据流
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟|Niagara CPU vs GPU 模拟]] — VectorVM(CPU) 与 Compute Shader(GPU) 的分工、能力边界与源码分叉点

### AI 美术
- [[Wiki/Concepts/AIArt/Lora|LoRA]] — 冻结主模型,只训低秩矩阵;游戏美术 DNA 固化的核心技术 (来源:1)
- [[Wiki/Concepts/AIArt/Base-model-selection|基座模型选型]] — Illustrious/NoobAI/Flux 对比,选错全白干 (来源:1)
- [[Wiki/Concepts/AIArt/Caption-strategy|Caption 策略]] — 想让 LoRA 永远带的不写,按需触发的写清楚（反常识） (来源:1)
- [[Wiki/Concepts/AIArt/Trigger-word|Trigger Word]] — 激活 LoRA 的独特词,命名约定 (来源:1)
- [[Wiki/Concepts/AIArt/Multi-lora-composition|Multi-LoRA 组合]] — ComfyUI 真正威力,MJ 做不到的模块化风格控件 (来源:1)

## Sources(源摘要)
*每个 raw 文件或代码源对应一页摘要。按时间倒序。*

### 文章 / 笔记
- [[Wiki/Sources/Methodology/Karpathy-llm-wiki]] — Karpathy 的 LLM Wiki idea file (2026-04,note)
- [[Wiki/Sources/AIArt/Lora-deep-dive]] — LoRA 深度指南：给鸣潮美术向流水线的落地方案 (2026-04,note)

## Syntheses(综合/专题)
*跨源分析、对比、好答案的沉淀。*

### Niagara 源码学习
- [[Wiki/Syntheses/Niagara/Niagara-learning-path|Niagara 源码学习路径]] — UE 4.26 Niagara 插件 10 阶段学习路线图，含 ~69 个文件待 ingest (stock)

---

## 快速导航

- 方法论源文档:[[Raw/Notes/Karpathy Wiki 方法论]]
- 操作规程:[[CLAUDE]]
- 时间线日志:[[log]]
