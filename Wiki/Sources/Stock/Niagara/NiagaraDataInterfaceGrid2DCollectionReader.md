---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, grid2d-reader, rw-di]
sources: 1
aliases: [NiagaraDataInterfaceGrid2DCollectionReader.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceGrid2DCollectionReader.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceGrid2DCollectionReader.h

- **Repo**: stock · **规模**: 95 行 · **Phase**: 10.4

## 职责

**只读**访问另一个 emitter 的 `Grid2DCollection`。让多 emitter 协作(例如 emitter A 写热图,emitter B 读热图来 spawn)。不存数据,只是"别人 grid 的代理"。

## 主类(L43)

```cpp
UCLASS(EditInlineNew, Category = "Grid", meta = (DisplayName = "Grid2D Collection Reader", Experimental),
       Blueprintable, BlueprintType, hidecategories = (Grid,RW))
class UNiagaraDataInterfaceGrid2DCollectionReader : public UNiagaraDataInterfaceGrid2D
{
    UPROPERTY(EditAnywhere, Category = "Reader")
    FString EmitterName;      // 目标 emitter

    UPROPERTY(EditAnywhere, Category = "Reader")
    FString DIName;           // 目标 emitter 上的 Grid2DCollection DI 名

    virtual void GetEmitterDependencies(UNiagaraSystem*, TArray<UNiagaraEmitter*>& Deps) const override;
    // 仅 GetValue / SampleGrid VM 函数,无 Set
};
```

**`hidecategories = (Grid, RW)`**:Reader 继承 Grid2D 但 Grid 参数无意义(尺寸继承自被读的 grid),editor 隐藏。

## `FGrid2DCollectionReaderInstanceData_GameThread`

```cpp
struct FGrid2DCollectionReaderInstanceData_GameThread {
    FNiagaraSystemInstance* SystemInstance;
    FNiagaraEmitterInstance* EmitterInstance;   // 目标 emitter 的 instance 引用
    FString EmitterName, DIName;
};
```

## Proxy

```cpp
struct FNiagaraDataInterfaceProxyGrid2DCollectionReaderProxy : public FNiagaraDataInterfaceProxyRW
{
    FIntVector GetElementCount(FNiagaraSystemInstanceID) const;

    // RT 侧保存对目标 proxy 的指针
    struct FGrid2DCollectionReaderInstanceData_RenderThread {
        FNiagaraDataInterfaceProxyGrid2DCollectionProxy* ProxyToUse;
    };
    TMap<FNiagaraSystemInstanceID, FGrid2DCollectionReaderInstanceData_RenderThread> SystemInstancesToProxyData_RT;
};
```

## VM 函数

```cpp
static const FName GetValueFunctionName;
static const FName SampleGridFunctionName;
```

只两个——只读。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceGrid2DCollection]](被读的源)
- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceRWBase]]
