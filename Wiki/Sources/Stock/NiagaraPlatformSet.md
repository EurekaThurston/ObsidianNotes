---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, platform-set, scalability, device-profile]
sources: 1
aliases: [NiagaraPlatformSet.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraPlatformSet.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraPlatformSet.h

- **Repo**: stock · **路径**: `Classes/NiagaraPlatformSet.h` · **规模**: 378 行(扒前 200)
- **Phase**: 9.6

## 职责

定义 **`FNiagaraPlatformSet`**——**跨平台 × 质量等级** 的 enable/disable 决策系统。每个 Scalability Settings 条目 / 每个 Renderer Properties / 每个 Emitter 的 Platform 过滤都用本类。

## 3 种状态

### `ENiagaraPlatformSelectionState`(L11)

```cpp
enum class ENiagaraPlatformSelectionState : uint8 {
    Default,    // 按 PlatformSet 其他规则
    Enabled,    // 显式启用
    Disabled,   // 显式禁用
};
```

### `ENiagaraPlatformSetState`(L52)

```cpp
enum class ENiagaraPlatformSetState : uint8 {
    Disabled,   // 本 platform set 禁用
    Enabled,    // 本 device profile 启用但未激活
    Active,     // 启用且当前激活
    Unknown,
};
```

## `FNiagaraDeviceProfileStateEntry`(L19)

```cpp
USTRUCT() struct FNiagaraDeviceProfileStateEntry {
    FName ProfileName;              // 对应 UE DeviceProfile 名
    uint32 QualityLevelMask;        // 每 quality level 的启用 bit
    uint32 SetQualityLevelMask;     // 哪些 quality level 被显式设置(未设 → Default)

    ENiagaraPlatformSelectionState GetState(int32 QualityLevel) const;
    void SetState(int32 QualityLevel, ENiagaraPlatformSelectionState);
    bool AllDefaulted() const { return SetQualityLevelMask == 0; }
};
```

**双 mask 设计**:`QualityLevelMask` 是启用值,`SetQualityLevelMask` 是"是否显式设过"——三态压成两个 bit 的方法。

## Conflict 检测

```cpp
USTRUCT() struct FNiagaraPlatformSetConflictEntry { FName ProfileName; int32 QualityLevelMask; };
USTRUCT() struct FNiagaraPlatformSetConflictInfo { int32 SetAIndex / SetBIndex; TArray<FNiagaraPlatformSetConflictEntry> Conflicts; };
```

多个 `FNiagaraPlatformSet` 同时启用同 platform × quality → 冲突(两个 scalability 规则都适用)。Niagara editor 会检测并警告。

## `FNiagaraPlatformSetCVarCondition`(L124)

```cpp
USTRUCT() struct FNiagaraPlatformSetCVarCondition {
    FName CVarName;                 // CVar 名
    bool Value;                     // bool 类型必须等于
    int32 MinInt / MaxInt;          // int 类型范围
    float MinFloat / MaxFloat;
    uint32 bUseMinInt : 1 / bUseMaxInt : 1 / bUseMinFloat : 1 / bUseMaxFloat : 1;

    bool IsEnabledForPlatform(const FString& PlatformName) const;
    bool IsEnabledForDeviceProfile(const UDeviceProfile*, bool bCheckCurrentStateOnly) const;

    bool CheckValue(bool) / CheckValue(int32) / CheckValue(float);
    static void OnCVarChanged(IConsoleVariable*);
};
```

PlatformSet 可以**额外**要求某 CVar 值符合条件才启用。支持 bool / int-range / float-range。

## `FDeviceProfileValueCache`(L104,editor-only)

```cpp
struct FDeviceProfileValueCache {
    static void Empty();
    template<typename T>
    static bool GetValue(const UDeviceProfile*, FName CVarName, T& OutValue);

    static TMap<const UDeviceProfile*, FCVarValueMap> CachedDeviceProfileValues;
    static TMap<FName, FCVarValueMap> CachedPlatformValues;
};
```

Editor 里缓存 device profile 的 CVar 值——避免反复解析 ini 文件。

## 后 180 行未读

包括 `FNiagaraPlatformSet` 主类、实际 `IsActive / IsEnabled` 判断逻辑、编辑器 UI 辅助。

## 涉及实体

- [[Wiki/Entities/Stock/FNiagaraPlatformSet]]
- [[Wiki/Entities/Stock/UNiagaraEffectType]] — 每 ScalabilitySettings 一个 PlatformSet
- Phase 6 `UNiagaraRendererProperties::Platforms`
