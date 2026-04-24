---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, properties, asset]
sources: 1
aliases: [NiagaraRendererProperties.h, UNiagaraRendererProperties 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraRendererProperties.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererProperties.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 259 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.1(**基类**)

## 职责

定义 Niagara 渲染器的 **Asset 侧基类** `UNiagaraRendererProperties`,以及两个辅助结构:
- `FNiagaraRendererVariableInfo` — 单个 VF 变量的 dataset→GPU buffer offset 映射
- `FNiagaraRendererLayout` — 所有 VF 变量布局的 GT+RT 双 copy

本文件是 Phase 6 所有 Asset 侧 Properties 的共同基类(Sprite/Ribbon/Mesh/Light Properties 都继承)。

## 关键类型

### `UNiagaraRendererProperties`(L157)

```cpp
UCLASS(ABSTRACT)
class NIAGARA_API UNiagaraRendererProperties : public UNiagaraMergeable
```

**纯虚方法**(子类必须实现):

```cpp
virtual FNiagaraRenderer* CreateEmitterRenderer(ERHIFeatureLevel::Type, const FNiagaraEmitterInstance*, const UNiagaraComponent*) PURE_VIRTUAL(...);
virtual FNiagaraBoundsCalculator* CreateBoundsCalculator() PURE_VIRTUAL(...);
virtual void GetUsedMaterials(const FNiagaraEmitterInstance*, TArray<UMaterialInterface*>&) const PURE_VIRTUAL(...);
// Editor-only:
virtual void GetRendererWidgets(...) const PURE_VIRTUAL(...);
virtual void GetRendererTooltipWidgets(...) const PURE_VIRTUAL(...);
```

**默认虚方法**(子类可覆盖):
- `IsSimTargetSupported(ENiagaraSimTarget)` — Sprite/Mesh 都支持,Ribbon/Light 只支持 CPU
- `PopulateRequiredBindings(FNiagaraParameterStore&)` — 把必要的参数绑定注册到 store
- `CacheFromCompiledData(const FNiagaraDataSetCompiledData*)` — 编译数据到 runtime offset 缓存
- `NeedsMIDsForMaterials()` — 需要 `UMaterialInstanceDynamic` 吗
- `GetNumIndicesPerInstance()` — GPU DrawIndirect 用(Phase 8)
- `GetRendererFeedback(...)` — editor 错误警告收集

**核心字段**:

```cpp
UPROPERTY(EditAnywhere, Category = "Scalability")
FNiagaraPlatformSet Platforms;                    // 哪些平台启用此 renderer(Phase 9)

UPROPERTY(EditAnywhere, Category = "Sort Order")
int32 SortOrderHint;                              // 同 Emitter 内多 renderer 的绘制顺序

UPROPERTY()
bool bIsEnabled;                                  // 非 UI 字段,运行时可切

UPROPERTY(EditAnywhere, Category = "Rendering")
bool bMotionBlurEnabled;                          // 材质也要开运动模糊

TArray<const FNiagaraVariableAttributeBinding*> AttributeBindings;
TArray<FNiagaraVariable> CurrentBoundAttributes;  // GetBoundAttributes() 缓存
```

### `FNiagaraRendererVariableInfo`(L100)

```cpp
struct FNiagaraRendererVariableInfo
{
    int32 DatasetOffset = INDEX_NONE;   // 在 DataSet component 里的 offset
    int32 GPUBufferOffset = INDEX_NONE; // 在 VF 准备的 GPU buffer 里的 offset
    int32 NumComponents = 0;
    bool bUpload = false;               // 需要 CPU→GPU 上传?
    bool bHalfType = false;             // half float 存储?

    int32 GetGPUOffset() const          // 如启用 GbEnableMinimalGPUBuffers 用 GPU offset;
                                        // half 类型在最高位置 1 作标记
    {
        int32 Offset = GbEnableMinimalGPUBuffers ? GPUBufferOffset : DatasetOffset;
        if (bHalfType) Offset |= 1 << 31;
        return Offset;
    }
};
```

关键全局 CVar `GbEnableMinimalGPUBuffers` — 开启后只上传 VF 需要的变量(不上传整个 DataSet),省带宽。

### `FNiagaraRendererLayout`(L130)

```cpp
struct FNiagaraRendererLayout
{
    void Initialize(int32 NumVariables);
    bool SetVariable(const FNiagaraDataSetCompiledData*, const FNiagaraVariable&, int32 VFVarOffset);
    bool SetVariableFromBinding(const FNiagaraDataSetCompiledData*, const FNiagaraVariableAttributeBinding&, int32 VFVarOffset);
    void Finalize();

    // RT 访问(check IsInRenderingThread)
    TConstArrayView<FNiagaraRendererVariableInfo> GetVFVariables_RenderThread() const;
    int32 GetTotalFloatComponents_RenderThread() const;
    int32 GetTotalHalfComponents_RenderThread() const;

private:
    TArray<FNiagaraRendererVariableInfo> VFVariables_GT;  // GT 侧
    TArray<FNiagaraRendererVariableInfo> VFVariables_RT;  // RT 侧(Finalize 后拷贝)
    int32 TotalFloatComponents_GT / _RT;
    int32 TotalHalfComponents_GT / _RT;
};
```

**GT+RT 双 copy**机制——Finalize 时把 GT 侧计算结果拷贝到 RT 侧(RT 只读)。GT 修改不影响 RT 当前帧读。

### Editor Feedback(L29-L97)

```cpp
DECLARE_DELEGATE(FNiagaraRendererFeedbackFix);

class FNiagaraRendererFeedback
{
    FText DescriptionText;          // 完整描述
    FText SummaryText;              // 简短
    FText FixDescription;           // fix 操作描述
    FNiagaraRendererFeedbackFix Fix;  // 如果 fix 是可执行的 delegate
    bool Dismissable;

    bool IsFixable() const { return Fix.IsBound(); }
    void TryFix() const;
};
```

Editor 里面显示的"这个 renderer 有问题,点这里自动修复"对话框的背后数据。

## 继承体系

```
UNiagaraMergeable
    └── UNiagaraRendererProperties (本文件,ABSTRACT)
            ├── UNiagaraSpriteRendererProperties
            ├── UNiagaraRibbonRendererProperties
            ├── UNiagaraMeshRendererProperties
            ├── UNiagaraLightRendererProperties
            └── UNiagaraComponentRendererProperties(本学习路径未覆盖,用于 Component Renderer)
```

## 依赖与被依赖

**上游**:`NiagaraTypes.h` / `NiagaraCommon.h` / `NiagaraMergeable.h` / `NiagaraGPUSortInfo.h` / `NiagaraPlatformSet.h`
**下游**:所有具体 Renderer Properties 子类(Phase 6.3/5/7/9)+ `UNiagaraEmitter::RendererProperties` 数组(Phase 1)

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraRendererProperties]] — 本文件主类
- [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]] — 对偶运行时基类(Phase 6.2)
- [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — 持有 renderer properties 数组
