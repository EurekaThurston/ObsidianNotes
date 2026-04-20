---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, grid2d, rw-di]
sources: 1
aliases: [UNiagaraDataInterfaceGrid2DCollection, UNiagaraDataInterfaceGrid2DCollectionReader, FGrid2DBuffer]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceGrid2DCollection.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceGrid2DCollection(+ Reader)

> **2D 网格 + N 属性 + 双缓冲**。典型用途:2D 烟雾、热图、2D 流体。合并 Collection 主类 + Reader 只读版。

## 一句话角色

- **Grid2DCollection**:主类,存数据用 `Texture2DArray`(每 slice 一个 attribute)
- **Grid2DCollectionReader**:只读代理,访问另一 emitter 的 Grid2DCollection(emitter 间数据交互)

## `FGrid2DBuffer`

```cpp
class FGrid2DBuffer {
    FTexture2DArrayRHIRef GridTexture;       // ← Texture2DArray,slice = attribute
    FShaderResourceViewRHIRef GridSRV;
    FUnorderedAccessViewRHIRef GridUAV;
};
```

## 双缓冲

```cpp
TArray<TUniquePtr<FGrid2DBuffer>> Buffers;
FGrid2DBuffer* CurrentData;
FGrid2DBuffer* DestinationData;
void BeginSimulate() / EndSimulate();
```

同 Phase 4 DataSet 双缓冲思路。

## 主字段

```cpp
FNiagaraUserParameterBinding RenderTargetUserParameter;
ENiagaraGpuBufferFormat OverrideBufferFormat;
uint8 bOverrideFormat : 1;

#if WITH_EDITORONLY_DATA
uint8 bPreviewGrid : 1;
FName PreviewAttribute;
#endif
```

## Proxy SimStage Hooks

```cpp
virtual void PreStage(...) / PostStage(...) / PostSimulate(...) / ResetData(...);
virtual FIntVector GetElementCount(...) const override;
```

`PreStage` 切 current/destination;`PostStage` 收尾;`PostSimulate` 复制到 user RT。

## VM 函数家族(按属性类型重载)

```
ClearCell / CopyPreviousToCurrentForCell
Set/Get/SampleGrid × {Float, Vector2, Vector3, Vector4}
SetNumCells / GetNumCells / GetCellSize / GetWorldBBoxSize
GetXxxAttributeIndex
```

## Editor HLSL 自动生成

```cpp
virtual bool SupportsSetupAndTeardownHLSL() const override { return true; }
virtual bool SupportsIterationSourceNamespaceAttributesHLSL() const override { return true; }

GenerateSetupHLSL / GenerateTeardownHLSL
GenerateIterationSourceNamespaceRead/WriteAttributesHLSL
```

让 Grid 能作为 SimStage iteration source —— Phase 7 DI 基类的 `SupportsIterationSourceNamespaceAttributesHLSL` 接口在此兑现。

## Reader 不同点

```cpp
UCLASS(hidecategories = (Grid, RW))
class UNiagaraDataInterfaceGrid2DCollectionReader : public UNiagaraDataInterfaceGrid2D
{
    FString EmitterName;     // 目标 emitter
    FString DIName;          // 目标 DI 名
    virtual void GetEmitterDependencies(UNiagaraSystem*, TArray<UNiagaraEmitter*>&) const override;
    // 只有 GetValue / SampleGrid,无 Set
};
```

Reader 的 proxy 存目标 `FNiagaraDataInterfaceProxyGrid2DCollectionProxy*` 指针,读时 follow。

## 陷阱

- ⚠️ Experimental 标记 —— API 可能变
- ⚠️ 多 attribute 打成 Texture2DArray,不是多个 Texture2D —— shader 采样用 `slice = AttributeIndex`
- ⚠️ Reader 需要 `GetEmitterDependencies` 声明依赖,否则 tick 顺序错

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceRWBase]]
- [[Wiki/Entities/Stock/UNiagaraSimulationStage]]

## 深入阅读

- 源 × 2:[[Wiki/Sources/Stock/NiagaraDataInterfaceGrid2DCollection]] / [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid2DCollectionReader]]
- 读本:[[Readers/Niagara/Phase10-advanced-features-读本]]
