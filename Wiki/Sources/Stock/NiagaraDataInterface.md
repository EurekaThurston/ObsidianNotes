---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, di, core]
sources: 1
aliases: [NiagaraDataInterface.h, UNiagaraDataInterface 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterface.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterface.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterface.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 890 行(本次扒了前 400 行的核心部分)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.2(**主基类**,继承 Phase 5 的 `UNiagaraDataInterfaceBase`)

## 职责

定义 Niagara **DataInterface 生态的主基类** `UNiagaraDataInterface`(继承 Phase 5 的 `UNiagaraDataInterfaceBase`)+ RT 替身基类 `FNiagaraDataInterfaceProxy` + 一套 VM 函数绑定模板/宏,让 DI 能把"脚本里的函数调用"路由到 C++ 实现或 HLSL 生成。

DI 是 Niagara **最强扩展点**:想让脚本读外部数据(相机、骨骼网格、曲线、碰撞、RenderTarget),就写一个 DI 子类。

## 关键类型

### `FNiagaraDataInterfaceProxy`(L211)— RT 替身

```cpp
struct FNiagaraDataInterfaceProxy : TSharedFromThis<FNiagaraDataInterfaceProxy, ESPMode::ThreadSafe>
{
    virtual int32 PerInstanceDataPassedToRenderThreadSize() const = 0;
    virtual void ConsumePerInstanceDataFromGameThread(void* PerInstanceData, const FNiagaraSystemInstanceID&);

    FName SourceDIName;

    // Simulation Stage hooks(Phase 10)
    virtual void ResetData(FRHICommandList&, const FNiagaraDataInterfaceArgs&);
    virtual void PreStage(FRHICommandList&, const FNiagaraDataInterfaceStageArgs&);
    virtual void PostStage(FRHICommandList&, const FNiagaraDataInterfaceStageArgs&);
    virtual void PostSimulate(FRHICommandList&, const FNiagaraDataInterfaceArgs&);

    virtual FNiagaraDataInterfaceProxyRW* AsIterationProxy();  // RW 派生(Phase 10)
};
```

RT 替身——每个 DI 对应一个 Proxy 子类,GT 推数据给 RT,RT 在 compute shader 前后通过 `PreStage/PostStage/PostSimulate` 响应。

### `UNiagaraDataInterface`(L244)— 主基类

```cpp
UCLASS(abstract, EditInlineNew)
class NIAGARA_API UNiagaraDataInterface : public UNiagaraDataInterfaceBase
```

### Per-Instance Data 接口(L272)

```cpp
virtual bool InitPerInstanceData(void* PerInstanceData, FNiagaraSystemInstance*);
virtual void DestroyPerInstanceData(void* PerInstanceData, FNiagaraSystemInstance*);
virtual bool PerInstanceTick(void*, FNiagaraSystemInstance*, float DeltaSeconds);
virtual bool PerInstanceTickPostSimulate(void*, FNiagaraSystemInstance*, float);

virtual int32 PerInstanceDataSize() const { return 0; }         // CPU 侧 per-instance blob 大小
virtual int32 PerInstanceDataPassedToRenderThreadSize() const   // GT→RT 传递大小
    { return Proxy ? Proxy->PerInstanceDataPassedToRenderThreadSize() : 0; }

virtual void ProvidePerInstanceDataForRenderThread(void* DataForRenderThread, void* PerInstanceData, const FNiagaraSystemInstanceID&);
```

这回答了 Phase 3 `FNiagaraSystemInstance::DataInterfaceInstanceData` blob 的产生机制——每个 DI 声明 `PerInstanceDataSize()`,SystemInstance 按此分配对齐(16-byte)blob,InitPerInstanceData 写入初始数据。

### VM 函数注册(L321)

```cpp
virtual void GetFunctions(TArray<FNiagaraFunctionSignature>& OutFunctions);
virtual void GetVMExternalFunction(const FVMExternalFunctionBindingInfo& BindingInfo, void* InstanceData, FVMExternalFunction &OutFunc);
```

`GetFunctions` 在编译期被调——返回这个 DI 提供哪些函数(名字 + 参数签名)。编译器生成字节码时按签名匹配。
`GetVMExternalFunction` 运行时被调——按 `BindingInfo`(含函数名、参数位置信息)返回一个 `FVMExternalFunction`(lambda 封装),VM 遇到 DI 调用时跳过来。

### GPU HLSL 生成(L355)

```cpp
virtual void GetCommonHLSL(FString& OutHLSL);
virtual void GetParameterDefinitionHLSL(const FNiagaraDataInterfaceGPUParamInfo& ParamInfo, FString& OutHLSL);
virtual bool GetFunctionHLSL(const FNiagaraDataInterfaceGPUParamInfo&, const FNiagaraDataInterfaceGeneratedFunction&, int FunctionInstanceIndex, FString& OutHLSL);
```

GPU 路径靠字符串 HLSL 生成—— `GetParameterDefinitionHLSL` 产出 shader 参数声明(SamplerState, SRV 等),`GetFunctionHLSL` 按函数签名产出函数体。`UNiagaraScript` 编译时把这些拼接到生成的 compute shader 里。

### 渲染资源依赖

```cpp
virtual bool RequiresDistanceFieldData() const { return false; }
virtual bool RequiresDepthBuffer() const { return false; }
virtual bool RequiresEarlyViewData() const { return false; }
```

被 `NiagaraEmitterInstanceBatcher` 查询,决定 tick 放在哪个 `ETickStage`。

### Tick Group 约束

```cpp
virtual bool HasTickGroupPrereqs() const { return false; }
virtual ETickingGroup CalculateTickGroup(const void* PerInstanceData) const;
```

DI 可以强制 System Instance 的 TickGroup(比如 Camera DI 要求在 view 数据就绪后 tick)。

### VM 函数绑定模板族(L38-L110)

```cpp
struct TNDINoopBinder;
template<typename DirectType, typename NextBinder> struct TNDIExplicitBinder;
template<int32 ParamIdx, typename DataType, typename NextBinder> struct TNDIParamBinder;
```

**目的**:在运行时根据 `FVMExternalFunctionBindingInfo` 的 `InputParamLocations`(标记每个参数是常量还是寄存器)选择正确的 VM handler(`FExternalFuncConstHandler<T>` vs `FExternalFuncRegisterHandler<T>`),**编译期**拼出模板参数列表。

### 绑定宏

```cpp
#define NDI_FUNC_BINDER(ClassName, FuncName) T##ClassName##_##FuncName##Binder

#define DEFINE_NDI_FUNC_BINDER(ClassName, FuncName)\
struct NDI_FUNC_BINDER(ClassName, FuncName) {\
    template<typename... ParamTypes>\
    static void Bind(UNiagaraDataInterface* Interface, const FVMExternalFunctionBindingInfo&, void* InstanceData, FVMExternalFunction &OutFunc) {\
        auto Lambda = [Interface](FVectorVMContext& Context) {\
            static_cast<ClassName*>(Interface)->FuncName<ParamTypes...>(Context);\
        };\
        OutFunc = FVMExternalFunction::CreateLambda(Lambda);\
    }\
};

#define DEFINE_NDI_DIRECT_FUNC_BINDER(ClassName, FuncName)           // 无模板参数
#define DEFINE_NDI_DIRECT_FUNC_BINDER_WITH_PAYLOAD(ClassName, FuncName) // 带 payload
```

子类在 .cpp 用宏生成 binder 样板代码,然后在 `GetVMExternalFunction` 里按函数名 dispatch。

### Transform Handler(L21-L33)

```cpp
struct FNDITransformHandlerNoop {
    void TransformPosition(FVector& V, const FMatrix&) {}       // 不变换(world space 直接用)
    void TransformVector(FVector& V, const FMatrix&) {}
    void TransformRotation(FQuat& Q1, const FQuat&) {}
};

struct FNDITransformHandler {
    void TransformPosition(FVector& P, const FMatrix& M) { P = M.TransformPosition(P); }
    void TransformVector(FVector& V, const FMatrix& M);
    void TransformRotation(FQuat& Q1, const FQuat& Q2) { Q1 = Q2 * Q1; }
};
```

DI 采样方法模板化,根据 Emitter 是否 local-space 选择 Noop 或真变换——编译期多态。

### Editor 错误系统(L115-L206)

```cpp
DECLARE_DELEGATE_RetVal(bool, FNiagaraDataInterfaceFix);
class FNiagaraDataInterfaceError / FNiagaraDataInterfaceFeedback;  // UI 错误/警告 + 自动 fix delegate
```

### 其他虚方法(L320+)

- `CanExecuteOnTarget(ENiagaraSimTarget)` — CPU/GPU 支持
- `HasPreSimulateTick / HasPostSimulateTick`
- `IsUsedWithGPUEmitter(FNiagaraSystemInstance*)` — 决定是否创建 GPU 资源
- `IsDataInterfaceType(const FNiagaraTypeDefinition&)` — 静态判别
- `Equals / CopyTo / CopyToInternal` — 复制/比较(用于 mergeable)
- `UpgradeFunctionCall(FNiagaraFunctionSignature&)` editor — API 变化时改 pin
- `CanExposeVariables / GetExposedVariables` — 参数暴露给材质/BP

### 后半未读内容(L400-L890)

按推断还有:
- 更多 editor-only helpers
- CompileHash 相关
- 与 Simulation Stages 的细节接口(Phase 10)
- 若干 protected 方法

本 Phase 读本不依赖后半详细细节,Phase 10 再按需补读。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraDataInterface]] — 本文件主类
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceBase]] — Phase 5 基类
- [[Wiki/Entities/Stock/FNiagaraSystemInstance]] — 持有 DI per-instance data blob

## 开放问题

- 后 490 行的内容(本次未扒)→ 按需 offset 读
- `FNiagaraDataInterfaceProxyRW` 是什么 → Phase 10
- `FNiagaraDataInterfaceGeneratedFunction` 具体结构 → 编译器内部,按需看
