---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, skeletal-mesh]
sources: 1
aliases: [UNiagaraDataInterfaceSkeletalMesh, Skeletal Mesh DI]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceSkeletalMesh.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceSkeletalMesh

> **最复杂的 DI**(976 行)。从骨骼网格采样位置/法线/骨骼/socket。比 StaticMesh 多一层 CPU skinning 问题。

## 一句话角色

`UCLASS UNiagaraDataInterfaceSkeletalMesh : UNiagaraDataInterface`。典型用途:角色特效、布料、肌肉粒子、跟随骨骼动画的效果。

## 三种 Skinning Mode

```cpp
enum class ENDISkeletalMesh_SkinningMode : uint8 {
    None,          // 引用 pose only,按需算
    SkinOnTheFly,  // 骨骼前置,顶点按需(CPU Access)
    PreSkin,       // 全部前置(CPU Access,读大量顶点时最快)
};
```

性能/内存权衡。

## Source 模式

```cpp
Default / Source / AttachParent  // 比 StaticMesh 少 DefaultMeshOnly
```

## Filter 模式

```cpp
enum class ENDISkeletalMesh_FilterMode : uint8 {
    None, SingleRegion, MultiRegion  // 按 UE Sampling Region 过滤
};
```

## 共享骨骼缓存机制

```
FNDI_SkeletalMesh_GeneratedData (全局,住 WorldManager)
    ↓ TMap<MeshComponent, TSharedPtr<FSkeletalMeshSkinningData>>
FSkeletalMeshSkinningData (引用计数共享)
    ├─ BoneRefToLocals[2]                       // 双缓冲骨骼矩阵
    ├─ ComponentTransforms[2]
    ├─ FLODData { SkinnedCPUPositions[2], SkinnedTangentBasis }
    └─ FRWLock RWGuard
```

**核心思想**:多个 DI 指向同一 MeshComponent 时**共享**蒙皮数据——不重复 skinning。`BoneMatrixUsers / TotalPreSkinnedVertsUsers` 引用计数控制何时释放。

双缓冲 `[2]` 给**速度计算**(Prev - Curr 顶点差 / DeltaSeconds)。

## Usage Handle

```cpp
struct FSkeletalMeshSkinningDataUsage {
    int32 LODIndex;
    bool bUsesBoneMatrices;
    bool bUsesPreSkinnedVerts;
};
```

每 DI 注册一个 Usage(要用哪 LOD + 要骨骼还是顶点)→ GeneratedData 决定实际要算什么。

## 典型 VM 函数家族

- **Triangle/Vertex**:`RandomTriangle / GetVertexData(Position/Normal/UV/Color)`
- **Bone**:`RandomBone / GetBoneCount / GetBoneTransform{Current,Prev,Reference}`
- **Socket**:`GetSocketTransform / RandomSocket`
- **Skin Weight**:读顶点的骨骼 indices + weights

## 陷阱

- ⚠️ **Mesh CPU Access 要求**:`SkinOnTheFly / PreSkin` 都需要开 CPU Access(Asset 属性)
- ⚠️ **PreSkin 内存**:一个高多边形角色每帧 full skin 是 MB 级开销

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterface]]
- Phase 2 `UNiagaraFunctionLibrary::OverrideSystemUserVariableSkeletalMeshComponent / SetSkeletalMeshDataInterfaceSamplingRegions`

## 深入阅读

- 源:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceSkeletalMesh]](976 行,前 300 扒)
- 读本:[[Readers/UE/Niagara/Phase 7 - 最强扩展点 Data Interface]] § 7
