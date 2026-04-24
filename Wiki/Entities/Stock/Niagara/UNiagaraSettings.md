---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, settings, developer-settings, uclass]
sources: 1
aliases: [UNiagaraSettings]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSettings.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraSettings

> Niagara **项目级配置**。`UDeveloperSettings` 子类,落在 Project Settings > Plugins > Niagara。

## 一句话角色

`UCLASS(config = Niagara, defaultconfig) UNiagaraSettings : UDeveloperSettings`。编辑器配置项统一入口。

## 核心字段

| 字段 | 作用 |
|---|---|
| `DefaultEffectType` | Asset 没配 EffectType 时用的默认 |
| `QualityLevels` | 质量等级名字数组(Low/Medium/High/Epic/...) |
| `ComponentRendererWarningsPerClass` | Component Renderer 按 class 展示的警告 |
| `DefaultRenderTargetFormat` | Niagara RT DI 默认格式(默认 RGBA16f) |
| `DefaultGridFormat` | Niagara Grid DI 默认 buffer format(默认 HalfFloat) |
| (editor) `AdditionalParameterTypes / PayloadTypes / ParameterEnums` | 把自定义 UStruct/UEnum 注册进 Niagara 类型系统 |

## Delegate

```cpp
#if WITH_EDITOR
DECLARE_MULTICAST_DELEGATE_TwoParams(FOnNiagaraSettingsChanged, const FName&, const UNiagaraSettings*);
static FOnNiagaraSettingsChanged& OnSettingsChanged();
#endif
```

其他系统订阅——质量等级改名、Default EffectType 换了都能响应。

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraEffectType]]
- Phase 10 Grid DI 格式
- Phase 7 RenderTarget DI 格式

## 深入阅读

- 源:[[Wiki/Sources/Stock/Niagara/NiagaraSettings]]
