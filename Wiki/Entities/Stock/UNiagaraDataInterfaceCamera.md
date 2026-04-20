---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, camera]
sources: 1
aliases: [UNiagaraDataInterfaceCamera, Camera DI]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCamera.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceCamera

> 让脚本**读当前相机信息**(位置/方向/FOV/视图矩阵)并**按距离排序粒子**。CPU + GPU。

## 一句话角色

`UCLASS UNiagaraDataInterfaceCamera : UNiagaraDataInterface`。典型用途:粒子朝相机行为、距离排序、近处 LOD。

## Per-Instance Data

```cpp
struct FCameraDataInterface_InstanceData {
    FVector CameraLocation;
    FRotator CameraRotation;
    float CameraFOV;

    TQueue<FDistanceData, EQueueMode::Mpsc> DistanceSortQueue;  // VM 并行 push
    TArray<FDistanceData> ParticlesSortedByDistance;
};
```

**MPSC 队列**:多 VM worker 并行写,GT 单线程消费排序。

## 关键字段

```cpp
int32 PlayerControllerIndex = 0;          // 默认第一个 controller
bool bRequireCurrentFrameData = true;     // false = 用上一帧数据换性能
```

## 重要虚方法返回

- `CanExecuteOnTarget(Target)` → true(CPU+GPU)
- `HasTickGroupPrereqs()` → true(要求特定 TickGroup)
- `RequiresEarlyViewData()` → true(需要早期 view 数据)
- `HasPreSimulateTick()` → true(tick 前采集相机数据)

## VM 函数

- `CalculateParticleDistances`
- `GetClosestParticles`
- `GetCameraFOV / GetCameraProperties`
- GPU 专用:`GetViewPropertiesGPU / GetClipSpaceTransformsGPU / GetViewSpaceTransformsGPU`

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterface]]

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraDataInterfaceCamera]]
- 读本:[[Readers/Niagara/Phase7-data-interface-读本]] § 4
