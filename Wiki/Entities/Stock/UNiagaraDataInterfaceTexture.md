---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, texture, gpu-only]
sources: 1
aliases: [UNiagaraDataInterfaceTexture, Texture Sample DI]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceTexture.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceTexture

> 让 **GPU 脚本**采样 2D 纹理。最简单的 GPU-only DI。

## 一句话角色

`UCLASS UNiagaraDataInterfaceTexture : UNiagaraDataInterface`。单字段 `UTexture* Texture`,3 个 VM 函数(Sample/Dimensions/PseudoVolume)。

## 陷阱

- ⚠️ **GPU only**:`CanExecuteOnTarget(Target) { return Target == GPUComputeSim; }`。CPU 脚本用不了。想要 CPU 采样?只能 CurveBase 的 ExposedTexture 反向走(曲线→texture)

## VM 函数

- `SampleTexture(UV) → Color`
- `GetTextureDimensions() → Size`
- `SamplePseudoVolumeTexture(UVW) → Color` — atlas 作伪 volume

## GPU shader 生成

```cpp
static const FString TextureName, SamplerName, DimensionsBaseName;
```

`GetParameterDefinitionHLSL` 拼出 `Texture2D + SamplerState + uint2 Dimensions` 声明,`GetFunctionHLSL` 生成函数体。

## 相关 API

`SetTexture(UTexture*)` — Phase 2 `UNiagaraFunctionLibrary::SetTextureObject` 用这个路径替换。

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterface]]
- Phase 2 `UNiagaraFunctionLibrary::SetTextureObject`

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraDataInterfaceTexture]]
- 读本:[[Readers/Niagara/Phase 7 - 最强扩展点 Data Interface]] § 8
