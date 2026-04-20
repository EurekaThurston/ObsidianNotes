---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, di-base]
sources: 1
aliases: [NiagaraDataInterfaceBase.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraCore/Public/NiagaraDataInterfaceBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceBase.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraCore/Public/NiagaraDataInterfaceBase.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 136 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 5 — CPU 脚本执行 5.2(**Phase 7 详展**)

## 职责

定义 Niagara **DataInterface(DI)机制**的最底层抽象:
- `UNiagaraDataInterfaceBase` — DI 基类 UCLASS(在 NiagaraCore 模块,Phase 7 的 `UNiagaraDataInterface` 继承它)
- `FNiagaraDataInterfaceParametersCS` — GPU compute shader 参数绑定基类(non-virtual,layout-aware)
- 3 个辅助 args struct:`FNiagaraDataInterfaceArgs / StageArgs / SetArgs`(渲染线程调用上下文)
- 宏 `DECLARE_NIAGARA_DI_PARAMETER()` / `IMPLEMENT_NIAGARA_DI_PARAMETER(T, ParameterType)`

本 Phase 只做**登记存在**——Phase 7 系统讲 DI 生态。

## 关键类型

### `FNiagaraDataInterfaceArgs` 家族(L28-L79)

GPU 场景下把 DI 调用的上下文打包成 struct 传给 shader:

```cpp
struct FNiagaraDataInterfaceArgs {
    FNiagaraDataInterfaceProxy* DataInterface;
    FNiagaraSystemInstanceID SystemInstanceID;
    const NiagaraEmitterInstanceBatcher* Batcher;
};

struct FNiagaraDataInterfaceStageArgs : FNiagaraDataInterfaceArgs {
    const FNiagaraComputeInstanceData* ComputeInstanceData;
    uint32 SimulationStageIndex;
    bool IsOutputStage, IsIterationStage;
};

struct FNiagaraDataInterfaceSetArgs : FNiagaraDataInterfaceArgs {
    FShaderReference Shader;
    ...
};
```

### `FNiagaraDataInterfaceParametersCS`(L86)

**non-virtual** 参数绑定基类,用 `LAYOUT_FIELD` 宏配合 `FTypeLayoutDesc` 支持**二进制布局序列化**(memory image)。

```cpp
struct FNiagaraDataInterfaceParametersCS {
    DECLARE_EXPORTED_TYPE_LAYOUT(FNiagaraDataInterfaceParametersCS, NIAGARACORE_API, NonVirtual);

    void Bind(const FNiagaraDataInterfaceGPUParamInfo& ParameterInfo, const FShaderParameterMap& ParameterMap) {}
    void Set(FRHICommandList& RHICmdList, const FNiagaraDataInterfaceSetArgs& Context) const {}
    void Unset(FRHICommandList& RHICmdList, const FNiagaraDataInterfaceSetArgs& Context) const {}

    LAYOUT_FIELD(TIndexedPtr<UNiagaraDataInterfaceBase>, DIType);
};
```

三个方法非虚,子类通过"继承 + 特化 `FTypeLayoutDesc`"方式替代虚表。这是为了**让 shader 参数绑定能进 shader cache(binary memory image)**——虚表指针无法序列化。

### `UNiagaraDataInterfaceBase`(L100)

```cpp
UCLASS(abstract, EditInlineNew)
class NIAGARACORE_API UNiagaraDataInterfaceBase : public UNiagaraMergeable
{
    GENERATED_UCLASS_BODY()

    // 虚方法,被子类重写连接到 non-virtual 的 ParametersCS 子类
    virtual FNiagaraDataInterfaceParametersCS* CreateComputeParameters() const { return nullptr; }
    virtual const FTypeLayoutDesc* GetComputeParametersTypeDesc() const { return nullptr; }

    virtual void BindParameters(FNiagaraDataInterfaceParametersCS* Base, ...) {}
    virtual void SetParameters(const FNiagaraDataInterfaceParametersCS* Base, ...) const {}
    virtual void UnsetParameters(const FNiagaraDataInterfaceParametersCS* Base, ...) const {}

    virtual bool HasInternalAttributeReads(const UNiagaraEmitter* OwnerEmitter, const UNiagaraEmitter* Provider) const { return false; }
};
```

### 宏 `DECLARE_NIAGARA_DI_PARAMETER()` / `IMPLEMENT_NIAGARA_DI_PARAMETER(T, ParameterType)`

DI 子类要自定义参数类型时用这组宏(L121-L135)。`IMPLEMENT` 宏把 `CreateComputeParameters` / `BindParameters` / `SetParameters` / `UnsetParameters` 四个虚方法实现连接到 non-virtual ParametersCS 子类的对应方法。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceBase]] — 本文件主类(Phase 7 真正展开)

## 开放问题 / 待 Phase 7

- `UNiagaraDataInterface`(在 Niagara 主模块,继承 Base)的完整接口 → Phase 7.1
- 具体 DI 实现(Curve / Mesh / RenderTarget 等)→ Phase 7
- `FNiagaraDataInterfaceProxy`(RT 替身)→ Phase 8
- per-instance data 的分配与查找 → Phase 7
