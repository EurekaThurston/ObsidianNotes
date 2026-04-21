---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, di-base, uclass]
sources: 1
aliases: [UNiagaraDataInterfaceBase, DI 抽象基类]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraCore/Public/NiagaraDataInterfaceBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceBase

> Niagara **DataInterface(DI)**的 UObject 抽象基类。位于 `NiagaraCore` 模块(方便 Shader 模块依赖)。所有 DI(Phase 7 详)都继承本类。

## 一句话角色

`UCLASS(abstract, EditInlineNew) UNiagaraDataInterfaceBase : public UNiagaraMergeable`。提供 DI 的 GPU compute shader 参数绑定虚接口,搭配 non-virtual 的 `FNiagaraDataInterfaceParametersCS` 实现"可序列化的 shader 参数布局"。

## 为什么 DI 基类要在 NiagaraCore

模块分层:`NiagaraShader`(compute shader 编译)和 `NiagaraVertexFactories` 需要引用 DI 类型,但不想拉入 `Niagara` 主模块(242 文件)。把 DI 抽象基类单独塞到小型的 NiagaraCore 模块,按依赖最小化原则。

## 虚接口

```cpp
virtual FNiagaraDataInterfaceParametersCS* CreateComputeParameters() const { return nullptr; }
virtual const FTypeLayoutDesc* GetComputeParametersTypeDesc() const { return nullptr; }

virtual void BindParameters(FNiagaraDataInterfaceParametersCS* Base, ...);
virtual void SetParameters(const FNiagaraDataInterfaceParametersCS* Base, ...) const;
virtual void UnsetParameters(const FNiagaraDataInterfaceParametersCS* Base, ...) const;

virtual bool HasInternalAttributeReads(const UNiagaraEmitter* OwnerEmitter, const UNiagaraEmitter* Provider) const { return false; }
```

## 配套结构

- `FNiagaraDataInterfaceParametersCS` — non-virtual shader 参数绑定(`LAYOUT_FIELD` 支持 binary memory image)
- `FNiagaraDataInterfaceArgs / StageArgs / SetArgs` — shader 调用上下文
- `DECLARE_NIAGARA_DI_PARAMETER()` / `IMPLEMENT_NIAGARA_DI_PARAMETER(T, ParameterType)` — 子类的参数类型注册宏

## 相关

- `UNiagaraDataInterface`(Niagara 主模块)— 继承本类,Phase 7 主角
- 所有具体 DI(Curve / Camera / StaticMesh / SkeletalMesh 等)— Phase 7 详
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — DI 被 Emitter Asset 引用

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraDataInterfaceBase]]
- 主题读本:[[Readers/Niagara/Phase 5 - Niagara 脚本如何跑起来]] § 1(Phase 7 完整展开)

## 开放问题(Phase 7)

- 全部 → Phase 7
