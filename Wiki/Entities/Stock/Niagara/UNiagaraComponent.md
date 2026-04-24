---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, component, uclass]
sources: 1
aliases: [UNiagaraComponent, NiagaraComponent, Niagara Particle System Component]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraComponent.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraComponent

> 把 Niagara 特效资产挂进游戏世界的**桥**。继承 `UFXSystemComponent`,Actor/BP/C++ 三种生成路径最终都指向一个此类实例。

## 一句话角色

`UNiagaraComponent : public UFXSystemComponent`。Niagara 的"承载层"——持有一个 `UNiagaraSystem* Asset`,独占一个 `TUniquePtr<FNiagaraSystemInstance> SystemInstance`,在两者之间叠加一层参数覆盖,对接 UE 的 Primitive/Attachment/Tick/Scalability/Pool 全部基建。Phase 2 的主角。

## 五大职责

1. **Asset 持有者**:`UNiagaraSystem* Asset`(引用磁盘资产)
2. **Instance 管理者**:`TUniquePtr<FNiagaraSystemInstance> SystemInstance`(独占,Component 死则 Instance 死)
3. **参数覆盖层**:`FNiagaraUserRedirectionParameterStore OverrideParameters` + 18 个 `SetVariable*` BP 函数
4. **场景集成点**:UPrimitiveComponent 链(Bounds / SceneProxy / Auto-Attachment)
5. **生命周期调度**:`Activate/Deactivate` + `TickComponent` + `OnRegister/OnUnregister/BeginDestroy` + Pool + Scalability 接线

## 关键字段速查

| 字段 | 类型 | 作用 |
|---|---|---|
| `Asset` | `UNiagaraSystem*` | 被引用的 Niagara System 资产 |
| `SystemInstance` | `TUniquePtr<FNiagaraSystemInstance>` | 独占的运行时实例 |
| `OverrideParameters` | `FNiagaraUserRedirectionParameterStore` | 本实例的 User.* 参数覆盖 |
| `bForceSolo` | `uint32 : 1` | ⚠️ 绕开批量 Tick,性能陷阱 |
| `PoolingMethod` | `ENCPoolMethod` | 池化策略(Phase 9) |
| `bAutoDestroy` | `uint32 : 1` | 播完自销毁 |
| `ScalabilityManagerHandle` | `int32` | Scalability Manager 索引,`INDEX_NONE`=未注册 |
| `bAutoManageAttachment` + `AutoAttach*` | 一组 | `SpawnSystemAttached` 的自动挂载子系统 |
| `AgeUpdateMode` + `DesiredAge` 等 | 一组 | Sequencer 时间轴回放支持 |
| `OnSystemFinished` | `FOnNiagaraSystemFinished` multicast delegate | BP 可绑的"播完"事件 |

常用 BP 接口:`SetAsset/GetAsset`、`ResetSystem/ReinitializeSystem`、`SetPaused`、`SetVariable*(18 个)`、`SetAutoDestroy`、`SetForceSolo`、`AdvanceSimulation`。

## 陷阱

- ⚠️ **`bForceSolo=true`**:绕开 `FNiagaraSystemSimulation` 的批量 Tick,对同 Asset 多实例场景性能退化显著。仅调试用
- ⚠️ **`PoolingMethod` 默认 None**:频繁生成的小特效不显式启用池化会有严重 GC 压力
- ⚠️ **`SystemInstance` 是 `TUniquePtr`**:不可共享,Component 销毁即 Instance 销毁

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraSystem]] — 引用的 Asset 类型
- [[Wiki/Entities/Stock/Niagara/ANiagaraActor]] — 把本类封装成 Actor 的 wrapper
- [[Wiki/Entities/Stock/Niagara/UNiagaraFunctionLibrary]] — 返回本类的 BP 入口
- [[Wiki/Concepts/UE/UE4-资产与实例]] — 本类是 Asset→Instance 的**承载者**
- [[Wiki/Concepts/UE/Niagara/Niagara-vs-cascade]] — `UFXSystemComponent` 基类与 Cascade 的 `UParticleSystemComponent` 同源

## 深入阅读

- 全字段清单 + 代码片段:[[Wiki/Sources/Stock/Niagara/NiagaraComponent]]
- 主题读本(推荐初读):[[Readers/UE/Niagara/Phase 2 - Component 层的五职责]] § 3

## 开放问题

- `FNiagaraSceneProxy` 的渲染流程?→ Phase 6
- `FNiagaraScalabilityManager` friend 访问设计细节?→ Phase 9
- `ENCPoolMethod` 五种取值决策表?→ Phase 9
- `TemplateParameterOverrides` / `InstanceParameterOverrides` 两层覆盖在 BP 继承里怎么协作?→ 开放
