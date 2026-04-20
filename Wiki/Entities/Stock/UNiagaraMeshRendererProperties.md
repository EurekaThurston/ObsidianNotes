---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, renderer, mesh]
sources: 1
aliases: [UNiagaraMeshRendererProperties, FNiagaraRendererMeshes, Mesh Renderer]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraMeshRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraMeshRendererProperties / FNiagaraRendererMeshes

> **每粒子实例化一个 StaticMesh**。依赖 GPU Instancing。CPU + GPU 都支持。碎片、岩石、模型群效果。

## 一句话角色

- Asset:`UNiagaraMeshRendererProperties`(288 行)
- 运行时:`FNiagaraRendererMeshes`(74 行)

## 核心字段

| 字段 | 作用 |
|---|---|
| `ParticleMesh` | `UStaticMesh*` — 必须 "Niagara Mesh Particles" |
| `bOverrideMaterials` + `OverrideMaterials` | `TArray<FNiagaraMeshMaterialOverride>` 覆盖每 material slot |
| `FacingMode` | 4 种(Default/Velocity/CameraPosition/CameraPlane) |
| `bLockedAxisEnable` + `LockedAxis` + `LockedAxisSpace` | 非默认 facing 时锁定某旋转轴 |
| `PivotOffset` + `PivotOffsetSpace` | 4 种空间(Mesh/Simulation/World/Local) |
| `SubImageSize` + `bSubImageBlend` | SubUV |

## 14 个 VF slot

`Position / Velocity / Color / Scale / Transform / MaterialRandom / NormalizedAge / CustomSorting / SubImage / DynamicParam0-3 / CameraOffset`

**`Transform` slot**:粒子可以有自由 transform(位置 + 旋转 + 缩放),通过 Particles.Transform 绑定。

## 绑定(13 个)

包含 `MeshOrientationBinding`(FQuat)—— Mesh 专有,给每粒子独立旋转。

## LOD 处理

- `MeshMinimumLOD` 控制起点
- Runtime `GetLODIndex()` 基于相机距离选 —— 所有 instance 共享同 LOD,不支持 per-instance LOD

## `FNiagaraMeshMaterialOverride`

```cpp
USTRUCT()
struct FNiagaraMeshMaterialOverride {
    UMaterialInterface* ExplicitMat;
    FNiagaraUserParameterBinding UserParamBinding;  // User 参数优先
};
```

## 相关

- [[Wiki/Entities/Stock/UNiagaraRendererProperties]] / [[Wiki/Entities/Stock/FNiagaraRenderer]]
- Phase 2 `UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMesh` —— DI-backed mesh 覆盖
- Phase 8 `FNiagaraMeshVertexFactory`

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraMeshRendererProperties]] / [[Wiki/Sources/Stock/NiagaraRendererMeshes]]
- 读本:[[Readers/Niagara/Phase6-rendering-读本]] § 6
