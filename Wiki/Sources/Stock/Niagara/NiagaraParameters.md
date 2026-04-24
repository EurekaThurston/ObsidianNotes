---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, parameters, editor-only]
sources: 1
aliases: [NiagaraParameters.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraParameters.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraParameters.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraParameters.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 82 行(Phase 4 最小)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 4 — 数据模型 4.6

## 职责

定义 `FNiagaraParameters` USTRUCT——一个**简单的 `TArray<FNiagaraVariable>` 包装**,加一些 editor-only 的 CRUD/merge 工具方法。

> [!note] 与 `FNiagaraParameterStore` 的区别
> `FNiagaraParameters`(本文件)是 editor-only 的简单参数列表,只在编辑期间 / 编译中用来汇总参数。
> `FNiagaraParameterStore`(Phase 4.7,`NiagaraParameterStore.h`)是运行时带 offset 表 + 绑定系统的正牌**参数存储**——两者不是对等物。

## 唯一的 struct

```cpp
// NiagaraParameters.h:12-20
USTRUCT()
struct FNiagaraParameters
{
    GENERATED_USTRUCT_BODY()

    // TODO: Sort the array so we can binary search, do not change to a TMap to avoid memory bloat!
    UPROPERTY(EditAnywhere, EditFixedSize, Category = "Uniform")
    TArray<FNiagaraVariable> Parameters;
    ...
};
```

**TODO 注释很诚实**——作者明确拒绝 TMap(内存占用更多),打算用排序+二分。当前是线性搜索。

## 方法(全部 WITH_EDITORONLY_DATA)

- `NIAGARA_API void Empty()` / `DumpParameters()`
- `void AppendToConstantsTable(uint8* ConstantsTable, const FNiagaraParameters& Externals) const` / overload — 编译后把参数拼到 constants 字节表
- `int32 GetTableSize() const` — 算所有参数占几字节
- `NIAGARA_API FNiagaraVariable* SetOrAdd(const FNiagaraVariable&)` — 改/加
- `NIAGARA_API FNiagaraVariable* FindParameter(FNiagaraVariable)` / `FindParameter(FGuid)` — 两种查找
- `void Merge(FNiagaraParameters&)` — 合并另一份
- `void Allocate()` — 遍历 Parameters 给每个 `VarData` 分配数据(simulation 前必须)
- `void Set(FNiagaraParameters&)` — 清空后整体赋值

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraParameters]] — 本文件的唯一 struct
- [[Wiki/Entities/Stock/Niagara/FNiagaraVariable]] — 元素类型
- [[Wiki/Entities/Stock/Niagara/FNiagaraParameterStore]] — 运行时参数存储(**不是本类**)

## 开放问题

- `SetOrAdd` 的 matching 语义(按 Name?Type+Name?GUID?)→ 看 cpp
- `AppendToConstantsTable` 与 Phase 5 VM constants table 的对应 → Phase 5
- 为什么不直接用 `FNiagaraParameterStore`?→ `FNiagaraParameterStore` 需要 offset 表 + 绑定,editor 编辑期间用不上全部,太重;但也说明有技术债 —— editor 里同时用两套
