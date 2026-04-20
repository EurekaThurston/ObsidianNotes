---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, type-system, data-model]
sources: 1
aliases: [FNiagaraTypeDefinition, Niagara 类型定义]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraTypes.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraTypeDefinition

> Niagara 类型系统的身份牌。每个参数/变量都带一个 TypeDefinition,标识它是 float / Vec3 / UStruct / UEnum / DataInterface 类 等。

## 一句话角色

对外暴露的 Niagara 基础类型:float / int / bool / Vec2/3/4 / Color / Quat / Matrix / Half 家族 / Numeric(多态)/ ParameterMap / 各种 UStruct / UEnum / UClass(DI)。通过 `FNiagaraTypeDefinition::GetFloatDef()` 等静态工厂访问全局单例。

所有 `FNiagaraVariable` 都携带 TypeDef——它决定 DataSet 里占几个 float/int/half component、VM 里占几个寄存器、GPU 里对应什么 HLSL 类型。

## 常用静态工厂

| 工厂方法 | 返回 |
|---|---|
| `GetFloatDef() / GetIntDef() / GetBoolDef()` | 标量 |
| `GetVec2Def() / GetVec3Def() / GetVec4Def()` | 向量 |
| `GetColorDef() / GetQuatDef() / GetMatrix4Def()` | 结构 |
| `GetHalfDef() / GetHalfVec2Def() / HalfVec3Def / HalfVec4Def` | 半精度 |
| `GetNumericDef()` | 多态占位 |
| `GetParameterMapDef()` | 参数图伪类型 |
| `GetExecutionStateEnum()` | `ENiagaraExecutionState` 的 UEnum |

## 相关

- [[Wiki/Entities/Stock/FNiagaraVariable]] — 用本类作身份
- [[Wiki/Entities/Stock/FNiagaraTypeLayoutInfo]] — 描述本类在 DataBuffer 里的 byte layout
- [[Wiki/Entities/Stock/FNiagaraDataSet]] — 按 TypeDef 组织 component

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraTypes]]
- 主题读本:[[Readers/Niagara/Phase4-data-model-读本]] § 2

## 开放问题

- 完整字段(本次未扒 1739 行后半)→ offset 读
