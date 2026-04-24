---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, runtime, emitter-instance, particle-simulation]
sources: 1
aliases: [NiagaraEmitterInstance.h, FNiagaraEmitterInstance 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterInstance.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraEmitterInstance.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterInstance.h`
- **快照**: commit `b6ab0dee9`(分支 `Eureka` / 基于 4.26)
- **文件规模**: 239 行
- **同仓协作文件**: `NiagaraEmitterInstance.cpp`、`NiagaraSystemInstance.h`(持有者)、`NiagaraScriptExecutionContext.h`(持有 ExecContext,Phase 5)、`NiagaraDataSet.h`(持有粒子数据,Phase 4)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 3 — 运行时实例层 3.2

## 职责 / 这个文件干什么的

定义 `FNiagaraEmitterInstance`——**单个 Emitter 在一个 System Instance 下的运行时对象**。如果 `UNiagaraEmitter`(Phase 1)是 "Emitter 资产",`FNiagaraEmitterInstance` 就是 "正在跑的 Emitter 实例",负责:

1. 持有当前活跃粒子数据(`FNiagaraDataSet* ParticleDataSet`)
2. 持有 Spawn/Update 脚本的执行上下文(`SpawnExecContext / UpdateExecContext`),或 GPU 情况下的计算上下文(`GPUExecContext`)
3. 执行每帧的 `PreTick / Tick / PostTick / HandleCompletion`
4. 计算 spawn 数量、派发 event、管理 bounds、cache 统计

也**非 UObject**——和 `FNiagaraSystemInstance` 同源设计,生命周期由 `TSharedRef` 管(其实就是 SystemInstance 的 `Emitters` 数组)。

> [!warning] 命名陷阱提醒
> `FNiagaraEmitterInstance`(本类,Phase 3,**运行时**)**不是** `FNiagaraEmitterHandle::Instance`(Phase 1,**Asset 层 Emitter 资产副本**)。两者同名但位于不同层。Phase 1 读本已专门强调,这里再强调一次。

## 关键类型 / 函数 / 宏

### 主类

- `class FNiagaraEmitterInstance`(L25):非 UObject,非 NIAGARA_API 整个类(部分方法单独标 API)

### 内嵌结构

- `FEventInstanceData`(L28-L42,私有 struct):Event 相关数据打包
  - `TArray<FNiagaraScriptExecutionContext> EventExecContexts`:事件脚本各自一个 ExecContext
  - `TArray<FNiagaraParameterDirectBinding<int32>> EventExecCountBindings`:事件执行 count 绑定
  - `TArray<FNiagaraDataSet*> UpdateScriptEventDataSets / SpawnScriptEventDataSets`:事件 DataSet
  - `TArray<bool> UpdateEventGeneratorIsSharedByIndex / SpawnEventGeneratorIsSharedByIndex`
  - `TArray<FNiagaraEventHandlingInfo> EventHandlingInfo`:event handler 的运行时状态
  - `int32 EventSpawnTotal = 0`
  - 存放在 `TUniquePtr<FEventInstanceData> EventInstanceData`(L184,只在有事件时分配)

### 构造与生命周期

- `explicit FNiagaraEmitterInstance(FNiagaraSystemInstance* InParentSystemInstance)`(L45):构造需要 System 指针
- `virtual ~FNiagaraEmitterInstance()`(L46):虚析构
- `void Init(int32 InEmitterIdx, FNiagaraSystemInstanceID SystemInstanceID)`(L48):初始化 —— 在 parent SystemInstance 里的索引 + System ID
- `void ResetSimulation(bool bKillExisting = true)`(L50):重置模拟,默认杀现有粒子
- `void OnPooledReuse()`(L52):池复用时调
- `void DirtyDataInterfaces()`(L54):DI 状态变化通知

### 三阶段 Tick(Emitter 级)

```cpp
// NiagaraEmitterInstance.h:63-66
void PreTick();                  // 准备阶段:事件 count 计算等
void Tick(float DeltaSeconds);   // 主体:执行 Spawn/Update 脚本,更新粒子
void PostTick();                 // 收尾:bounds 计算等
bool HandleCompletion(bool bForce = false);  // 判断是否完成
```

Emitter 的 Tick 在 System Instance 的 `Tick_Concurrent` 里被调用(一批批 emitter 一起算)。

- `bool ShouldTick() const`(L70):`ExecutionState == Active || NumParticles > 0`
- `bool IsAllowedToExecute() const`(L61):有资格跑吗(不被 scalability cull 等)
- `uint32 CalculateEventSpawnCount(const FNiagaraEventScriptProperties&, TArray<int32>&, FNiagaraDataSet*)`(L72):事件触发的 spawn 数

### 状态机

- `ENiagaraExecutionState ExecutionState = Inactive`(L232):**单一状态**(比 SystemInstance 的双状态简单)
- `GetExecutionState() / SetExecutionState()`(L120-L121)
- `IsDisabled / IsInactive / IsComplete`(L83-L85)

### 粒子数据访问

- `FNiagaraDataSet* ParticleDataSet = nullptr`(L201):**核心字段**——粒子数据(Phase 4 的 `FNiagaraDataSet` 主角)
- `FNiagaraDataSet& GetData() const`(L81)
- `int32 GetNumParticles() const`(L93-L108):**特殊**——GPU 情况下从 GPUExecContext 读,否则从 DataSet 读
  ```cpp
  if (GPUExecContext) { return GetNumParticlesGPUInternal(); }
  if (ParticleDataSet->GetCurrentData()) { return ParticleDataSet->GetCurrentData()->GetNumInstances(); }
  return 0;
  ```
  注释承认 GPU 数据有 latency,"读到的 count 可能不对,但对 system script update 够用"
- `int32 GetNumParticlesGPUInternal()`(L91,私有):GPU 专用路径
- `int32 GetTotalSpawnedParticles() const`(L110) / `int32 TotalSpawnedParticles`(L218):累计 spawn 数

### 脚本执行上下文(Phase 5 预告)

```cpp
// NiagaraEmitterInstance.h:170-172
FNiagaraScriptExecutionContext SpawnExecContext;     // CPU 的 Spawn 脚本
FNiagaraScriptExecutionContext UpdateExecContext;    // CPU 的 Update 脚本
FNiagaraComputeExecutionContext* GPUExecContext = nullptr;  // GPU 情况下用这个(指针,非 owned)
```

访问:
- `FNiagaraScriptExecutionContext& GetSpawnExecutionContext()`(L125)
- `FNiagaraScriptExecutionContext& GetUpdateExecutionContext()`(L126)
- `TArrayView<FNiagaraScriptExecutionContext> GetEventExecutionContexts()`(L127):来自 `EventInstanceData`
- `FNiagaraComputeExecutionContext* GetGPUContext() const`(L140)

### Direct Bindings(Phase 5 参数绑定)

```cpp
FNiagaraParameterDirectBinding<float> SpawnIntervalBinding;        // L174
FNiagaraParameterDirectBinding<float> InterpSpawnStartBinding;
FNiagaraParameterDirectBinding<int32> SpawnGroupBinding;
FNiagaraParameterDirectBinding<int32> SpawnExecCountBinding;
FNiagaraParameterDirectBinding<int32> UpdateExecCountBinding;
```

这些是参数存储到执行上下文的"直接绑定"——不走字符串查表,直接指针引用。典型的 Niagara 编译期绑定思路。

### Spawn Info

- `TArray<FNiagaraSpawnInfo> SpawnInfos`(L168):当前帧所有 spawn 调度(每个 source 一条:rate-based / burst / event-based)
- `TArray<FNiagaraSpawnInfo>& GetSpawnInfo()`(L132)

### 参数存储

- `FNiagaraParameterStore RendererBindings`(L182):**给 Renderer 读的参数**——Renderer 每帧从这里取它需要的属性(Phase 6)
- `const FNiagaraParameterStore& GetRendererBoundVariables() const`(L152-L153):两个重载
- `FNiagaraParameterStore ScriptDefinedDataInterfaceParameters`(L187):脚本里定义的 DI 参数
- `BindParameters(bool bExternalOnly)` / `UnbindParameters(bool bExternalOnly)`(L58-L59)
- `bool GetBoundRendererValue_GT(const FNiagaraVariableBase&, const FNiagaraVariableBase&, void*) const`(L79):从 RendererBindings 里取一个值(GT 上下文)

### Material Binding

- `bool FindBinding(const FNiagaraUserParameterBinding&, TArray<UMaterialInterface*>&) const`(L147):查 User 参数里的材质绑定

### Bounds

- `FBox GetBounds()`(L123):外部取
- `FBox CachedBounds`(L190):计算后缓存
- `TOptional<FBox> CachedSystemFixedBounds`(L194):System 的 fixed bounds 优先
- `void SetSystemFixedBoundsOverride(FBox)`(L145)
- `FBox InternalCalculateDynamicBounds(int32 ParticleCount) const`(L165,私有)
- `#if WITH_EDITOR void CalculateFixedBounds(const FTransform& ToWorldSpace)`(L76):**editor 专用,会引发 GPU readback stall**

### Persistent ID / Random

- `bool RequiresPersistentIDs() const`(L68):粒子需要唯一 ID 吗(Emitter 资产设置)
- `int32 InstanceSeed = FGenericPlatformMath::Rand()`(L215):随机种子(本 Instance 专属)
- `int32 GetInstanceSeed() const`(L155)

### 统计 / 调试

- `float GetTotalCPUTimeMS()`(L117):cycles → ms
- `int GetTotalBytesUsed()`(L118)
- `uint32 CPUTimeCycles = 0`(L221):累计 cycles
- `uint32 MaxRuntimeAllocation / MaxAllocationCount / MinOverallocation / ReallocationCount / MaxInstanceCount`(L223-L229):分配统计
- `void Dump() const`(L136)
- `bool WaitForDebugInfo()`(L138):editor 调试同步

### 其他字段

- `FNiagaraSystemInstance* ParentSystemInstance`(L203):反向指针
- `UNiagaraEmitter* CachedEmitter`(L206):**裸指针**缓存 Asset,注释说"System 级保证 lifetime"
- `FName CachedIDName`(L207)
- `int32 EmitterIdx = INDEX_NONE`(L210):在 ParentSystemInstance->Emitters 里的索引
- `float EmitterAge = 0.0f`(L213)
- `int32 TickCount = 0`(L216)
- `TSharedPtr<const FNiagaraEmitterCompiledData> CachedEmitterCompiledData`(L181):Asset 编译产物(Phase 1 结构体)
- `NiagaraEmitterInstanceBatcher* Batcher`(L198):GPU 批处理器(Phase 8)
- `FNiagaraSystemInstanceID OwnerSystemInstanceID`(L196)
- `const FNiagaraEmitterHandle& GetEmitterHandle() const`(L113):Asset 层的 handle
- `const FNiagaraEmitterScalabilitySettings& GetScalabilitySettings() const`(L111):Phase 9 scalability 条款

### Flag bitfield

```cpp
uint32 bResetPending : 1;          // L235  延迟到 tick 再真正 reset(RT 可能还在读)
uint32 bCombineEventSpawn : 1;     // L237  优化:合并事件 spawn(不安全于 ExecIndex)
```

### 私有方法

- `void CheckForErrors()`(L158)
- `void BuildConstantBufferTable(const FNiagaraScriptExecutionContext&, FScriptExecutionConstantBufferTable&) const`(L160)

## 依赖与被依赖

**上游:**
- `NiagaraCommon.h` / `NiagaraDataSet.h` / `NiagaraEmitterHandle.h` / `NiagaraEmitter.h` / `NiagaraScriptExecutionContext.h`

**下游:**
- `FNiagaraSystemInstance`(持有一组 emitter instance 作为 SharedRef)
- `FNiagaraRenderer`(Phase 6,从 RendererBindings + ParticleDataSet 取数据渲染)
- `NiagaraEmitterInstanceBatcher`(Phase 8,GPU 情况下通过 GPUExecContext 交互)

## 关键代码片段

> `NiagaraEmitterInstance.h:L63-L66` @ `b6ab0dee9` — Emitter 级 Tick 生命周期
> ```cpp
> void PreTick();
> void Tick(float DeltaSeconds);
> void PostTick();
> bool HandleCompletion(bool bForce = false);
> ```

> `NiagaraEmitterInstance.h:L93-L108` @ `b6ab0dee9` — GetNumParticles 的 CPU/GPU 双路径
> ```cpp
> FORCEINLINE int32 GetNumParticles() const
> {
>     if (GPUExecContext)
>     {
>         return GetNumParticlesGPUInternal();
>     }
>     if (ParticleDataSet->GetCurrentData())
>     {
>         return ParticleDataSet->GetCurrentData()->GetNumInstances();
>     }
>     return 0;
> }
> ```

> `NiagaraEmitterInstance.h:L170-L172` @ `b6ab0dee9` — 三路执行上下文
> ```cpp
> FNiagaraScriptExecutionContext SpawnExecContext;
> FNiagaraScriptExecutionContext UpdateExecContext;
> FNiagaraComputeExecutionContext* GPUExecContext = nullptr;
> ```

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/Niagara/FNiagaraEmitterInstance]] — 本文件定义的主类
- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemInstance]] — 持有者
- [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — 对应的 Asset(CachedEmitter)
- [[Wiki/Entities/Stock/Niagara/FNiagaraEmitterHandle]] — Asset 层的 handle
- [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]] — CPU/GPU 分叉就在本类的 ExecContext 分支

## 开放问题 / 待深入

- `FEventInstanceData` 里的 DataSet 所有权?`FNiagaraDataSet*` 裸指针 —— 由谁分配和释放?→ Phase 4 / SystemInstance 的 `EmitterEventDataSetMap`
- GPU 情况下 `GPUExecContext` 的分配/归属?`FNiagaraComputeExecutionContext*` 看是指针非 owning —— 实际 owner?→ Phase 8
- `CachedEmitter` 裸指针的安全性 —— "System lifetime 保证"是够,但 Asset 热重载时?→ Phase 1 `UNiagaraEmitter` + editor 流程
- `bCombineEventSpawn` 具体不安全于 `ExecIndex()` 的根本原因?→ Phase 5 脚本执行
- `CalculateFixedBounds` 引发 GPU readback stall —— editor only,但开发中如何避免 hit?→ 实操经验累积
