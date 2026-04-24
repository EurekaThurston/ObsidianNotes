---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, shader-map, stub]
sources: 1
aliases: [NiagaraShaderMap.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShaderMap.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraShaderMap.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShaderMap.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: **8 行**(stub 文件)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.4

## 职责

实质上是**空文件**。注释是 "MaterialShared.h: Shared material definitions."(复制错了,明显是占位)。只有 `#pragma once` 和空行。

```cpp
// Copyright Epic Games, Inc. All Rights Reserved.

/*=============================================================================
    MaterialShared.h: Shared material definitions.
=============================================================================*/

#pragma once
```

## 为什么保留

学习路径 8.4 列它,是因为**学习时需要知道 `FNiagaraShaderMap` 类存在**。但实际声明在 [[Wiki/Sources/Stock/Niagara/NiagaraShared]] L393。

推测该头文件被规划作公开入口但内容最终并入 `NiagaraShared.h`,stub 保留兼容 include。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraShader]](Shader 家族合并页,ShaderMap 在此)
