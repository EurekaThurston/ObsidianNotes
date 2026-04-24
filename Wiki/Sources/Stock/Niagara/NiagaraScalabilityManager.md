---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, scalability, significance, culling]
sources: 1
aliases: [NiagaraScalabilityManager.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraScalabilityManager.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraScalabilityManager.h

- **Repo**: stock · **路径**: `Public/NiagaraScalabilityManager.h` · **规模**: 140 行
- **Phase**: 9.2

## 职责

定义 **`FNiagaraScalabilityManager`**——按 `UNiagaraEffectType` 分组的 scalability 状态管理器。每个 `UNiagaraEffectType` 在 `FNiagaraWorldManager::ScalabilityManagers` 有一份,负责该 effect type 所有 components 的距离/可见性/instance count/significance 排序 cull。

## `FNiagaraScalabilityState`(L83)

```cpp
struct FNiagaraScalabilityState {
    float Significance = 1.0f;     // 重要度(SignificanceHandler 算出)
    uint8 bDirty : 1;
    uint8 bCulled : 1;

#if DEBUG_SCALABILITY_STATE
    uint8 bCulledByDistance : 1;
    uint8 bCulledByInstanceCount : 1;
    uint8 bCulledByVisibility : 1;
#endif
};
```

每 Component 一份,反映当前是否被 cull + 为什么 cull。

## `FNiagaraScalabilityManager` USTRUCT(L108)

```cpp
USTRUCT()
struct FNiagaraScalabilityManager {
    UPROPERTY(transient) UNiagaraEffectType* EffectType;
    UPROPERTY(transient) TArray<UNiagaraComponent*> ManagedComponents;   // 注册的 components

    TArray<FNiagaraScalabilityState> State;         // 并行 State[i] 对应 ManagedComponents[i]
    TArray<int32> SignificanceSortedIndices;        // 按 Significance 降序排序的索引
    float LastUpdateTime;

    void Update(FNiagaraWorldManager* Owner, bool bNewOnly);  // 主刷新
    void Register(UNiagaraComponent*);
    void Unregister(UNiagaraComponent*);

    void AddReferencedObjects(FReferenceCollector&);
    void PreGarbageCollectBeginDestroy();

private:
    void UnregisterAt(int32 IndexToRemove);
};
```

## 工作流

```
Component Activate → WorldManager::RegisterWithScalabilityManager → 对应 EffectType 的 Manager.Register
    → ManagedComponents.Add(Component) + State.Add({}) + Component->ScalabilityManagerHandle = Idx

每 tick(按 EffectType::UpdateFrequency):
    WorldManager::UpdateScalabilityManagers
        → 每个 ScalabilityManager::Update
            → 对每个 component:
                → CalculateScalabilityState(...)
                    → DistanceCull / VisibilityCull / InstanceCountCull
                → 写入 State[i]
            → 如 SignificanceHandler 存在:
                → SignificanceHandler->CalculateSignificance(Components, State)
                → Sort SignificanceSortedIndices by Significance desc
            → SortedSignificanceCull:按 MaxInstances 从低 significance 开始 cull

Component Deactivate → Unregister → Swap-pop ManagedComponents + State
```

## `FNiagaraScopedRuntimeCycleCounter`(L63)

RAII 性能计数器,`ENABLE_NIAGARA_RUNTIME_CYCLE_COUNTERS` 默认关(0),未来可能开启。用于 perf-based scalability 反馈。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraScalabilityManager]]
- [[Wiki/Entities/Stock/Niagara/FNiagaraWorldManager]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraEffectType]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraComponent]] — `ScalabilityManagerHandle`,friend 访问(Phase 2)
