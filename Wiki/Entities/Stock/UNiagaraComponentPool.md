---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, component-pool, uclass, pooling]
sources: 1
aliases: [UNiagaraComponentPool, FNCPool, FNCPoolElement, ENCPoolMethod]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraComponentPool.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraComponentPool

> Niagara Component **池化系统**。复用 `UNiagaraComponent` 避免频繁 GC。Phase 2 `SpawnSystemAt*` 的 `PoolingMethod` 参数背后。

## 一句话角色

`UCLASS(Transient) UNiagaraComponentPool : UObject`。每 `FNiagaraWorldManager` 一个。按 `TMap<UNiagaraSystem*, FNCPool>` 分组——一个 Asset 一组池。

## `ENCPoolMethod` 5 取值

| 方法 | 语义 | 使用场景 |
|---|---|---|
| `None` | 不进池 | 高级用户自管 |
| `AutoRelease` | 从池取,播完自动回 | ⭐ fire-and-forget(爆炸、命中) |
| `ManualRelease` | 从池取,用户必须 ReleaseToPool | 需要持续配置参数 |
| `ManualRelease_OnComplete` | 手动释放但等 complete 才真回池 | 折中 |
| `FreeInPool` | 已在池里待用(内部) | 不直接用 |

## `FNCPool`(单 Asset 的池)

```cpp
USTRUCT() struct FNCPool {
    TArray<FNCPoolElement> FreeElements;              // 空闲池(FNCPoolElement = Component + LastUsedTime)
    TArray<UNiagaraComponent*> InUseComponents_Auto;
    TArray<UNiagaraComponent*> InUseComponents_Manual;
    int32 MaxUsed;

    UNiagaraComponent* Acquire(UWorld*, Template, ENCPoolMethod, bForceNew=false);
    void Reclaim(UNiagaraComponent*, CurrentTimeSeconds);
    void KillUnusedComponents(float KillTime, UNiagaraSystem* Template);
};
```

## 主类 API

```cpp
UNiagaraComponent* CreateWorldParticleSystem(UNiagaraSystem* Template, UWorld*, ENCPoolMethod);
void ReclaimWorldParticleSystem(UNiagaraComponent*);
void PrimePool(UNiagaraSystem* Template, UWorld*);   // 预热
void ReclaimActiveParticleSystems();
```

## `PrimePool` 的意义

世界加载时预分配 N 个 Component,避免战斗首次 spawn 卡顿。`FNiagaraWorldManager::PrimePoolForAllWorlds` 触发。

## 陷阱

- ⚠️ `ManualRelease` 忘了 `ReleaseToPool` 会泄露(Component 永远在 `InUseComponents_Manual`)
- ⚠️ `AutoRelease` 不能在 tick 后访问 NC(可能已回池被复用)
- ⚠️ 同 Asset 的 pool `TInlineAllocator` 没用——规模大时 FreeElements/InUseComponents 堆分配

## 相关

- [[Wiki/Entities/Stock/UNiagaraComponent]] — 被池化的对象(Phase 2)
- [[Wiki/Entities/Stock/FNiagaraWorldManager]] — 持有者
- [[Wiki/Entities/Stock/UNiagaraFunctionLibrary]] — `SpawnSystemAt*` 用本 pool

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraComponentPool]]
- 读本:[[Readers/Niagara/Phase9-world-management-读本]]
