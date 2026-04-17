---
type: overview
created: 2026-04-17
updated: 2026-04-17
tags: [overview]
sources: 3
---

# Overview

> Wiki 的顶层综合视图。每次 ingest 有重大影响时,LLM 会在这里更新"当前的最佳理解"。

最后更新:2026-04-17(Batch 7 完成)

---

## 当前主题

**Wiki 现在覆盖两个主题**:

### 1. 知识库方法论本身(meta / bootstrap)

自举阶段产出:基于 [[raw/notes/Karpathy Wiki 方法论]] 建的三层架构(raw/wiki/schema);详见 [[wiki/concepts/llm-wiki-方法论|LLM Wiki 方法论]]。

### 2. **Kuro EffectSystem(Aki 游戏项目的特效系统)**—— 当前主要内容

通过 7 批 ingest 完成对 `project-game` 仓的 `KuroGameplay/Plugins` 插件中特效系统的**完整文档化**。覆盖:
- **老版**(`KuroEffectSystem/`,Aki 第一代,已冻结)
- **新版**(`KuroEffectSystemNew/`,当前主力,~23k 行)
- 两版架构对比

---

## Kuro EffectSystem 速览

### 产出规模
- **43 个 wiki 实体页**(入口类、Handle、LifeTime、Spec、各种 Handle/Container/Bridge、Reflection、BP Lib)
- **2 个 source overview**(Old / New 各一)
- **1 个大 synthesis**:[[syntheses/kuro-effect-system-old-vs-new|Old vs New 架构对比]]
- **2 个跨仓 concept**:[[concepts/scripting-bridge-pattern|脚本桥模式]]、[[concepts/pending-init-pattern|Pending Init 模式]]

### 核心结论(来自 [[syntheses/kuro-effect-system-old-vs-new|Old vs New]])

Old → New 是**架构级重写**,解决 5 个结构性问题:

1. **JS 耦合贯穿全链** → **ScriptBridge 层 + Holder 抽象**(Spec/Handle/LifeTime 零 v8)
2. **裸指针所有权** → **TSharedPtr/TWeakPtr/TStrongObjectPtr/TUniquePtr** 精细所有权
3. **类型化硬编码**(7 Init 重载) → **数据驱动**(FEffectSpecData + FEffectSpecFactory)
4. **同步 Spawn** → **4 阶段异步管线**(Spawn→LoadData→Init→Play,Pending 缓存)
5. **无联机支持** → **FPlayerEffectContainer** 分池 + **FContinuousEffectController** 连续技 + C# Bridge

### 规模对比
| 指标 | Old | New |
|---|---|---|
| 总代码量 | ~9k 行 | ~23k 行(2.5x) |
| Spec 接口规模 | 6 纯虚 | 80+ 纯虚 |
| 运行时 CVars | 0 | 15+ |
| 脚本宿主支持 | 1(Puerts) | 2(Puerts + C#) |

### 关键架构改变

**JS 解耦**:v8::Isolate* 从 Handle/Info/LifeTime/Spec 5 个类中抹除,只剩 `FEffectSystem::Initialize` 入口一处。

**Pending Init 管线**(Batch 2/6):业务调 SpawnEffect → 立即返回 HandleId → 业务可对 Pending Handle 做任意操作(被缓存到 `FEffectInitHandle::EffectActorHandle` 的 Action 队列)→ 资源 ready 后 flush。

**Addition TimeScale**(Batch 3):多来源 TimeScale 乘法叠加(`TMap<int32 SourceType, float Value>`),每个 source 独立开关,帧内缓存。

**3 段 Tick**(Batch 6):`TG_PrePhysics` / `TG_PostPhysics` / `TG_PostUpdateWork` 各一条 Tick,粒度细化 3 倍。

**Handle Id 位编码**:32 bit = Version + Index,编辑器(15+17)vs Shipping(12+20)差异化。Version 递增防 ABA。

---

## 其他论点(来自方法论 ingest)

**关于 LLM Wiki 本身**(基于 Karpathy 那篇 idea file):

1. **持续整合 > 查询时检索**:相比 [[wiki/concepts/rag|RAG]] 每次重拼碎片,[[wiki/concepts/llm-wiki-方法论|LLM Wiki]] 在 ingest 时就完成跨源整合
2. **三层架构**:raw(只读源)/ wiki(LLM 写的 md)/ schema(规则)
3. **人类提供方向,LLM 负责 bookkeeping**:一次 ingest 触达 15 个文件不是 bug,是特性
4. **思想溯源到 [[wiki/concepts/memex|Memex]]**(1945)

本 wiki 本身就是这个方法论的**实证**——7 批 ingest 产出 43+ entity 页 + 1 大 synthesis + 2 concept + 2 source overview,全程 LLM 维护。

---

## 开放问题 / 待观察

### 关于 KuroEffectSystem
- 4 个 "待兼容" Spec(MaterialController/SequencePose/NDC/CurveTrailDecal)**什么时候完全就绪**?当前状态下业务使用这些类型会怎样?
- Private/.cpp 主力文件仍有大半未读(EffectSystem.cpp 剩 3400 行、EffectHandle.cpp 全 1926 行)——按需读
- 同一系统的 **业务侧 TS 代码**(不在本 wiki 内,在 Client 项目外部)如何使用这套 API?Spawn 调用模式?

### 关于 Wiki 本身
- **规模边界**:当前 ~50 个实体页,index.md 还够用;当超过 200 页时怎么办?
- **Concept 页跨仓共享**:已产出 2 个项目性 concept(Bridge、Pending),未来 `stock` 仓 ingest 时能否复用?
- **同名类孪生管理**:当 `stock` 仓也 ingest 后,`FEffectHandle` 等类会和 Kuro 里的类重名,靠目录隔离 + `twin:` 链。这个机制要实测。

---

## 主要分支(目前)

```
当前 wiki 的知识图 ≈ 3 条线

methodology (meta)
    ├── karpathy.md
    ├── llm-wiki-方法论
    ├── memex
    └── rag

project-game / kuro-effect-system
    ├── Old 系统(10 entity + 1 source)
    ├── New 系统(19 entity + 1 source)
    └── 对比 synthesis

cross-cutting concepts
    ├── scripting-bridge-pattern
    └── pending-init-pattern
```

`stock` 仓(UE 公版引擎)**尚未 ingest**——将来 ingest 时应该:
1. 新 entity 页进 `entities/stock/`
2. 同名类(如 UE 的 `UNiagaraSystem`)可能和 Kuro 里的 Niagara 相关页建 twin 链
3. Concept 页重用(`scripting-bridge-pattern` 在公版 UE 里也有?BlueprintVM 算不算?)

---

## 入口

- 所有页面目录:[[index]]
- 操作规程:[[CLAUDE]]
- 时间线:[[log]]
- **核心合成**:[[syntheses/kuro-effect-system-old-vs-new|KuroEffectSystem Old vs New]]
- **方法论原文**:[[raw/notes/Karpathy Wiki 方法论]]

---

## 下一步建议

### 短期(可触手可及的)
1. **在 Rider 打开一个具体业务用例**,对照 wiki 里的 Spawn 流程、AdditionTimeScale、PendingInit 看实际运行
2. **读 EffectHandle.cpp** 解答 OwnerEffect/BodyEffect 的完整实现
3. **关注"待兼容" Spec**(NDC/MaterialController/SequencePose/CurveTrailDecal)—— 观察它们在实际运行中的状态

### 中期
4. **纳入 `stock`(UE 公版引擎)** —— 特别是 Niagara 公版对应 Kuro 这边的使用方式
5. **拉一个 Kuro 其他插件进来**(如 `KuroGAS` / `KuroTimerSystem`)—— 一个 wiki 多个插件能共享多少概念
6. **写 BodyEffect / OwnerEffect 专题** —— 两个独立小概念值得单独总结

### 长期
7. **业务代码侧 ingest** —— 把 TS 业务代码(`F:\Aki\dev\Source\Client\TypeScript/`)作为 raw source 吸收,对照 C++ 侧看真正的调用模式
8. **跨插件性能分析** —— 如果系统里的特效 Tick 慢,从这里的 AdditionTimeScale/LRU/Tick 分组 文档能快速定位瓶颈
