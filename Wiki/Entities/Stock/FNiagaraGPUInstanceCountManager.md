---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, gpu, instance-count, draw-indirect]
sources: 1
aliases: [FNiagaraGPUInstanceCountManager]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraGPUInstanceCountManager.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraGPUInstanceCountManager

> **全局 GPU 粒子计数管理器**。解决"GPU 上改 count,CPU 不知道"——维护一个共享 count buffer,按 offset 索引,用 draw indirect 避免 readback。

## 一句话角色

`class FNiagaraGPUInstanceCountManager`。由 Phase 5 `NiagaraEmitterInstanceBatcher` 持有,RT 驻留。

## 核心 Buffer

| Buffer | 用途 |
|---|---|
| `CountBuffer`(`FRWBuffer`) | 所有 emitter 的粒子 count,按 offset 共享 |
| `CulledCountBuffer` | view-culled 后的 count |
| `DrawIndirectBuffer` | 最终 draw indirect args(每条 5 uint32) |
| `CountReadback` | 异步 GPU→CPU readback(stats 用) |

## 核心 API

```cpp
uint32 AcquireEntry();                                  // 分配 offset
void FreeEntry(uint32& BufferOffset);                   // 归还,参数置 INDEX_NONE
void FreeEntryArray(TConstArrayView<uint32>);

uint32 AddDrawIndirect(uint32 InstanceCountBufferOffset, uint32 NumIndicesPerInstance,
                        uint32 StartIndexLocation,
                        bool bIsInstancedStereoEnabled, bool bCulled);  // 登记 draw indirect task
void UpdateDrawIndirectBuffer(FRHICommandList&, ERHIFeatureLevel::Type);  // 一帧一次,flush 所有 task

void ResizeBuffers(FRHICommandListImmediate&, ERHIFeatureLevel::Type, int32 ReservedInstanceCounts);

const uint32* GetGPUReadback();                         // 读回,stats 用
void EnqueueGPUReadback(FRHICommandListImmediate&);
```

## Draw Indirect 机制

GPU 粒子数量只 GPU 知道 → draw call 要"按需"发:

```cpp
// CPU-side indirect args buffer 里每条:
struct { uint32 IndexCountPerInstance, InstanceCount, StartIndex, BaseVertex, StartInstance; }
```

`Phase 6 Renderer::CreateRenderThreadResources` → `AddDrawIndirect` → 一帧末 `UpdateDrawIndirectBuffer` → 通过 `FNiagaraDrawIndirectArgsGenCS`(Phase 8.14)compute shader 把 GPU count 翻译成 indirect args。

## `FNiagaraGPURendererCount`

引用计数小类,`TRefCountPtr` 被 `FNiagaraRenderer`(Phase 6)和 `FNiagaraEmitterInstance`(Phase 3)持有。总数决定 `DrawIndirectBuffer` 容量。

## 相关

- [[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher]] — 持有者
- [[Wiki/Entities/Stock/FNiagaraDrawIndirect]] — 对偶 compute shader
- [[Wiki/Entities/Stock/FNiagaraDataBuffer]] — `GPUInstanceCountBufferOffset` 引用本 manager 的 offset

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraGPUInstanceCountManager]]
- 读本:[[Readers/Niagara/Phase8-gpu-simulation-读本]] § 4
