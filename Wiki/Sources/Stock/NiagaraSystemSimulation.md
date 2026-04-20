---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, runtime, simulation, batched-tick]
sources: 1
aliases: [NiagaraSystemSimulation.h, FNiagaraSystemSimulation 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSystemSimulation.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraSystemSimulation.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSystemSimulation.h`
- **快照**: commit `b6ab0dee9`(分支 `Eureka` / 基于 4.26)
- **文件规模**: 429 行
- **同仓协作文件**: `NiagaraSystemSimulation.cpp`、`NiagaraSystemInstance.h`、`NiagaraWorldManager.h`(Phase 9,持有 Simulation map)、`NiagaraScriptExecutionContext.h`(Phase 5)、`NiagaraDataSet.h`(Phase 4)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 3 — 运行时实例层 3.3

## 职责 / 这个文件干什么的

定义 `FNiagaraSystemSimulation`——**同一个 `UNiagaraSystem` 资产在一个 `UWorld`+`ETickingGroup` 下所有实例的批量 Tick 调度器**。回答 Phase 2 反复提到的"批量 Tick 做了什么"这个问题。

设计哲学:Niagara 假设**同 Asset 多实例在场景里很常见**(50 个爆炸、100 个子弹拖尾),批量处理把"System 级脚本执行 + 参数 → DataSet 拉取 + emitter ticktask 调度"合并成一次流水线。

一个 `FNiagaraSystemSimulation` 的身份 = (Asset, World, TickGroup)。`FNiagaraWorldManager`(Phase 9)按这个三元组索引。Solo 实例是退化情形——每个 solo Instance 各自一个 1 成员的 Simulation。

本文件还定义两个关键辅助结构体,供 Simulation 用:
- `FNiagaraParameterStoreToDataSetBinding`(L33-L192):**参数存储 → DataSet 的预计算偏移映射表** —— Phase 1 `FNiagaraSystemCompiledData` 里的 binding 在这里被消费
- `FNiagaraSystemSimulationTickContext`(L207-L236):Tick 一轮所需的上下文打包

继承 `FGCObject` 因为它持有 UObject 引用(Asset、EffectType)但自身非 UObject,需要显式注册 GC。

## 关键类型 / 函数 / 宏

### 主类

- `class FNiagaraSystemSimulation : public TSharedFromThis<FNiagaraSystemSimulation, ESPMode::ThreadSafe>, FGCObject`(L239-L241):
  - `TSharedFromThis` 给 SystemInstance 能安全持 `TSharedPtr` 回指
  - `FGCObject` 让 UE GC 看到它持有的 UObject 引用
  - `friend FNiagaraSystemSimulationTickContext`

### 配套枚举

- `enum class ENiagaraGPUTickHandlingMode`(L17-L24):GPU Tick 的 5 种调度策略
  - `None` — 无 GPU tick
  - `GameThread` — 每个 system 在 GT 单独提交
  - `Concurrent` — 每个 system 在 concurrent tick 单独提交
  - `GameThreadBatched` — 批量提交,GT
  - `ConcurrentBatched` — 批量提交,concurrent
  - `GetGPUTickHandlingMode() const`(L308):运行时决策

### Tick Batch

```cpp
// NiagaraSystemSimulation.h:28-29
#define NiagaraSystemTickBatchSize 4
typedef TArray<FNiagaraSystemInstance*, TInlineAllocator<NiagaraSystemTickBatchSize>> FNiagaraSystemTickBatch;
```

**4 个实例打包成一批**,`TInlineAllocator<4>` 小批时免堆分配。TODO 注释写着"如果能根据 system 平均执行时间动态调整 batch size 会更好"。

- `FNiagaraSystemTickBatch TickBatch`(L418):当前在填的 batch
- `void AddSystemToTickBatch(FNiagaraSystemInstance*, FNiagaraSystemSimulationTickContext&)`(L332,protected)
- `void FlushTickBatch(FNiagaraSystemSimulationTickContext&)`(L333,protected)

### 辅助结构体 1:`FNiagaraParameterStoreToDataSetBinding`(L33-L192)

这是 Niagara 性能秘诀的**具体消费点**——编译期算好的"ParameterStore 中每个值 → DataSet 中该值的内存偏移",运行时直接按偏移 memcpy,不查字符串/FName。

字段:
- `struct FDataOffsets { int32 ParameterOffset; int32 DataSetComponentOffset; }`(L36-L43)
- `struct FHalfDataOffsets : public FDataOffsets { bool ApplyAsFloat; }`(L44-L48):half float 要不要当 float 写
- `TArray<FDataOffsets> FloatOffsets / Int32Offsets`(L50-L51)
- `TArray<FHalfDataOffsets> HalfOffsets`(L52)

核心方法:
- `void Init(FNiagaraDataSet&, const FNiagaraParameterStore&)`(L61-L111):初始化时遍历 DataSet 的变量布局,为每个变量算出在 ParameterStore 里的 offset 和在 DataSet 的 component offset。这段代码完整展示了"从 `UNiagaraSystem::SystemCompiledData` 到运行时 offset 表"的转换。
- `void DataSetToParameterStore(FNiagaraParameterStore&, FNiagaraDataSet&, int32 DataSetInstanceIndex)`(L113-L149):**反向**——把一份 DataSet instance 的数据写回 ParameterStore(Simulation 结果扔回参数)
- `void ParameterStoreToDataSet(const FNiagaraParameterStore&, FNiagaraDataSet&, int32 DataSetInstanceIndex)`(L151-L191):**正向**——把 ParameterStore 数据灌进 DataSet 的一个 instance(Simulation 前的输入)

两个方向用 `FORCEINLINE_DEBUGGABLE`,性能热点。`NIAGARA_NAN_CHECKING` 宏开启时会全程查 NaN。

### 辅助结构体 2:`FNiagaraConstantBufferToDataSetBinding`(L194-L205)

把 `FNiagaraSystemCompiledData` 里的绑定表(`FNiagaraParameterDataSetBindingCollection`)拷到 DataSet——前面那个结构的特化版。

- `static void CopyToDataSets(const FNiagaraSystemCompiledData&, const FNiagaraSystemInstance&, FNiagaraDataSet& SpawnDataSet, FNiagaraDataSet& UpdateDataSet, int32 DataSetInstanceIndex)`:一次性把 System Instance 的所有参数灌进 spawn + update 两份 DataSet
- `protected: static void ApplyOffsets(const FNiagaraParameterDataSetBindingCollection&, const uint8*, FNiagaraDataSet&, int32)`

### 辅助结构体 3:`FNiagaraSystemSimulationTickContext`(L207-L236)

一次 Tick 的所有上下文打包成一个 context 对象:

```cpp
class FNiagaraSystemSimulation*   Owner;
UNiagaraSystem*                   System;
TArray<FNiagaraSystemInstance*>&  Instances;      // 引用要 tick 的实例列表
FNiagaraDataSet&                  DataSet;        // 用哪个 DataSet
float                             DeltaSeconds;
int32                             SpawnNum;       // 本轮要 spawn 多少
int                               EffectsQuality; // 效果质量层
FGraphEventRef                    MyCompletionGraphEvent;
FGraphEventArray*                 FinalizeEvents;
bool                              bTickAsync;     // 整体是否异步
bool                              bTickInstancesAsync;
```

构造走工厂:
- `static FNiagaraSystemSimulationTickContext MakeContextForTicking(...)`(L217)
- `static FNiagaraSystemSimulationTickContext MakeContextForSpawning(...)`(L218)

### 构造与初始化

- `FNiagaraSystemSimulation()` / `~FNiagaraSystemSimulation()`(L248-L249)
- `bool Init(UNiagaraSystem*, UWorld*, bool bInIsSolo, ETickingGroup)`(L250):**主 init**——确定身份(Asset / World / Solo / TickGroup)
- `void Destroy()`(L251)
- `bool IsValid() const`(L253):WeakSystem 还有效 + bCanExecute + World 非空
- `virtual void AddReferencedObjects(FReferenceCollector&) override`(L245):FGCObject 接口,告诉 GC "我持有这些 UObject"

### Tick 两阶段(System Simulation 级)

```cpp
// NiagaraSystemSimulation.h:256-258
void Tick_GameThread(float DeltaSeconds, const FGraphEventRef& MyCompletionGraphEvent);
void Tick_Concurrent(FNiagaraSystemSimulationTickContext& Context);
```

注意 Simulation 级只有 **2 阶段**(GT/Concurrent),Finalize 在 Instance 级做。

辅助:
- `void UpdateTickGroups_GameThread()`(L261):收集哪些 instance 要换 TickGroup,走 promotion
- `void Spawn_GameThread(float, bool bPostActorTick)` / `Spawn_Concurrent(FNiagaraSystemSimulationTickContext&)`(L263, L265):额外的 spawn 阶段,处理不在常规 tick 里的生成(level streaming)
- `void WaitForSystemTickComplete(bool bEnsureComplete = false)`(L270):GT 等 Simulation 级 tick 结束
- `void WaitForInstancesTickComplete(bool bEnsureComplete = false)`(L272):GT 等所有 instance 的 tick 结束

### Instance 管理

- `void AddInstance(FNiagaraSystemInstance*)` / `RemoveInstance(FNiagaraSystemInstance*)`(L274-L275)
- `void PauseInstance(FNiagaraSystemInstance*)` / `UnpauseInstance(FNiagaraSystemInstance*)`(L277-L278)
- `void TransferInstance(FNiagaraSystemSimulation* Source, FNiagaraSystemInstance*)`(L287):在 Simulation 之间迁移(比如 Solo ↔ Batched 切换)
- `void AddTickGroupPromotion(FNiagaraSystemInstance*)`(L299):标记 instance 换 TickGroup
- `int32 AddPendingSystemInstance(FNiagaraSystemInstance*)`(L300):添加到 pending 队列

**实例数组分类**(L393-L403):

```cpp
TArray<FNiagaraSystemInstance*> SystemInstances;          // 正在模拟
TArray<FNiagaraSystemInstance*> SpawningInstances;        // 将要在非常规 tick spawn 的
TArray<FNiagaraSystemInstance*> PausedSystemInstances;    // 暂停中
TArray<FNiagaraSystemInstance*> PendingSystemInstances;   // 等待 spawn
TArray<FNiagaraSystemInstance*> PendingTickGroupPromotions;
```

### 多份 DataSet(按用途分开)

```cpp
// NiagaraSystemSimulation.h:348-360
FNiagaraDataSet MainDataSet;                      // 常规 tick
FNiagaraDataSet SpawningDataSet;                  // 额外 spawn(tick 外)
FNiagaraDataSet PausedInstanceData;               // 暂停实例
FNiagaraDataSet SpawnInstanceParameterDataSet;    // System 参数聚合(spawn 输入)
FNiagaraDataSet UpdateInstanceParameterDataSet;   // System 参数聚合(update 输入)
```

### Script 执行上下文

- `TUniquePtr<FNiagaraScriptExecutionContextBase> SpawnExecContext`(L362):System Spawn 脚本
- `TUniquePtr<FNiagaraScriptExecutionContextBase> UpdateExecContext`(L363):System Update 脚本
- `FNiagaraScriptExecutionContextBase* GetSpawnExecutionContext()`(L296)
- `FNiagaraScriptExecutionContextBase* GetUpdateExecutionContext()`(L297)
- `static bool UseLegacySystemSimulationContexts()` / `OnChanged_UseLegacySystemSimulationContexts(IConsoleVariable*)`(L311-L312):历史兼容 CVar

### 绑定(多组)

```cpp
// NiagaraSystemSimulation.h:366-379
FNiagaraParameterStoreToDataSetBinding SpawnInstanceParameterToDataSetBinding;
FNiagaraParameterStoreToDataSetBinding UpdateInstanceParameterToDataSetBinding;

TArray<FNiagaraParameterStoreToDataSetBinding> DataSetToEmitterSpawnParameters;      // System→Emitter Spawn
TArray<FNiagaraParameterStoreToDataSetBinding> DataSetToEmitterUpdateParameters;     // System→Emitter Update
TArray<TArray<FNiagaraParameterStoreToDataSetBinding>> DataSetToEmitterEventParameters;  // System→Emitter 每个事件
TArray<FNiagaraParameterStoreToDataSetBinding> DataSetToEmitterGPUParameters;        // System→Emitter GPU
TArray<FNiagaraParameterStoreToDataSetBinding> DataSetToEmitterRendererParameters;   // System→Emitter Renderer
```

System 脚本算完的结果通过这些 binding 派发到各个 Emitter——就是 "Emitter 之间能读 System attribute" 的运行时机制。

### Direct Bindings(快速访问)

```cpp
// NiagaraSystemSimulation.h:383-390
FNiagaraParameterDirectBinding<int32> SpawnNumSystemInstancesParam;
FNiagaraParameterDirectBinding<int32> UpdateNumSystemInstancesParam;

FNiagaraParameterDirectBinding<float> SpawnGlobalSpawnCountScaleParam;
FNiagaraParameterDirectBinding<float> UpdateGlobalSpawnCountScaleParam;

FNiagaraParameterDirectBinding<float> SpawnGlobalSystemCountScaleParam;
FNiagaraParameterDirectBinding<float> UpdateGlobalSystemCountScaleParam;
```

Engine-level 参数(当前场景里有多少 system / scale),给 script 直接访问。

### 主 Tick 的内部阶段(protected)

```cpp
// NiagaraSystemSimulation.h:315-330
void SetupParameters_GameThread(float DeltaSeconds);           // GT 阶段,刷常量
void PrepareForSystemSimulate(...);                             // 把 instance 参数拉到 DataSet
void SpawnSystemInstances(...);                                 // 跑 System Spawn 脚本
void UpdateSystemInstances(...);                                // 跑 System Update 脚本
void TransferSystemSimResults(...);                             // 把结果派发到 Emitter
void BuildConstantBufferTable(...);                             // 构建脚本常量表
```

这是 System 级脚本执行的完整 pipeline。

### 其他

- `TWeakObjectPtr<UNiagaraSystem> WeakSystem`(L336):弱引用,GC 后不阻止析构
- `UNiagaraEffectType* EffectType`(L339):缓存,GC 仍然 reachable(见 `AddReferencedObjects`)
- `ETickingGroup SystemTickGroup = TG_MAX`(L342)
- `UWorld* World`(L345)
- `FNiagaraParameterStore ScriptDefinedDataInterfaceParameters`(L413)
- `TOptional<float> MaxDeltaTime`(L415)
- `FGraphEventRef SystemTickGraphEvent`(L421):当前运行的 tick 任务
- `NiagaraEmitterInstanceBatcher* Batcher`(L425):GPU 批处理器(Phase 8)
- `GetBatcher() const`(L306)
- `mutable FString CrashReporterTag`(L423) / `GetCrashReporterTag() const`(L302)

### Flag

```cpp
uint32 bCanExecute : 1;                   // L407
uint32 bBindingsInitialized : 1;
uint32 bInSpawnPhase : 1;
uint32 bIsSolo : 1;
static bool bUseLegacyExecContexts;       // L427
```

### Parameter Collection

- `UNiagaraParameterCollectionInstance* GetParameterCollectionInstance(UNiagaraParameterCollection*)`(L282):跨 Simulation 共享的 parameter collection

### 调试

- `void DumpInstance(const FNiagaraSystemInstance*) const`(L289)
- `void DumpTickInfo(FOutputDevice&)`(L292)

## 依赖与被依赖

**上游:**
- `Engine/World.h`、`UObject/GCObject.h`(FGCObject 基类)
- `NiagaraParameterCollection.h`、`NiagaraDataSet.h`、`NiagaraScriptExecutionContext.h`、`NiagaraSystem.h`

**下游:**
- `FNiagaraWorldManager`(Phase 9,按 (Asset, TickGroup) 索引持有一批 Simulation)
- `FNiagaraSystemInstance`(通过 `SystemSimulation` 指针反向引用)

## 关键代码片段

> `NiagaraSystemSimulation.h:L28-L29` @ `b6ab0dee9` — Tick batch 核心常量
> ```cpp
> #define NiagaraSystemTickBatchSize 4
> typedef TArray<FNiagaraSystemInstance*, TInlineAllocator<NiagaraSystemTickBatchSize>> FNiagaraSystemTickBatch;
> ```

> `NiagaraSystemSimulation.h:L239-L242` @ `b6ab0dee9` — 继承 / 声明
> ```cpp
> class FNiagaraSystemSimulation
>     : public TSharedFromThis<FNiagaraSystemSimulation, ESPMode::ThreadSafe>, FGCObject
> {
>     friend FNiagaraSystemSimulationTickContext;
> ```

> `NiagaraSystemSimulation.h:L36-L43` @ `b6ab0dee9` — 参数 offset 的核心结构
> ```cpp
> struct FDataOffsets
> {
>     int32 ParameterOffset;       // 在 ParameterStore 的 offset
>     int32 DataSetComponentOffset; // 在 DataSet component 的 offset
> };
> ```

> `NiagaraSystemSimulation.h:L315-L325` @ `b6ab0dee9` — System 级 Tick 的内部 pipeline
> ```cpp
> void SetupParameters_GameThread(float DeltaSeconds);
> void PrepareForSystemSimulate(FNiagaraSystemSimulationTickContext& Context);
> void SpawnSystemInstances(FNiagaraSystemSimulationTickContext& Context);
> void UpdateSystemInstances(FNiagaraSystemSimulationTickContext& Context);
> void TransferSystemSimResults(FNiagaraSystemSimulationTickContext& Context);
> ```

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/FNiagaraSystemSimulation]] — 本文件定义的主类
- [[Wiki/Entities/Stock/FNiagaraSystemInstance]] — 被调度的 instance
- [[Wiki/Entities/Stock/UNiagaraSystem]] — Asset,Simulation 身份的一部分
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — Niagara 为性能牺牲黑盒模块化的典型产物

## 开放问题 / 待深入

- **`NiagaraSystemTickBatchSize=4` 的选择依据**:不是 8 不是 16?cache line 大小?TaskGraph 调度开销?→ cpp 可能有 profiling 注释
- **`bInSpawnPhase` 怎么影响 Tick**:spawn 和 update 是同一个 tick 里分开两次,还是不同帧?→ 看 `Tick_GameThread` 实现
- **`bUseLegacyExecContexts` 的现状**:legacy 模式下不能 per-instance DI 调用,全部强制 solo——什么情况下还要用?→ CVar 历史/向后兼容
- **`DataSetToEmitterEventParameters` 嵌套 TArray**:外层是 emitter,内层是事件?还是反过来?→ `.cpp` 确认
- **`FGCObject::AddReferencedObjects` 都加了什么**:WeakSystem 不需要(weak),EffectType 肯定要,还有什么?→ `.cpp`
- **Solo 与 Batched 切换(`TransferInstance`)的触发场景**:用户改 `bForceSolo`、CVar 变化、legacy 模式?→ 与 Phase 2 `SetForceSolo` 对接确认
