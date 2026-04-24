---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, component-pool, pool, pooling]
sources: 1
aliases: [NiagaraComponentPool.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraComponentPool.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraComponentPool.h

- **Repo**: stock · **路径**: `Public/NiagaraComponentPool.h` · **规模**: 149 行
- **Phase**: 9.3

## 职责

定义 Niagara **Component 池化系统**——复用 `UNiagaraComponent` 实例,避免频繁 GC。Phase 2 `UNiagaraFunctionLibrary::SpawnSystemAt*` 的 `PoolingMethod` 参数的具体实现。

## `ENCPoolMethod`(L14)

```cpp
enum class ENCPoolMethod : uint8
{
    None,                      // 全新 new,不进池
    AutoRelease,               // 从池取 + 用户无需管理释放;fire-and-forget
    ManualRelease,             // 从池取,用户 own + 显式 ReleaseToPool
    ManualRelease_OnComplete,  // 手动 release 但等 complete 时才真的回池
    FreeInPool,                // 标记当前在池里待用(内部用)
};
```

用户选择:

| 场景 | 方法 |
|---|---|
| 爆炸、命中、一次性 | `AutoRelease`(最简) |
| 需要配置参数后保留 | `ManualRelease`(必须记得 release!) |
| 玩家可能放弃控制的 effect | `ManualRelease_OnComplete`(折中) |
| 高级用户 | `None`(自管 GC) |

## `FNCPoolElement`(L47)

```cpp
USTRUCT()
struct FNCPoolElement {
    UPROPERTY(transient) UNiagaraComponent* Component;
    float LastUsedTime;
};
```

## `FNCPool`(L70)— 单 Asset 的池

```cpp
USTRUCT()
struct FNCPool {
    TArray<FNCPoolElement> FreeElements;                  // 空闲池
    TArray<UNiagaraComponent*> InUseComponents_Auto;      // Auto 类正在用的
    TArray<UNiagaraComponent*> InUseComponents_Manual;    // Manual 类正在用的
    int32 MaxUsed;                                        // 最大同时使用数

    void Cleanup(bool bFreeOnly);
    UNiagaraComponent* Acquire(UWorld*, UNiagaraSystem* Template, ENCPoolMethod, bool bForceNew=false);
    void Reclaim(UNiagaraComponent*, float CurrentTimeSeconds);
    bool RemoveComponent(UNiagaraComponent*);
    void KillUnusedComponents(float KillTime, UNiagaraSystem* Template);
    int32 NumComponents();
};
```

TODO 注释:考虑改 FIFO queue(TCircularQueue)避免总用最后一个元素。

## `UNiagaraComponentPool`(L111,world 级容器)

```cpp
UCLASS(Transient)
class UNiagaraComponentPool : public UObject
{
    UPROPERTY() TMap<UNiagaraSystem*, FNCPool> WorldParticleSystemPools;

    UNiagaraComponent* CreateWorldParticleSystem(UNiagaraSystem* Template, UWorld*, ENCPoolMethod);
    void ReclaimWorldParticleSystem(UNiagaraComponent*);
    void ReclaimActiveParticleSystems();

    void Cleanup(bool bFreeOnly=false);
    void ClearPool(UNiagaraSystem*);
    void PrimePool(UNiagaraSystem* Template, UWorld*);

    void PooledComponentDestroyed(UNiagaraComponent*);
    void RemoveComponentsBySystem(UNiagaraSystem*);

    void Dump();
};
```

被 `FNiagaraWorldManager::ComponentPool` 持有(每 World 一个)。按 `UNiagaraSystem*` 分 FNCPool—— 一个 Asset 一组池。

## `PrimePool`

预分配——世界加载时提前创建 N 个 Component 避免战斗中首次 spawn 卡顿。触发点:`OnWorldInit` 后手动 `PrimePoolForAllWorlds`。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraComponentPool]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraComponent]] — `PoolingMethod` 字段(Phase 2)
