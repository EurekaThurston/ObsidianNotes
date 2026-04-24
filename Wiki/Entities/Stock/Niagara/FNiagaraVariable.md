---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, variable, data-model]
sources: 1
aliases: [FNiagaraVariable, FNiagaraVariableBase, Niagara 变量]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraTypes.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraVariable

> Niagara 世界里**命名的一段数据**。`TypeDefinition + Name` 组成 `FNiagaraVariableBase`,带上默认值数据就是 `FNiagaraVariable`。

## 一句话角色

- `FNiagaraVariableBase`:`FNiagaraTypeDefinition Type` + `FName Name`——纯身份,无数据
- `FNiagaraVariable : FNiagaraVariableBase`:+ `TArray<uint8> VarData`(默认值数据)
- `FNiagaraVariableWithOffset : FNiagaraVariableBase`:+ `int32 Offset`(用于 `FNiagaraParameterStore::SortedParameterOffsets`)
- `FNiagaraVariableMetaData`:UI 展示(Description / Category / Properties / MetaData)

`FNiagaraVariable` 几乎无处不在:Asset `ExposedParameters` 里的参数、DataSet 里的粒子属性、VM 脚本的 Input/Output、DI 的 function signature 参数……全都是 `FNiagaraVariable`。

## 命名约定(§ NiagaraConstants)

Name 通常带命名空间前缀:

- `User.MyParam`
- `Engine.DeltaTime`
- `Particles.Position`
- `Emitter.Age`
- `System.NumEmittersAlive`
- `Module.LocalVar`
- `Initial.Particles.Position`(粒子初始值)
- `Previous.Particles.Position`(上帧值)

## 常用方法

- `GetSizeInBytes() const` — 通过 TypeDef 算
- `AllocateData()` — 给 VarData 按 size 分配
- `const uint8* GetData() const` / `uint8* GetData()` — 访问 raw byte
- `SetValue<T>(T)` / `T GetValue<T>() const` — 按模板 reinterpret
- `operator==` / `GetTypeHash` — 用于 TMap 键

## 相关

- [[Wiki/Entities/Stock/Niagara/FNiagaraTypeDefinition]] — 身份组成
- [[Wiki/Entities/Stock/Niagara/FNiagaraParameterStore]] — 运行时容器
- [[Wiki/Entities/Stock/Niagara/FNiagaraConstants]] — 预注册的命名空间变量

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/Niagara/NiagaraTypes]]
- 主题读本:[[Readers/UE/Niagara/Phase 4 - Niagara 的数据语言]] § 2-3
