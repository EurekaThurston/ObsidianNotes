---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, batcher, gpu, render-thread]
sources: 1
aliases: [NiagaraEmitterInstanceBatcher.h, NiagaraEmitterInstanceBatcher]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterInstanceBatcher.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraEmitterInstanceBatcher.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterInstanceBatcher.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 317 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 5 — CPU 脚本执行 5.5(**实际是 GPU Batcher,Phase 8 详展**)

## 职责

> [!warning] 文件名有误导性
> 文件头注释写 "Queueing and batching for Niagara simulation" —— 看起来像 CPU/GPU 通用。**实际上本类是 GPU Compute 专用的批处理器**——CPU VM 不走 Batcher(CPU 模拟在 `FNiagaraSystemInstance::Tick_Concurrent` 里直接跑)。学习路径把它归在 Phase 5 是为了提前登记存在。

定义 `NiagaraEmitterInstanceBatcher` —— GPU Niagara 的**渲染线程驻留的核心调度器**,继承 `FFXSystemInterface`。职责:

1. 接收 GT 产生的 `FNiagaraGPUSystemTick`,在 RT 按渲染 pipeline 阶段(`ETickStage`)分批 dispatch compute shader
2. 管理 GPU 实例计数(`FNiagaraGPUInstanceCountManager`,Phase 8 详)
3. 管理 GPU 排序(`FGPUSortManager` 接入)
4. 绑定 DI 参数、constant buffer、unbind 资源
5. Dummy UAV 池(懒分配的"占位"UAV)

## 主类声明

```cpp
class NiagaraEmitterInstanceBatcher : public FFXSystemInterface
```

`FFXSystemInterface` 是 UE 通用 FX 的 RT 调度接口(Cascade 的 `FFXSystem` 也实现它)。`GetInterface(FName)` 按 name lookup,让 RT 能按插件 id 找到 Niagara 的 batcher。

## 关键枚举

```cpp
enum class ETickStage { PreInitViews, PostInitViews, PostOpaqueRender, Max };
```

GPU tick 可以在三个渲染阶段执行,依 tick 是否需要 View 数据 / 早期数据 / 深度缓冲决定:
- **PreInitViews** — 最早,view 数据还没有
- **PostInitViews** — 有 view,但 opaque 还没渲
- **PostOpaqueRender** — opaque 渲完,可以读深度/场景纹理

## 生命周期

- GT 通过 `GetInterface(Name)` 拿到 batcher
- GT 每帧产生 `FNiagaraGPUSystemTick`(见 Phase 5.3),调 `GiveSystemTick_RenderThread` 推进 RT 队列
- RT 在各 FFXSystemInterface 钩子(`PreInitViews / PostInitViews / PreRender / PostRenderOpaque`)里**处理对应 Stage 的 tick**

## 核心 RT 入口(FFXSystemInterface 实现)

```cpp
virtual void PreInitViews(FRHICommandListImmediate&, bool bAllowGPUParticleUpdate) override;
virtual void PostInitViews(FRHICommandListImmediate&, FRHIUniformBuffer* ViewUniformBuffer, bool bAllowGPUParticleUpdate) override;
virtual void PreRender(FRHICommandListImmediate&, const FGlobalDistanceFieldParameterData*, bool bAllowGPUParticleUpdate) override;
virtual void PostRenderOpaque(FRHICommandListImmediate&, FRHIUniformBuffer* ViewUniformBuffer, ...) override;
```

`bAllowGPUParticleUpdate` 是整体开关(某些情境下会关,比如 cinematic 录制预渲染)。

对应的查询(告诉渲染器 "我需要这些资源"):

```cpp
virtual bool UsesGlobalDistanceField() const override;
virtual bool UsesDepthBuffer() const override;
virtual bool RequiresEarlyViewUniformBuffer() const override;
```

## Tick 调度内部

```cpp
private:
    struct FDispatchInstance {
        FNiagaraGPUSystemTick* Tick;
        FNiagaraComputeInstanceData* InstanceData;
        int32 StageIndex;
        bool bFinalStage;
    };
    struct FDispatchGroup {
        TArray<FDispatchInstance, TMemStackAllocator<>> DispatchInstances;
        TArray<FRHITransitionInfo, TMemStackAllocator<>> TransitionsBefore / TransitionsAfter;
        ...
    };

    void ExecuteAll(FRHICommandList&, FRHIUniformBuffer*, ETickStage);
    void BuildDispatchGroups(FOverlappableTicks&, FRHICommandList&, FDispatchGroupList&, FEmitterInstanceList&);
    void DispatchStage(FDispatchInstance&, uint32 StageIndex, FRHICommandList&, FRHIUniformBuffer*);
    void DispatchAllOnCompute(FDispatchInstanceList&, FRHICommandList&, FRHIUniformBuffer*);
```

`TMemStackAllocator` 意味着这些临时结构在每帧 stack 分配,不触发 malloc。

## `Run` — 实际 dispatch

```cpp
void Run(const FNiagaraGPUSystemTick& Tick,
         const FNiagaraComputeInstanceData* Instance,
         uint32 UpdateStartInstance, uint32 TotalNumInstances,
         const FNiagaraShaderRef& Shader,
         FRHICommandList& RHICmdList,
         FRHIUniformBuffer* ViewUniformBuffer,
         const FNiagaraGpuSpawnInfo& SpawnInfo,
         bool bCopyBeforeStart = false,
         uint32 DefaultSimulationStageIndex = 0,
         uint32 SimulationStageIndex = 0,
         FNiagaraDataInterfaceProxyRW* IterationInterface = nullptr,
         bool HasRunParticleStage = false);
```

这是 **GPU compute shader 的真正 dispatch 入口**。参数多到吓人,反映了 Niagara GPU tick 的复杂:shader 绑定、spawn info、SimStage 索引、迭代 DI(Phase 10 SimStages)。

## GPU 实例计数器

```cpp
FNiagaraGPUInstanceCountManager GPUInstanceCounterManager;
FORCEINLINE FNiagaraGPUInstanceCountManager& GetGPUInstanceCounterManager() { check(IsInRenderingThread()); ... }
```

GPU 粒子数量维护在一个共享 buffer 上,按 offset 索引——避免每个 Emitter 单独 CPU Readback。Phase 8 的 `NiagaraGPUInstanceCountManager.h` 专门讲。

## 持久化的 Uniform Buffer Layout

```cpp
TRefCountPtr<FNiagaraRHIUniformBufferLayout> GlobalCBufferLayout;
TRefCountPtr<FNiagaraRHIUniformBufferLayout> SystemCBufferLayout;
TRefCountPtr<FNiagaraRHIUniformBufferLayout> OwnerCBufferLayout;
TRefCountPtr<FNiagaraRHIUniformBufferLayout> EmitterCBufferLayout;
```

四种 constant buffer(对应 Phase 5.3 的 5 种 `EUniformBufferType` 的前四种 + External 从 Shader 生成)的 layout 缓存。创建一次,所有 Niagara GPU Tick 复用。

## GPU 排序接入

```cpp
TRefCountPtr<FGPUSortManager> GPUSortManager;
TArray<FNiagaraGPUSortInfo> SimulationsToSort;

bool AddSortedGPUSimulation(FNiagaraGPUSortInfo& SortInfo);
void GenerateSortKeys(FRHICommandListImmediate&, int32 BatchId, int32 NumElementsInBatch, EGPUSortFlags Flags, FRHIUnorderedAccessView* KeysUAV, FRHIUnorderedAccessView* ValuesUAV);
```

半透明粒子需要按深度排序(透明混合正确性)。Niagara 把 sort 任务注册到 `FGPUSortManager`(UE 通用服务),`GenerateSortKeys` 是回调,在 PreRender 阶段由 Manager 调用生成 key/value UAV。详见 Phase 8.7 `NiagaraGPUSortInfo.h`。

## Dummy UAV 池

```cpp
struct DummyUAV { FVertexBufferRHIRef Buffer; FTexture2DRHIRef Texture; FUnorderedAccessViewRHIRef UAV; };
struct DummyUAVPool { int32 NextFreeIndex; TArray<DummyUAV> UAVs; };
mutable TMap<EPixelFormat, DummyUAVPool> DummyBufferPool / DummyTexturePool;

FRHIUnorderedAccessView* GetEmptyRWBufferFromPool(FRHICommandList&, EPixelFormat Format);
FRHIUnorderedAccessView* GetEmptyRWTextureFromPool(FRHICommandList&, EPixelFormat Format);
```

某些 shader 需要绑定"UAV slot",但本次 tick 没实际用——绑一个空 UAV 满足 RHI 验证。池化避免重复分配。

## 其他

- `DeferredIDBufferUpdates` — 需要更新 Free ID buffer 的 emitter 列表
- `UpdateFreeIDBuffers / ClearFreeIDsListSizesBuffer / ResizeFreeIDsListSizesBuffer` — Persistent ID GPU 侧维护
- `PreStageInterface / PostStageInterface / PostSimulateInterface` — DI 的 pre/post stage hooks
- Editor debug:`GpuComputeDebugPtr / GpuDebugReadbackInfos / ProcessDebugReadbacks`

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/NiagaraEmitterInstanceBatcher]] — 本文件主类
- [[Wiki/Entities/Stock/Niagara/FNiagaraGPUSystemTick]] — 被调度的 tick(Phase 8 详)
- [[Wiki/Entities/Stock/Niagara/FNiagaraComputeExecutionContext]] — tick 里引用(Phase 8 详)

## 开放问题(留 Phase 8)

- `FNiagaraGPUInstanceCountManager` 的 buffer 布局 → Phase 8.6
- `FGPUSortManager` 的 batching 规则 → Phase 8.7 `NiagaraGPUSortInfo`
- Overlap compute 机制(`UseOverlapCompute`)→ 看 cpp
- DI 的 pre/post stage hook 执行时机 → Phase 7/10
- Readback manager 的非阻塞读回 → Phase 8
