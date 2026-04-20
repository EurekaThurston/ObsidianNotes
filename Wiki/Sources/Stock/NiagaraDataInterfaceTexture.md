---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, texture, gpu-only]
sources: 1
aliases: [NiagaraDataInterfaceTexture.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceTexture.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceTexture.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceTexture.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 63 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.9

## 职责

让 GPU 脚本**采样 2D 纹理**。**GPU only**——CPU VM 不支持 texture 采样。

## 类

```cpp
UCLASS(EditInlineNew, Category = "Texture", meta = (DisplayName = "Texture Sample"))
class NIAGARA_API UNiagaraDataInterfaceTexture : public UNiagaraDataInterface
{
    UPROPERTY(EditAnywhere)
    UTexture* Texture;                        // 可以是 UTexture2D / UTextureRenderTarget2D / UTexture2DArray...

    virtual bool CanExecuteOnTarget(ENiagaraSimTarget Target) const override
    { return Target == GPUComputeSim; }      // ← GPU only

    // VM stub(CPU 不可达,但保留签名)
    void SampleTexture(FVectorVMContext&);
    void GetTextureDimensions(FVectorVMContext&);
    void SamplePseudoVolumeTexture(FVectorVMContext&);

    // GPU HLSL 生成
    virtual void GetParameterDefinitionHLSL(...) override;
    virtual bool GetFunctionHLSL(...) override;

    void SetTexture(UTexture* InTexture);     // Phase 2 UNiagaraFunctionLibrary::SetTextureObject 用
    virtual void PushToRenderThreadImpl() override;
};
```

## GPU shader 生成

```cpp
static const FString TextureName;
static const FString SamplerName;
static const FString DimensionsBaseName;
```

`GetParameterDefinitionHLSL` 生成:

```hlsl
Texture2D {TextureName}_{DIName};
SamplerState {SamplerName}_{DIName};
uint2 {DimensionsBaseName}_{DIName};
```

`GetFunctionHLSL` 生成函数体,比如:

```hlsl
void {FunctionName}_{DIName}(float2 UV, out float4 OutColor) {
    OutColor = {TextureName}_{DIName}.SampleLevel({SamplerName}_{DIName}, UV, 0);
}
```

## VM 函数

- `SampleTexture2D(UV) → Color`
- `GetTextureDimensions() → Size`
- `SamplePseudoVolumeTexture(UVW) → Color` — 把 2D texture 当 volume 采样(通过 atlas)

## 类似 DI

Niagara 家族还有:
- `NiagaraDataInterfaceVolumeTexture`(2D volume)
- `NiagaraDataInterfaceCubeTexture`
- `NiagaraDataInterfaceRenderTarget2D`(7.10,可写!)

本 DI 是**只读**texture 采样,Phase 2 `UNiagaraFunctionLibrary::SetTextureObject` 可替换引用的 `Texture`。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceTexture]]
- [[Wiki/Entities/Stock/UNiagaraDataInterface]]
