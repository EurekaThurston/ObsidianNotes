---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, module, core]
sources: 1
aliases: [NiagaraCore.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraCore/Public/NiagaraCore.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraCore.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraCore/Public/NiagaraCore.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: **6 行**(仅一个 typedef)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 5 — CPU 脚本执行 5.1

## 职责

定义 **`NiagaraCore` 模块**的公开入口头文件。内容出奇地少:

```cpp
#pragma once

typedef uint64 FNiagaraSystemInstanceID;
```

## 为什么 NiagaraCore 模块存在

Niagara 运行时模块树结构:

```
NiagaraCore  ← 最底层,只含 DI 抽象基类 + 极少类型(本文件)
    ↑
Niagara      ← 主模块,242 文件,运行时和绝大多数业务代码
NiagaraShader
NiagaraVertexFactories
```

`NiagaraCore` 存在的理由:**让 RHI / Shader 模块能依赖 DI 接口而不拖累主 Niagara 模块**。例如 `FNiagaraShader` 需要绑定 DI 参数 —— 它依赖 `UNiagaraDataInterfaceBase`(NiagaraCore 模块),不需要依赖 `UNiagaraSystem`(Niagara 主模块)。这是典型的**接口/实现模块分离**。

NiagaraCore 模块的全部内容:
- `NiagaraCore.h` — typedef `FNiagaraSystemInstanceID = uint64`(本文件)
- `NiagaraDataInterfaceBase.h` — DI 抽象基类 + CS 参数绑定框架
- `NiagaraMergeable.h` — Niagara 可合并对象基类(DI 继承它)
- 及少量其他共享类型(~10 个文件)

## `FNiagaraSystemInstanceID`

```cpp
typedef uint64 FNiagaraSystemInstanceID;
```

单个 Niagara System Instance 的 **64-bit 全局唯一 ID**,Phase 3 `FNiagaraSystemInstance::ID` 就是这个。用于崩溃报告标签、GPU tick 打包、DI per-instance 数据索引等场景。跨进程/世界都不重复(大约按 incremental + mix 生成)。

## 涉及实体

- [[Wiki/Entities/Stock/FNiagaraSystemInstance]] — 持有 `ID` 字段
- 几乎所有 GPU 相关结构(Phase 8)都以此 ID 作 key

## 开放问题

- ID 具体怎么生成(uint64 空间分配、跨进程唯一性)?→ `.cpp`
