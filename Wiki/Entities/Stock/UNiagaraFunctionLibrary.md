---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, function-library, blueprint]
sources: 1
aliases: [UNiagaraFunctionLibrary, NiagaraFunctionLibrary]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraFunctionLibrary.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraFunctionLibrary

> Niagara 给 Blueprint 和游戏代码的**静态工具集**。所有函数都是 static,自身不持有状态——是 `UNiagaraComponent` 的 BP 外立面。

## 一句话角色

`UNiagaraFunctionLibrary : public UBlueprintFunctionLibrary`。按用途分四组:

| 组 | 代表函数 | 用途 |
|---|---|---|
| 1. 动态生成特效 | `SpawnSystemAtLocation` / `SpawnSystemAttached` | BP 里最常用的两个 Niagara 函数;返回 `UNiagaraComponent*` |
| 2. 重型 User 参数覆盖 | `OverrideSystemUserVariable{StaticMesh,SkeletalMesh,...}` / `SetTextureObject` / `SetVolumeTextureObject` | 覆盖 DI 承载的对象参数(轻量参数走 Component 的 SetVariable*) |
| 3. Parameter Collection | `GetNiagaraParameterCollection` | 类比 `MaterialParameterCollection`,跨特效共享参数 |
| 4. VectorVM Fast Path | `GetVectorVMFastPathOps` / `DefineFunctionHLSL` / `GetVectorVMFastPathExternalFunction` | **C++ internal**,CPU VM 性能优化;Phase 5 深入 |

## Spawn 函数签名速查

```cpp
// 世界坐标生成(爆炸、命中)
static UNiagaraComponent* SpawnSystemAtLocation(
    const UObject* WorldContextObject,
    UNiagaraSystem* SystemTemplate,
    FVector Location, FRotator Rotation, FVector Scale,
    bool bAutoDestroy = true, bool bAutoActivate = true,
    ENCPoolMethod PoolingMethod = ENCPoolMethod::None,  // ⚠️ 默认不用池
    bool bPreCullCheck = true);

// 挂载到父组件 socket(武器刀光、Buff)
static UNiagaraComponent* SpawnSystemAttached(
    UNiagaraSystem* SystemTemplate,
    USceneComponent* AttachToComponent, FName AttachPointName,
    FVector Location, FRotator Rotation,
    EAttachLocation::Type LocationType,
    bool bAutoDestroy, bool bAutoActivate = true,
    ENCPoolMethod PoolingMethod = ENCPoolMethod::None,
    bool bPreCullCheck = true);
```

## 陷阱

- ⚠️ **`PoolingMethod::None` 是默认值**:高频 Spawn 必须显式传 `ENCPoolMethod::AutoRelease` 等,否则 GC 压力大
- ⚠️ **`UnsafeDuringActorConstruction`**:不能在构造阶段调用,World 未初始化
- ⚠️ **`bPreCullCheck=true`** 默认开:scalability 可能直接拒绝生成,调试时注意

## 相关

- [[Wiki/Entities/Stock/UNiagaraComponent]] — Spawn 的返回类型
- [[Wiki/Entities/Stock/UNiagaraSystem]] — Spawn 的 Template 参数
- [[Wiki/Entities/Stock/ANiagaraActor]] — 另一种"静态放置"路径(本类是"动态生成"路径)

## 深入阅读

- 全字段清单 + 代码片段:[[Wiki/Sources/Stock/NiagaraFunctionLibrary]]
- 主题读本:[[Readers/Niagara/Phase 2 - Component 层的五职责]] § 5

## 开放问题

- VectorVM Fast Path 注册了哪些算子?→ Phase 5
- `SpawnSystemAttached` 两个重载参数顺序不一致,历史债务?→ 非关键
- 默认 `bPreCullCheck=true` 的影响?→ Phase 9
