---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, gpu, draw-indirect, compute-shader]
sources: 1
aliases: [FNiagaraDrawIndirect, FNiagaraDrawIndirectArgsGenCS, FNiagaraDrawIndirectResetCountsCS, FNiagaraDrawIndirectArgGenTaskInfo]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraDrawIndirect.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraDrawIndirect 家族

> **GPU 生成 draw indirect args** 的 compute shader。解决"粒子数 GPU 才知道"的问题。合并 `ArgsGenCS` + `ResetCountsCS` + `ArgGenTaskInfo`。

## 一句话角色

- **`FNiagaraDrawIndirectArgGenTaskInfo`**:一条 task 描述,放 `(countOffset, numIndicesPerInstance, startIndex, flags)`
- **`FNiagaraDrawIndirectArgsGenCS`**:compute shader,把 N 个 task 并行转成 `DrawIndirectBuffer`(5 uint32/条)
- **`FNiagaraDrawIndirectResetCountsCS`**:不支持 RW texture buffer 的平台用的 reset-only fallback

## 常量

```cpp
NIAGARA_DRAW_INDIRECT_ARGS_GEN_THREAD_COUNT = 64;
NIAGARA_DRAW_INDIRECT_ARGS_SIZE = 5;          // 对应 (IndexCountPerInstance, InstanceCount, StartIndex, BaseVertex, StartInstance)
NIAGARA_DRAW_INDIRECT_TASK_INFO_SIZE = 4;     // 对应 TaskInfo 4 uint32
```

## `FNiagaraDrawIndirectArgGenTaskInfo`

```cpp
struct FNiagaraDrawIndirectArgGenTaskInfo {
    enum Flags : uint32 {
        Flag_UseCulledCounts = 1 << 0,
        Flag_InstancedStereo = 1 << 1,
    };

    uint32 InstanceCountBufferOffset;
    uint32 NumIndicesPerInstance;              // -1 = 只清 counter
    uint32 StartIndexLocation;
    uint32 Flags;
};
```

**"`NumIndicesPerInstance = -1`" 双用**:同一 task struct 既描述 arg gen,也描述 instance count clear。节省调度开销。

## ArgsGenCS Parameters

```cpp
LAYOUT_FIELD(FShaderResourceParameter, TaskInfosParam);         // SRV: 所有 tasks
LAYOUT_FIELD(FShaderResourceParameter, CulledInstanceCountsParam); // SRV
LAYOUT_FIELD(FRWShaderParameter, InstanceCountsParam);          // UAV: 共享 count buffer
LAYOUT_FIELD(FRWShaderParameter, DrawIndirectArgsParam);        // UAV: 输出
LAYOUT_FIELD(FShaderParameter, TaskCountParam);                 // uint
```

`FSupportsTextureRW` permutation(2)—— 不同平台 RW texture 支持差异。

## 工作流

```
Phase 6 Renderer::CreateRenderThreadResources:
    → InstanceCountMgr.AddDrawIndirect(countOffset, numIndices, ...)
    → 登记 TaskInfo 到 DrawIndirectArgGenTasks

一帧末 InstanceCountMgr.UpdateDrawIndirectBuffer:
    → TaskInfos 上传成 SRV
    → Dispatch FNiagaraDrawIndirectArgsGenCS
    → CS 并行生成 DrawIndirectBuffer 所有条目
    → (clear task 把 count 设 0)

Renderer GetDynamicMeshElements:
    → MeshBatch.IndirectArgsBuffer = DrawIndirectBuffer
    → MeshBatch.IndirectArgsOffset = AddDrawIndirect 返回的 offset
    → UE 最终 RHIDrawIndexedPrimitiveIndirect
```

## 相关

- [[Wiki/Entities/Stock/FNiagaraGPUInstanceCountManager]] — 拥有 tasks 队列
- Phase 6 `FNiagaraRenderer` — 注册 tasks

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraDrawIndirect]]
- 读本:[[Readers/Niagara/Phase8-gpu-simulation-读本]] § 7
