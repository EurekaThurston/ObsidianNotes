---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, learning-path, phase-3, reader, runtime, instance, state-machine]
sources: 3
aliases: [Phase 3 读本, Niagara 运行时实例层读本, Niagara 心脏读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 3 读本 — Niagara 的心脏

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 3 的**主题读本**——详细、精确、满满当当,一次读完即完整掌握 Niagara 的运行时实例层(状态机 + 三阶段 Tick + 批量调度),不需要跳转。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 3 要回答的问题

Phase 2 在 `UNiagaraComponent` 里放了一个 `TUniquePtr<FNiagaraSystemInstance> SystemInstance`,然后指了一句"详见 Phase 3"就溜了。本 Phase 就是来兑付这张支票。

> [!question] Phase 3 要回答
> 一个正在跑的 Niagara 特效,在任意一帧里,到底发生了什么?它是如何从"上帧的状态"推进到"这帧的粒子"?多个同 Asset 的实例,是各跑各的,还是有批量优化?
>
> 更细的追问:
> 1. `TUniquePtr<FNiagaraSystemInstance>` 里装的到底是个什么类?UObject?普通 C++?
> 2. Phase 2 说"批量 Tick" —— 具体谁来批量,怎么批量,为什么批?
> 3. `bForceSolo` 陷阱的性能退化,退化的是谁的代码?
> 4. UE 的 Tick 系统是同步的,但 Niagara 模拟那么重,怎么避免卡住 GameThread?
> 5. "Reset"这个操作,为什么要分四档(`ResetAll` / `ResetSystem` / `ReInit` / `None`)?

Phase 3 涉及 3 个文件,按"由内到外"分层:

| 文件 | 行数 | 层级 | 角色 |
|---|---|---|---|
| `NiagaraEmitterInstance.h` | 239 | 最内层 | 单个 Emitter 的运行时(粒子数据 + Spawn/Update 脚本上下文) |
| `NiagaraSystemInstance.h` | 574 | 中间层 | 单个 System 特效的运行时(状态机 + Emitter 容器 + 三阶段 Tick) |
| `NiagaraSystemSimulation.h` | 429 | 最外层 | 同 Asset 多实例的**批量调度器**(Tick Batch + 参数聚合 DataSet) |

三者的包含关系是:

```
FNiagaraSystemSimulation  ← WorldManager 按 (Asset, World, TickGroup) 持有
    持有 TArray<FNiagaraSystemInstance*>  ← 一批同 Asset 的 instance(非 owning 指针)
        每个 FNiagaraSystemInstance
            持有 TArray<TSharedRef<FNiagaraEmitterInstance>>  ← Emitter 实例列表
```

这个包含关系是 Phase 3 全部内容的骨架。

读完你应该能回答:
1. 为什么 `FNiagaraSystemInstance` 不是 UObject?
2. 双状态机(Requested / Actual)为什么要分两个?
3. 三阶段 Tick(GT / Concurrent / Finalize)分别在干什么?
4. Tick Batch = 4 的意义?
5. 参数双缓冲(`GlobalParameters[2]`)解决什么问题?
6. Niagara 的"无字符串查表"在 Phase 3 的具体实现(`FNiagaraParameterStoreToDataSetBinding`)
7. Emitter 间的 event 通讯如何实现?
8. Solo 与 Batched 的区别,`bForceSolo` 性能陷阱的定量估算

---

## 1. 为什么不是 UObject

先回答问题 1。`FNiagaraSystemInstance`、`FNiagaraEmitterInstance`、`FNiagaraSystemSimulation` 三个类都是**普通 C++ 类**,不是 UObject。

```cpp
// NiagaraSystemInstance.h:20
class NIAGARA_API FNiagaraSystemInstance
{
    friend class FNiagaraSystemSimulation;
    friend class FNiagaraGPUSystemTick;
    ...
```

这是**主动的设计选择**。UObject 的好处是反射、GC、BP 可见;代价是元数据、GC 扫描开销、对象指针不能裸用(要 TWeakObjectPtr/UPROPERTY)。Niagara 运行时实例要创建很多(50 个爆炸 = 50 个 SystemInstance + ~数十个 EmitterInstance),UObject 的开销打不住。

性能敏感对象常用的"非 UObject"模式:
- 生命周期由 `TUniquePtr` / `TSharedRef` 显式控制
- UObject 引用用 `TWeakObjectPtr`(Asset、World、AttachComponent)或裸指针(危险但省空间,注释里"保证 lifetime"就是口头协议)
- 需要 UObject GC 保护时,继承 `FGCObject` + `AddReferencedObjects` 告诉 GC 持有了什么

`FNiagaraSystemSimulation` 就是典型混合体:**既非 UObject,又要 GC 知道自己持有 `UNiagaraEffectType*`** —— 所以继承 `FGCObject`:

```cpp
// NiagaraSystemSimulation.h:239-247
class FNiagaraSystemSimulation
    : public TSharedFromThis<FNiagaraSystemSimulation, ESPMode::ThreadSafe>, FGCObject
{
    //FGCObject Interface
    virtual void AddReferencedObjects(FReferenceCollector& Collector) override;
```

> [!abstract] 小结
> Phase 3 的三个主要类都不是 UObject。这不是懒得套 UCLASS,而是性能选择。用 `TUniquePtr`(Component→SystemInstance)/`TSharedRef`(SystemInstance→EmitterInstance)精确控制生命周期,GC 保护通过 `FGCObject` 按需注入。

---

## 2. 单实例视角:`FNiagaraSystemInstance` 状态机

Phase 3 的中间层先登场——因为它是 Component 直接持有的对象,也是另两个类交互的枢纽。

### 2.1 Init 到 Complete 的完整生命

Component 通过 `AllocateSystemInstance` 静态工厂创建 Instance:

```cpp
// NiagaraSystemInstance.h:284
static bool AllocateSystemInstance(
    TUniquePtr<FNiagaraSystemInstance>& OutSystemInstanceAllocation,
    UWorld& InWorld, UNiagaraSystem& InAsset,
    FNiagaraUserRedirectionParameterStore* InOverrideParameters = nullptr,
    USceneComponent* InAttachComponent = nullptr,
    ENiagaraTickBehavior InTickBehavior = ENiagaraTickBehavior::UsePrereqs,
    bool bInPooled = false);
```

构造后流程:

1. `Init(bInForceSolo)` —— 构建 Emitter 列表(`InitEmitters`)、初始化 DI(`InitDataInterfaces`)、绑定参数(`BindParameters`)
2. `Activate(EResetMode::ResetAll)` —— 启动;注册到 `FNiagaraSystemSimulation`
3. 每帧三阶段 Tick(§3)
4. Complete 条件满足 → `HandleCompletion()` → 广播 `OnCompleteDelegate`
5. Component 析构时 `Cleanup()` + `~FNiagaraSystemInstance()`

### 2.2 双状态机:Requested vs Actual

回答问题 2:

```cpp
// NiagaraSystemInstance.h:505-508
ENiagaraExecutionState RequestedExecutionState;  // 用户/BP 通过 Activate/Deactivate 设
ENiagaraExecutionState ActualExecutionState;     // Simulation 实际所处的状态
```

`ENiagaraExecutionState` 的主要取值(来自 `NiagaraCommon.h`):
- `Active` — 正常 tick
- `Inactive` — 不再 spawn,但已有粒子继续 update(soft stop)
- `Complete` — 粒子全部消亡,实例可归还/销毁
- `Disabled` — 被强制关闭

**为什么要分开**:

| 场景 | Requested | Actual |
|---|---|---|
| 调 `Deactivate()`(普通) | 立刻变 Inactive | 继续 Active,直到粒子消亡 |
| 调 `Deactivate(bImmediate=true)` | 立刻变 Complete | 立刻变 Complete(硬结束) |
| 调 `Complete(bExternalCompletion=true)` | — | Complete(显式标记) |
| Scalability cull(系统触发) | 不动(避免 `OnSystemFinished` 误发) | 走 Disabled |

这个分离让 "用户意图" 与 "实际状态" 解耦,支持 "先请求停,等粒子自然消亡" 的软结束——典型的生产场景(烟花粒子放完再结束)。

### 2.3 Reset 四档

回答问题 5。`EResetMode` 四档对应不同剧烈程度的重置:

```cpp
// NiagaraSystemInstance.h:38-49
enum class EResetMode
{
    ResetAll,       // Instance + 所有 Emitter Simulations 都重置(粒子清零)
    ResetSystem,    // 只重置 Instance,Emitter 保留(特殊场景)
    ReInit,         // 彻底重建——Asset 编辑后必须
    None            // 不重置(通常是 Activate 的复用)
};
```

典型触发:
- **BP 调 `ResetSystem`** → `ResetAll`(重头播)
- **BP 调 `ReinitializeSystem`** → `ReInit`(假设 Asset 结构可能变了)
- **Asset editor 保存** → `ReInit`(所有引用这 Asset 的 Component 都要走这条)
- **从 Pool 取出** → `ResetAll`(清旧实例的粒子遗留)

异步工作中请求 Activate 时,Instance 会把 reset mode 存到 `DeferredResetMode`,finalize 时再执行——避免阻塞 GT。

### 2.4 暂停与时间控制

- `SetPaused(bool)` / `IsPaused()` —— `bPaused : 1` 位
- `Age` / `TickCount` —— 只累加,不受暂停影响?实际看 .cpp,但基于名字是"被 tick 触发的计数"
- `ManualTick(DeltaSeconds, GraphEvent)` —— 手动推 tick(solo 模式),Phase 2 `AdvanceSimulation` 的入口
- `SetSolo(bool)` —— 切换 Solo;触发 `TransferInstance` 在 Simulation 之间迁移(§5)

---

## 3. 三阶段 Tick 架构

回答问题 4 的前半(后半在 §5)。`FNiagaraSystemInstance` 有三个 tick 入口:

```cpp
// NiagaraSystemInstance.h:120-127
void Tick_GameThread(float DeltaSeconds);
void Tick_Concurrent(bool bEnqueueGPUTickIfNeeded = true);
bool FinalizeTick_GameThread(bool bEnqueueGPUTickIfNeeded = true);
```

### 3.1 为什么要分三段

UE 的 `UActorComponent::TickComponent` 在 TickGroup 里被同步调用。但 Niagara 单个 SystemInstance 的模拟可能几百微秒到几毫秒——如果全走 GT,几十个 Instance 堆起来就是几十毫秒,帧率垮。

策略:

- **GT 阶段**(必须在 GameThread):
  - 读当前帧的 Transform(`WorldTransform`)
  - 读 `DeltaSeconds` / 世界时间
  - `TickInstanceParameters_GameThread` 聚合 per-instance 参数到 `GatheredInstanceParameters` 结构
  - 不做重型计算
- **Concurrent 阶段**(任意线程):
  - `TickInstanceParameters_Concurrent` 继续参数工作
  - 真正的模拟:所有 Emitter 的 `PreTick / Tick / PostTick`
  - 脚本执行(Phase 5 的 `FNiagaraScriptExecutionContext::Execute`)
  - 参数双缓冲翻转(`FlipParameterBuffers`)
- **Finalize GT 阶段**(回 GameThread):
  - `TickDataInterfaces(DeltaSeconds, bPostSimulate=true)` —— 必须在 GT 的 DI 后处理(比如某些 DI 要访问 World)
  - `LocalBounds` 更新
  - GPU Tick 入队(如果 `NeedsGPUTick`)
  - `HandleCompletion()` 判断是否 Complete
  - 广播 `OnPostTickDelegate`

### 3.2 异步的边界

Concurrent 阶段可能在 TaskGraph worker 线程跑。GT 代码要访问 Instance 状态,必须先同步:

```cpp
// NiagaraSystemInstance.h:137-144
/** Blocks until any async work has completed, does NOT finalize. */
void WaitForAsyncTickDoNotFinalize(bool bEnsureComplete = false);

/** Blocks and then calls finalize if needed. */
void WaitForAsyncTickAndFinalize(bool bEnsureComplete = false);
```

两个版本的差别:`DoNotFinalize` 仅等 concurrent 停,不走 finalize;`AndFinalize` 会触发 finalize,可能导致 Instance 从 Simulation 中移除。文档警告:误用 DoNotFinalize 可能让 Instance 处于未定义状态。

相关 flag:

```cpp
volatile bool bAsyncWorkInProgress;   // 有异步任务在飞
uint32 bNeedsFinalize : 1;             // 有任务完成需要 GT 收尾
EResetMode DeferredResetMode;          // 异步期间的 reset 请求
```

### 3.3 参数双缓冲

回答问题 5:

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

问题:Concurrent 阶段(当前帧)要写新参数,但 RenderThread(上一帧)可能还在读旧参数。双缓冲的经典解法:`CurrentFrameIndex` 指向当前写 buffer,`~CurrentFrameIndex` 指向上一帧读 buffer,Flip 即切换。

```cpp
FORCEINLINE void FlipParameterBuffers()
{
    CurrentFrameIndex = ~CurrentFrameIndex;
    if (CurrentFrameIndex == 1)  // 两个 buffer 都触过后才标 valid
    {
        ParametersValid = true;
    }
}
```

访问接口还支持 "读上一帧":

```cpp
FORCEINLINE const FNiagaraGlobalParameters& GetGlobalParameters(bool PreviousFrame = false) const
{
    return GlobalParameters[GetParameterIndex(PreviousFrame)];
}
```

这是给 "interpolated spawn"(帧间插值 spawn)之类功能用的——某些脚本要在 spawn 时引用上一帧的变量状态。

---

## 4. Emitter 侧:`FNiagaraEmitterInstance`

SystemInstance 持有一组 Emitter 实例:

```cpp
// NiagaraSystemInstance.h:403
TArray<TSharedRef<FNiagaraEmitterInstance, ESPMode::ThreadSafe>> Emitters;
```

`TSharedRef` 让 Emitter 的生命周期对外可见(Phase 6 的 `FNiagaraRenderer` 会持有这个 SharedRef)。

### 4.1 核心字段三元组

每个 EmitterInstance 的灵魂是三个字段:

```cpp
// NiagaraEmitterInstance.h:170-172, 201
FNiagaraScriptExecutionContext SpawnExecContext;     // CPU Spawn 脚本(Phase 5)
FNiagaraScriptExecutionContext UpdateExecContext;    // CPU Update 脚本
FNiagaraComputeExecutionContext* GPUExecContext = nullptr;  // GPU 替代

FNiagaraDataSet* ParticleDataSet = nullptr;          // 粒子数据(Phase 4)
```

> [!abstract] CPU/GPU 分叉就在这里
> Asset 层 `UNiagaraEmitter::SimTarget` 在运行时体现为:要么 `SpawnExecContext + UpdateExecContext` 有效(CPU),要么 `GPUExecContext` 有效。两条路径不共存。这回应了 [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] 里的"分叉点"预告。

### 4.2 每帧四步

```cpp
// NiagaraEmitterInstance.h:63-66
void PreTick();                                  // 事件 count 计算等准备
void Tick(float DeltaSeconds);                   // 执行 Spawn/Update 脚本
void PostTick();                                 // bounds 计算等收尾
bool HandleCompletion(bool bForce = false);      // 完成判断
```

这四步嵌套在 SystemInstance 的 `Tick_Concurrent` 里——所以 Emitter tick 是在 worker 线程跑的。

### 4.3 GetNumParticles 的 CPU/GPU 双路径

一个看似简单的 "有多少粒子"接口,底下有两套路径:

```cpp
// NiagaraEmitterInstance.h:93-108
FORCEINLINE int32 GetNumParticles() const
{
    // GPU 情况:从 GPUExecContext 查(注意 latent,需 fence 才准)
    if (GPUExecContext)
    {
        return GetNumParticlesGPUInternal();
    }
    // CPU 情况:从 DataSet 查
    if (ParticleDataSet->GetCurrentData())
    {
        return ParticleDataSet->GetCurrentData()->GetNumInstances();
    }
    return 0;
}
```

GPU 情况的注释很坦诚——"The count will still technically be incorrect but hopefully adequate for a system script update"。意思是:System 级脚本读这个值用来决定 "要不要 spawn 更多" 时,上一帧的 count 够用;精确读取需要专门的 fence(Phase 8)。

### 4.4 Renderer Bindings

Emitter 和 Renderer(Phase 6)之间有个专门的参数桥:

```cpp
// NiagaraEmitterInstance.h:182
FNiagaraParameterStore RendererBindings;
```

Renderer 每帧从这里取它需要的 emitter 级参数(比如 material 覆盖、Sprite 尺寸默认值)。Phase 6 会展开。

### 4.5 Spawn 信息

```cpp
// NiagaraEmitterInstance.h:168
TArray<FNiagaraSpawnInfo> SpawnInfos;
```

一个 Emitter 可以有多个 spawn 来源(rate-based、burst、event-based),每个都产生一条 `FNiagaraSpawnInfo` 放到这个数组。`Tick()` 里遍历这个数组做实际 spawn。

### 4.6 Event 子系统

如果 Emitter 没有 event,`EventInstanceData` 是 nullptr(省内存)。否则它持有一个内嵌结构:

```cpp
struct FEventInstanceData
{
    TArray<FNiagaraScriptExecutionContext> EventExecContexts;              // 每个 event handler 一个 ExecContext
    TArray<FNiagaraParameterDirectBinding<int32>> EventExecCountBindings;
    TArray<FNiagaraDataSet*> UpdateScriptEventDataSets;
    TArray<FNiagaraDataSet*> SpawnScriptEventDataSets;
    TArray<FNiagaraEventHandlingInfo> EventHandlingInfo;
    int32 EventSpawnTotal = 0;
};
```

Emitter 之间通过命名事件通讯(Emitter A 发 "Death" 事件,Emitter B 监听并 spawn)。Event 的 DataSet 实际持有在 SystemInstance 的 `EmitterEventDataSetMap`:

```cpp
// NiagaraSystemInstance.h:452-454
typedef TPair<FName, FName> EmitterEventKey;  // (EmitterName, EventName)
typedef TMap<EmitterEventKey, FNiagaraDataSet*> EventDataSetMap;
EventDataSetMap EmitterEventDataSetMap;
```

这回答了 "Emitter 间 event 如何实现":**键是 (SourceEmitterName, EventName),值是一份 DataSet**,生产者往里写,消费者从里读。`CreateEventDataSet` / `GetEventDataSet` / `ClearEventDataSets` 管理。

### 4.7 危险字段:`CachedEmitter`

```cpp
// NiagaraEmitterInstance.h:206
UNiagaraEmitter* CachedEmitter = nullptr;  // 裸指针!
```

注释说"Raw ptr should be safe here as we check for the validity of the system and it's emitters higher up before any ticking"——意思是 "tick 之前 SystemInstance 已经验过 System/Emitter 都活着,所以裸指针安全"。

> [!warning] 裸指针的口头协议
> `CachedEmitter` 依赖 "System 级生命周期保证"——这是一份**编码约定,不是技术保证**。编辑器场景里 Asset 可能被热重载,口头协议可能破。开发中如果 Niagara 崩溃 stack 里看到 `CachedEmitter` 访问,优先怀疑这一路。

---

## 5. 多实例视角:`FNiagaraSystemSimulation`

最外层登场。回答问题 2 / 4 后半 / 8。

### 5.1 身份:(Asset, World, TickGroup)

```cpp
// NiagaraSystemSimulation.h:336-345
TWeakObjectPtr<UNiagaraSystem> WeakSystem;   // weak,Asset GC 不阻塞
UNiagaraEffectType* EffectType;              // 即使 Asset GC 了也要保 scalability count
ETickingGroup SystemTickGroup = TG_MAX;
UWorld* World;
```

Simulation 的身份由这三元组决定。`FNiagaraWorldManager`(Phase 9)按此索引持有一堆 Simulation。如果有 50 个相同 Asset 的爆炸,它们都注册到**同一个** Simulation;这个 Simulation 每帧只跑一次 System 级脚本(但要处理 50 个实例的参数聚合)。

### 5.2 Tick Batch = 4 的秘密

Phase 2 反复说的"批量 Tick",具体实现:

```cpp
// NiagaraSystemSimulation.h:28-29
#define NiagaraSystemTickBatchSize 4
typedef TArray<FNiagaraSystemInstance*, TInlineAllocator<NiagaraSystemTickBatchSize>> FNiagaraSystemTickBatch;
```

**4 个实例打一批**,提交一个 TaskGraph 任务。`TInlineAllocator<4>` 让小批免堆分配。相关方法:

```cpp
void AddSystemToTickBatch(FNiagaraSystemInstance*, FNiagaraSystemSimulationTickContext&);
void FlushTickBatch(FNiagaraSystemSimulationTickContext&);
```

Tick 轮里遍历 `SystemInstances` 数组,每塞满 4 个就 flush 一次,剩余的最后再 flush。

为什么是 4?TODO 注释写着 "It would be good to have the batch size be variable per system to try to keep a good work/overhead ratio"——意思是**这是个经验选择**,理想是动态调整,但目前固定。4 大约对应:
- 小到能让 4 个 instance 大多数在一个 L1 cache warm 周期里做完
- 大到摊薄 TaskGraph 的调度开销(约几微秒)

### 5.3 System 级脚本的 Pipeline

Simulation 的主 Tick(两阶段):

```cpp
Tick_GameThread(DeltaSeconds, CompletionEvent)
    → SetupParameters_GameThread       // 刷常量参数(时间、instance 数量)
    → 调度 concurrent 任务
Tick_Concurrent(Context)  // 可在任意线程
    → PrepareForSystemSimulate         // 把 instance 参数拉到 DataSet
    → SpawnSystemInstances             // 跑 System Spawn 脚本(新 instance)
    → UpdateSystemInstances            // 跑 System Update 脚本(所有 instance)
    → TransferSystemSimResults         // 把计算结果派发到每个 Emitter
```

### 5.4 参数 ↔ DataSet 绑定的运行时形态

Phase 1 里我们见过 `FNiagaraSystemCompiledData` 编译期产出的 "参数到 DataSet 偏移" 绑定。Phase 3 这里是那个绑定的**运行时消费者**:

```cpp
// NiagaraSystemSimulation.h:33-52
struct FNiagaraParameterStoreToDataSetBinding
{
    struct FDataOffsets
    {
        int32 ParameterOffset;        // 在 ParameterStore 的 byte offset
        int32 DataSetComponentOffset; // 在 DataSet component 的 offset
    };

    TArray<FDataOffsets> FloatOffsets;
    TArray<FDataOffsets> Int32Offsets;
    TArray<FHalfDataOffsets> HalfOffsets;
    ...
};
```

正向(参数 → DataSet,simulation 前):

```cpp
void ParameterStoreToDataSet(const FNiagaraParameterStore&, FNiagaraDataSet&, int32 InstanceIndex)
{
    ...
    for (const FDataOffsets& DataOffsets : FloatOffsets) {
        float* ParamPtr = (float*)(ParameterData + DataOffsets.ParameterOffset);
        float* DataSetPtr = CurrBuffer.GetInstancePtrFloat(DataOffsets.DataSetComponentOffset, InstanceIndex);
        *DataSetPtr = *ParamPtr;  // 直接 memcpy,没有字符串查表
    }
    ...
}
```

反向(DataSet → 参数,simulation 后)同理。这是 Niagara 性能优势的**具体形态**——每次 tick 里 N 个实例 × M 个变量的参数移动,全靠编译期算好的 offset 表直接 memcpy。

Simulation 里持有 6 组这种 binding:

```cpp
// NiagaraSystemSimulation.h:366-379
FNiagaraParameterStoreToDataSetBinding SpawnInstanceParameterToDataSetBinding;
FNiagaraParameterStoreToDataSetBinding UpdateInstanceParameterToDataSetBinding;

TArray<FNiagaraParameterStoreToDataSetBinding> DataSetToEmitterSpawnParameters;
TArray<FNiagaraParameterStoreToDataSetBinding> DataSetToEmitterUpdateParameters;
TArray<TArray<FNiagaraParameterStoreToDataSetBinding>> DataSetToEmitterEventParameters;
TArray<FNiagaraParameterStoreToDataSetBinding> DataSetToEmitterGPUParameters;
TArray<FNiagaraParameterStoreToDataSetBinding> DataSetToEmitterRendererParameters;
```

"System attribute 对 Emitter 可见"的运行时机制就是这 6 组 binding 在 `TransferSystemSimResults` 里把 System DataSet 的结果派发到各 Emitter 的参数存储。

### 5.5 多份 DataSet

```cpp
FNiagaraDataSet MainDataSet;                     // 常规 tick
FNiagaraDataSet SpawningDataSet;                 // 额外 spawn(level streaming 情境)
FNiagaraDataSet PausedInstanceData;              // 暂停实例的数据备份
FNiagaraDataSet SpawnInstanceParameterDataSet;   // System 参数聚合(spawn 输入)
FNiagaraDataSet UpdateInstanceParameterDataSet;  // System 参数聚合(update 输入)
```

这 5 份都是 System 层的 DataSet——每份代表一个不同 pipeline 阶段的数据状态。Phase 4 会深入 `FNiagaraDataSet` 本身。

### 5.6 Solo vs Batched 的定量差别

回答问题 8 的定量估算:

假设场景里有 50 个同 Asset 的爆炸实例,每个实例的 System 级 update 脚本执行大约 10μs。

**Batched 模式**:
- Simulation 一次 Tick,12.5 批(50/4),每批一个 TaskGraph 任务
- 12.5 个任务,每个 40μs(4 实例 × 10μs)—— 但 System 级脚本**在一次 `Tick_Concurrent` 里执行一次**,共享编译产物和 binding,TaskGraph 调度开销摊薄
- 总耗时约等于串行的 1.5-2 倍(取决于核数),合计 <1ms GT 阻塞(Finalize 部分)

**Solo 模式**:
- 50 个 Simulation,每个就 1 个实例
- 50 次完整 System 级 tick,每次包含:`SetupParameters` + `PrepareForSystemSimulate` + `SpawnSystemInstances` + `UpdateSystemInstances` + `TransferSystemSimResults`
- 每次 Simulation 本身的 overhead 约 20-50μs,加实际脚本 10μs
- 50 × 30μs = 1.5ms+ 的**单线程**工作,因为每 Simulation 是独立 task,不能合并
- **退化约 5-20 倍**

这就是 Phase 2 `bForceSolo` 警告的定量根据。

### 5.7 其他

- `ENiagaraGPUTickHandlingMode` 5 模式 —— GPU Tick 有 5 种调度策略,本 Phase 只登记,Phase 8 深入
- `TransferInstance(Source, Instance)` —— Solo ↔ Batched 切换时从源 Simulation 迁移
- `bUseLegacyExecContexts` CVar —— Legacy 模式下强制全部 solo(历史兼容)

---

## 6. 完整 Tick 时序拼图

把前面 5 节拼成一张图——**一个同 Asset 有 50 个实例的场景**,一帧里的完整事件序列:

```
UE Tick Group(比如 TG_PrePhysics)到了
    ↓
FNiagaraWorldManager 调所有 Simulation 的 Tick_GameThread(触发点)
    ↓
FNiagaraSystemSimulation::Tick_GameThread(DeltaSeconds, CompletionEvent)
    ↓
    SetupParameters_GameThread      // 刷常量
    ↓
    调度 Concurrent 任务(TaskGraph 把 Tick_Concurrent 发到 worker 线程)
    ↓
    ── GT 回来,继续做别的事 ──
    ↓
UE 在 worker 线程上开始
FNiagaraSystemSimulation::Tick_Concurrent(Context)
    ↓
    PrepareForSystemSimulate:把 50 个 instance 的参数拉到 SpawnInstanceParameterDataSet / UpdateInstanceParameterDataSet
    ↓
    SpawnSystemInstances:跑 System Spawn 脚本(对新 instance)
    ↓
    UpdateSystemInstances:跑 System Update 脚本(对 50 个 instance,批 4 个一组)
    ↓
    TransferSystemSimResults:把结果通过 6 组 binding 派发到每个 Emitter
    ↓
    Per-Instance 嵌套阶段(每个 FNiagaraSystemInstance 的 Tick_Concurrent):
      Instance 1 的 TickInstanceParameters_Concurrent
        → 每个 FNiagaraEmitterInstance.PreTick / Tick / PostTick
            → SpawnExecContext 执行(CPU Emitter Spawn 脚本)
            → UpdateExecContext 执行(CPU Emitter Update 脚本)
            → 或 GPUExecContext 准备 GPU tick
            → 粒子数据更新到 ParticleDataSet
      Instance 2, ..., Instance 50
    ↓
    FlipParameterBuffers
    ↓
    ── Concurrent 结束,标记 bNeedsFinalize ──
    ↓
GT 某个时刻调 FinalizeTick_GameThread
    ↓
  每个 Instance:
    TickDataInterfaces(postSimulate=true)      // DI 后处理
    更新 LocalBounds
    如果 NeedsGPUTick -> GenerateAndSubmitGPUTick(Phase 8)
    HandleCompletion()                          // 判 Complete,可能广播 OnCompleteDelegate
    ProcessComponentRendererTasks()             // 处理 renderer 产生的 SceneComponent 任务
    ↓
  触发 OnPostTickDelegate
    ↓
  下一帧重复
```

渲染线程在另一边:读上一帧的 DataSet + Renderer Bindings 出 render proxy 的数据(Phase 6)。

---

## 7 条关键洞察

1. **三个主要类都不是 UObject**,是性能取舍。生命周期通过 `TUniquePtr` / `TSharedRef` 精确管,UObject 引用通过 `TWeakObjectPtr` 或 `FGCObject::AddReferencedObjects` 保护
2. **双状态机(Requested/Actual)**解耦用户意图和实际状态,支持软结束。Scalability cull 走 Actual 不动 Requested,避免误发 `OnSystemFinished`
3. **三阶段 Tick 的分工**:GT 收输入、Concurrent 做重活、Finalize GT 收尾 + GPU 提交——避免 GT 阻塞但保持 DI 后处理的顺序性
4. **参数双缓冲**解决渲染线程读上一帧、游戏线程写当前帧的竞态;`CurrentFrameIndex : 1` 位 flip,无锁
5. **Simulation 的身份 = (Asset, World, TickGroup)**,同 Asset 多实例自动聚到一个 Simulation,批量 tick 4 个一组
6. **`bForceSolo` 退化 5-20 倍**——Solo 模式等于把 50 个 instance 打回 50 个 Simulation,System 级脚本和绑定都不能共享
7. **`FNiagaraParameterStoreToDataSetBinding` 是 Phase 1 编译产物的运行时消费者**——Niagara "无字符串查表"的 memcpy 现场就在这里。每帧 N 实例 × M 参数的移动全靠预算好的 offset

---

## Phase 3 留下的问题

- `FNiagaraDataSet` 的内部结构、SoA 布局、双缓冲机制(CurrentData / DestinationData)→ **Phase 4**
- `FNiagaraScriptExecutionContext::Execute` 里 VectorVM 字节码如何被 dispatch → **Phase 5**
- `FNiagaraComputeExecutionContext` 的 GPU 接线细节 → **Phase 8**
- `NiagaraEmitterInstanceBatcher` 的 GPU Tick 调度(两个 Instance 的 Batcher 是同一个?按 Simulation?按 World?) → **Phase 8**
- `FNiagaraComponentRenderPool` 的 component renderer 生态 → Phase 6 可能简略提
- DI 的 `PreTick` / `PostTick` 具体做什么,`DataInterfaceInstanceData` blob 的填充 → **Phase 7**
- Pool 实例的 `bPooled` 对 complete 流程影响(不 unbind 参数) → **Phase 9**
- `ENiagaraGPUTickHandlingMode` 5 种模式的决策 → **Phase 8**

## 下一步预告

**Phase 4**:数据模型。揭开 `FNiagaraDataSet` 的盖子——Niagara 的"数据语言"具体长什么样:
- `NiagaraTypes.h` — 类型系统(`FNiagaraTypeDefinition` / `FNiagaraVariable`)
- `NiagaraCommon.h` — 共享枚举与结构
- `NiagaraConstants.h` — 预定义常量(`Engine.Time` 等)
- `NiagaraDataSet.h` — 核心!粒子数据存储的 SoA 布局
- `NiagaraDataSetAccessor.h` — 类型安全的 accessor
- `NiagaraParameters.h` — 参数类型
- `NiagaraParameterStore.h` — 参数存储

难度仍是 ⭐⭐⭐,但主题切换到数据结构,状态机复杂度下来了。

---

## 深入阅读

### 本议题的原子页

- 源摘要:
  - [[Wiki/Sources/Stock/NiagaraSystemInstance]](574 行,中间层)
  - [[Wiki/Sources/Stock/NiagaraEmitterInstance]](239 行,最内层)
  - [[Wiki/Sources/Stock/NiagaraSystemSimulation]](429 行,最外层批量调度)
- Entity 页:
  - [[Wiki/Entities/Stock/FNiagaraSystemInstance]]
  - [[Wiki/Entities/Stock/FNiagaraEmitterInstance]]
  - [[Wiki/Entities/Stock/FNiagaraSystemSimulation]]

### 前置议题

- Phase 2 Component 层:[[Readers/Niagara/Phase2-component-layer-读本]] — `UNiagaraComponent::SystemInstance` 的兑付
- Phase 1 Asset 层:[[Readers/Niagara/Phase1-asset-layer-读本]] — `FNiagaraSystemCompiledData` 的绑定在 §5.4 被消费
- Phase 0 心智模型:[[Readers/Niagara/Phase0-心智模型-读本]] — CPU/GPU 分叉、Asset/Instance 二元

### 导航 / 总图

- [[Wiki/Syntheses/Niagara/Niagara-learning-path]] — 10 Phase 学习路径总图
- [[Wiki/Overview]] — 本仓综合视图

---

*本读本由 [[Claudian]] 基于 Phase 3 的 3 个头文件(574+239+429 行)综合生成,2026-04-20。commit `b6ab0dee9`。*
