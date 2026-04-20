---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, gpu, instance-count, draw-indirect]
sources: 1
aliases: [NiagaraGPUInstanceCountManager.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraGPUInstanceCountManager.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraGPUInstanceCountManager.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraGPUInstanceCountManager.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 127 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.6

## 职责

`FNiagaraGPUInstanceCountManager` —— **一个全局共享的 GPU 粒子计数 buffer 管理器**。解决:GPU 粒子数量 CPU 不知道(在 GPU 上改),避免频繁 CPU readback。

由 Phase 5 `NiagaraEmitterInstanceBatcher` 持有(`FORCEINLINE FNiagaraGPUInstanceCountManager& GetGPUInstanceCounterManager()`)。

## 核心字段

```cpp
FRWBuffer CountBuffer;                          // 主 count buffer(所有 emitter 的计数混在一起,按 offset 索引)
FRWBuffer CulledCountBuffer;                    // 经 view cull 后的计数

int32 UsedInstanceCounts = 0;                   // 当前已分配
int32 AllocatedInstanceCounts = 0;              // buffer 容量

int32 RequiredCulledCounts = 0;
int32 AllocatedCulledCounts = 0;
bool bAcquiredCulledCounts = false;

TArray<uint32> FreeEntries;                     // 可复用的 offset 池

FRHIGPUMemoryReadback* CountReadback = nullptr;
int32 CountReadbackSize = 0;

TRefCountPtr<FNiagaraGPURendererCount> NumRegisteredGPURenderers;

int32 AllocatedDrawIndirectArgs = 0;
TArray<FNiagaraDrawIndirectArgGenTaskInfo> DrawIndirectArgGenTasks;
TMap<FNiagaraDrawIndirectArgGenTaskInfo, uint32> DrawIndirectArgMap;
TArray<uint32> InstanceCountClearTasks;
FRWBuffer DrawIndirectBuffer;                    // 5-uint32 per entry(per drawcall)
```

## 核心 API

```cpp
// 分配 / 释放 count buffer 中的 entry(offset)
uint32 AcquireEntry();
void FreeEntry(uint32& BufferOffset);                // 把 offset 放回 FreeEntries 池,传入参数设 INDEX_NONE
void FreeEntryArray(TConstArrayView<uint32>);

// Culled count 系统
uint32 AcquireCulledEntry();
FRWBuffer* AcquireCulledCountsBuffer(FRHICommandListImmediate&, ERHIFeatureLevel::Type);

// GPU readback
const uint32* GetGPUReadback();
void ReleaseGPUReadback();
void EnqueueGPUReadback(FRHICommandListImmediate&);
bool HasPendingGPUReadback() const;

// Draw indirect args 生成
uint32 AddDrawIndirect(uint32 InstanceCountBufferOffset, uint32 NumIndicesPerInstance,
                        uint32 StartIndexLocation,
                        bool bIsInstancedStereoEnabled, bool bCulled);

FRWBuffer& GetDrawIndirectBuffer();

void ResizeBuffers(FRHICommandListImmediate&, ERHIFeatureLevel::Type, int32 ReservedInstanceCounts);

void UpdateDrawIndirectBuffer(FRHICommandList&, ERHIFeatureLevel::Type);   // 一帧一次,生成所有 indirect args
```

## `FNiagaraGPURendererCount`(L18)

```cpp
class FNiagaraGPURendererCount : public FRefCountedObject {
    int32 Value = 0;
};
```

所有 GPU renderer 的总数——决定 `DrawIndirectBuffer` 需要多大(每 renderer 最多一条 indirect arg)。`FNiagaraRenderer`(Phase 6)持 `TRefCountPtr`,register/release 时 + / -。

## `kCountBufferDefaultState`

```cpp
static const ERHIAccess kCountBufferDefaultState = ERHIAccess::SRVMask | ERHIAccess::CopySrc;
```

Count buffer 默认状态——可 SRV 读、可复制,但**不可写**。Compute shader 要写时需要 transition。

## Draw Indirect 机制

粒子数量在 GPU 上算(Compute shader 写 count buffer),CPU 不知道具体值。Draw 时用 **`DrawIndexedPrimitiveIndirect`**,参数从 `DrawIndirectBuffer` 里读(5 个 uint32:IndexCountPerInstance / InstanceCount / StartIndex / BaseVertex / StartInstance)。

`AddDrawIndirect` 在 GT 调(Phase 6 Renderer `CreateRenderThreadResources` 里),登记一条 task;一帧 flush 时 `UpdateDrawIndirectBuffer` 把所有 task 转化成 `DrawIndirectBuffer` 内容(通过 `FNiagaraDrawIndirectArgsGenCS` shader,Phase 8.14)。

## 涉及实体

- [[Wiki/Entities/Stock/FNiagaraGPUInstanceCountManager]]
- [[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher]]
- [[Wiki/Entities/Stock/FNiagaraDrawIndirect]](Phase 8.14)
