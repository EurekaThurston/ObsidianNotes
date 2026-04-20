---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, static-mesh]
sources: 1
aliases: [NiagaraDataInterfaceStaticMesh.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceStaticMesh.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceStaticMesh.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceStaticMesh.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 491 行(本次扒了前 250 行)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.7

## 职责

让脚本**从 StaticMesh 采样**位置/法线/UV/顶点颜色/socket——粒子可以"贴在 mesh 表面"。支持面积加权采样、section 过滤。CPU+GPU。

## 关键枚举与辅助

### `ENDIStaticMesh_SourceMode`(L27)

```cpp
enum class ENDIStaticMesh_SourceMode : uint8 {
    Default,            // Source 优先,否则 attach parent,否则 DefaultMesh
    Source,             // 只用 Source(UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMesh 设置)
    AttachParent,       // 只用父 actor/component
    DefaultMeshOnly,    // 只用 DefaultMesh 配置
};
```

### `FNDIStaticMeshSectionFilter`(L54)

```cpp
USTRUCT()
struct FNDIStaticMeshSectionFilter {
    TArray<int32> AllowedMaterialSlots;   // 白名单:按 material slot 过滤 sections
    void Init(UNiagaraDataInterfaceStaticMesh* Owner, bool bAreaWeighted);
    bool CanEverReject() const { return AllowedMaterialSlots.Num() > 0; }
};
```

### `FStaticMeshFilteredAreaWeightedSectionSampler`(L13)

```cpp
struct FStaticMeshFilteredAreaWeightedSectionSampler : FWeightedRandomSampler
{
    virtual float GetWeights(TArray<float>& OutWeights) override;
    TRefCountPtr<const FStaticMeshLODResources> Res;
    FNDIStaticMesh_InstanceData* Owner;
};
```

**Alias 方法**的加权随机采样器(`FWeightedRandomSampler`),按 section 面积加权让每三角形被 spawn 概率 ∝ 面积。

## `FStaticMeshGpuSpawnBuffer`(L72)— GPU 资源

```cpp
class FStaticMeshGpuSpawnBuffer : public FRenderResource
{
    // Triangle alias sampling
    struct SectionInfo { uint32 FirstIndex, NumTriangles; float Prob; uint32 Alias; };
    TArray<SectionInfo> ValidSections;
    FShaderResourceViewRHIRef BufferSectionSRV;
    FShaderResourceViewRHIRef BufferUniformTriangleSamplingSRV;

    // Cached mesh SRV
    FShaderResourceViewRHIRef MeshIndexBufferSrv;
    FShaderResourceViewRHIRef MeshVertexBufferSrv;
    FShaderResourceViewRHIRef MeshTangentBufferSrv;
    FShaderResourceViewRHIRef MeshTexCoordBufferSrv;
    FShaderResourceViewRHIRef MeshColorBufferSRV;
    uint32 NumTexCoord;

    // Sockets
    TResourceArray<FVector4> SocketTransformsResourceArray;
    FShaderResourceViewRHIRef SocketTransformsSRV;
    FShaderResourceViewRHIRef FilteredAndUnfilteredSocketsSRV;
    uint32 NumSockets, NumFilteredSockets;
};
```

每 (DI, mesh) 元组一份 GPU buffer:section 信息 + mesh 顶点/索引/UV/颜色 SRV + socket transforms。

## `FNDIStaticMesh_InstanceData`(L148)— per-instance CPU

```cpp
struct FNDIStaticMesh_InstanceData
{
    TWeakObjectPtr<USceneComponent> SceneComponent;  // 挂载的组件
    TWeakObjectPtr<UStaticMesh> StaticMesh;

    FMatrix Transform / TransformInverseTransposed / PrevTransform;
    float DeltaSeconds;

    FVector PhysicsVelocity;
    uint32 bUsePhysicsVelocity : 1;

    uint32 bComponentValid : 1;
    uint32 bMeshValid : 1;
    uint32 bMeshAllowsCpuAccess : 1;          // 没 CPU access 就只能 GPU 采样
    uint32 bIsCpuUniformlyDistributedSampling : 1;
    uint32 bIsGpuUniformlyDistributedSampling : 1;

    TArray<int32> ValidSections;              // 过滤结果
    FStaticMeshFilteredAreaWeightedSectionSampler Sampler;

    TSharedPtr<FDynamicVertexColorFilterData> DynamicVertexColorSampler;  // 按顶点颜色过滤

    uint32 ChangeId;
    int32 MinLOD, CachedLODIdx;

    TArray<FTransform> CachedSockets;
    int32 NumFilteredSockets;
    TArray<uint16> FilteredAndUnfilteredSockets;

    bool Init(...);
    bool Tick(...);
    void Release();
};
```

每 SystemInstance 一份。注意**双 Transform**(Current / Prev)给速度计算。

## 主类

```cpp
UCLASS(EditInlineNew, Category = "Meshes", meta = (DisplayName = "Static Mesh"))
class NIAGARA_API UNiagaraDataInterfaceStaticMesh : public UNiagaraDataInterface
{
    enum class ESampleMode : int32 { Invalid, Default, AreaWeighted };
    UPROPERTY(EditAnywhere) ENDIStaticMesh_SourceMode SourceMode;
    // ... 更多 UPROPERTY 在未读部分
};
```

## 典型 VM 函数(从 GetFunctions 推断,本次未扒后半)

- `RandomSection / RandomTriangle / RandomVertex`
- `GetTriangleNormal / Position / UV / Color`(按索引)
- `GetSocketTransform`
- 面积加权版本 vs 均匀版本

## 后 250 行未读内容(L250-L491)

- 主类剩余 UPROPERTY(DefaultMesh 等)
- 所有 VM / HLSL 函数定义
- 可能更多 Editor 支持

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceStaticMesh]]
- [[Wiki/Entities/Stock/UNiagaraDataInterface]]
- Phase 2 `UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMesh` 路径

## 开放问题

- 完整 VM 函数列表 → 按需 offset 读
- GPU 侧 ComputePassMask 如何限制采样 → cpp
