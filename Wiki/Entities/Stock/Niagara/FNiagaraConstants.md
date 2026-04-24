---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, constants, namespaces]
sources: 1
aliases: [FNiagaraConstants]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraConstants.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraConstants

> Niagara 脚本能开箱即用的所有 Engine/System/Emitter/Particle 级变量的**注册中心**。同时定义参数命名空间的 FName 常量。

## 一句话角色

`struct NIAGARA_API FNiagaraConstants`(静态类,无实例)。管理约 70+ 个预定义 `FNiagaraVariable`(如 `Engine.DeltaTime` / `Particles.Position`)+ 一组命名空间 FName(`UserNamespace` / `EngineNamespace` 等)。

Asset/编译器/脚本面板查"哪些变量已经存在"都通过本类的 getter。

## 三大入口

```cpp
static const TArray<FNiagaraVariable>& GetEngineConstants();        // Engine.*
static const TArray<FNiagaraVariable>& GetCommonParticleAttributes(); // Particles.*
static const TArray<FNiagaraVariable>& GetTranslatorConstants();    // 编译器 translator 内部
static const TArray<FNiagaraVariable>& GetStaticSwitchConstants();  // Static switch
```

## 命名空间(FName 常量)

`User / Engine / System / Emitter / ParticleAttribute / Module / Output / Transient / StackContext / DataInstance / StaticSwitch / Array / ParameterCollection / Local / Initial / Previous / Owner`

作用域名:`EngineOwnerScope / EngineSystemScope / EngineEmitterScope / ScriptTransientScope / ScriptPersistentScope / InputScope / OutputScope / UniqueOutputScope / CustomScope`

## SYS_PARAM 宏族

约 70+ 个 `SYS_PARAM_*` 宏提供便捷访问,例如 `SYS_PARAM_ENGINE_DELTA_TIME` 展开为 `INiagaraModule::GetVar_Engine_DeltaTime()`。

## 相关

- [[Wiki/Entities/Stock/Niagara/FNiagaraVariable]] — 每个常量都是它
- [[Wiki/Entities/Stock/Niagara/UNiagaraSystem]] / [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — 常量命名空间在它们身上体现为 `User.` `System.` `Emitter.` 等分组

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/Niagara/NiagaraConstants]]
- 主题读本:[[Readers/UE/Niagara/Phase 4 - Niagara 的数据语言]] § 5
