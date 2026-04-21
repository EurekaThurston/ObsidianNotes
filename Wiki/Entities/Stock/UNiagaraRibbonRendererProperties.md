---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, renderer, ribbon]
sources: 1
aliases: [UNiagaraRibbonRendererProperties, FNiagaraRendererRibbons, Ribbon Renderer]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRibbonRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraRibbonRendererProperties / FNiagaraRendererRibbons

> **把粒子按顺序连成带状**的 renderer。用于闪电、拖尾、能量带。**CPU only**。本页合并 Asset + 运行时。

## 一句话角色

- Asset:`UNiagaraRibbonRendererProperties`(361 行,Phase 6 最大)
- 运行时:`FNiagaraRendererRibbons`(103 行),每帧排序粒子 + 生成 index buffer + 自适应 tessellation

## 陷阱 / 限制

- ⚠️ **CPU only**:`IsSimTargetSupported` 只返回 `CPUSim`
- ⚠️ **`CustomSideVector` facing 不支持 ribbon twist**
- ⚠️ **Tessellation 开销大**:粒子数 × InterpCount

## 核心能力

### 3 种 FacingMode

- `Screen`、`Custom`、`CustomSideVector`

### Multi-ribbon per emitter

- `RibbonIdBinding` — 同 emitter 多条独立 ribbon
- `RibbonLinkOrderBinding` — 同 ribbon 内连接顺序(默认按 age)
- `RibbonFullIDDataSetAccessor` — `FNiagaraID` 版,跨帧追踪

### Tessellation

```cpp
ENiagaraRibbonTessellationMode TessellationMode;  // Automatic / Custom / Disabled
float CurveTension;                                // 0-1
int32 TessellationFactor;                          // 1-16
bool bUseConstantFactor;                           // vs 自适应
float TessellationAngle;                           // 自适应触发角度
bool bScreenSpaceTessellation;                     // 屏幕空间自适应
```

Runtime `mutable` 字段按帧统计段角度/twist/长度,反馈下帧 tessellation。

### 双 UV channel

- `FNiagaraRibbonUVSettings UV0Settings / UV1Settings` — 各自有 LeadingEdgeMode / TrailingEdgeMode / DistributionMode / Tiling / Offset / Scale / per-particle Override
- 3 种 EdgeMode:SmoothTransition / Locked
- 3 种 DistributionMode:ScaledUniformly / ScaledUsingSegmentLength / TiledOverRibbonLength

### 16 个 VF slot

`Position / Velocity / Color / Width / Twist / Facing / NormalizedAge / MaterialRandom / MaterialParam0-3 / U0Override / V0RangeOverride / U1Override / V1RangeOverride`

## 运行时字段(部分 mutable 自适应)

```cpp
mutable float TessellationAngle = 0;
mutable float TessellationCurvature = 0;
mutable float TessellationTwistAngle = 0;
mutable float TessellationTwistCurvature = 0;
mutable float TessellationTotalSegmentLength = 0;
```

## Runtime Accessor(强类型缓存)

```cpp
FNiagaraDataSetAccessor<float> SortKey / Size / Twist;
FNiagaraDataSetAccessor<FVector> Position / Facing;
FNiagaraDataSetAccessor<FVector4> MaterialParam0-3;
FNiagaraDataSetAccessor<int32> RibbonId;
FNiagaraDataSetAccessor<FNiagaraID> RibbonFullID;
```

## 相关

- [[Wiki/Entities/Stock/UNiagaraRendererProperties]] / [[Wiki/Entities/Stock/FNiagaraRenderer]]
- Phase 4 `FNiagaraID` — RibbonFullID
- Phase 8 `FNiagaraRibbonVertexFactory`

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraRibbonRendererProperties]] / [[Wiki/Sources/Stock/NiagaraRendererRibbons]]
- 读本:[[Readers/Niagara/Phase 6 - Niagara 粒子如何变成屏幕像素]] § 5
