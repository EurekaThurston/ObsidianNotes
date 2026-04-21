---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, type-layout, data-model]
sources: 1
aliases: [FNiagaraTypeLayoutInfo]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraTypes.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraTypeLayoutInfo

> USTRUCT。把一个 Niagara 类型(通常对应一个 UStruct)**拆成** Float / Int32 / Half 三类 component 的 byte offset 数组 + register offset 数组。

## 一句话角色

Niagara 类型系统"**把任何数据拍平成三路 SoA**"的编译期产物。例如 `FVector`(3 个 float)生成 `FloatComponentByteOffsets = [0, 4, 8]`(对应 x/y/z 在 struct 内偏移)。

## 字段

```cpp
TArray<uint32> FloatComponentByteOffsets;
TArray<uint32> FloatComponentRegisterOffsets;
TArray<uint32> Int32ComponentByteOffsets;
TArray<uint32> Int32ComponentRegisterOffsets;
TArray<uint32> HalfComponentByteOffsets;
TArray<uint32> HalfComponentRegisterOffsets;
```

两套 offset 的区别:
- **ByteOffsets** 是在**结构体内**的 byte 位置(用于把 UStruct 字段读进 DataBuffer)
- **RegisterOffsets** 是在**VM 寄存器表**里的索引(用于生成 VM 指令)

## 静态方法

```cpp
static void GenerateLayoutInfo(FNiagaraTypeLayoutInfo& Layout, const UScriptStruct* Struct);
```

通过反射遍历 UStruct 的 FProperty 自动生成 layout,递归处理 nested struct。是 Phase 1 `FNiagaraEmitterCompiledData` / `FNiagaraDataSetCompiledData` 的生成源。

## 相关

- [[Wiki/Entities/Stock/FNiagaraTypeDefinition]] — 类型身份,本类是其伴生数据
- [[Wiki/Entities/Stock/FNiagaraDataSet]] — 按本类的 offset 组织 SoA
- [[Wiki/Entities/Stock/FNiagaraVariable]]

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraTypes]]
- 主题读本:[[Readers/Niagara/Phase 4 - Niagara 的数据语言]] § 3
