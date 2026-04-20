---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, common, enums, shared-structures]
sources: 1
aliases: [NiagaraCommon.h, Niagara 共享头]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraCommon.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraCommon.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraCommon.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 1200 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 4 — 数据模型 4.2

## 职责

Niagara 的**共享声明集合**——常量、大量 enum、跨多个子系统用到的小型 USTRUCT(DataSetID、FunctionSignature、SpawnInfo-like)。几乎所有 Niagara 头文件都 `include` 它。

## 关键常量

```cpp
#define NIAGARA_NAN_CHECKING 0                                      // L25
#define INTERPOLATED_PARAMETER_PREFIX TEXT("PREV_")                 // L27
constexpr uint32 NiagaraComputeMaxThreadGroupSize = 64;             // L30 GPU thread group 上限
constexpr uint32 NiagaraMaxThreadGroupCountPerDimension = 65535;    // L33
constexpr uint32 NIAGARA_MAX_GPU_SPAWN_INFOS = 8;                   // L36 GPU 可并行的 spawn info 上限
constexpr ETickingGroup NiagaraFirstTickGroup = TG_PrePhysics;      // L39
constexpr ETickingGroup NiagaraLastTickGroup = TG_LastDemotable;    // L40
constexpr int NiagaraNumTickGroups = ...;                           // L41
```

## 关键枚举

- `ENiagaraTickBehavior`(L45):`UsePrereqs / UseComponentTickGroup / ForceTickFirst / ForceTickLast` —— Phase 2 Component 用
- `ENiagaraBaseTypes`(L57):`NBT_Half / NBT_Float / NBT_Int32 / NBT_Bool / NBT_Max` —— VM/GPU 寄存器基础分类
- `ENiagaraGpuBufferFormat`(L68):`Float / HalfFloat / UnsignedNormalizedByte` —— GPU buffer 位精度选项
- `ENiagaraDefaultMode`(L83):`Value / Binding / Custom` —— 默认值来源
- `ENiagaraSimTarget`(L94):**`CPUSim` / `GPUComputeSim`** —— Asset 的 SimTarget 字段在这定义
- `ENiagaraAgeUpdateMode`(L103):`TickDeltaTime / DesiredAge / DesiredAgeNoSeek` —— Phase 2 Component 的时间控制
- `ENiagaraDataSetType`(L131):`ParticleData / Shared / Event`
- `ENiagaraInputNodeUsage`(L139):`Undefined / Parameter / Attribute / SystemConstant / TranslatorConstant / RapidIterationParameter`
- `ENiagaraScriptCompileStatus`(L153):`NCS_Unknown / Dirty / Error / UpToDate / BeingCreated / UpToDateWithWarnings / ComputeUpToDateWithWarnings`
- 更多(文件后半未读):`ENiagaraScriptUsage`、`ENiagaraSystemSimulationScript`、`ENiagaraBindingSource` 等

## 关键结构体

### `FNiagaraDataSetID`(L173)

USTRUCT,`FName Name` + `ENiagaraDataSetType Type`——两个 DataSet 的身份键(Phase 3 `EmitterEventDataSetMap` 的 key 用这个)。

### `FNiagaraDataSetProperties`(L217)

USTRUCT,`ID` + `TArray<FNiagaraVariable> Variables`。

### `FNiagaraFunctionSignature`(L269) — 大结构体

USTRUCT,NIAGARA_API。VM/DI 函数的完整签名:

```cpp
FName Name;                                     // 函数名
TArray<FNiagaraVariable> Inputs;                // 输入参数
TArray<FNiagaraVariable> Outputs;               // 输出参数
FName OwnerName;                                // 是否有 owner(member function)
uint32 bRequiresContext : 1;                    // 需要执行上下文
uint32 bRequiresExecPin : 1;                    // 有副作用,不能被 VM 优化掉
uint32 bMemberFunction : 1;                     // 成员函数(第一个输入是 owner)
uint32 bExperimental : 1;
uint32 bSupportsCPU : 1;
uint32 bSupportsGPU : 1;
uint32 bWriteFunction : 1;
uint32 bSoftDeprecatedFunction : 1;
int32 ModuleUsageBitmask;                       // 哪些 script usage 允许用
int32 ContextStageMinIndex / ContextStageMaxIndex;  // SimStages 相关(Phase 10)
TMap<FName, FName> FunctionSpecifiers;          // 编译期 specifier(如采样模式)
```

这是 Phase 7 DataInterface 注册函数时用的核心结构;Phase 5 VM 脚本中调用 DI 函数也靠它匹配。

### `FNiagaraOpInOutInfo`(L229)

普通 C++,VM op 的 input/output 信息:Name + DataType + FriendlyName + Description + Default + HlslSnippet。

### `FNiagaraScriptDataUsageInfo`(L254)

USTRUCT,描述脚本用了什么(`bReadsAttributeData`)——编译优化提示。

## 文件后半未完整读的内容(按推断)

按 Niagara 代码库的一般组织,`NiagaraCommon.h` 后半还包括:
- 更多 `ENiagaraScriptUsage` / `ENiagaraSystemSimulationScript` 等 script 上下文枚举
- VM 相关:`FNiagaraCompileEvent` / `FVMExternalFunction` 类型别名
- Scalability 相关:`FNiagaraEmitterScalabilitySettings` / `FNiagaraSystemScalabilitySettings`
- Per-instance DI:`FNiagaraPerInstanceDIFuncInfo`

这些留在 Phase 5/7/9 深入时再看。

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/FNiagaraFunctionSignature]] — 本文件的核心结构体
- [[Wiki/Entities/Stock/FNiagaraDataSetID]] — DataSet 身份
- 大量 enum 分散引用于:[[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]](SimTarget)、Phase 2(TickBehavior/AgeUpdateMode)、Phase 3(ExecutionState)

## 开放问题

- 文件后半的 scalability/VM 相关结构(本次未扒)→ 实际需要时 offset 读
- `ENiagaraScriptUsage` 与 Phase 1 `UNiagaraScript` 的对应 → Phase 5 可能重访
