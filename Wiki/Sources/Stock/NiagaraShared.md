---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, shader, gpu, shader-map]
sources: 1
aliases: [NiagaraShared.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShared.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraShared.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShared.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 902 行(Phase 8 最大,扒前 400)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.1

## 职责

`NiagaraShader` 模块的**共享定义**。放 Shader Map、编译 Id、DI GPU 参数信息、Compile Event、`FNiagaraShaderScript`(GPU script 的 shader 侧代理)等。Phase 8 其他文件大多 include 本文件。

## 关键类型

### `FNiagaraShaderMapPointerTable`(L82)

```cpp
class FNiagaraShaderMapPointerTable : public FShaderMapPointerTable {
    TPtrTable<UNiagaraDataInterfaceBase> DITypes;   // DI 类型表,binary memory image 序列化用
};

using FNiagaraShaderRef = TShaderRefBase<FNiagaraShader, FNiagaraShaderMapPointerTable>;
```

### `FNiagaraCompileEvent`(L47)

USTRUCT。编辑器编译事件(Log/Warning/Error + NodeGuid + PinGuid + CallstackGuids),用于 UI 报错定位到节点。

### `FNiagaraDataInterfaceGeneratedFunction`(L104)

```cpp
USTRUCT()
struct FNiagaraDataInterfaceGeneratedFunction {
    FName DefinitionName;      // DI 定义的函数名
    FString InstanceName;      // 按 DI 实例 + specifier 唯一化
    TArray<TTuple<FName, FName>> Specifiers;   // 编译期 specifier(采样模式等)
};
```

编译器为每个 DI 函数调用生成一个 instance,支持同名但不同 specifier 的多次调用。

### `FNiagaraDataInterfaceGPUParamInfo`(L165)

```cpp
USTRUCT()
struct FNiagaraDataInterfaceGPUParamInfo {
    FString DataInterfaceHLSLSymbol;         // HLSL 符号(唯一后缀)
    FString DIClassName;                     // DI 类名
    TArray<FNiagaraDataInterfaceGeneratedFunction> GeneratedFunctions;
};
```

编译期产物,给 shader 侧绑定用。

### `FNiagaraDataInterfaceParamRef`(L195)

```cpp
struct FNiagaraDataInterfaceParamRef {
    DECLARE_TYPE_LAYOUT(FNiagaraDataInterfaceParamRef, NonVirtual);
    void Bind(const FNiagaraDataInterfaceGPUParamInfo&, const FShaderParameterMap&);

    LAYOUT_FIELD(TIndexedPtr<UNiagaraDataInterfaceBase>, DIType);
    LAYOUT_FIELD_WITH_WRITER(TMemoryImagePtr<FNiagaraDataInterfaceParametersCS>, Parameters, WriteFrozenParameters);
};
```

**FShader 侧**的 DI 参数绑定——`LAYOUT_FIELD` + `TMemoryImagePtr` 支持**binary memory image** 序列化(进 shader cache)。

### `FNiagaraShaderMapId`(L224)

编译 id,决定 shader 是否需要重编:

```cpp
class FNiagaraShaderMapId {
    LAYOUT_FIELD(FGuid, CompilerVersionID);
    LAYOUT_FIELD(ERHIFeatureLevel::Type, FeatureLevel);
    LAYOUT_FIELD(TMemoryImageArray<FMemoryImageString>, AdditionalDefines);
    LAYOUT_FIELD(FSHAHash, BaseCompileHash);
    LAYOUT_FIELD(TMemoryImageArray<FSHAHash>, ReferencedCompileHashes);
    LAYOUT_FIELD(FPlatformTypeLayoutParameters, LayoutParams);
    LAYOUT_FIELD_INITIALIZED(bool, bUsesRapidIterationParams, true);
};
```

DDC key 的一部分——脚本改了、编译器版本改了、平台变了,都让 Id 变,shader 重编。

### `FNiagaraCompilationQueue`(L312,editor-only)

单例编译队列。`FNiagaraShaderQueueTickable`(别处)按帧从队列取任务编译。

### `FNiagaraShaderMapContent`(L369)

```cpp
class FNiagaraShaderMapContent : public FShaderMapContent {
    LAYOUT_FIELD(FMemoryImageString, FriendlyName);
    LAYOUT_FIELD(FMemoryImageString, DebugDescription);
    LAYOUT_FIELD(FNiagaraShaderMapId, ShaderMapId);
    LAYOUT_FIELD(FNiagaraComputeShaderCompilationOutput, NiagaraCompilationOutput);
};
```

### `FNiagaraShaderMap`(L393)

```cpp
class FNiagaraShaderMap :
    public TShaderMap<FNiagaraShaderMapContent, FNiagaraShaderMapPointerTable>,
    public FDeferredCleanupInterface
```

**每个 GPU NiagaraScript 对应一份 ShaderMap**。缓存已编译 shader。类比 UE `FMaterialShaderMap`。

### 后 500 行未读内容

- `FNiagaraShaderScript`(Shader 侧的脚本代理 — Phase 3 / Phase 5 `GPUScript_RT` 的类型)
- 更多 editor 编译流程
- 序列化接口

## 涉及实体

- 合并到 [[Wiki/Entities/Stock/FNiagaraShader]](Phase 8.2)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceBase]]

## 开放问题

- 后 500 行的 `FNiagaraShaderScript` 具体字段
- DDC 流程接入点
