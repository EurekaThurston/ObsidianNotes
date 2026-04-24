---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, parameter-store, data-model, runtime]
sources: 1
aliases: [FNiagaraParameterStore, Niagara 参数存储, 参数 Store]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraParameterStore.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraParameterStore

> Niagara 运行时的**参数存储**统一容器。同时管理参数 byte 数据 + DataInterface 对象 + UObject 引用,外加 store 间的绑定图。

## 一句话角色

`USTRUCT FNiagaraParameterStore`。Phase 2/3 几乎所有参数字段的类型:`UNiagaraComponent::OverrideParameters`(继承 `FNiagaraUserRedirectionParameterStore`)、`FNiagaraSystemInstance::InstanceParameters`、`FNiagaraEmitterInstance::RendererBindings`……都是它或其子类。

## 三类管理

| 类别 | 字段 |
|---|---|
| 参数 byte 数据 | `TArray<uint8> ParameterData` |
| DataInterface | `TArray<UNiagaraDataInterface*> DataInterfaces` |
| 其他 UObject | `TArray<UObject*> UObjects` |

所有类别通过 `SortedParameterOffsets: TArray<FNiagaraVariableWithOffset>` 找到——同一个 Offset 在三个 TArray 里解释不同。

## 绑定图

```cpp
TArray<BindingPair> Bindings;                  // 我 push 到的 Dest
TArray<FNiagaraParameterStore*> SourceStores;  // 推给我的 Source
```

核心 API:`Bind / Unbind / UnbindAll / Rebind / TransferBindings / Tick`。

`FNiagaraParameterStoreBinding` 内部有三类绑定(Parameter / Interface / UObject),16-bit offset 压缩。

## 三路 Dirty 标记

```cpp
uint32 bParametersDirty : 1;
uint32 bInterfacesDirty : 1;
uint32 bUObjectsDirty : 1;
```

Tick 时按标记增量 push——改了参数不重传 DI,反之亦然。性能关键优化。

## 关键 API

- `AddParameter / RemoveParameter / RenameParameter / Empty / Reset`
- `IndexOf / FindParameterOffset / FindVariable(for DI)`
- 模板 `GetParameterValue<T>` / 裸 `GetParameterData(Offset)` / `GetDataInterface` / `GetUObject`
- `Tick()` — 按 dirty 推到所有 bound stores
- `InitFromSource / TransferBindings` — 迁移

## 子类

- `FNiagaraUserRedirectionParameterStore` — 自动加 `User.` 前缀,Component `OverrideParameters` 的类型
- `FNiagaraScriptExecutionParameterStore`(Phase 5)— VM 执行时的特化

## 相关

- [[Wiki/Entities/Stock/Niagara/FNiagaraVariable]] — 参数身份
- [[Wiki/Entities/Stock/Niagara/UNiagaraComponent]] — 通过子类持有
- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemInstance]] / [[Wiki/Entities/Stock/Niagara/FNiagaraEmitterInstance]] — 持有多个
- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemSimulation]] — 持有 `ScriptDefinedDataInterfaceParameters`

## 深入阅读

- 全字段:[[Wiki/Sources/Stock/Niagara/NiagaraParameterStore]]
- 主题读本:[[Readers/UE/Niagara/Phase 4 - Niagara 的数据语言]] § 7

## 开放问题

- 绑定生命周期 —— Dest Store 析构时谁 unbind?→ cpp
- `LayoutVersion` 变化触发 Rebind 的检测点 → cpp
