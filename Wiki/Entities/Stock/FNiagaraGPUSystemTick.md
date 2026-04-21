---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, gpu-tick, render-thread]
sources: 1
aliases: [FNiagaraGPUSystemTick]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScriptExecutionContext.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraGPUSystemTick

> **GT 产生、RT 消费**的 GPU tick 打包结构。代表单次 `FNiagaraSystemInstance` 的 GPU 更新所需的全部信息。**Phase 8 详**。

## 一句话角色

`class FNiagaraGPUSystemTick`(非 UObject)。每帧每个有 GPU emitter 的 System Instance 产生一份,送到 Batcher 的 RT 队列。

持有:
- 每个 Emitter 的 `FNiagaraComputeInstanceData` 列表
- 所有 DI 的 per-instance 数据(`FNiagaraDataInterfaceInstanceData`)
- 4 路 Global/System/Owner/Emitter 参数数据 + 5 路 Uniform Buffer

## 5 种 Uniform Buffer 类型

```cpp
enum EUniformBufferType {
    UBT_Global,       // System-level:time, quality, count 等
    UBT_System,       // System-level:system age, num emitters
    UBT_Owner,        // System-level:owner transform, velocity
    UBT_Emitter,      // Per-Emitter
    UBT_External,     // DI 或 script 额外的 custom cbuffer
    UBT_NumTypes = 5
};
```

## 核心字段

| 字段 | 作用 |
|---|---|
| `SystemInstanceID` | 64-bit 身份 |
| `SharedContext` | GT/RT 共享状态 |
| `DIInstanceData` | DI per-instance 数据(打包) |
| `InstanceData_ParamData_Packed` | 所有 Emitter 数据 + 参数紧凑 packed |
| `GlobalParamData / SystemParamData / OwnerParamData` | 3 种 System-level 参数 |
| `UniformBuffers` | RT 侧创建的 UniformBuffer refs |
| `Count / TotalDispatches / NumInstancesWithSimStages` | 调度统计 |
| `bRequiresDistanceFieldData / DepthBuffer / EarlyViewData / ViewUniformBuffer` | 依赖查询 |
| `bNeedsReset / bIsFinalTick` | 状态标记 |

## `InstanceData_ParamData_Packed` 布局

```
+---------------------------------------+
| uint32 EmitterDispatchCount (count)   |
+---------------------------------------+
| FNiagaraComputeInstanceData[0]        |
| FNiagaraComputeInstanceData[1]        |
| ...                                    |
+---------------------------------------+
| 16-byte aligned padding                |
+---------------------------------------+
| ParamData blob(GlobalParamData 等)    |
+---------------------------------------+
```

16-byte 对齐是为了 ParamData 能直接 upload 成 UniformBuffer。

## 相关

- [[Wiki/Entities/Stock/FNiagaraSystemInstance]] — `friend class FNiagaraGPUSystemTick`
- [[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher]] — RT 消费者
- [[Wiki/Entities/Stock/FNiagaraComputeExecutionContext]] — 被引用

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraScriptExecutionContext]] § 块 D
- 主题读本:[[Readers/Niagara/Phase 5 - Niagara 脚本如何跑起来]] § 5(Phase 8 详展)
