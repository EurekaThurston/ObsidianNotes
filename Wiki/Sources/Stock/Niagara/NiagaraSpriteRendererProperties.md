---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, sprite, properties]
sources: 1
aliases: [NiagaraSpriteRendererProperties.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSpriteRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraSpriteRendererProperties.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSpriteRendererProperties.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 316 行(Properties 类最大)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.3

## 职责

定义 **最常用的 Niagara renderer**——`UNiagaraSpriteRendererProperties`。每粒子一个 billboard quad,支持 SubUV、cutout、8 种 facing 模式、多 dynamic material param。CPU 和 GPU 都支持。

## 关键枚举

### `ENiagaraSpriteAlignment`(L14)

```cpp
enum class ENiagaraSpriteAlignment : uint8 {
    Unaligned,           // 只看 SpriteRotation + FacingMode
    VelocityAligned,     // "上" 轴朝 Velocity 方向
    CustomAlignment,     // "上" 轴朝 Particles.SpriteAlignment
};
```

### `ENiagaraSpriteFacingMode`(L27)

```cpp
enum class ENiagaraSpriteFacingMode : uint8 {
    FaceCamera,              // 朝摄像机原点,保持 UP 对齐摄像机 UP
    FaceCameraPlane,         // 完全平行摄像机平面(最"扁")
    CustomFacingVector,      // 朝 Particles.SpriteFacing 向量
    FaceCameraPosition,      // 朝摄像机位置,但不依赖摄像机旋转(VR 稳定)
    FaceCameraDistanceBlend  // 远距离 FaceCameraPosition,近距离 FaceCamera
};
```

Shader 里 `NiagaraSpriteVertexFactory.ush` 必须和本枚举同步。

### `ENiagaraSpriteVFLayout`(L41)

VF 变量的 slot 枚举(**17 个**):`Position / Color / Velocity / Rotation / Size / Facing / Alignment / SubImage / MaterialParam0-3 / CameraOffset / UVScale / MaterialRandom / CustomSorting / NormalizedAge / Num`。

## 主类

```cpp
UCLASS(editinlinenew, meta = (DisplayName = "Sprite Renderer"))
class NIAGARA_API UNiagaraSpriteRendererProperties : public UNiagaraRendererProperties
```

**支持 CPU + GPU**:`IsSimTargetSupported(...) { return true; }`

## 字段分组

### 渲染基础

- `UMaterialInterface* Material` — 必须开 "Use with Niagara Sprites"
- `FNiagaraUserParameterBinding MaterialUserParamBinding` — User 参数覆盖材质
- `ENiagaraRendererSourceDataMode SourceMode` — 每粒子画 / 每 emitter 画一个
- `ENiagaraSpriteAlignment Alignment` / `ENiagaraSpriteFacingMode FacingMode`
- `FVector2D PivotInUVSpace` — 旋转轴位置,UE UV 空间(0,0 左上,1,1 右下)

### SubUV

- `FVector2D SubImageSize` — 列 x 行
- `bSubImageBlend : 1` — 子图间 alpha 混合

### 排序

- `ENiagaraSortMode SortMode`
- `bSortOnlyWhenTranslucent : 1` — 透明材质才排序(不透明不必)
- `bGpuLowLatencyTranslucency : 1` — GPU emitter 是否用当前帧数据(半透明低延迟)

### 视觉调节

- `bRemoveHMDRollInVR : 1` — VR 里去除头部 roll
- `MinFacingCameraBlendDistance` / `MaxFacingCameraBlendDistance` — `FaceCameraDistanceBlend` 用

### 可见性 / Culling

- `bEnableCameraDistanceCulling : 1` + `MinCameraDistance / MaxCameraDistance`
- `uint32 RendererVisibility` — 对应 `Particles.VisibilityTag` 绑定

### 属性绑定(约 15 个 `FNiagaraVariableAttributeBinding`)

`Position / Color / Velocity / SpriteRotation / SpriteSize / SpriteFacing / SpriteAlignment / SubImageIndex / DynamicMaterial / DynamicMaterial1/2/3 / CameraOffset / UVScale / MaterialRandom / CustomSorting / NormalizedAge / RendererVisibilityTag`

### Material Parameter 绑定

```cpp
TArray<FNiagaraMaterialAttributeBinding> MaterialParameterBindings;
```

如果有 → `NeedsMIDsForMaterials()` 返 true,会给每 emitter instance 创建 `MaterialInstanceDynamic`。

### Cutout 子系统(Editor-only,WITH_EDITORONLY_DATA)

```cpp
bool bUseMaterialCutoutTexture;
UTexture2D* CutoutTexture;
TEnumAsByte<ESubUVBoundingVertexCount> BoundingMode;  // 4 or 8 顶点
TEnumAsByte<EOpacitySourceMode> OpacitySourceMode;
float AlphaThreshold;

void UpdateCutoutTexture();
void CacheDerivedData();

FSubUVDerivedData DerivedData;  // 生成的 bounding geometry
```

**目的**:半透明 sprite 默认矩形 alpha = 0 区域会造成 overdraw。Cutout 生成一个贴合不透明部分的多边形(4 或 8 顶点),减少 overdraw。

### Runtime 缓存

```cpp
FNiagaraRendererLayout RendererLayoutWithCustomSort;
FNiagaraRendererLayout RendererLayoutWithoutCustomSort;  // 两份 layout
uint32 MaterialParamValidMask;                           // 哪些动态材质参数真的有绑定
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraSpriteRendererProperties]](对偶 Renderer 在 [[Wiki/Sources/Stock/Niagara/NiagaraRendererSprites]])
- [[Wiki/Entities/Stock/Niagara/UNiagaraRendererProperties]] — 基类
