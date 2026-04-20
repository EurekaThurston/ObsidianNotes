---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, accessor, template, data-model]
sources: 1
aliases: [FNiagaraDataSetAccessor, FNiagaraDataSetReader, FNiagaraDataSetWriter]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataSetAccessor.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraDataSetAccessor

> **模板家族**:C++ 代码类型安全地读写 `FNiagaraDataSet` 的粒子属性。通过 `TypeInfo<T>` 特化选择 Reader/Writer 基类。

## 一句话角色

用户用 `FNiagaraDataSetAccessor<FVector>(DataSet, "Position")` 就能拿到一个按 FVector 读取 Position 属性的 accessor。构造一次(查变量 offset),之后 `Reader[i]` 是 O(1)。

## 三个家族

| 家族 | Reader | Writer | 支持类型 |
|---|---|---|---|
| Float | `FNiagaraDataSetReaderFloat<T>` | ~~`Writer`~~(注释掉) | `float / FVector2D / FVector / FVector4 / FLinearColor / FQuat` |
| Int32 | `FNiagaraDataSetReaderInt32<T>` | `FNiagaraDataSetWriterInt32<T>` | `int32 / FNiagaraBool / ENiagaraExecutionState` |
| Struct | `FNiagaraDataSetReaderStruct<T>` | 无 | `FNiagaraSpawnInfo / FNiagaraID`(默认)+ 用户扩展 |

## Half 支持

Float 家族内:`float / Vec2 / Vec3 / Vec4` 支持 half(自动 `LoadHalf` 转 float);`Color / Quat` **不支持**(check(false))。

## 扩展用户类型

给自己的 UStruct 添加 Accessor 支持,需要特化 `FNiagaraDataSetAccessorTypeInfo<YourStruct>` 提供:
- `NumFloatComponents / NumInt32Components`
- `static FNiagaraTypeDefinition GetStructType()`
- `static YourStruct ReadStruct(int32, const float*const*, const int32*const*)`

## 陷阱

- ⚠️ Float writer 被**注释掉**,不能用 C++ 直接写粒子 float 属性(设计选择:属性由 VM 脚本写)
- ⚠️ `Reader` 基于 `GetCurrentData()`,`Writer` 基于 `GetDestinationData()` ——后者只在 `BeginSimulate/EndSimulate` 之间有效

## 相关

- [[Wiki/Entities/Stock/FNiagaraDataSet]] — 操作对象
- [[Wiki/Entities/Stock/FNiagaraVariable]] — 按名字查
- [[Wiki/Entities/Stock/FNiagaraTypeDefinition]] — TypeInfo 基于此

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraDataSetAccessor]]
- 主题读本:[[Readers/Niagara/Phase4-data-model-读本]] § 6
