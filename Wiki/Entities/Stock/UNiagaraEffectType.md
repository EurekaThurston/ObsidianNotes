---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, effect-type, scalability, significance]
sources: 1
aliases: [UNiagaraEffectType, FNiagaraSystemScalabilitySettings, FNiagaraEmitterScalabilitySettings, UNiagaraSignificanceHandler]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEffectType.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraEffectType 家族

> **特效分类单元**(如 Blood / Fire / Gameplay VFX)+ 共享 scalability 预算。本页合并 EffectType + 4 种 ScalabilitySettings struct + SignificanceHandler 家族。

## 一句话角色

`UCLASS UNiagaraEffectType : UObject`。每 `UNiagaraSystem::EffectType` 引用一个,同 EffectType 的所有 system 共享 instance count 预算、cull 规则、significance handler。

## 关键枚举

### `ENiagaraCullReaction`

```cpp
Deactivate           // "Kill" — 停 spawn,自然消亡,不自动恢复
DeactivateImmediate  // "Kill and Clear" — 立刻清,不恢复
DeactivateResume     // "Asleep" — 停 spawn,条件好恢复
DeactivateImmediateResume  // "Asleep and Clear"
```

### `ENiagaraScalabilityUpdateFrequency`

```cpp
SpawnOnly / Low / Medium / High / Continuous
```

## 核心字段

```cpp
ENiagaraScalabilityUpdateFrequency UpdateFrequency;
ENiagaraCullReaction CullReaction;
UNiagaraSignificanceHandler* SignificanceHandler;

FNiagaraSystemScalabilitySettingsArray SystemScalabilitySettings;    // 多 entry,按 Platforms 选
FNiagaraEmitterScalabilitySettingsArray EmitterScalabilitySettings;

int32 NumInstances;                                          // 跨 world 总数
uint32 bNewSystemsSinceLastScalabilityUpdate : 1;

// Perf stats
float AvgTimeMS_GT / GT_CNC / RT;
FInGameCycleHistory CycleHistory_GT / GT_CNC / RT;
```

## `FNiagaraSystemScalabilitySettings`

```cpp
FNiagaraPlatformSet Platforms;

uint32 bCullByDistance : 1;
uint32 bCullMaxInstanceCount : 1;            // EffectType 级
uint32 bCullPerSystemMaxInstanceCount : 1;    // System 级
uint32 bCullByMaxTimeWithoutRender : 1;       // 可见性

float MaxDistance;
int32 MaxInstances / MaxSystemInstances;
float MaxTimeWithoutRender;
```

配套 `Override`(子类)让单个 System 覆盖 EffectType 默认。

## `FNiagaraEmitterScalabilitySettings`

简单——只有 `bScaleSpawnCount` + `SpawnCountScale`。

## `UNiagaraSignificanceHandler`

```cpp
UCLASS(abstract, EditInlineNew)
class UNiagaraSignificanceHandler {
    virtual void CalculateSignificance(TArray<Component*>&, TArray<ScalabilityState>&) = 0;
};

// 两个默认实现:
UNiagaraSignificanceHandlerDistance  // 距离越近越重要
UNiagaraSignificanceHandlerAge       // 越新越重要
```

用户可自实现,"玩家相关更重要"等。

## Active Settings 选择

```cpp
GetActiveSystemScalabilitySettings()
    → 遍历 SystemScalabilitySettings.Settings
    → 按 Platforms(FNiagaraPlatformSet)匹配当前平台 × quality
    → 返回第一个匹配的
```

## Runtime Cycle Counter

```cpp
int32* GetCycleCounter(bool bGameThread, bool bConcurrent);
void ProcessLastFrameCycleCounts();
```

future:perf-based dynamic budget(TODO 注释)。

## 相关

- [[Wiki/Entities/Stock/FNiagaraScalabilityManager]] — 按 EffectType 管理的 manager
- [[Wiki/Entities/Stock/UNiagaraSystem]] — 引用 EffectType
- [[Wiki/Entities/Stock/FNiagaraPlatformSet]] — Settings 里的 Platforms

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraEffectType]]
- 读本:[[Readers/Niagara/Phase 9 - Niagara 的世界管理与可扩展性]]
