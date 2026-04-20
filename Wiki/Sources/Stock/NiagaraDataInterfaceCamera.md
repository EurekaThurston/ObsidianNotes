---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, camera]
sources: 1
aliases: [NiagaraDataInterfaceCamera.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCamera.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceCamera.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCamera.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 107 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.5

## 职责

让脚本**读取当前相机信息**(位置/方向/FOV)并**按距离排序粒子**。典型用途:"近处的粒子更亮"、"面向相机的效果"、"根据相机距离决定 LOD"。

## `FCameraDataInterface_InstanceData`(L15)— per-instance

```cpp
struct FCameraDataInterface_InstanceData
{
    FVector CameraLocation;
    FRotator CameraRotation;
    float CameraFOV;

    TQueue<FDistanceData, EQueueMode::Mpsc> DistanceSortQueue;   // VM 并行 push
    TArray<FDistanceData> ParticlesSortedByDistance;             // tick 时排序结果
};

struct FDistanceData { FNiagaraID ParticleID; float DistanceSquared; };
```

**MPSC 队列**:多个 VM worker 并行写粒子距离(Multi-Producer),GT tick 时单线程消费(Single-Consumer)排序。

## 主类

```cpp
UCLASS(EditInlineNew, Category = "Camera", meta = (DisplayName = "Camera Query"))
class NIAGARA_API UNiagaraDataInterfaceCamera : public UNiagaraDataInterface
{
    UPROPERTY(EditAnywhere)
    int32 PlayerControllerIndex = 0;   // 默认第一个 controller

    UPROPERTY(EditAnywhere, Category = "Performance")
    bool bRequireCurrentFrameData = true;  // 用上一帧数据换性能
```

`bRequireCurrentFrameData = false` 的性能权衡:用上一帧相机数据,Niagara 可以提前在 tick group 里跑(不用等 view 更新),GT 总 cost 降低,但特效和相机可能差一帧。

### 覆盖的接口

```cpp
virtual bool CanExecuteOnTarget(ENiagaraSimTarget Target) const override { return true; }  // CPU+GPU
virtual bool HasTickGroupPrereqs() const override { return true; }         // 有 TickGroup 约束
virtual ETickingGroup CalculateTickGroup(const void* PerInstanceData) const override;
virtual bool RequiresEarlyViewData() const override { return true; }       // 需要早期 view
virtual bool HasPreSimulateTick() const override { return true; }          // tick 前采集相机数据
```

### VM 函数

```cpp
void CalculateParticleDistances(FVectorVMContext& Context);   // 压距离到 DistanceSortQueue
void GetClosestParticles(FVectorVMContext& Context);          // 查询排序结果
void GetCameraFOV(FVectorVMContext& Context);
void GetCameraProperties(FVectorVMContext& Context);

// GPU only
void GetViewPropertiesGPU(FVectorVMContext& Context);
void GetClipSpaceTransformsGPU(FVectorVMContext& Context);
void GetViewSpaceTransformsGPU(FVectorVMContext& Context);
```

### 函数名静态 FName

```cpp
static const FName CalculateDistancesName;
static const FName QueryClosestName;
static const FName GetViewPropertiesName;
static const FName GetClipSpaceTransformsName;
static const FName GetViewSpaceTransformsName;
static const FName GetCameraPropertiesName;
static const FName GetFieldOfViewName;
```

## GPU 侧

```cpp
struct FNiagaraDataIntefaceProxyCameraQuery : public FNiagaraDataInterfaceProxy
{
    // 空 — 相机数据从场景 UniformBuffer 读
};

struct FNiagaraDataInterfaceParametersCS_CameraQuery : public FNiagaraDataInterfaceParametersCS
{
    LAYOUT_FIELD(FShaderUniformBufferParameter, PassUniformBuffer);  // 场景深度/相机
};
```

GPU compute shader 绑定 `PassUniformBuffer`(场景纹理 + 相机数据),HLSL 函数从中读取。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceCamera]]
- [[Wiki/Entities/Stock/UNiagaraDataInterface]]
