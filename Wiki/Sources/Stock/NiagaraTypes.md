---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, types, type-system, data-model]
sources: 1
aliases: [NiagaraTypes.h, Niagara 类型系统源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraTypes.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraTypes.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraTypes.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 1739 行(Phase 4 最大)
- **同仓协作文件**: `NiagaraTypes.cpp`、`NiagaraCommon.h`、`NiagaraConstants.h`、`NiagaraDataSet.h`(layout 由本文件类型系统驱动)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 4 — 数据模型 4.1

## 职责

定义 Niagara 的**类型系统**——Niagara 脚本能操作的所有基本类型、Variable 抽象、Variable 的 Layout 信息、编译哈希访问者、执行状态枚举。Phase 4 的其他文件都依赖本文件的 `FNiagaraTypeDefinition` / `FNiagaraVariable` / `FNiagaraTypeLayoutInfo` 三大类。

## 关键类型

### 基础值类型(USTRUCT 包装基础类型)

- `FNiagaraFloat`(L21):`float Value`
- `FNiagaraInt32`(L30):`int32 Value`
- `FNiagaraBool`(L39):**特殊** —— VM 约定 True=`INDEX_NONE`(-1)/False=`0`,方便 VM compare+select 运算
- `FNiagaraHalf`(L70):`uint16 Value`(半精度存储)
- `FNiagaraHalfVector2/3/4`(L79, L91, L106):三个半精度向量
- `FNiagaraNumeric`(L124):空 USTRUCT,标记 "数值多态类型"(脚本中运算符自适应)
- `FNiagaraParameterMap`(L131):空 USTRUCT,标记 "参数图" 伪类型(脚本中参数流的汇总)
- `FNiagaraMatrix`(L138):4×FVector4 行

### 特殊数据结构

- `FNiagaraSpawnInfo`(L157):BlueprintType,`Count` + `InterpStartDt` + `IntervalDt` + `SpawnGroup` —— 一条 spawn 调度指令;`NiagaraClearEachFrame="true"` 元标志每帧清
- `FNiagaraID`(L180):BlueprintType,`Index` + `AcquireTag` —— **持久 ID 双整数** 机制(粒子死后 Index 复用,AcquireTag 区别历史)

### 类型 Layout(核心)

- `FNiagaraTypeLayoutInfo`(L215):USTRUCT。一个类型在 DataBuffer/Register 中如何排布:
  ```cpp
  TArray<uint32> FloatComponentByteOffsets;   // 每个 float 成员在结构体内的 byte offset
  TArray<uint32> FloatComponentRegisterOffsets;  // 每个 float 在 register table 中的 offset
  TArray<uint32> Int32ComponentByteOffsets;
  TArray<uint32> Int32ComponentRegisterOffsets;
  TArray<uint32> HalfComponentByteOffsets;
  TArray<uint32> HalfComponentRegisterOffsets;
  ```
  - 静态方法 `GenerateLayoutInfo(Layout, Struct)`(L246)通过反射遍历 UStruct 字段自动生成 layout(处理 nested struct 递归)。这是 Phase 1 `FNiagaraEmitterCompiledData` 的生成源。

### `FNiagaraTypeDefinition` / `FNiagaraVariable`(文件后半)

按代码推断(本次未完整读 1739 行,从用法可确认):
- `FNiagaraTypeDefinition` —— 类型的身份(UStruct/UEnum/UClass/基础类型之一 + 可选限定)
- `FNiagaraVariableBase` —— `TypeDef + Name`
- `FNiagaraVariable : FNiagaraVariableBase` —— 带默认值数据(`TArray<uint8> VarData`)
- `FNiagaraVariableWithOffset : FNiagaraVariableBase` —— 带 offset(见 `NiagaraParameterStore.h`)
- `FNiagaraVariableMetaData` —— UI 展示元数据(Description / Category)

常用静态 factory:`FNiagaraTypeDefinition::GetFloatDef() / GetVec2Def() / GetVec3Def() / GetVec4Def() / GetColorDef() / GetQuatDef() / GetIntDef() / GetBoolDef() / GetHalfDef() / GetHalfVec2Def() / GetHalfVec3Def() / GetHalfVec4Def() / GetExecutionStateEnum()`。

### 执行状态枚举

- `ENiagaraExecutionState`(L330):`Active` / `Inactive` / `InactiveClear` / `Complete` / `Disabled`(Phase 3 SystemInstance/EmitterInstance 都用这个)
- `ENiagaraExecutionStateSource`(L321):**状态源 precedence 系统**——`Scalability < Internal < Owner < InternalCompletion`。低 precedence 的源不能覆盖高 precedence 已设的状态。这解释了 Phase 3 双状态机(Requested vs Actual)的底层机制

### 数值输出模式

- `ENiagaraNumericOutputTypeSelectionMode`(L304):`None / Largest / Smallest / Scalar`——函数基于 Numeric 输入选择输出类型的策略(脚本中 `1+2.5` 该是 int 还是 float)

### 编译哈希

- `FNiagaraCompileHashVisitor`(L367):访问者模式,走遍编译相关字段计算 SHA1——Phase 1 DDC key 的产生器
- `FNiagaraCompileHashVisitorDebugInfo`(L350):Editor-only 调试数据

### 日志类别

- `DECLARE_LOG_CATEGORY_EXTERN(LogNiagara, Log, Verbose)`(L16)

## 依赖与被依赖

**上游**:`Engine/UserDefinedStruct.h`、`Misc/SecureHash.h`、`UObject/GCObject.h`
**下游**:几乎所有 Niagara 头文件——类型系统是基石

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/FNiagaraTypeDefinition]]
- [[Wiki/Entities/Stock/FNiagaraVariable]]
- [[Wiki/Entities/Stock/FNiagaraTypeLayoutInfo]]
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — Half/Float/Int32 三类就是 VM/GPU 的三大寄存器类型

## 开放问题

- `FNiagaraTypeDefinition` 具体字段(本次未完整扒 1739 行后半段)→ 可 offset 读
- `FNiagaraVariableMetaData` 完整字段 → 同上
- `GenerateLayoutInfoInternal` 对 nested struct 递归 —— Matrix(4×Vec4)layout 该长什么样 → 看 cpp 或示例
