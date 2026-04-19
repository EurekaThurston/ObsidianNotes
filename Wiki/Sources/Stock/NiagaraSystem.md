---
type: source
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, source-code, asset, system]
sources: 1
aliases: [NiagaraSystem.h, UNiagaraSystem 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraSystem.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraSystem.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraSystem.h`
- **快照**: commit `b6ab0dee9`(分支 `Eureka` / 基于 4.26)
- **同仓协作文件**: `NiagaraSystem.cpp`、`NiagaraEmitterHandle.h`、`NiagaraEmitter.h`、`NiagaraScript.h`、`NiagaraEffectType.h`、`NiagaraSystemInstance.h`(运行时)
- **Ingest 日期**: 2026-04-19
- **学习阶段**: Phase 1 — 资产层 1.1

## 职责 / 这个文件干什么的

定义 `UNiagaraSystem`——**Niagara 特效的顶层资产类**(`.uasset` 文件)。一个 System 是若干 Emitter 的容器,持有它们的 handle、System 级别的 Spawn/Update 脚本、预计算的编译产物、可扩展性(scalability)与预热(warmup)配置,以及暴露给外部的参数。

从使用者视角:在内容浏览器里右键 "Create → Niagara System",产生的就是 `UNiagaraSystem` 资产。运行时由 `UNiagaraComponent` 引用它来驱动 `FNiagaraSystemInstance`。

## 关键类型 / 函数 / 宏

### 主类

- `UNiagaraSystem : public UFXSystemAsset`(L190):顶层资产类,`UCLASS(BlueprintType)`
  - 继承 `UFXSystemAsset`(与 Cascade 的 `UParticleSystem` 共享的 UE 通用特效资产基类),印证了 Niagara 没完全抛开 Cascade 遗产

### 配套结构体

- `FNiagaraEmitterCompiledData`(L26):**每 Emitter** 的编译产物,记录了脚本读写的粒子属性、绑定变量(EmitterAge、RandomSeed 等)、粒子 DataSet 的布局
- `FNiagaraSystemCompiledData`(L111):**System 级** 的编译产物,含 `InstanceParamStore`、三份 DataSet CompiledData(System/SpawnParams/UpdateParams)、四类参数到 DataSet 的绑定集合(Global/System/Owner/Emitter)
- `FNiagaraParameterDataSetBinding`(L69) + `FNiagaraParameterDataSetBindingCollection`(L81):编译期产生的"参数内存偏移映射表"——这是 Niagara **脚本执行时不需要字符串查找** 的根本机制
- `FEmitterCompiledScriptPair`(L147) + `FNiagaraSystemCompileRequest`(L160):异步编译流水线的任务包
- `FNiagaraEmitterExecutionIndex`(L178):用 bitfield 压缩的"执行顺序 + 是否开启新 overlap 组"标志;`bStartNewOverlapGroup` 用于数据依赖时强制 Emitter 之间不并行

### 核心字段(protected)

- `TArray<FNiagaraEmitterHandle> EmitterHandles`(L549):**System 的核心状态** —— 所有 Emitter 的引用包装
- `UNiagaraScript* SystemSpawnScript / SystemUpdateScript`(L566, L571):System 级的两个脚本(Emitter 级脚本放在 `UNiagaraEmitter` 里)
- `FNiagaraUserRedirectionParameterStore ExposedParameters`(L582):暴露给外界(BP/Sequencer)的"User Parameters"
- `UNiagaraEffectType* EffectType`(L536):scalability 预算归属,见 Phase 9
- `FBox FixedBounds`(L600)+ `bFixedBounds`(L478):可选的固定包围盒;跳过每帧包围盒计算
- `WarmupTime / WarmupTickCount / WarmupTickDelta`(L607-L615):开场预热,快进 N 步再展示
- `TArray<FNiagaraEmitterExecutionIndex> EmitterExecutionOrder`(L626):预计算的 Emitter 执行顺序(考虑依赖)
- `TArray<int32> RendererDrawOrder`(L629):渲染器排序

### 关键公共接口(选)

- `GetEmitterHandles()`(L221):取所有 Emitter handle,只读/可写双重载
- `GetSystemSpawnScript() / GetSystemUpdateScript()`(L270-L271)
- `ForEachScript(TAction)`(L279, L697 实现):遍历 System 脚本 + 所有 Emitter 脚本;模板方法,编辑器很多地方靠它
- `AddEmitterHandle / DuplicateEmitterHandle / RemoveEmitterHandle`(L233-L243,仅 `WITH_EDITORONLY_DATA`):在编辑器里增删 Emitter
- `RequestCompile / PollForCompilationComplete / WaitForCompilationComplete`(L312-L318):异步编译入口(仅 editor)
- `ComputeEmitterPriority / FindDataInterfaceDependencies / FindEventDependencies / ComputeEmittersExecutionOrder`(L407-L416):依赖分析 + 执行顺序求解
- `CacheFromCompiledData()`(L422):把编译数据缓存成运行时可直接用的 accessor,避免每 instance 重复查找

### 性能/归档相关开关

- `bBakeOutRapidIteration / bBakeOutRapidIterationOnCook`(L385-L389):是否在编译/烘焙时把 RapidIteration 参数烘死
- `bCompressAttributes`(L394):粒子属性是否压缩(精度换内存/带宽)
- `bTrimAttributes / bTrimAttributesOnCook`(L397-L402):剔除 VM 从不读的属性
- `bRequireCurrentFrameData`(L452):决定 Tick 依赖策略(影响 TickGroup)

### 编辑器/烘焙分支

头文件到处是 `#if WITH_EDITOR` / `#if WITH_EDITORONLY_DATA`。运行时版本(shipping)里:
- 无异步编译、无 merge、无 EditorData
- `bIsValidCached / bIsReadyToRunCached` 取代重新计算(L631-L632)——cooked 数据路径

## 依赖与被依赖

**上游(此文件 include / 使用):**
- `NiagaraCommon.h`(类型系统、枚举)
- `NiagaraDataSet.h`(粒子数据存储)
- `NiagaraEmitterHandle.h`(`TArray<FNiagaraEmitterHandle>`)
- `NiagaraEmitterInstance.h`(运行时 include 进来是反模式之一,实际代码里是为了 `TSharedRef` 的前向声明便利)
- `NiagaraParameterCollection.h`、`NiagaraUserRedirectionParameterStore.h`、`NiagaraEffectType.h`
- `Particles/ParticleSystem.h`、`Particles/ParticlePerfStats.h`(UFXSystemAsset 基类链)

**下游(谁用它):**
- `NiagaraComponent`:持有 `UNiagaraSystem*`,激活时创建 SystemInstance
- `NiagaraSystemInstance`:运行时镜像,直接读 System 的 `EmitterCompiledData` / `SystemCompiledData`
- `NiagaraFunctionLibrary::SpawnSystemAt* / SpawnSystemAttached`:BP 入口,参数就是 `UNiagaraSystem*`
- `NiagaraEffectType`:scalability 规则依赖 System 归类
- 编辑器:几乎所有的 Emitter 面板、Stack 面板最终都在编辑这个 System 里的 `EmitterHandles`

## 关键代码片段

> `NiagaraSystem.h:L188-L192` @ `b6ab0dee9` — 类声明与继承
> ```cpp
> /** Container for multiple emitters that combine together to create a particle system effect.*/
> UCLASS(BlueprintType)
> class NIAGARA_API UNiagaraSystem : public UFXSystemAsset
> {
>     GENERATED_UCLASS_BODY()
> ```

> `NiagaraSystem.h:L547-L549` @ `b6ab0dee9` — System 最核心的那一行
> ```cpp
> /** Handles to the emitter this System will simulate. */
> UPROPERTY()
> TArray<FNiagaraEmitterHandle> EmitterHandles;
> ```

> `NiagaraSystem.h:L697-L710` @ `b6ab0dee9` — `ForEachScript` 体现 System 脚本树的形状
> ```cpp
> template<typename TAction>
> void UNiagaraSystem::ForEachScript(TAction Func) const
> {
>     Func(SystemSpawnScript);
>     Func(SystemUpdateScript);
>     for (const FNiagaraEmitterHandle& Handle : EmitterHandles)
>     {
>         if (UNiagaraEmitter* Emitter = Handle.GetInstance())
>         {
>             Emitter->ForEachScript(Func);
>         }
>     }
> }
> ```
> 一个 System 的完整脚本树 = 2 个 System 脚本 + ∑(每个 Emitter 的 N 个脚本)

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/UNiagaraSystem]] — 本文件定义的主类
- [[Wiki/Entities/Stock/FNiagaraEmitterHandle]] — `EmitterHandles` 元素类型
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — 被 handle 引用的 Emitter
- [[Wiki/Entities/Stock/UNiagaraScript]] — SystemSpawn/SystemUpdate 脚本类型
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — System 是 Asset,Instance 见 Phase 3
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — `UFXSystemAsset` 基类同源于 Cascade

## 与既有 wiki 的关系

- 印证了 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] 的三层架构图:System → EmitterHandle → Emitter → Script
- 印证了 [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] 里 "SystemSpawnScript / SystemUpdateScript 对应 Niagara 的固定执行阶段"
- 新增认识:Niagara 的 "无字符串查表" 性能优势,核心实现就是 `FNiagaraSystemCompiledData`(L111)里预计算的四类 binding collection

## 开放问题 / 待深入

- `ForEachScript` 里只对 System 脚本和 **Enabled Emitter** 的脚本调用?实际看起来遍历所有 handle 的 `GetInstance()`——disabled 的也会进去,只是运行时不会跑。这和 `FNiagaraEmitterHandle::bIsEnabled` 的真实生效点需要在 Phase 3(SystemInstance)时再确认
- `EmitterExecutionOrder` 里 `kStartNewOverlapGroupBit`(最高位) 的意图很清楚,但实际 parallel emitter tick 在哪里读取这个标志?在 Phase 3 / Phase 9 里看 `FNiagaraSystemSimulation` 时再追
- `bBakeOutRapidIteration` 的 "RapidIteration parameters" 是什么?按命名应该是"可以快速迭代调整的实时参数",在 Phase 5 读 `NiagaraScriptExecutionParameterStore` 时会明朗
- `HasSystemScriptDIsWithPerInstanceData()` / `UpdateDITickFlags()` 涉及 DataInterface 的 per-instance 数据,留给 Phase 7
