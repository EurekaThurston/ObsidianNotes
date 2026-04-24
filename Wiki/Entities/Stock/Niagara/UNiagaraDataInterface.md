---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, di]
sources: 1
aliases: [UNiagaraDataInterface, DI 主基类]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterface.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterface

> Niagara **DataInterface 主基类**(Niagara 主模块)。继承 Phase 5 的 `UNiagaraDataInterfaceBase`。所有具体 DI 子类(Curve/Camera/CollisionQuery/StaticMesh/SkeletalMesh/Texture/RenderTarget2D/Grid/...)都继承本类。

## 一句话角色

`UCLASS(abstract, EditInlineNew) UNiagaraDataInterface : UNiagaraDataInterfaceBase`。让脚本能"读/写外部数据"——相机、mesh、曲线、RenderTarget、碰撞查询。

## 核心虚方法家族

| 类别 | 方法 |
|---|---|
| Per-Instance | `InitPerInstanceData / DestroyPerInstanceData / PerInstanceTick / PerInstanceTickPostSimulate / PerInstanceDataSize` |
| VM 注册 | `GetFunctions / GetVMExternalFunction` |
| GPU HLSL | `GetCommonHLSL / GetParameterDefinitionHLSL / GetFunctionHLSL` |
| 资源需求 | `RequiresDistanceFieldData / RequiresDepthBuffer / RequiresEarlyViewData` |
| Tick 依赖 | `HasTickGroupPrereqs / CalculateTickGroup` |
| 执行目标 | `CanExecuteOnTarget(CPU/GPU)` |
| 渲染线程 | `ProvidePerInstanceDataForRenderThread / PerInstanceDataPassedToRenderThreadSize / PushToRenderThreadImpl` |
| 比较复制 | `Equals / CopyTo / CopyToInternal` |

## 配套结构

- **`FNiagaraDataInterfaceProxy`** — RT 替身基类(`TSharedFromThis ThreadSafe`)。带 `PreStage / PostStage / PostSimulate` 钩子(SimStage)
- **`FNDITransformHandler / FNDITransformHandlerNoop`** — local/world space 变换策略
- **`FNiagaraDataInterfaceError / FNiagaraDataInterfaceFeedback`**(editor)— UI 错误 + 自动 fix delegate
- **绑定模板**:`TNDINoopBinder / TNDIExplicitBinder / TNDIParamBinder` + 宏 `DEFINE_NDI_FUNC_BINDER(_DIRECT)(_WITH_PAYLOAD)` —— 把 DI 方法编译期绑到 `FVMExternalFunction`

## DI 编译期三路径

```
脚本编译:GetFunctions → 注册可用函数签名
     ↓ 编译器根据脚本里的调用生成字节码
运行时 CPU:GetVMExternalFunction(BindingInfo) → 返回 lambda → VM 遇 DI 调用执行
运行时 GPU:GetFunctionHLSL / GetParameterDefinitionHLSL → 拼接进 compute shader
```

## 具体子类(Phase 7 本批)

- Curve 家族(Float/Vector/Color — 本学习路径只覆盖 Float)
- Camera
- CollisionQuery
- StaticMesh
- SkeletalMesh
- Texture(GPU only)
- RenderTarget2D(RW,继承 UNiagaraDataInterfaceRWBase — 非本基类直系)
- Grid 家族(Phase 10)

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceBase]] — 父类(Phase 5)
- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemInstance]] — `DataInterfaceInstanceData` blob
- [[Wiki/Entities/Stock/Niagara/FNiagaraScriptExecutionContext]] — VM 调用处(Phase 5)

## 深入阅读

- 源:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterface]](890 行,本次扒前 400)
- 读本:[[Readers/UE/Niagara/Phase 7 - 最强扩展点 Data Interface]]
