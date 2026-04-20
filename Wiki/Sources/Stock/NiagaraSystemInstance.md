---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, runtime, instance, state-machine]
sources: 1
aliases: [NiagaraSystemInstance.h, FNiagaraSystemInstance 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSystemInstance.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraSystemInstance.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSystemInstance.h`
- **快照**: commit `b6ab0dee9`(分支 `Eureka` / 基于 4.26)
- **文件规模**: 574 行
- **同仓协作文件**: `NiagaraSystemInstance.cpp`、`NiagaraEmitterInstance.h`、`NiagaraSystemSimulation.h`、`NiagaraComponent.h`(持有者)、`NiagaraScriptExecutionContext.h`(Phase 5)、`NiagaraDataSet.h`(Phase 4)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 3 — 运行时实例层 3.1

## 职责 / 这个文件干什么的

定义 `FNiagaraSystemInstance`——**单个 Niagara 特效实例的运行时"心脏"**。它是 Phase 2 中 `UNiagaraComponent::SystemInstance` 这个 `TUniquePtr` 指向的东西。

关键设计选择:**不是 UObject**。`FNiagaraSystemInstance` 是裸 C++ 类,不走 UE 反射/GC,生命周期完全被 `TUniquePtr` 控制(Component 死则 Instance 死)。性能敏感对象常用此模式:免去 UObject 元数据、免 GC 扫描、用户不能从 BP 直接持有。

Instance 承担 4 大职责:

1. **状态机**:`ENiagaraExecutionState` 双状态(Requested/Actual)+ `EResetMode` 四种重置模式
2. **三阶段 Tick**:GT → Concurrent → Finalize,配合异步任务系统
3. **Emitter 实例容器**:`TArray<TSharedRef<FNiagaraEmitterInstance>>`,一个 System 下的所有 Emitter 实例
4. **参数 / DI 实例数据中枢**:持有 per-instance 参数、DI 实例数据、事件 DataSet、参数双缓冲

## 关键类型 / 函数 / 宏

### 主类

- `class NIAGARA_API FNiagaraSystemInstance`(L20):**非 UObject**,裸 C++ 类
  - `friend class FNiagaraSystemSimulation` / `friend class FNiagaraGPUSystemTick`——批量 Tick 调度器和 GPU Tick 数据结构可以直接访问私有字段

### 配套类型 / 枚举

- `enum class EResetMode { ResetAll, ResetSystem, ReInit, None }`(L39):重置的四档
  - `ResetAll`:重置 Instance + 所有 Emitter Simulations(粒子清零,状态归零)
  - `ResetSystem`:只重置 Instance 不碰 Emitter(保留粒子)
  - `ReInit`:彻底重建 Instance 和所有 Emitter(结构变了才用,如 Asset 被编辑)
  - `None`:不重置
- `DECLARE_DELEGATE(FOnPostTick)`(L26):每帧 tick 完 GT 回调
- `DECLARE_DELEGATE_OneParam(FOnComplete, bool /*bExternalCompletion*/)`(L27):完成回调,参数标是否外部触发
- `FInstanceParameters`(L537-L564,内嵌结构):聚合 per-tick 输入
  - `FTransform ComponentTrans` / `DeltaSeconds` / `TimeSeconds` / `RealTimeSeconds` / `EmitterCount` / `NumAlive` / `TransformMatchCount` / `RequestedExecutionState`

### 构造与生命周期

- `FNiagaraSystemInstance(UWorld&, UNiagaraSystem&, FNiagaraUserRedirectionParameterStore* = nullptr, USceneComponent* = nullptr, ENiagaraTickBehavior = UsePrereqs, bool bInPooled = false)`(L54):构造
  - 参数来自 Component:World、Asset、OverrideParameters(Component 那层)、AttachComponent、TickBehavior、是否池化
- `virtual ~FNiagaraSystemInstance()`(L58):虚析构
- `void Cleanup()`(L60):显式清理(析构前可能被调)
- `void Init(bool bInForceSolo=false)`(L63):**初始化**——构建 Emitter 实例列表、InitDataInterfaces、绑定参数
- `static bool AllocateSystemInstance(TUniquePtr<FNiagaraSystemInstance>&, UWorld&, UNiagaraSystem&, ...)`(L284):静态工厂,Component 用这个
- `static bool DeallocateSystemInstance(TUniquePtr<FNiagaraSystemInstance>&)`(L287):对偶

### 状态机

**双状态设计**(核心洞察):

- `ENiagaraExecutionState RequestedExecutionState`(L505):**用户/BP 请求**的状态(Activate / Deactivate 设置)
- `ENiagaraExecutionState ActualExecutionState`(L508):**模拟实际**的状态(Instance/Simulation 决定)
- `GetRequestedExecutionState() / SetRequestedExecutionState()`(L154-L155):访问器
- `GetActualExecutionState() / SetActualExecutionState()`(L157-L158)
- `IsComplete()`(L167):`ActualExecutionState == Complete || Disabled`
- `IsDisabled()`(L168):`ActualExecutionState == Disabled`

用户 Deactivate → RequestedExecutionState 立刻变,但 Actual 可能要等 emitter 的粒子全部消亡才变 Complete。这个分离让"软结束"和"硬结束"都能表达。

### 生命周期命令

- `void Activate(EResetMode = ResetAll)`(L65):激活,并指定 reset 模式
- `void Deactivate(bool bImmediate = false)`(L66):关闭,immediate=true 则立即硬结束
- `void Complete(bool bExternalCompletion)`(L67):标记完成
- `void Reset(EResetMode)`(L115):请求下一 tick 重置(延迟到 tick 执行,避免破坏当前数据)
- `void OnPooledReuse(UWorld& NewWorld)`(L69):被池复用时的重置入口
- `void SetPaused(bool)` / `bool IsPaused()`(L71-L72):暂停
- `void SetSolo(bool)`(L74):切换 Solo 模式
- `void HandleCompletion()`(L147):判断是否真的 Complete 并走完成流程

### 三阶段 Tick(最重要)

```cpp
// NiagaraSystemInstance.h:120-127
void Tick_GameThread(float DeltaSeconds);      // GT 阶段:收集输入、决定是否要 tick
void Tick_Concurrent(bool bEnqueueGPUTickIfNeeded = true);  // 任意线程:重型 CPU 模拟
bool FinalizeTick_GameThread(bool bEnqueueGPUTickIfNeeded = true);  // GT 阶段:DI 后处理、提交 GPU tick、bounds
```

**为什么三阶段**:UE 的 Tick 系统允许 Actor Component 在指定 TickGroup 执行,但 Niagara 的模拟工作量大,阻塞 GT 不可接受。策略是:
1. **GT 阶段**只收集当前帧的输入(Transform、DeltaSeconds、InstanceParameters_GameThread)
2. **Concurrent 阶段**在 TaskGraph 的其他线程跑真正的模拟(脚本执行、粒子更新)——可能跑几毫秒
3. **Finalize 阶段**回到 GT,处理那些必须在 GT 做的事(DI 后 tick、bounds 更新、GPU Tick 入队、Complete 判断)

相关辅助:
- `void TickInstanceParameters_GameThread(float)` / `TickInstanceParameters_Concurrent()`(L297-L299):两阶段的参数聚合
- `void ManualTick(float, const FGraphEventRef&)`(L117):手动 tick 入口(solo 模式下 `UNiagaraComponent::AdvanceSimulation` 用)
- `void TickDataInterfaces(float, bool bPostSimulate)`(L152):Pre/Post DI tick,**不能并行**(可能触发 Complete)
- `void WaitForAsyncTickDoNotFinalize(bool)` / `WaitForAsyncTickAndFinalize(bool)`(L137, L144):等待异步工作完成的两个版本——不 finalize / 要 finalize

**异步状态字段**:
- `volatile bool bAsyncWorkInProgress`(L493):当前有异步任务在飞
- `uint32 bNeedsFinalize : 1`(L471):有任务完成需要 GT 收尾
- `EResetMode DeferredResetMode = EResetMode::None`(L490):异步进行中时用户请求 Activate,把 reset 模式存到这里,finalize 时再做

### Tick Group

- `ETickingGroup CalculateTickGroup() const`(L320):基于 DI prereqs、Solo 状态等决定 TickGroup
- `ENiagaraTickBehavior TickBehavior`(L388)/ `GetTickBehavior() / SetTickBehavior()`(L315-L317)
- `UpdatePrereqs()`(L78):重新计算 DI 依赖带来的 TickGroup 约束
- `UActorComponent* PrereqComponent`(L384):依赖的 Component

### Emitter 管理

- `TArray<TSharedRef<FNiagaraEmitterInstance, ESPMode::ThreadSafe>> Emitters`(L403):**核心字段**——持有一组 Emitter 实例
- `GetEmitters()`(L177-L178):只读/读写双重载
- `TSharedPtr<FNiagaraEmitterInstance> GetSimulationForHandle(const FNiagaraEmitterHandle&)`(L171):按 handle 查 Emitter 实例
- `FNiagaraEmitterInstance* GetEmitterByID(FGuid)`(L182):按 GUID 查
- `TConstArrayView<FNiagaraEmitterExecutionIndex> GetEmitterExecutionOrder() const`(L180):拿 System 预计算的 tick 顺序(Phase 1 `UNiagaraSystem::EmitterExecutionOrder`)
- `void SetEmitterEnable(FName, bool)`(L149):单个 Emitter 开关
- `void InitEmitters()`(L354,私有):构造所有 Emitter 实例

### 参数双缓冲(性能关键)

```cpp
// NiagaraSystemInstance.h:441-449
static constexpr int32 ParameterBufferCount = 2;
FNiagaraGlobalParameters GlobalParameters[ParameterBufferCount];
FNiagaraSystemParameters SystemParameters[ParameterBufferCount];
FNiagaraOwnerParameters OwnerParameters[ParameterBufferCount];
TArray<FNiagaraEmitterParameters> EmitterParameters;  // 每 emitter × 2

uint32 CurrentFrameIndex : 1;
uint32 ParametersValid : 1;
```

**为什么双缓冲**:Concurrent 阶段(上一帧的渲染)和当前帧的模拟可能同时读写这些参数。双缓冲保证"当前帧写 buffer A,上一帧继续读 buffer B",避免加锁。

访问接口(L86-L106):
- `GetParameterIndex(bool PreviousFrame = false)`:根据 `CurrentFrameIndex` 算当前索引
- `FlipParameterBuffers()`:切换 + 两个 buffer 都触过后标记 `ParametersValid=true`
- `GetGlobalParameters / GetSystemParameters / GetOwnerParameters / GetEmitterParameters / EditEmitterParameters`:可带 `PreviousFrame` 参数读上一帧

- `FNiagaraParameterStore InstanceParameters`(L439):per-instance 参数,会被拉到 DataSet

### DataInterface 实例数据

DataInterface(DI,Phase 7 主角)可能需要 per-instance 数据。Instance 这里集中存:

```cpp
// NiagaraSystemInstance.h:426-436
TArray<uint8, TAlignedHeapAllocator<16>> DataInterfaceInstanceData;     // 所有 DI 的 instance data 拼成的大 blob
TArray<int32> PreTickDataInterfaces;                                    // 需要 pre-tick 的 DI 索引
TArray<int32> PostTickDataInterfaces;                                   // 需要 post-tick 的 DI 索引
TArray<TPair<TWeakObjectPtr<UNiagaraDataInterface>, int32>> DataInterfaceInstanceDataOffsets;
                                                                         // DI → 在 blob 里的 offset
TArray<FNiagaraPerInstanceDIFuncInfo> PerInstanceDIFunctions[(int32)ENiagaraSystemSimulationScript::Num];
                                                                         // 需要 per-instance 绑定的 DI 函数
```

- `void* FindDataInterfaceInstanceData(UNiagaraDataInterface*)`(L212):查某个 DI 的 instance data
- `const FNiagaraPerInstanceDIFuncInfo& GetPerInstanceDIFunction(ScriptType, FuncIndex)`(L221)
- `void InitDataInterfaces()`(L365,私有):初始化时遍历所有 DI 调 `PrepareForSimulation`
- `void DestroyDataInterfaceInstanceData()`(L351,私有)
- `bool bDataInterfacesInitialized : 1`(L473)
- `bool bDataInterfacesHaveTickPrereqs : 1`(L468):DI 是否有 TickGroup 约束

### Event DataSet

Emitter 之间可以通过命名 Event 通讯(Emitter A 发 Death 事件,Emitter B 监听)。Instance 管理 Event DataSet:

```cpp
typedef TPair<FName, FName> EmitterEventKey;                     // (EmitterName, EventName)
typedef TMap<EmitterEventKey, FNiagaraDataSet*> EventDataSetMap;
EventDataSetMap EmitterEventDataSetMap;                          // L452-L454
```

- `FNiagaraDataSet* CreateEventDataSet(FName EmitterName, FName EventName)`(L301)
- `FNiagaraDataSet* GetEventDataSet(FName, FName) const`(L302)
- `void ClearEventDataSets()`(L303)

### GPU 支持

- `uint32 bHasGPUEmitters : 1`(L466)
- `int32 ActiveGPUEmitterCount = 0`(L530)
- `uint32 TotalGPUParamSize = 0`(L529)
- `int32 GPUDataInterfaceInstanceDataSize = 0`(L533)
- `bool GPUParamIncludeInterpolation = false`(L534)
- `TArray<TPair<TWeakObjectPtr<UNiagaraDataInterface>, int32>> GPUDataInterfaces`(L535)
- `TUniquePtr<FNiagaraComputeSharedContext, FNiagaraComputeSharedContextDeleter> SharedContext`(L531):GPU 共享上下文
- `FNiagaraComputeSharedContext* GetComputeSharedContext()`(L188)
- `bool NeedsGPUTick() const`(L186):需要 GPU tick?
- `void GenerateAndSubmitGPUTick()`(L129):制造 GPU tick 并提交
- `void InitGPUTick(FNiagaraGPUSystemTick& OutTick)`(L130):填充 GPU tick 结构
- `NiagaraEmitterInstanceBatcher* GetBatcher() const`(L282):取 GPU 批处理器(Phase 8)
- `bool HasGPUEmitters()`(L289)
- `bool RequiresDistanceFieldData() / RequiresDepthBuffer() / RequiresEarlyViewData() / RequiresViewUniformBuffer()`(L109-L112):渲染资源依赖查询

### 其他关键字段

- `FNiagaraSystemInstanceID ID` + `FName IDName`(L422-L423):唯一标识,崩溃报告/调试用
- `int32 SystemInstanceIndex`(L373):在 SystemSimulation 数组里的索引
- `int32 SignificanceIndex`(L376):重要性排名,Scalability Manager 维护
- `UWorld* World`(L380) / `TWeakObjectPtr<UNiagaraSystem> Asset`(L381) / `TWeakObjectPtr<USceneComponent> AttachComponent`(L383)
- `FNiagaraUserRedirectionParameterStore* OverrideParameters`(L382):指向 Component 的 OverrideParameters,不是 owning
- `float Age` / `int32 TickCount`(L391, L397)
- `float LastRenderTime`(L394)
- `float LODDistance / MaxLODDistance`(L400-L401)
- `FBox LocalBounds`(L502)
- `FTransform WorldTransform`(L386)
- `ERHIFeatureLevel::Type FeatureLevel`(L516)
- `TSharedPtr<FNiagaraSystemSimulation, ESPMode::ThreadSafe> SystemSimulation`(L378):**反向指回**自己归属的批量 Simulation

### Flag bitfield(多)

```cpp
uint32 bSolo : 1;                           // 当前是否 solo
uint32 bForceSolo : 1;                      // 用户/bForceSolo 字段强制 solo
uint32 bPendingSpawn : 1;                   // 刚被加入 simulation 还没 spawn
uint32 bNotifyOnCompletion : 1;             // Complete 时是否触发 delegate
uint32 bPaused : 1;
uint32 bHasGPUEmitters : 1;
uint32 bDataInterfacesHaveTickPrereqs : 1;
uint32 bNeedsFinalize : 1;
uint32 bDataInterfacesInitialized : 1;
uint32 bAlreadyBound : 1;
uint32 bLODDistanceIsValid : 1;
uint32 bPooled : 1;                         // 来自池的实例,complete 时不解绑参数
uint32 bHasSimulationReset : 1;             // 需要硬重置
```

### Component Renderer Pool

Niagara 还允许 renderer 动态 spawn `USceneComponent`(比如 `NiagaraComponentRendererProperties`)。Instance 管这些任务:

```cpp
TQueue<FNiagaraComponentUpdateTask, EQueueMode::Mpsc> ComponentTasks;  // L519
FNiagaraComponentRenderPool ComponentRenderPool;                        // L520
mutable FRWLock ComponentPoolLock;                                      // L521
```

- `bool EnqueueComponentUpdateTask(const FNiagaraComponentUpdateTask&)`(L322)
- `TSet<int32> GetParticlesWithActiveComponents(USceneComponent*)`(L327)
- `void ResetComponentRenderPool()`(L522)
- `void ProcessComponentRendererTasks()`(L367)

### Editor-only

- `DECLARE_MULTICAST_DELEGATE(FOnInitialized/OnReset/OnDestroyed)`(L30-L33)
- Script debugger capture(L416-L420, L257-L273):`RequestCapture / QueryCaptureResults / GetActiveCaptureResults / FinishCapture / ShouldCaptureThisFrame / GetActiveCaptureWrite` —— 脚本调试器的帧捕获机制
- `UsesEmitter / UsesScript / UsesCollection`(L224-L227):反向查询
- `bool GetIsolateEnabled() const`(L206):编辑器里的 "Isolate Emitter"

### 调试 / 诊断

- `void Dump() const` / `DumpTickInfo(FOutputDevice&)`(L277, L280)
- `const FString& GetCrashReporterTag() const` / `FString CrashReporterTag`(L307, L513)

## 依赖与被依赖

**上游:**
- `NiagaraEmitterInstance.h` / `NiagaraEmitter.h` / `NiagaraEmitterHandle.h`(Phase 1)
- `NiagaraSystem.h`(Phase 1)
- `NiagaraDataInterfaceBindingInstance.h`(Phase 7 相关)
- `NiagaraCommon.h` / `NiagaraDataInterface.h`

**下游:**
- `UNiagaraComponent`(Phase 2,TUniquePtr 拥有者)
- `FNiagaraSystemSimulation`(本 Phase,批量调度)
- `FNiagaraGPUSystemTick`(Phase 8,通过 friend 访问)
- `FNiagaraScalabilityManager`(Phase 9)

## 关键代码片段

> `NiagaraSystemInstance.h:L20-L24` @ `b6ab0dee9` — 主类定义
> ```cpp
> class NIAGARA_API FNiagaraSystemInstance
> {
>     friend class FNiagaraSystemSimulation;
>     friend class FNiagaraGPUSystemTick;
> ```

> `NiagaraSystemInstance.h:L119-L127` @ `b6ab0dee9` — 三阶段 Tick 架构
> ```cpp
> /** Initial phase of system instance tick. Must be executed on the game thread. */
> void Tick_GameThread(float DeltaSeconds);
> /** Secondary phase of the system instance tick that can be executed on any thread. */
> void Tick_Concurrent(bool bEnqueueGPUTickIfNeeded = true);
> /** Final phase of system instance tick. Must be executed on the game thread. */
> bool FinalizeTick_GameThread(bool bEnqueueGPUTickIfNeeded = true);
> ```

> `NiagaraSystemInstance.h:L38-L49` @ `b6ab0dee9` — Reset 四模式
> ```cpp
> enum class EResetMode
> {
>     /** Resets the System instance and simulations. */
>     ResetAll,
>     /** Resets the System instance but not the simulations */
>     ResetSystem,
>     /** Full reinitialization of the system and emitters.  */
>     ReInit,
>     /** No reset */
>     None
> };
> ```

> `NiagaraSystemInstance.h:L91-L100` @ `b6ab0dee9` — 参数双缓冲翻转
> ```cpp
> FORCEINLINE void FlipParameterBuffers()
> {
>     CurrentFrameIndex = ~CurrentFrameIndex;
>     if (CurrentFrameIndex == 1)
>     {
>         ParametersValid = true;
>     }
> }
> ```

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/FNiagaraSystemInstance]] — 本文件定义的主类
- [[Wiki/Entities/Stock/FNiagaraEmitterInstance]] — Instance 持有的 emitter 实例
- [[Wiki/Entities/Stock/FNiagaraSystemSimulation]] — Instance 归属的批量调度器
- [[Wiki/Entities/Stock/UNiagaraComponent]] — Instance 的拥有者(TUniquePtr)
- [[Wiki/Entities/Stock/UNiagaraSystem]] — Instance 对应的 Asset
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — 本类是"Instance"的典型

## 开放问题 / 待深入

- **三阶段 Tick 的跨线程数据传递细节**:`Tick_Concurrent` 读 GT 写入的什么?→ 看 `.cpp` 实现或 Phase 8 GPU 那部分
- **`DataInterfaceInstanceData` 的 blob 对齐策略**:`TAlignedHeapAllocator<16>`——为什么是 16?SIMD?→ Phase 7
- **`PerInstanceDIFunctions` 的 binding 时机**:Init 时绑定一次?每次 Reset?→ Phase 5/7
- **事件 DataSet 的生命周期**:`EmitterEventDataSetMap` 里的 `FNiagaraDataSet*` 谁 own?→ Phase 4 看 DataSet 分配策略
- **`bPooled` flag 的 UnbindParameters 影响**:注释说 pool 时不解绑——这是 pool 性能关键,具体怎么做?→ Phase 9 Pool
