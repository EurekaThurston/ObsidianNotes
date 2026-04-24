---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, effect-type, scalability, significance]
sources: 1
aliases: [NiagaraEffectType.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEffectType.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraEffectType.h

- **Repo**: stock · **路径**: `Classes/NiagaraEffectType.h` · **规模**: 337 行
- **Phase**: 9.5(**核心 scalability**)

## 职责

定义 **`UNiagaraEffectType`**——特效的**分类单元**(如 "Blood"、"Fire"、"Gameplay VFX")及其共享 scalability 设置。每个 `UNiagaraSystem::EffectType` 引用一个本类实例。同个 EffectType 的所有 system 共享 instance count 预算、cull 规则、significance handler。

## 关键枚举

### `ENiagaraCullReaction`(L14)

```cpp
enum class ENiagaraCullReaction
{
    Deactivate,                  // "Kill" — 停 spawn,粒子自然消亡,不自动恢复
    DeactivateImmediate,         // "Kill and Clear" — 立刻清粒子,不恢复
    DeactivateResume,            // "Asleep" — 停 spawn,通过 cull 检查后自动恢复
    DeactivateImmediateResume,   // "Asleep and Clear" — 立刻清,自动恢复
};
```

### `ENiagaraScalabilityUpdateFrequency`(L30)

```cpp
enum class ENiagaraScalabilityUpdateFrequency {
    SpawnOnly,   // 仅 spawn 时检查
    Low,
    Medium,
    High,
    Continuous,  // 每帧
};
```

## `FNiagaraSystemScalabilitySettings`(L48)

```cpp
USTRUCT() struct FNiagaraSystemScalabilitySettings {
    FNiagaraPlatformSet Platforms;             // 哪些平台生效

    uint32 bCullByDistance : 1;
    uint32 bCullMaxInstanceCount : 1;           // EffectType 级总数
    uint32 bCullPerSystemMaxInstanceCount : 1;   // System 级总数
    uint32 bCullByMaxTimeWithoutRender : 1;      // 可见性 cull

    float MaxDistance;
    int32 MaxInstances;                          // EffectType 级
    int32 MaxSystemInstances;                    // 同 System 级
    float MaxTimeWithoutRender;
};
```

TODO 注释提到 `ScreenFraction`(屏幕占比 cull,未实现)。

### 配套 Override

```cpp
USTRUCT() struct FNiagaraSystemScalabilityOverride : public FNiagaraSystemScalabilitySettings {
    uint32 bOverrideDistanceSettings : 1;
    uint32 bOverrideInstanceCountSettings : 1;
    uint32 bOverridePerSystemInstanceCountSettings : 1;
    uint32 bOverrideTimeSinceRendererSettings : 1;
};
```

EffectType 设默认,某 System 可以 override。

## `FNiagaraEmitterScalabilitySettings`(L145)

```cpp
USTRUCT() struct FNiagaraEmitterScalabilitySettings {
    FNiagaraPlatformSet Platforms;
    uint32 bScaleSpawnCount : 1;
    float SpawnCountScale;
};
```

简单——Emitter 级只有 spawn count scale。

## `UNiagaraSignificanceHandler`(L208)

```cpp
UCLASS(abstract, EditInlineNew)
class UNiagaraSignificanceHandler : public UObject {
    virtual void CalculateSignificance(TArray<UNiagaraComponent*>&, TArray<FNiagaraScalabilityState>&) PURE_VIRTUAL(...);
};
```

两个默认实现:
- `UNiagaraSignificanceHandlerDistance`(L219): 距离越近越重要
- `UNiagaraSignificanceHandlerAge`(L229): 越新越重要

用户可自写 handler——"玩家相关特效更重要" 等。

## `UNiagaraEffectType` 主类(L241)

```cpp
UCLASS()
class UNiagaraEffectType : public UObject
{
    UPROPERTY(EditAnywhere)
    ENiagaraScalabilityUpdateFrequency UpdateFrequency;

    UPROPERTY(EditAnywhere)
    ENiagaraCullReaction CullReaction;

    UPROPERTY(EditAnywhere, Instanced)
    UNiagaraSignificanceHandler* SignificanceHandler;

    UPROPERTY(EditAnywhere)
    FNiagaraSystemScalabilitySettingsArray SystemScalabilitySettings;

    UPROPERTY(EditAnywhere)
    FNiagaraEmitterScalabilitySettingsArray EmitterScalabilitySettings;

    int32 NumInstances;                                     // 跨 world 总数
    uint32 bNewSystemsSinceLastScalabilityUpdate : 1;

    // Perf monitoring
    float AvgTimeMS_GT / AvgTimeMS_GT_CNC / AvgTimeMS_RT;
    FInGameCycleHistory CycleHistory_GT / CycleHistory_GT_CNC / CycleHistory_RT;
    int32 FramesSincePerfSampled;
    bool bSampleRunTimePerfThisFrame;

    const FNiagaraSystemScalabilitySettings& GetActiveSystemScalabilitySettings() const;
    const FNiagaraEmitterScalabilitySettings& GetActiveEmitterScalabilitySettings() const;

    int32* GetCycleCounter(bool bGameThread, bool bConcurrent);
    void ProcessLastFrameCycleCounts();

    FRenderCommandFence ReleaseFence;
};
```

## Active Settings 选择

```cpp
GetActiveSystemScalabilitySettings()
    → 遍历 SystemScalabilitySettings.Settings
    → 按 Platforms(FNiagaraPlatformSet)匹配当前平台/quality level
    → 返回第一个匹配的 entry
```

每个 Settings 有 `FNiagaraPlatformSet Platforms`(Phase 9.6),决定在哪些平台生效。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraEffectType]]
- [[Wiki/Entities/Stock/Niagara/FNiagaraPlatformSet]]
- [[Wiki/Entities/Stock/Niagara/FNiagaraScalabilityManager]]
