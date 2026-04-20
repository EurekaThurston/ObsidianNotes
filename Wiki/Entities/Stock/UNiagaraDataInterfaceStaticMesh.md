---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, static-mesh]
sources: 1
aliases: [UNiagaraDataInterfaceStaticMesh, Static Mesh DI]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceStaticMesh.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceStaticMesh

> 让脚本**从 StaticMesh 采样**位置/法线/UV/顶点颜色/socket。支持面积加权随机采样、material slot 过滤。CPU + GPU。

## 一句话角色

`UCLASS UNiagaraDataInterfaceStaticMesh : UNiagaraDataInterface`。典型用途:粒子"贴附"mesh 表面、沿 mesh 生成效果。

## Source 模式

```cpp
enum ENDIStaticMesh_SourceMode : uint8 {
    Default,          // Source 优先 → AttachParent → DefaultMesh
    Source,           // 只 Source(Phase 2 FunctionLibrary 设置)
    AttachParent,     // 只 parent actor/component
    DefaultMeshOnly   // 只 DefaultMesh
};
```

## 采样模式

```cpp
enum class ESampleMode : int32 { Invalid, Default, AreaWeighted };
```

Area weighted 用 `FStaticMeshFilteredAreaWeightedSectionSampler`(alias 方法),每三角形概率 ∝ 面积。

## Section 过滤

```cpp
USTRUCT() struct FNDIStaticMeshSectionFilter {
    TArray<int32> AllowedMaterialSlots;  // 白名单 material slots
};
```

## Per-Instance Data(丰富)

```cpp
struct FNDIStaticMesh_InstanceData {
    TWeakObjectPtr<USceneComponent> SceneComponent;
    TWeakObjectPtr<UStaticMesh> StaticMesh;
    FMatrix Transform / PrevTransform / TransformInverseTransposed;
    float DeltaSeconds;

    FVector PhysicsVelocity;
    uint32 bUsePhysicsVelocity : 1;

    uint32 bMeshAllowsCpuAccess : 1;              // 没就只能 GPU 采样
    uint32 bIsCpuUniformlyDistributedSampling : 1;
    uint32 bIsGpuUniformlyDistributedSampling : 1;

    TArray<int32> ValidSections;
    FStaticMeshFilteredAreaWeightedSectionSampler Sampler;
    TSharedPtr<FDynamicVertexColorFilterData> DynamicVertexColorSampler;

    TArray<FTransform> CachedSockets;
    TArray<uint16> FilteredAndUnfilteredSockets;
};
```

双 Transform 给速度计算,socket 缓存便于高频读。

## GPU Spawn Buffer

`FStaticMeshGpuSpawnBuffer : FRenderResource` 持 `SectionInfo`(alias sampling)+ mesh SRV + socket transforms。每 (DI, mesh) 一份。

## 典型 VM 函数

- `RandomSection / RandomTriangle / RandomVertex`
- `GetTriangle{Position,Normal,UV,Color}`
- `GetSocketTransform`
- 均匀 vs 面积加权版本

## 陷阱

- ⚠️ **Mesh 需要 CPU Access** 才能 CPU 采样三角形
- ⚠️ `DynamicVertexColorSampler` 的数据要在运行时维护——编辑 mesh 顶点色后需要 `Tick` 感知

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterface]]
- Phase 2 `UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMesh{,Component}`

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraDataInterfaceStaticMesh]]
- 读本:[[Readers/Niagara/Phase7-data-interface-读本]] § 6
