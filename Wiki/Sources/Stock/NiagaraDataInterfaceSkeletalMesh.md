---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, skeletal-mesh, skinning]
sources: 1
aliases: [NiagaraDataInterfaceSkeletalMesh.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceSkeletalMesh.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceSkeletalMesh.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceSkeletalMesh.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: **976 行**(Phase 7 最大,本次扒前 300 行)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.8(**最复杂 DI**)

## 职责

**最复杂的 DI**。让脚本从骨骼网格采样 —— 比 StaticMesh 多一层 CPU 蒙皮(skinning)问题。角色特效、布料、肌肉粒子。

## 共享骨骼蒙皮数据

### `FSkeletalMeshSkinningDataUsage`(L24)

```cpp
struct FSkeletalMeshSkinningDataUsage {
    int32 LODIndex;
    uint32 bUsesBoneMatrices : 1;
    uint32 bUsesPreSkinnedVerts : 1;

    bool NeedBoneMatrices() const { return bUsesBoneMatrices || bUsesPreSkinnedVerts; }
};
```

### `FSkeletalMeshSkinningData`(L62)— 共享缓存

```cpp
struct FSkeletalMeshSkinningData
{
    TWeakObjectPtr<USkeletalMeshComponent> MeshComp;
    float DeltaSeconds;
    int32 CurrIndex;                              // ← 双缓冲索引

    volatile int32 BoneMatrixUsers;               // 引用计数
    volatile int32 TotalPreSkinnedVertsUsers;

    TArray<FMatrix> BoneRefToLocals[2];           // 双缓冲骨骼矩阵
    TArray<FTransform> ComponentTransforms[2];    // 双缓冲组件变换

    struct FLODData {
        volatile int32 PreSkinnedVertsUsers;
        TArray<FVector> SkinnedCPUPositions[2];   // 双缓冲蒙皮顶点位置
        TArray<FVector> SkinnedTangentBasis;
    };
    TArray<FLODData> LODData;

    FRWLock RWGuard;
    bool bForceDataRefresh;

    // ... getter/setter 系列
};
```

**双缓冲** `[2]` 是为了**速度计算**—— Prev + Curr 两帧顶点位置的差除以 DeltaSeconds。

`RWLock` 保护跨线程访问。

### `FNDI_SkeletalMesh_GeneratedData`(L223)— 全局缓存

```cpp
class FNDI_SkeletalMesh_GeneratedData
{
    FRWLock CachedSkinningDataGuard;
    TMap<TWeakObjectPtr<USkeletalMeshComponent>, TSharedPtr<FSkeletalMeshSkinningData>> CachedSkinningData;

    FSkeletalMeshSkinningDataHandle GetCachedSkinningData(
        TWeakObjectPtr<USkeletalMeshComponent>& InComponent,
        FSkeletalMeshSkinningDataUsage Usage,
        bool bNeedsDataImmediately);

    void TickGeneratedData(ETickingGroup TickGroup, float DeltaSeconds);
};
```

**全局一份**(住在 WorldManager)——多个 DI 引用同一 MeshComponent 时**共享**蒙皮缓存,避免重复 skinning。

## 关键枚举

### `ENDISkeletalMesh_SourceMode`(L236)

```cpp
Default / Source / AttachParent  // 3 种(比 StaticMesh 少 DefaultMeshOnly)
```

### `ENDISkeletalMesh_SkinningMode`(L257)

```cpp
enum class ENDISkeletalMesh_SkinningMode : uint8 {
    None,          // 引用 pose only — 骨骼/socket 按需算,顶点按需算(CPU)
    SkinOnTheFly,  // 骨骼/socket 前置算,顶点按需(需要 CPU Access)
    PreSkin,       // 全部前置算,最适合"读大量顶点"(需要 CPU Access)
};
```

性能/内存权衡:PreSkin 花 CPU 时间一次性算完,之后采样 O(1);None 最省内存但每次采样要实时 skin。

### `ENDISkeletalMesh_FilterMode`(L281)

```cpp
enum class ENDISkeletalMesh_FilterMode : uint8 {
    None, SingleRegion, MultiRegion
};
```

`FSkeletalMeshSamplingRegion` 是 UE 资产级的采样区域(骨骼附近一组三角形)。本 DI 支持 single 或 multi region 过滤。

### `ENDISkelMesh_AreaWeightingMode`

```cpp
None / AreaWeighted
```

## `FSkeletalMeshSamplingRegionAreaWeightedSampler`(L298)

```cpp
struct FSkeletalMeshSamplingRegionAreaWeightedSampler : FWeightedRandomSampler
{
    // 跨多 region 的面积加权采样
};
```

## 主类(本次未扒具体字段,在 L300+)

`UCLASS UNiagaraDataInterfaceSkeletalMesh : public UNiagaraDataInterface`

### 典型 VM 函数(按家族推断)

- **Triangle/Vertex**:`RandomTriangle / GetTriangleData / RandomVertex / GetVertexData(Position/Normal/UV/Color/SkinWeight)`
- **Bone**:`RandomBone / GetBoneCount / GetBoneTransform(Current/Prev/Reference)`
- **Socket**:`GetSocketTransform / RandomSocket / GetFilteredSocketCount`
- **Skin Weight**:读某顶点的 bone indices + weights

## 后 676 行未读内容

- 主类的所有 UPROPERTY + VM 函数实现声明
- GPU HLSL 生成
- Editor 支持
- 各种辅助 struct

本 Phase 读本只讲架构,具体 VM API 按需 offset 读。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceSkeletalMesh]]
- [[Wiki/Entities/Stock/UNiagaraDataInterface]]
- Phase 2 `UNiagaraFunctionLibrary::OverrideSystemUserVariableSkeletalMeshComponent / SetSkeletalMeshDataInterfaceSamplingRegions`
