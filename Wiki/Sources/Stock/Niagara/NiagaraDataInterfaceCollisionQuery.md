---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, collision]
sources: 1
aliases: [NiagaraDataInterfaceCollisionQuery.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCollisionQuery.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceCollisionQuery.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCollisionQuery.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 93 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.6

## 职责

让脚本做**物理碰撞查询**。CPU 侧:同步/异步 physics trace。GPU 侧:直接读场景深度 buffer + mesh distance field。

## `CQDIPerInstanceData`(L16)

```cpp
struct CQDIPerInstanceData {
    FNiagaraSystemInstance* SystemInstance;
    FNiagaraDICollisionQueryBatch CollisionBatch;   // 异步查询结果队列
};
```

## 主类

```cpp
UCLASS(EditInlineNew, Category = "Collision", meta = (DisplayName = "Collision Query"))
class NIAGARA_API UNiagaraDataInterfaceCollisionQuery : public UNiagaraDataInterface
{
    FNiagaraSystemInstance* SystemInstance;

    virtual bool InitPerInstanceData(void*, FNiagaraSystemInstance*) override;
    virtual void DestroyPerInstanceData(void*, FNiagaraSystemInstance*) override;
    virtual bool PerInstanceTick(void*, FNiagaraSystemInstance*, float) override;
    virtual bool PerInstanceTickPostSimulate(void*, FNiagaraSystemInstance*, float) override;  // 异步查询回收
    virtual int32 PerInstanceDataSize() const override { return sizeof(CQDIPerInstanceData); }

    virtual bool CanExecuteOnTarget(ENiagaraSimTarget Target) const override { return true; }
    virtual bool RequiresDistanceFieldData() const override { return true; }    // GPU DF
    virtual bool RequiresDepthBuffer() const override { return true; }          // GPU depth

    virtual bool HasPreSimulateTick() const override { return true; }
    virtual bool HasPostSimulateTick() const override { return true; }
```

### VM 函数

```cpp
void PerformQuerySyncCPU(FVectorVMContext&);    // 同步 line trace(阻塞当前 VM)
void PerformQueryAsyncCPU(FVectorVMContext&);   // 异步(本帧发起,下帧读结果)
void QuerySceneDepth(FVectorVMContext&);        // GPU:深度缓冲读
void QueryMeshDistanceField(FVectorVMContext&); // GPU:distance field 读
```

### 函数 FName

```cpp
const static FName SceneDepthName;
const static FName DistanceFieldName;
const static FName SyncTraceName;
const static FName AsyncTraceName;
```

### Editor

```cpp
#if WITH_EDITOR
virtual void ValidateFunction(const FNiagaraFunctionSignature&, TArray<FText>&) override;
virtual bool UpgradeFunctionCall(FNiagaraFunctionSignature&) override;
#endif
```

### 全局资源

```cpp
static FCriticalSection CriticalSection;  // 异步查询保护
UEnum* TraceChannelEnum;                   // BP 侧 trace channel 选择
```

## GPU 侧

```cpp
struct FNiagaraDataIntefaceProxyCollisionQuery : public FNiagaraDataInterfaceProxy
{
    // 空 — 直接读场景深度/DF,无 per-instance 数据
};
```

## 性能权衡

- **Sync CPU**:最精确,但 VM 阻塞。少量粒子可接受
- **Async CPU**:本帧发起,下帧读结果。高吞吐但结果延迟 1 帧
- **GPU depth**:只能查**屏幕内**的深度(depth buffer 限制),屏幕外或遮挡粒子查不到
- **GPU DF**:全场景的 mesh distance field,更可靠但需要 distance field 开启(全局 DF 资源)

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceCollisionQuery]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterface]]
