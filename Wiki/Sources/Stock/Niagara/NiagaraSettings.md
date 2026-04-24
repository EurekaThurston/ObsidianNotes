---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, settings, project-settings]
sources: 1
aliases: [NiagaraSettings.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSettings.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraSettings.h

- **Repo**: stock · **路径**: `Public/NiagaraSettings.h` · **规模**: 75 行
- **Phase**: 9.4

## 职责

`UNiagaraSettings : UDeveloperSettings` —— Niagara **项目配置**,落在 `Project Settings > Plugins > Niagara`。存 DefaultEffectType、QualityLevels、default RT/Grid format、component renderer 警告等。

## 类

```cpp
UCLASS(config = Niagara, defaultconfig, meta=(DisplayName="Niagara"))
class NIAGARA_API UNiagaraSettings : public UDeveloperSettings
{
#if WITH_EDITORONLY_DATA
    // 编辑器允许添加自定义类型到 Niagara 类型系统
    UPROPERTY(config, EditAnywhere)
    TArray<FSoftObjectPath> AdditionalParameterTypes;
    UPROPERTY(config, EditAnywhere)
    TArray<FSoftObjectPath> AdditionalPayloadTypes;
    UPROPERTY(config, EditAnywhere)
    TArray<FSoftObjectPath> AdditionalParameterEnums;
#endif

    /** 默认 EffectType,Asset 没配就用这个 */
    UPROPERTY(config, EditAnywhere, meta = (AllowedClasses = "NiagaraEffectType"))
    FSoftObjectPath DefaultEffectType;

    /** 质量等级名字数组(Low/Medium/High/Epic/Cinematic...)*/
    UPROPERTY(config, EditAnywhere, Category = Scalability)
    TArray<FText> QualityLevels;

    /** Component renderer 按 component class 展示的警告 */
    UPROPERTY(config, EditAnywhere, Category = Renderer)
    TMap<FString, FText> ComponentRendererWarningsPerClass;

    /** Niagara RenderTarget DI 的默认格式 */
    UPROPERTY(config, EditAnywhere, Category = Renderer)
    TEnumAsByte<ETextureRenderTargetFormat> DefaultRenderTargetFormat = RTF_RGBA16f;

    /** Niagara Grid DI 的默认 buffer format(Phase 10)*/
    UPROPERTY(config, EditAnywhere, Category = Renderer)
    ENiagaraGpuBufferFormat DefaultGridFormat = ENiagaraGpuBufferFormat::HalfFloat;

    UNiagaraEffectType* GetDefaultEffectType() const;

#if WITH_EDITOR
    DECLARE_MULTICAST_DELEGATE_TwoParams(FOnNiagaraSettingsChanged, const FName&, const UNiagaraSettings*);
    static FOnNiagaraSettingsChanged& OnSettingsChanged();
#endif
};
```

## 使用点

- `UNiagaraSystem::GetEffectType()` 没配时 → `UNiagaraSettings::GetDefaultEffectType()`
- `FNiagaraWorldManager::CalculateScalabilityState` 读 `QualityLevels.Num()` 决定循环
- RenderTarget DI 初始化 fallback

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraSettings]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraEffectType]]
