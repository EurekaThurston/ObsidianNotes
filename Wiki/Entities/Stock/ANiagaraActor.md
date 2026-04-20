---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, actor, uclass]
sources: 1
aliases: [ANiagaraActor, NiagaraActor]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraActor.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# ANiagaraActor

> 一个**纯粹的 ComponentWrapperClass**——把 `UNiagaraComponent` 包装成可独立放置在关卡里的 Actor。整个类不承担任何特效逻辑。

## 一句话角色

`ANiagaraActor : public AActor`,`UCLASS(MinimalAPI, ComponentWrapperClass, hideCategories=(Activation, Input, Collision, "Game|Damage"))`。存在的唯一意义是:让关卡设计师能把 `UNiagaraSystem` 拖进视口得到一个 Outliner 条目,并通过 `OnSystemFinished` delegate 实现"播完自销毁"。

所有特效逻辑都在 [[Wiki/Entities/Stock/UNiagaraComponent]] 和 `FNiagaraSystemInstance` 里。是 Cascade 时代 `AEmitter` 的对位物。

## 核心字段速查

| 字段 | 类型 | 作用 |
|---|---|---|
| `NiagaraComponent` | `UNiagaraComponent*` | **唯一的逻辑字段**;私有但 `AllowPrivateAccess=true` |
| `bDestroyOnSystemFinish` | `uint32 : 1` | 特效播完是否 `Destroy()` 自己 |
| `SpriteComponent` | `UBillboardComponent*` | 编辑器视口图标(仅 editor) |
| `ArrowComponent` | `UArrowComponent*` | 编辑器方向箭头(仅 editor) |

核心方法:
- `PostRegisterAllComponents()` — 绑定 `NiagaraComponent->OnSystemFinished` 到下面的回调
- `OnNiagaraSystemFinished(UNiagaraComponent*)` — 收到事件后 `if (bDestroyOnSystemFinish) Destroy();`
- `SetDestroyOnSystemFinish(bool)` — BP 可调
- `GetNiagaraComponent()` — 对外访问 Component

## 相关

- [[Wiki/Entities/Stock/UNiagaraComponent]] — 唯一的逻辑字段类型
- [[Wiki/Entities/Stock/UNiagaraSystem]] — Component 引用的 Asset
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — 对应 Cascade 的 `AEmitter`

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraActor]]
- 主题读本:[[Readers/Niagara/Phase2-component-layer-读本]] § 6

## 开放问题

- Pool 的载体是 Component 还是 Actor?→ Phase 9
- `ResetInLevel()` 实现细节?→ 非关键,查 .cpp 可
