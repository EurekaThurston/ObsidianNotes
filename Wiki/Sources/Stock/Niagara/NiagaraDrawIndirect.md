---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, gpu, draw-indirect, compute-shader]
sources: 1
aliases: [NiagaraDrawIndirect.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraDrawIndirect.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDrawIndirect.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraDrawIndirect.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 130 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.14

## 职责

定义两个 compute shader(`FNiagaraDrawIndirectArgsGenCS` + `FNiagaraDrawIndirectResetCountsCS`)和任务描述 `FNiagaraDrawIndirectArgGenTaskInfo`。把 Phase 8.6 `FNiagaraGPUInstanceCountManager::DrawIndirectArgGenTasks` 在 GPU 上转化成真正的 draw indirect buffer。

## 常量

```cpp
#define NIAGARA_DRAW_INDIRECT_ARGS_GEN_THREAD_COUNT 64
#define NIAGARA_DRAW_INDIRECT_ARGS_SIZE 5                    // indirect arg 是 5 uint32
#define NIAGARA_DRAW_INDIRECT_TASK_INFO_SIZE 4               // task info 是 4 uint32
```

`NIAGARA_DRAW_INDIRECT_ARGS_SIZE = 5` 对应 `(IndexCountPerInstance, InstanceCount, StartIndex, BaseVertex, StartInstance)`。

## `FNiagaraDrawIndirectArgGenTaskInfo`(L26)

```cpp
struct FNiagaraDrawIndirectArgGenTaskInfo {
    enum Flags : uint32 {
        Flag_UseCulledCounts = 1 << 0,
        Flag_InstancedStereo = 1 << 1,
    };

    uint32 InstanceCountBufferOffset;
    uint32 NumIndicesPerInstance;          // -1 时意味着 "只清 counter,不生成 draw arg"
    uint32 StartIndexLocation;
    uint32 Flags;
};
```

"NumIndicesPerInstance = -1" 的双用——同一 task struct 既描述 draw arg gen,也描述 instance count clear。

## `FNiagaraDrawIndirectArgsGenCS`(L69)

```cpp
class FNiagaraDrawIndirectArgsGenCS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FNiagaraDrawIndirectArgsGenCS);

    class FSupportsTextureRW : SHADER_PERMUTATION_INT("SUPPORTS_TEXTURE_RW", 2);
    using FPermutationDomain = TShaderPermutationDomain<FSupportsTextureRW>;

    void SetOutput(FRHICommandList&, FRHIUnorderedAccessView* DrawIndirectArgsUAV, FRHIUnorderedAccessView* InstanceCountsUAV);
    void SetParameters(FRHICommandList&,
                       FRHIShaderResourceView* TaskInfosBuffer,
                       FRHIShaderResourceView* CulledInstanceCountsBuffer,
                       int32 NumArgGenTasks, int32 NumInstanceCountClearTasks);
    void UnbindBuffers(FRHICommandList&);

    LAYOUT_FIELD(FShaderResourceParameter, TaskInfosParam);
    LAYOUT_FIELD(FShaderResourceParameter, CulledInstanceCountsParam);
    LAYOUT_FIELD(FRWShaderParameter, InstanceCountsParam);
    LAYOUT_FIELD(FRWShaderParameter, DrawIndirectArgsParam);
    LAYOUT_FIELD(FShaderParameter, TaskCountParam);
};
```

### 工作

每 thread 处理一个 task:
1. 读 `TaskInfos[tid]` 拿到 `(InstanceCountOffset, NumIndicesPerInstance, StartIndex, Flags)`
2. 从 `InstanceCounts[InstanceCountOffset]`(或 Culled counts)读实际粒子数
3. 写到 `DrawIndirectArgs[tid * 5]`(5 uint32)
4. 如果 `NumIndicesPerInstance == -1`,改为把 count 置 0(clear)

## `FNiagaraDrawIndirectResetCountsCS`(L104)

```cpp
class FNiagaraDrawIndirectResetCountsCS : public FGlobalShader
{
    // 同上 + 不支持 RW texture 的平台用这个
};
```

Fallback 路径:不支持 RW texture buffer 的平台用纯 buffer 版。

## 工作流

```
Phase 6 FNiagaraRenderer::CreateRenderThreadResources:
    → InstanceCountMgr.AddDrawIndirect(countOffset, numIndicesPerInstance, ...)
    → 登记到 DrawIndirectArgGenTasks

一帧 flush 时 InstanceCountMgr.UpdateDrawIndirectBuffer:
    → 把 DrawIndirectArgGenTasks 上传成 SRV(TaskInfosBuffer)
    → Dispatch FNiagaraDrawIndirectArgsGenCS(thread count = NumTasks)
    → CS 并行生成 DrawIndirectBuffer 所有条目

Renderer 在 GetDynamicMeshElements:
    → MeshBatch 设 IndirectArgs = DrawIndirectBuffer, offset = AddDrawIndirect 返回的 offset
    → UE 渲染器最终调 RHIDrawIndexedPrimitiveIndirect
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraDrawIndirect]]
- [[Wiki/Entities/Stock/Niagara/FNiagaraGPUInstanceCountManager]]
