---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, vertex-factory, gpu, rendering]
sources: 1
aliases: [FNiagaraVertexFactory, FNiagaraVertexFactoryBase, FNiagaraSpriteVertexFactory, FNiagaraRibbonVertexFactory, FNiagaraMeshVertexFactory]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraVertexFactory.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraVertexFactory 家族

> Niagara **VertexFactory**——连接粒子 Buffer 与材质 Shader 的"桥"。合并基类 + 3 种具体 VF(Sprite/Ribbon/Mesh)。Light 没 VF(不产 mesh batch)。

## 一句话角色

- **`FNiagaraVertexFactoryBase : FVertexFactory`**:带 `LastFrameSetup` 帧缓存 + `CheckAndUpdateLastFrame` 避免同帧重复绑定
- **`FNiagaraSpriteVertexFactory`**:Billboard,2 个 UBO(stable + loose)+ cutout / sorted indices / facing / alignment
- **`FNiagaraRibbonVertexFactory`**:带状,特有 tangent/distance buffer + multi-ribbon indexing
- **`FNiagaraMeshVertexFactory`**:GPU Instancing 唯一,需要 `FStaticMeshDataType` mesh 顶点数据。**唯一支持 tessellation**

## `ENiagaraVertexFactoryType`

```cpp
NVFT_Sprite / NVFT_Ribbon / NVFT_Mesh / NVFT_MAX
```

## `GFNiagaraNullSortedIndicesVertexBuffer`(全局单例)

空 SRV(1 个 int32=0),渲染器没排序时绑它避免特殊分支。

## VF 共通字段模式

| 字段 | 类型 |
|---|---|
| `NiagaraParticleDataFloat / Half SRV` | CPU sim 粒子数据 |
| `SortedIndices SRV` | 可选排序结果 |
| `[Type]UniformBuffer` | Stable 参数 |
| `LooseParameterUniformBuffer` | Per-draw 参数 |

## Sprite vs Ribbon vs Mesh 特点

| 特性 | Sprite | Ribbon | Mesh |
|---|---|---|---|
| UBO 数 | 2(stable + loose) | 2 | 1(都塞一起) |
| 特有 Buffer | CutoutGeometry | TangentsAndDistances / MultiRibbonIndices / PackedPerRibbonDataByIndex | `FStaticMeshDataType Data`(mesh 顶点) |
| Tessellation | ❌ | ❌ | ✅ (`SupportsTessellationShaders()`) |
| Facing 模式 | 5 种(uint32 枚举) | 3 种(Screen/Custom/CustomSideVector) | 4 种(enum) |
| GPU Instancing | 隐式(billboard expand) | 索引展开 | 显式 SV_InstanceID |

## Shader define

```
FNiagaraVertexFactoryBase: NIAGARA_PARTICLE_FACTORY=1
FNiagaraMeshVertexFactory: NIAGARA_MESH_FACTORY=1 + NIAGARA_MESH_INSTANCED=1
```

材质 shader 根据这些 define 编译不同 vertex code path。

## 相关

- Phase 6 `FNiagaraRenderer` 持有 VF
- [[Wiki/Entities/Stock/FNiagaraShader]] — compute shader(不同体系)
- [[Wiki/Entities/Stock/FNiagaraDataBuffer]] — 数据源
- [[Wiki/Entities/Stock/FNiagaraGPUSort]] — SortedIndices 来源

## 深入阅读

- 源 × 4:[[Wiki/Sources/Stock/NiagaraVertexFactory]] / [[Wiki/Sources/Stock/NiagaraSpriteVertexFactory]] / [[Wiki/Sources/Stock/NiagaraRibbonVertexFactory]] / [[Wiki/Sources/Stock/NiagaraMeshVertexFactory]]
- 读本:[[Readers/Niagara/Phase8-gpu-simulation-读本]] § 6
