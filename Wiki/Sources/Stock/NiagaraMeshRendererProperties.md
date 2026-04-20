---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, mesh, properties]
sources: 1
aliases: [NiagaraMeshRendererProperties.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraMeshRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraMeshRendererProperties.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraMeshRendererProperties.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 288 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.7

## 职责

定义 `UNiagaraMeshRendererProperties`——**每粒子实例化一个 StaticMesh**。依赖 GPU Instancing。用于碎片、岩石、模型群效果。CPU 和 GPU 都支持。

## 关键枚举

### `ENiagaraMeshFacingMode`(L17)

```cpp
enum class ENiagaraMeshFacingMode : uint8 {
    Default,           // 按粒子 local-space X 轴 + Particles.Transform
    Velocity,          // X 轴对齐 Velocity
    CameraPosition,    // X 轴指向摄像机位置
    CameraPlane,       // X 轴指向摄像机视平面最近点
};
```

### Pivot / Axis 空间

```cpp
enum class ENiagaraMeshPivotOffsetSpace : uint8 { Mesh, Simulation, World, Local };
enum class ENiagaraMeshLockedAxisSpace : uint8 { Simulation, World, Local };
```

### `ENiagaraMeshVFLayout`(L80):14 个 slot

`Position / Velocity / Color / Scale / Transform / MaterialRandom / NormalizedAge / CustomSorting / SubImage / DynamicParam0-3 / CameraOffset / Num`

## 配套 USTRUCT:`FNiagaraMeshMaterialOverride`(L52)

```cpp
USTRUCT()
struct NIAGARA_API FNiagaraMeshMaterialOverride {
    UMaterialInterface* ExplicitMat;        // 直接指定材质
    FNiagaraUserParameterBinding UserParamBinding;  // User 参数优先
};
```

支持 `SerializeFromMismatchedTag` —— 从老版 `FNiagaraParameterStore` 升级。

## 主类字段

### Mesh 设置

- `UStaticMesh* ParticleMesh` — 必须开 "Niagara Mesh Particles"
- `bOverrideMaterials : 1` + `TArray<FNiagaraMeshMaterialOverride> OverrideMaterials` — 覆盖每 material slot

### 排序

- `ENiagaraSortMode SortMode`
- `bSortOnlyWhenTranslucent : 1`

### SubUV

- `FVector2D SubImageSize`
- `bSubImageBlend : 1`

### Facing

- `ENiagaraMeshFacingMode FacingMode`
- `bLockedAxisEnable : 1` + `FVector LockedAxis` + `ENiagaraMeshLockedAxisSpace LockedAxisSpace`
- `FVector PivotOffset` + `ENiagaraMeshPivotOffsetSpace PivotOffsetSpace`

### Culling

- `bEnableFrustumCulling : 1`
- `bEnableCameraDistanceCulling : 1` + `MinCameraDistance / MaxCameraDistance`
- `uint32 RendererVisibility` + 绑定 `RendererVisibilityTagBinding`

### 属性绑定(约 13 个)

`Position / Color / Velocity / MeshOrientation / Scale / SubImageIndex / DynamicMaterial / DynamicMaterial1/2/3 / MaterialRandom / CustomSorting / NormalizedAge / CameraOffset / RendererVisibilityTag`

**`MeshOrientation`** 是 Mesh 专有——每粒子的旋转用 `FQuat` 存。

### Runtime 缓存

```cpp
uint32 MaterialParamValidMask = 0;
FNiagaraRendererLayout RendererLayoutWithCustomSorting;
FNiagaraRendererLayout RendererLayoutWithoutCustomSorting;
```

## Editor 专属

`OnMeshChanged() / OnMeshPostBuild(UStaticMesh*) / CheckMaterialUsage()` — Editor 处理 mesh 资源变化。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraMeshRendererProperties]](合并 Renderer)
- [[Wiki/Entities/Stock/FNiagaraRenderer]]
