---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, script-execution, vm]
sources: 1
aliases: [FNiagaraScriptExecutionContext, FNiagaraScriptExecutionContextBase, FNiagaraSystemScriptExecutionContext]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScriptExecutionContext.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraScriptExecutionContext

> Niagara **CPU VM 脚本执行上下文**。Phase 3 `FNiagaraEmitterInstance::SpawnExecContext / UpdateExecContext` 的类型。持有脚本关联 + DI 函数表 + 参数存储 + DataSet 绑定,`Execute()` 触发 VectorVM dispatch。

## 一句话角色

**非 UObject** 的 struct 家族:

- `FNiagaraScriptExecutionContextBase`(虚基类,`Tick` 纯虚)
- `FNiagaraScriptExecutionContext : Base`(CPU Emitter Spawn/Update 用)
- `FNiagaraSystemScriptExecutionContext : Base`(System Spawn/Update 用,带 per-instance DI hook)

## 核心字段(Base)

| 字段 | 作用 |
|---|---|
| `UNiagaraScript* Script` | 关联的 Asset |
| `TArray<const FVMExternalFunction*> FunctionTable` | DI 外部函数绑定表(指针) |
| `TArray<void*> UserPtrTable` | per-instance user data 指针 |
| `FNiagaraScriptInstanceParameterStore Parameters` | 参数存储(Phase 4.7 子类) |
| `TArray<FNiagaraDataSetExecutionInfo, TInlineAllocator<2>> DataSetInfo` | DataSet 输入/输出绑定 |
| `HasInterpolationParameters : 1` | 是否有插值参数(CopyCurrToPrev) |
| `bAllowParallel : 1` | 粒子级并行允许(编译器决定) |

## 核心方法

```cpp
virtual bool Init(UNiagaraScript*, ENiagaraSimTarget);
virtual bool Tick(FNiagaraSystemInstance*, ENiagaraSimTarget) = 0;  // 纯虚
void BindData(int32 Index, FNiagaraDataSet&, int32 StartInstance, bool);
bool Execute(uint32 NumInstances, const FScriptExecutionConstantBufferTable&);  // ★ VM dispatch
```

## 子类差异

| 子类 | 特点 |
|---|---|
| `FNiagaraScriptExecutionContext`(普通 Emitter) | `LocalFunctionTable` 存 per-instance DI 函数对象 |
| `FNiagaraSystemScriptExecutionContext`(System) | 持有 `SystemInstances` 指针 + `PerInstanceFunctionHook` hook |

## `PerInstanceFunctionHook`(System 脚本特有)

System 脚本跑 50 instance 时,每个 instance 的 DI 可能不同。VM 遇到 DI 调用 → hook → 按当前 instance 索引找到 `FNiagaraPerInstanceDIFuncInfo { Function, InstData }` → 代为调用。

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraScript]] — 关联 Asset
- [[Wiki/Entities/Stock/Niagara/FNiagaraEmitterInstance]] — 持有 Emitter-level context
- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemSimulation]] — 持有 System-level context
- [[Wiki/Entities/Stock/Niagara/FNiagaraComputeExecutionContext]] — GPU 对位(Phase 8)
- [[Wiki/Entities/Stock/Niagara/FNiagaraParameterStore]]

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/Niagara/NiagaraScriptExecutionContext]]
- 主题读本:[[Readers/UE/Niagara/Phase 5 - Niagara 脚本如何跑起来]] § 2-3
