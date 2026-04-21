---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, collision]
sources: 1
aliases: [UNiagaraDataInterfaceCollisionQuery, Collision Query DI]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCollisionQuery.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceCollisionQuery

> 让脚本做**物理碰撞查询**。CPU 同步/异步 trace + GPU 深度/DF 查询。

## 一句话角色

`UCLASS UNiagaraDataInterfaceCollisionQuery : UNiagaraDataInterface`。典型用途:粒子撞地面弹起、雨滴触地、火花碰物体消失。

## Per-Instance Data

```cpp
struct CQDIPerInstanceData {
    FNiagaraSystemInstance* SystemInstance;
    FNiagaraDICollisionQueryBatch CollisionBatch;  // 异步查询结果队列
};
```

## 4 种查询方式

| 方法 | 目标 | 性能 | 精度 |
|---|---|---|---|
| `PerformQuerySyncCPU` | CPU | ⚠️ VM 阻塞 | 精确 |
| `PerformQueryAsyncCPU` | CPU | 高吞吐 | 结果延迟 1 帧 |
| `QuerySceneDepth` | GPU | 快 | 只能查屏幕内像素 |
| `QueryMeshDistanceField` | GPU | 较快 | 全场景 DF(需 DF 开启) |

## 资源需求

```cpp
RequiresDistanceFieldData() → true;    // GPU DF
RequiresDepthBuffer() → true;          // GPU depth
HasPreSimulateTick() → true;           // 发起异步查询
HasPostSimulateTick() → true;          // 读回异步结果
```

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterface]]

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraDataInterfaceCollisionQuery]]
- 读本:[[Readers/Niagara/Phase 7 - 最强扩展点 Data Interface]] § 5
