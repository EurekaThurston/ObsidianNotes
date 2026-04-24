---
type: source
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, source-code, asset, emitter]
sources: 1
aliases: [NiagaraEmitter.h, UNiagaraEmitter 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitter.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraEmitter.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitter.h`
- **快照**: commit `b6ab0dee9`
- **同仓协作文件**: `NiagaraEmitter.cpp`、`NiagaraEmitterInstance.h`(Phase 3)、`NiagaraRendererProperties.h`(Phase 6)、`NiagaraScript.h`、`NiagaraSimulationStageBase.h`(Phase 10)
- **Ingest 日期**: 2026-04-19
- **学习阶段**: Phase 1 — 资产层 1.2

## 职责 / 这个文件干什么的

定义 `UNiagaraEmitter`——**一个 Emitter 的静态描述**:要跑几个脚本(Spawn/Update/EventHandler/GPU Compute/SimulationStage)、如何渲染(RendererProperties 列表)、CPU 还是 GPU 模拟、是否 local space、是否确定性随机、是否插值 spawn、scalability 覆盖、预分配策略等。

相比 `UNiagaraSystem` 的 "容器"角色,Emitter 才是描述"怎么产生和演化粒子"的**单元**。多个 Emitter 可以共享同一个 System 资产(例如"爆炸 = 火花 Emitter + 烟雾 Emitter + 碎屑 Emitter")。

这个文件同时定义了 Emitter 持有的若干 USTRUCT:事件收发配置、脚本配置包装器等。

## 关键类型 / 函数 / 宏

### 主类

- `UNiagaraEmitter : public UObject`(L210),`UCLASS(MinimalAPI)`
  - `friend struct FNiagaraEmitterHandle`——handle 可直接访问 Emitter 内部,配合 merge 等操作

### 事件系统相关 USTRUCT

- `FNiagaraEventReceiverProperties`(L28):一个事件接收端,声明"我要接收来自哪个 Emitter 的哪个 EventGenerator 的事件"
- `FNiagaraEventGeneratorProperties`(L65):一个事件生成端,含 MaxEventsPerFrame 和它自己的 `FNiagaraDataSetCompiledData`(事件本质是"粒子级别的小消息队列",用 DataSet 存)
- `FNiagaraEmitterScriptProperties`(L113):**脚本的包装器** — `Script*` + `EventReceivers` + `EventGenerators`。Spawn/Update 脚本都是这个类型
- `FNiagaraEventScriptProperties : FNiagaraEmitterScriptProperties`(L137):事件处理器脚本,多出 `ExecutionMode`(EveryParticle / SpawnedParticles / SingleParticle)、`SpawnNumber`、`SourceEmitterID`、`SourceEventName` 等

### 枚举

- `EScriptExecutionMode`(L93):控制事件 handler 对哪些粒子执行
- `EParticleAllocationMode`(L104):`AutomaticEstimate`(运行时估算) / `ManualEstimate`(手填 `PreAllocationCount`)

### 核心字段(分组)

**模拟模式 / 空间**
- `bLocalSpace`(L259):粒子相对 Emitter 还是世界空间
- `SimTarget`(`ENiagaraSimTarget`,L304):`CPUSim` or `GPUComputeSim` — 这是 [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]] 里的核心分叉字段
- `bDeterminism`(L263)+ `RandomSeed`(L267):确定性随机

**生命周期 / 分配**
- `AllocationMode`(L275)+ `PreAllocationCount`(L282)
- `bInterpolatedSpawning`(L325):**插值 spawn**,帧间均匀分布 spawn 时间,防止节奏感被帧率拖抖;对应 `ENiagaraScriptUsage::ParticleSpawnScriptInterpolated`
- `bRequiresPersistentIDs`(L345):给粒子分配持久 ID
- `bCombineEventSpawn`(L349):合并事件 spawn 为单次执行(ExecIndex 语义会变)
- `bLimitDeltaTime`(L377)+ `MaxDeltaTimePerTick`(L353):限制 dt 防止大帧卡顿破坏模拟

**脚本四件套(运行时)**
- `SpawnScriptProps`(`FNiagaraEmitterScriptProperties`,L288):ParticleSpawn
- `UpdateScriptProps`(L285):ParticleUpdate
- `EventHandlerScriptProps`(`TArray<FNiagaraEventScriptProperties>`,L615):若干事件处理器
- `GPUComputeScript`(`UNiagaraScript*`,L621):GPU 模式下用的 compute script
- (editor-only)`EmitterSpawnScriptProps / EmitterUpdateScriptProps`(L292-L295):Emitter 级脚本;**注意**:他们实际不走 UNiagaraScript 编译路径,而是被合并进 SystemSpawn/UpdateScript(见 `UNiagaraScript::IsCompilable`:EmitterSpawn/EmitterUpdate **不可单独编译**)

**渲染 / 结构**
- `RendererProperties`(`TArray<UNiagaraRendererProperties*>`,L612):此 Emitter 的所有渲染器(Sprite/Mesh/Ribbon/Light…,Phase 6)
- `SimulationStages`(`TArray<UNiagaraSimulationStageBase*>`,L618):Simulation Stage 列表(Phase 10)
- `UniqueEmitterName`(`FString`,L609):在所属 System 内的唯一名

**Scalability**
- `Platforms`(`FNiagaraPlatformSet`,L318):允许跑在哪些平台
- `ScalabilityOverrides`(L321):对 EffectType 默认规则的 override

**Source(editor-only)**
- `GraphSource`(`UNiagaraScriptSourceBase*`,L392):所有此 Emitter 内脚本的共享图源,见 [[Wiki/Sources/Stock/Niagara/NiagaraScriptSourceBase]]

**继承 / Merge(editor-only,Niagara 的"继承"机制)**
- `Parent`(L628):此 Emitter 是从哪个"模板 Emitter"衍生来的
- `ParentAtLastMerge`(L631):上次 merge 时父的快照
- `MergeChangesFromParent()`(L454):从父 Emitter 合并变更下来——**这是 Niagara 继承机制的核心**,区别于 Cascade 的"拷贝就断了"

### 关键方法

- `GetScript(ENiagaraScriptUsage Usage, FGuid UsageId)`(L381):按 usage + usage id 查脚本
- `ForEachEnabledRenderer(TAction)`(L673 实现):遍历启用且 sim target 兼容的渲染器
- `ForEachScript(TAction)`(L683 实现):遍历 Spawn/Update/GPU/EventHandler 脚本
- `AddRenderer / RemoveRenderer`(L495-L497):编辑器 API
- `AddEventHandler / RemoveEventHandlerByUsageId`(L505-L507)
- `AddSimulationStage / RemoveSimulationStage / MoveSimulationStageToIndex`(L513-L517)
- `IsAllowedByScalability()`、`IsValid()`、`IsReadyToRun()`
- `CacheFromCompiledData / CacheFromShaderCompiled`(L386-L387):把编译/编译后的数据缓存成运行时可用形式
- `GetMaxParticleCountEstimate()`(L532):基于 `MemoryRuntimeEstimation` 估最大粒子数

### `MemoryRuntimeEstimation`(L198)

非 UCLASS/USTRUCT 的裸结构体,记录运行时看到的分配峰值 → 下一次 System 实例分配时参考。配合 `bInterpolatedSpawning` 与 `AllocationMode::AutomaticEstimate` 使用。

## 依赖与被依赖

**上游:**
- `NiagaraCommon.h`、`NiagaraScript.h`、`NiagaraRendererProperties.h`(渲染器基类)
- `NiagaraEffectType.h`(scalability 结构体)
- `NiagaraBoundsCalculator.h`(包围盒计算器)
- `NiagaraMessageDataBase.h`(编辑器消息)
- `INiagaraMergeManager.h`(merge 接口)

**下游:**
- `FNiagaraEmitterHandle`(通过 `friend` 关系直接持有 `Instance`)
- `FNiagaraEmitterInstance`(Phase 3,运行时按此 Emitter 的字段初始化)
- 编辑器 Stack UI(`NiagaraEditorWidgets`):几乎所有粒子编辑操作 = 改这里的字段
- `UNiagaraSystem::ForEachScript`:委托给此类的 `ForEachScript`

## 关键代码片段

> `NiagaraEmitter.h:L205-L215` @ `b6ab0dee9` — 类声明 + friend 关系
> ```cpp
> /**
>  *	UNiagaraEmitter stores the attributes of an FNiagaraEmitterInstance
>  *	that need to be serialized and are used for its initialization
>  */
> UCLASS(MinimalAPI)
> class UNiagaraEmitter : public UObject
> {
>     GENERATED_UCLASS_BODY()
>     friend struct FNiagaraEmitterHandle;
> ```
> 注释直说了:此类是"**Instance 需要序列化的那部分属性**" —— 明确的"Asset 描述初始化 Instance"模型。

> `NiagaraEmitter.h:L683-L695` @ `b6ab0dee9` — ForEachScript 脚本清单
> ```cpp
> template<typename TAction>
> void UNiagaraEmitter::ForEachScript(TAction Func) const
> {
>     Func(SpawnScriptProps.Script);
>     Func(UpdateScriptProps.Script);
>     Func(GPUComputeScript);
>     for (auto& EventScriptProps : EventHandlerScriptProps)
>     {
>         Func(EventScriptProps.Script);
>     }
> }
> ```
> 一个 Emitter 的脚本树 = ParticleSpawn + ParticleUpdate + GPUCompute + N × EventHandler。EmitterSpawn/EmitterUpdate 脚本**不**在这里,因为它们被合并进了 System 脚本。

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — 本文件定义的主类
- [[Wiki/Entities/Stock/Niagara/UNiagaraScript]] — 所有脚本字段的类型
- [[Wiki/Entities/Stock/Niagara/FNiagaraEmitterHandle]] — 持有 Emitter 的 `friend`
- [[Wiki/Entities/Stock/Niagara/UNiagaraSystem]] — 容器
- [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]] — `SimTarget` 字段是此分叉的根源
- [[Wiki/Concepts/UE/Niagara/Niagara-vs-cascade]] — Emitter 继承 + merge 机制是相对 Cascade 的主要架构升级之一

## 与既有 wiki 的关系

- 印证了 [[Wiki/Concepts/UE/Niagara/Niagara-vs-cascade]] 里提到的"`bInterpolatedSpawning` 是 Cascade 时代概念的延续"
- 印证了学习路径里 [[Wiki/Syntheses/UE/Niagara/Niagara-learning-path]] 的 "UNiagaraEmitter 持有渲染器列表 + 脚本列表" 描述
- 新增认识:**Emitter 级脚本(EmitterSpawn/Update)不是独立运行的,而是被编译合并进 System 的两个脚本**——Cascade 背景的人会误以为每 Emitter 有独立 tick 入口

## 开放问题 / 待深入

- `bSimulationStagesEnabled` vs `bDeprecatedShaderStagesEnabled`:4.26 里同时存在新旧两套机制,后者在新版已移除;实际使用看不到"deprecated" 字样为何还在
- `MergeChangesFromParent`:继承关系到底能传播什么?renderers? scripts? 只有源图?这需要读 `INiagaraMergeManager` 实现才能确认
- `SharedEventGeneratorIds`(L624):事件生成器在 Spawn/Update 脚本间共享的 Id;具体"共享"是什么语义?
- `AttributesToPreserve`(L300)和 `UNiagaraSystem::bTrimAttributes`(Phase 1 已见)协作,Phase 4/5 看 DataSet 编译路径时再确认细节
