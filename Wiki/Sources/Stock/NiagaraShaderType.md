---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, shader, shader-type, compile]
sources: 1
aliases: [NiagaraShaderType.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShaderType.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraShaderType.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShaderType.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 160 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.3

## 职责

`FNiagaraShaderType : FShaderType` —— Niagara compute shader 的 **ShaderType meta 描述符**。UE shader 系统通过 ShaderType 知道一种 shader 要怎么编译、怎么绑定。

## 关键类型

### `FNiagaraShaderPermutationParameters`(L40)

```cpp
struct FNiagaraShaderPermutationParameters : public FShaderPermutationParameters {
    const FNiagaraShaderScript* Script;
};
```

编译 permutation 时,带上 Script 指针——`FNiagaraShader::ShouldCompilePermutation` 能基于 script 内容决定编不编。

### `FNiagaraShaderType`(L53)

```cpp
class FNiagaraShaderType : public FShaderType
{
    struct CompiledShaderInitializerType : FGlobalShaderType::CompiledShaderInitializerType {
        const FString DebugDescription;
        TArray<FNiagaraDataInterfaceGPUParamInfo> DIParamInfo;
    };

    // 编译入口
    TSharedRef<FShaderCommonCompileJob, ESPMode::ThreadSafe> BeginCompileShader(
        uint32 ShaderMapId, int32 PermutationId,
        const FNiagaraShaderScript* Script,
        FShaderCompilerEnvironment*, EShaderPlatform,
        TArray<TSharedRef<FShaderCommonCompileJob, ESPMode::ThreadSafe>>& NewJobs,
        FShaderTarget, TArray<FNiagaraDataInterfaceGPUParamInfo>& InDIParamInfo);

    FShader* FinishCompileShader(const FSHAHash&, const FShaderCompileJob&, const FString& InDebugDescription);

    bool ShouldCache(EShaderPlatform, const FNiagaraShaderScript*) const;
};
```

### `IMPLEMENT_NIAGARA_SHADER_TYPE` 宏(L15)

对 UE 通用 `IMPLEMENT_SHADER_TYPE` 的薄包装。

### Global

```cpp
static TMap<const FShaderCompileJob*, TArray<FNiagaraDataInterfaceGPUParamInfo>> ExtraParamInfo;
```

编译期缓存——DI 参数信息随 compile job 存,编译完拿回。

## 编译流程粗线(推断)

```
Script 变化 →
FNiagaraShaderMapId 变化 →
FNiagaraShaderMap 发现没缓存 →
FNiagaraShaderType::BeginCompileShader(Script) →
加入编译队列(FNiagaraCompilationQueue)
    ↓
worker thread 编译 →
FinishCompileShader 创建 FShader ←
→ 加入 ShaderMap
```

## 涉及实体

- 合并到 [[Wiki/Entities/Stock/FNiagaraShader]]
