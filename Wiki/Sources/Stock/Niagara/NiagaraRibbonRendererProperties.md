---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, ribbon, properties]
sources: 1
aliases: [NiagaraRibbonRendererProperties.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRibbonRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraRibbonRendererProperties.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRibbonRendererProperties.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 361 行(Phase 6 最大)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.5

## 职责

定义 `UNiagaraRibbonRendererProperties`——把 emitter 的粒子按顺序连成**带状几何体**的 renderer。典型用途:闪电、拖尾、能量带。

> [!warning] CPU only
> `IsSimTargetSupported(ENiagaraSimTarget)` 只返回 `(InSimTarget == CPUSim)`。Ribbon 依赖跨粒子的**有序连接**和**tessellation**,GPU 路径在本引擎版本不支持。

## 关键枚举

### `ENiagaraRibbonFacingMode`(L17)

```cpp
enum class ENiagaraRibbonFacingMode : uint8 {
    Screen,              // 面屏幕
    Custom,              // Particles.RibbonFacing 作 facing
    CustomSideVector,    // Particles.RibbonFacing 作 side,facing 算出来
                         //   ⚠️ 不支持 RibbonTwist
};
```

### `ENiagaraRibbonDrawDirection`(L43):`FrontToBack / BackToFront`

### Tessellation 3 模式

```cpp
enum class ENiagaraRibbonTessellationMode : uint8 {
    Automatic,      // 自动参数
    Custom,         // 自定义
    Disabled        // 完全关(最快,直线段)
};
```

### UV 边缘模式

```cpp
enum class ENiagaraRibbonUVEdgeMode {
    SmoothTransition,   // 边缘 UV 按 normalized age 平滑过渡(UV 超 0-1)
    Locked,             // 边缘锁 0 或 1
};
```

### UV 分布模式

```cpp
enum class ENiagaraRibbonUVDistributionMode {
    ScaledUniformly,                       // 均匀 0-1
    ScaledUsingRibbonSegmentLength,        // 按段长比例分配 UV
    TiledOverRibbonLength,                 // 按 tiling 距离重复
};
```

### `ENiagaraRibbonVFLayout`(L126):16 个 slot

`Position / Velocity / Color / Width / Twist / Facing / NormalizedAge / MaterialRandom / MaterialParam0-3 / U0Override / V0RangeOverride / U1Override / V1RangeOverride / Num`

## `FNiagaraRibbonUVSettings` struct(L87)

每 UV channel 一份配置:

```cpp
USTRUCT()
struct FNiagaraRibbonUVSettings {
    ENiagaraRibbonUVEdgeMode LeadingEdgeMode;      // 前端(粒子新生成)
    ENiagaraRibbonUVEdgeMode TrailingEdgeMode;     // 后端(粒子消亡)
    ENiagaraRibbonUVDistributionMode DistributionMode;
    float TilingLength;                             // TiledOverRibbonLength 用
    FVector2D Offset;                               // 额外 UV 偏移
    FVector2D Scale;                                // 额外 UV 缩放
    bool bEnablePerParticleUOverride;               // 粒子级 U 覆盖
    bool bEnablePerParticleVRangeOverride;
};
```

## 主类字段

### 材质

- `UMaterialInterface* Material` — 必须开 "Use with Niagara Ribbons"
- `FNiagaraUserParameterBinding MaterialUserParamBinding`

### 渲染模式

- `ENiagaraRibbonFacingMode FacingMode`
- `FNiagaraRibbonUVSettings UV0Settings / UV1Settings` — 两个 UV channel
- `ENiagaraRibbonDrawDirection DrawDirection`

### Tessellation

```cpp
float CurveTension;                    // 曲线张力 0-1
ENiagaraRibbonTessellationMode TessellationMode;
int32 TessellationFactor;              // 1-16
bool bUseConstantFactor;               // 用上面常量,或基于下面自适应
float TessellationAngle;               // 自适应:拐角角度 <N 度就加细
bool bScreenSpaceTessellation;         // 屏幕空间自适应
```

### 15 个属性绑定

`Position / Color / Velocity / NormalizedAge / RibbonTwist / RibbonWidth / RibbonFacing / RibbonId / RibbonLinkOrder / MaterialRandom / DynamicMaterial / DynamicMaterial1/2/3 / U0Override / V0RangeOverride / U1Override / V1RangeOverride`

**`RibbonId` 和 `RibbonLinkOrder`** 是 Ribbon 专有:一个 emitter 可以有**多条独立 ribbon**(如粒子分成不同 RibbonId 的组),`LinkOrder` 控制粒子在同 ribbon 内的连接顺序(默认按 normalized age)。

### Runtime DataSetAccessor 缓存(L338)

```cpp
FNiagaraDataSetAccessor<float> SortKeyDataSetAccessor;
FNiagaraDataSetAccessor<FVector> PositionDataSetAccessor;
FNiagaraDataSetAccessor<float> SizeDataSetAccessor;
FNiagaraDataSetAccessor<float> TwistDataSetAccessor;
FNiagaraDataSetAccessor<FVector> FacingDataSetAccessor;
FNiagaraDataSetAccessor<FVector4> MaterialParam0-3 DataSetAccessor;
bool U0OverrideIsBound / U1OverrideIsBound;

FNiagaraDataSetAccessor<int32> RibbonIdDataSetAccessor;
FNiagaraDataSetAccessor<FNiagaraID> RibbonFullIDDataSetAccessor;
```

**直接持 DataSetAccessor** 作缓存(比每帧重查 offset 快)。和 Sprite 持 `VFBoundOffsetsInParamStore` 数组思路不同——Ribbon 复杂度更高,每个属性都用强类型 accessor 清晰。

## Deprecated 字段(L201-L226,editor only)

`UV0TilingDistance_DEPRECATED / UV0Scale_DEPRECATED / UV0Offset_DEPRECATED / UV0AgeOffsetMode_DEPRECATED`(UV1 同) —— 老版本 UV 设置,现在合并进 `UV0Settings / UV1Settings`。`PostLoad` 做迁移。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraRibbonRendererProperties]](合并 Renderer)
- [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]] — 基类
- Phase 4 Persistent ID(`FNiagaraID`)— RibbonFullID 绑定
