---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, shader, gpu, compute-shader]
sources: 1
aliases: [NiagaraShader.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShader.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraShader.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShader.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 168 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.2

## 职责

定义 `FNiagaraShader`——**每个 GPU Emitter 对应一个编译后 compute shader**。本类是 `FShader` 子类,持有大量 `LAYOUT_FIELD` shader 参数(粒子 buffer / constant buffer / event buffer / SimStage 参数)。

## 主类

```cpp
class NIAGARASHADER_API FNiagaraShader : public FShader
{
    DECLARE_SHADER_TYPE(FNiagaraShader, Niagara);
    using FPermutationParameters = FNiagaraShaderPermutationParameters;

    static uint32 GetGroupSize(EShaderPlatform Platform)
    {
        return Platform == SP_PS4 ? 64 : 32;  // PS4 用 64,其他 32
    }

    // Bind DI parameters + 大量 LAYOUT_FIELD(见下)
    void BindParams(const TArray<FNiagaraDataInterfaceGPUParamInfo>&, const FShaderParameterMap&);
    const TMemoryImageArray<FNiagaraDataInterfaceParamRef>& GetDIParameters();
};
```

`THREADGROUP_SIZE` define 写入 compilation environment,HLSL 里 `[numthreads(THREADGROUP_SIZE, 1, 1)]`。

## 粒子 Buffer 绑定(CPU 模拟数据 + GPU 双缓冲)

```cpp
LAYOUT_FIELD(FShaderResourceParameter, FloatInputBufferParam);   // SRV
LAYOUT_FIELD(FShaderResourceParameter, IntInputBufferParam);
LAYOUT_FIELD(FShaderResourceParameter, HalfInputBufferParam);
LAYOUT_FIELD(FRWShaderParameter, FloatOutputBufferParam);         // UAV
LAYOUT_FIELD(FRWShaderParameter, IntOutputBufferParam);
LAYOUT_FIELD(FRWShaderParameter, HalfOutputBufferParam);
```

三类(Float/Int/Half)× Input(SRV)/Output(UAV)= 6 个 buffer 参数。对应 Phase 4 `FNiagaraDataBuffer` 的 GPU 侧。

## Instance Count

```cpp
LAYOUT_FIELD(FRWShaderParameter, InstanceCountsParam);              // 全局计数 buffer
LAYOUT_FIELD(FShaderParameter, ReadInstanceCountOffsetParam);       // 读 offset
LAYOUT_FIELD(FShaderParameter, WriteInstanceCountOffsetParam);      // 写 offset
```

对应 Phase 8.6 `FNiagaraGPUInstanceCountManager`。

## Persistent IDs(GPU)

```cpp
LAYOUT_FIELD(FShaderResourceParameter, FreeIDBufferParam);
LAYOUT_FIELD(FRWShaderParameter, IDToIndexBufferParam);
```

对应 Phase 4 `FNiagaraDataBuffer::GPUFreeIDs / GPUIDToIndexTable`。

## Constant Buffers(5 种 × 2 Prev/Curr)

```cpp
LAYOUT_ARRAY(FShaderUniformBufferParameter, GlobalConstantBufferParam, 2);
LAYOUT_ARRAY(FShaderUniformBufferParameter, SystemConstantBufferParam, 2);
LAYOUT_ARRAY(FShaderUniformBufferParameter, OwnerConstantBufferParam, 2);
LAYOUT_ARRAY(FShaderUniformBufferParameter, EmitterConstantBufferParam, 2);
LAYOUT_ARRAY(FShaderUniformBufferParameter, ExternalConstantBufferParam, 2);  // DI + external
LAYOUT_FIELD(FShaderUniformBufferParameter, ViewUniformBufferParam);
```

对应 Phase 5 `EUniformBufferType` 5 种。`[2]` 给当前/上一帧(interpolated spawn)。

## Spawn / Tick 参数

```cpp
LAYOUT_FIELD(FShaderParameter, SimStartParam);
LAYOUT_FIELD(FShaderParameter, EmitterTickCounterParam);
LAYOUT_FIELD(FShaderParameter, EmitterSpawnInfoOffsetsParam);
LAYOUT_FIELD(FShaderParameter, EmitterSpawnInfoParamsParam);
LAYOUT_FIELD(FShaderParameter, CopyInstancesBeforeStartParam);
LAYOUT_FIELD(FShaderParameter, NumSpawnedInstancesParam);
LAYOUT_FIELD(FShaderParameter, UpdateStartInstanceParam);
```

## SimStage(Phase 10)

```cpp
LAYOUT_FIELD(FShaderParameter, DefaultSimulationStageIndexParam);
LAYOUT_FIELD(FShaderParameter, SimulationStageIndexParam);
LAYOUT_FIELD(FShaderParameter, SimulationStageIterationInfoParam);
LAYOUT_FIELD(FShaderParameter, SimulationStageNormalizedIterationIndexParam);
LAYOUT_FIELD(FShaderParameter, DispatchThreadIdToLinearParam);
```

## Event Buffers(最多 4 个并发)

```cpp
LAYOUT_ARRAY(FRWShaderParameter, EventIntUAVParams, MAX_CONCURRENT_EVENT_DATASETS);  // 4
LAYOUT_ARRAY(FRWShaderParameter, EventFloatUAVParams, MAX_CONCURRENT_EVENT_DATASETS);
LAYOUT_ARRAY(FShaderResourceParameter, EventIntSRVParams, MAX_CONCURRENT_EVENT_DATASETS);
LAYOUT_ARRAY(FShaderResourceParameter, EventFloatSRVParams, MAX_CONCURRENT_EVENT_DATASETS);
// 读写 stride 各 × 4
```

`MAX_CONCURRENT_EVENT_DATASETS = 4`。Phase 3 的 event 系统在 GPU 侧的对位。

## DI 参数

```cpp
LAYOUT_FIELD(TMemoryImageArray<FNiagaraDataInterfaceParamRef>, DataInterfaceParameters);
```

每个 DI 一条。

## 涉及实体

- [[Wiki/Entities/Stock/FNiagaraShader]](合并 FNiagaraShaderType + FNiagaraShaderMap)
