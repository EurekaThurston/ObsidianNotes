---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, platform-set, scalability, cvar]
sources: 1
aliases: [FNiagaraPlatformSet, FNiagaraDeviceProfileStateEntry, FNiagaraPlatformSetCVarCondition]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraPlatformSet.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraPlatformSet

> **跨 platform × quality level** 的启用/禁用决策。用于 EffectType Scalability Settings、Renderer Platforms、Emitter Platforms 等任何"此条目在哪些平台生效"的场景。

## 一句话角色

`struct FNiagaraPlatformSet`(USTRUCT)。条件组合:device profile 列表(每个带 quality mask)+ CVar 条件。查询"当前平台 × quality level 下本 set 是否启用"。

## 3 态(Selection State)

```cpp
enum class ENiagaraPlatformSelectionState : uint8 {
    Default,    // 按其他规则
    Enabled,    // 显式启用
    Disabled,   // 显式禁用
};
```

## `FNiagaraDeviceProfileStateEntry`

```cpp
struct FNiagaraDeviceProfileStateEntry {
    FName ProfileName;              // UE DeviceProfile
    uint32 QualityLevelMask;        // 启用 bit
    uint32 SetQualityLevelMask;     // 哪些 QL 被显式设置(双 mask 三态)
};
```

**双 mask 设计** = 三态压两个 bit:`Set` 表示"显式过","Set & QualityMask" 是 Enabled,"Set & ~QualityMask" 是 Disabled,"未 Set" 是 Default。

## `FNiagaraPlatformSetCVarCondition`

```cpp
struct FNiagaraPlatformSetCVarCondition {
    FName CVarName;
    bool Value;                                    // bool 要等于
    int32 MinInt / MaxInt;                          // int range
    float MinFloat / MaxFloat;                     // float range
    uint32 bUseMinInt / MaxInt / MinFloat / MaxFloat : 1;  // 哪些约束生效
};
```

**除了 device profile,还可以按 CVar 值过滤** —— 比如"r.MyCVar >= 3 才启用"。

## Conflict 检测

```cpp
FNiagaraPlatformSetConflictEntry / FNiagaraPlatformSetConflictInfo
```

多个 PlatformSet 同时启用同 platform × quality → 冲突,editor 警告。

## `FDeviceProfileValueCache`(editor)

缓存 device profile 的 CVar 值,避免反复解析 ini。

## 使用点

- `UNiagaraEffectType::SystemScalabilitySettings[].Platforms`
- `UNiagaraRendererProperties::Platforms`
- `UNiagaraEmitter` 的 Platforms

## 后 180 行未扒

实际 `IsActive / IsEnabled` 查询逻辑、editor UI 辅助在本次未读区。

## 相关

- [[Wiki/Entities/Stock/UNiagaraEffectType]]
- [[Wiki/Entities/Stock/UNiagaraRendererProperties]]
- [[Wiki/Entities/Stock/UNiagaraSettings]] — `QualityLevels` 来源

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraPlatformSet]]
- 读本:[[Readers/Niagara/Phase 9 - Niagara 的世界管理与可扩展性]]
