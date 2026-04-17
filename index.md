# Index — Wiki 目录

> 本文件是整个 wiki 的内容目录。LLM 每次 ingest 都会更新。
> 查询时先读这里,再深入相关页面。

最后更新:2026-04-17

---

## Overview
- [[wiki/overview]] — 当前主题:方法论 + KuroEffectSystem ingest(Batch 1 已完成)

## Entities(实体)
*人、组织、地点、产品、项目。*

### 方法论相关
- [[wiki/entities/karpathy|Andrej Karpathy]] — AI 研究者;LLM Wiki 方法论提出者 (来源:1)

### project-game / KuroEffectSystem(老版)
- [[wiki/entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem]] — Old 入口类(instance-based)
- [[wiki/entities/project-game/kuro-effect-system/effect-handle|FEffectHandle (Old)]] — Old 句柄类
- [[wiki/entities/project-game/kuro-effect-system/effect-handle-info|FEffectHandleInfo]] — Old Handle 的 JS 伴生体
- [[wiki/entities/project-game/kuro-effect-system/effect-life-time|FEffectLifeTime (Old)]] — Old 时间/播放状态机
- [[wiki/entities/project-game/kuro-effect-system/effect-define|EffectDefine]] — 枚举 & Delegate 参考(两版共享)
- [[wiki/entities/project-game/kuro-effect-system/effect-model-base|UEffectModelBase 家族]] — 18 种 DataAsset 配置(两版共享)
- [[wiki/entities/project-game/kuro-effect-system/effect-parameters|EffectParameters 家族]] — Old 运行时参数抽象
- [[wiki/entities/project-game/kuro-effect-system/effect-spec-base|FEffectSpecBase + FEffectSpec<T>]] — Old Spec 基类 + CRTP 模板
- [[wiki/entities/project-game/kuro-effect-system/effect-spec-subclasses|Old Spec 子类目录]] — 12 个 FEffectModelXxxSpec

### project-game / KuroEffectSystemNew(新版)
- [[wiki/entities/project-game/kuro-effect-system-new/effect-system|KuroEffect::FEffectSystem]] — New 入口静态类
- [[wiki/entities/project-game/kuro-effect-system-new/effect-handle|KuroEffect::FEffectHandle]] — New 句柄核心
- [[wiki/entities/project-game/kuro-effect-system-new/effect-life-time|KuroEffect::FEffectLifeTime]] — New 时间状态(Timer 驱动)
- [[wiki/entities/project-game/kuro-effect-system-new/effect-spec-data|FEffectSpecData]] — New 数据索引层
- [[wiki/entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitModel / FEffectInitHandle]] — New 异步初始化管线
- [[wiki/entities/project-game/kuro-effect-system-new/effect-context|FEffectContext 家族]] — New 创建上下文(4 层继承)
- [[wiki/entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]] — New Actor 门面 + Action 命令模式
- [[wiki/entities/project-game/kuro-effect-system-new/effect-spec-base|IEffectSpecBase + KuroEffect::FEffectSpec<T>]] — New 接口 + 模板基类(~1150 行)
- [[wiki/entities/project-game/kuro-effect-system-new/effect-spec-factory|FEffectSpecFactory]] — New Spec 工厂
- [[wiki/entities/project-game/kuro-effect-system-new/effect-spec-subclasses|New Spec 子类目录]] — 17 个(比 Old 多 5 个)
- [[wiki/entities/project-game/kuro-effect-system-new/player-effect-container|FPlayerEffectContainer]] — 玩家作用域 LRU 分池(队伍 N 人各自一池)
- [[wiki/entities/project-game/kuro-effect-system-new/continuous-effect-controller|FContinuousEffectController]] — 连续技特效的 AnsSlot 过渡管理
- [[wiki/entities/project-game/kuro-effect-system-new/effect-system-actor|AEffectSystemActor]] — 专用 UE Actor 类(含 HandleId + 蓝图 API)
- [[wiki/entities/project-game/kuro-effect-system-new/niagara-component-handle|FNiagaraComponentHandle]] — Niagara 参数缓存门面(Pending 期间延迟下发)

## Concepts(概念)
*想法、理论、方法、术语。*

- [[wiki/concepts/llm-wiki-方法论|LLM Wiki 方法论]] — 让 LLM 增量构建并维护持久化 wiki 的范式 (来源:1)
- [[wiki/concepts/rag|RAG]] — 查询时检索文档片段的主流 LLM+文档范式 (来源:1)
- [[wiki/concepts/memex|Memex]] — Vannevar Bush 1945 提出的个人关联式知识存储愿景 (来源:1)

## Sources(源摘要)
*每个 raw 文件或代码源对应一页摘要。按时间倒序。*

### 代码模块(project-game)
- [[wiki/sources/project-game/kuro-effect-system/overview|KuroEffectSystem overview (Old)]] — Aki 特效系统第一代,~9k 行,已冻结 (p4 CL 6985991)
- [[wiki/sources/project-game/kuro-effect-system-new/overview|KuroEffectSystemNew overview]] — Aki 特效系统第二代,~23k 行,当前版本 (p4 CL 7086024)

### 文章 / 笔记
- [[wiki/sources/karpathy-llm-wiki]] — Karpathy 的 LLM Wiki idea file (2026-04,note)

## Syntheses(综合/专题)
*跨源分析、对比、好答案的沉淀。*

_(Batch 7 将产出:`kuro-effect-system-old-vs-new`)_

---

## 快速导航

- 方法论源文档:[[raw/notes/Karpathy Wiki 方法论]]
- 操作规程:[[CLAUDE]]
- 时间线日志:[[log]]
