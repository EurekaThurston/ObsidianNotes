---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, vertex-factory, gpu, rendering]
sources: 1
aliases: [NiagaraVertexFactory.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraVertexFactory.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraVertexFactory.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraVertexFactory.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 135 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.9(**VF 基类**)

## 职责

Niagara **VertexFactory 基类** `FNiagaraVertexFactoryBase`——连接粒子 Buffer 与材质 Shader 的 "桥"。继承 UE 通用 `FVertexFactory`。

VertexFactory 的作用:告诉 UE 渲染器"每个顶点的数据从哪来",把不同来源数据(粒子属性、mesh 顶点)统一翻译成 material shader 能读的 vertex interpolants。

## `ENiagaraVertexFactoryType`(L49)

```cpp
enum ENiagaraVertexFactoryType { NVFT_Sprite, NVFT_Ribbon, NVFT_Mesh, NVFT_MAX };
```

3 种 VF 类型,对应 Phase 6 的 3 种有 VF 的 renderer(Light 没 VF)。

## `FNiagaraNullSortedIndicesVertexBuffer`(L18)

```cpp
class FNiagaraNullSortedIndicesVertexBuffer : public FVertexBuffer {
    FShaderResourceViewRHIRef VertexBufferSRV;  // 一个 int32=0 的 SRV
};
extern TGlobalResource<FNiagaraNullSortedIndicesVertexBuffer> GFNiagaraNullSortedIndicesVertexBuffer;
```

**全局单例的"空排序索引"**——渲染器没排序时绑这个 SRV,避免特殊分支。

## `FNiagaraVertexFactoryBase`(L62)

```cpp
class FNiagaraVertexFactoryBase : public FVertexFactory
{
    FNiagaraVertexFactoryBase(ENiagaraVertexFactoryType Type, ERHIFeatureLevel::Type);

    static void ModifyCompilationEnvironment(...) {
        // 设置 NIAGARA_PARTICLE_FACTORY=1 define(HLSL 里用来分支)
        OutEnvironment.SetDefine(TEXT("NIAGARA_PARTICLE_FACTORY"), TEXT("1"));
    }

    ENiagaraVertexFactoryType GetParticleFactoryType() const;
    void SetParticleFactoryType(ENiagaraVertexFactoryType);

    void SetInUse(bool);
    bool GetInUse() const;

    bool CheckAndUpdateLastFrame(const FSceneViewFamily&, const FSceneView* = nullptr) const;
    // 返回 true 则本帧首次被访问(需要刷新 per-view 数据)
};
```

### 关键字段

```cpp
mutable uint32 LastFrameSetup;              // 上次 setup 的帧号
mutable const FSceneViewFamily* LastViewFamily;
mutable const FSceneView* LastView;
mutable float LastFrameRealTime;

ENiagaraVertexFactoryType ParticleFactoryType;
bool bInUse;
```

`CheckAndUpdateLastFrame` 在每 view 调一次——返回 true 表示本帧首次见这个 view,需要重做 per-view 绑定。性能优化,多 view 渲染时避免重复。

## `bNeedsDeclaration = false`

构造里设为 false——Niagara VF 不需要传统 "vertex declaration"(CPU 侧 vertex buffer 布局),所有数据从 SRV 读。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraVertexFactory]](合并基类 + 3 种具体 VF)
- Phase 6 `FNiagaraRendererSprites / Ribbons / Meshes` 持有 VF
