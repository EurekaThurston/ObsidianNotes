---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, scalability, significance, cull]
sources: 1
aliases: [FNiagaraScalabilityManager, FNiagaraScalabilityState]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraScalabilityManager.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraScalabilityManager

> **按 `UNiagaraEffectType` 分组**的 scalability 状态管理器。Phase 2 Component 的 `ScalabilityManagerHandle` 指向这里的 `ManagedComponents` 数组索引。

## 一句话角色

`USTRUCT FNiagaraScalabilityManager`。每个 EffectType 在 `FNiagaraWorldManager::ScalabilityManagers` 有一份。负责该 effect type 的 distance / visibility / instance count 决策,按 significance 降序 cull。

## `FNiagaraScalabilityState`

```cpp
struct FNiagaraScalabilityState {
    float Significance = 1.0f;
    uint8 bDirty : 1;
    uint8 bCulled : 1;
#if DEBUG_SCALABILITY_STATE
    uint8 bCulledByDistance : 1;
    uint8 bCulledByInstanceCount : 1;
    uint8 bCulledByVisibility : 1;
#endif
};
```

每 Component 一份,反映当前 cull 状态 + debug 原因。

## 主字段

```cpp
UNiagaraEffectType* EffectType;
TArray<UNiagaraComponent*> ManagedComponents;   // 注册的 components
TArray<FNiagaraScalabilityState> State;          // 并行
TArray<int32> SignificanceSortedIndices;         // 按 Significance 降序
float LastUpdateTime;
```

## 生命周期

```cpp
void Register(UNiagaraComponent*);
void Unregister(UNiagaraComponent*);
void Update(FNiagaraWorldManager*, bool bNewOnly);   // 主刷新
void UnregisterAt(int32 IndexToRemove);              // swap-pop
```

## Update 流程

```
1. 遍历 ManagedComponents:
    - CalculateScalabilityState(Component, State[i])  // DistanceCull / VisibilityCull / InstanceCountCull
2. 如 EffectType 有 SignificanceHandler:
    - CalculateSignificance(Components, State)
    - Sort SignificanceSortedIndices by State[i].Significance desc
3. SortedSignificanceCull:
    - 按 MaxInstances 从低 significance 开始标 bCulled=true
4. 回写 Component 的 bIsCulledByScalability → ActivateInternal(bIsScalabilityCull=true) 或 DeactivateInternal(同)
```

## 相关

- [[Wiki/Entities/Stock/FNiagaraWorldManager]] — 持有者
- [[Wiki/Entities/Stock/UNiagaraEffectType]] — key + settings 来源
- [[Wiki/Entities/Stock/UNiagaraComponent]] — 被管理;`ScalabilityManagerHandle` 是索引

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraScalabilityManager]]
- 读本:[[Readers/Niagara/Phase9-world-management-读本]]
