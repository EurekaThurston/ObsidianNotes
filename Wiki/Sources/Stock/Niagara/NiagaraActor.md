---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, actor, runtime]
sources: 1
aliases: [NiagaraActor.h, ANiagaraActor 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraActor.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraActor.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraActor.h`
- **快照**: commit `b6ab0dee9`(分支 `Eureka` / 基于 4.26)
- **文件规模**: 66 行(Phase 2 最小的头文件)
- **同仓协作文件**: `NiagaraActor.cpp`、`NiagaraComponent.h`
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 2 — 场景入口 · Component 层 2.2

## 职责 / 这个文件干什么的

定义 `ANiagaraActor`——**一个纯粹的 ComponentWrapperClass**,把 `UNiagaraComponent` 包装成一个可独立放置在关卡里的 Actor。整个类只有:一个 `UNiagaraComponent*` 成员、一个"播完自销毁"开关、一条把 Component 的 `OnSystemFinished` 接到自销毁的回调路径、以及编辑器下的图标可视化(Billboard + Arrow)。

这个类**不承担任何特效逻辑**——真正的逻辑全在 `UNiagaraComponent` 和 `FNiagaraSystemInstance`。它的存在是为了:

1. 让关卡设计师能在 Content Browser 里拖 `UNiagaraSystem` 到视口,UE 自动生成这个 Actor
2. 让美术能给环境特效(火盆、瀑布等)一个独立的 Actor outliner 条目
3. 作为 Cascade 时代 `AEmitter` 的对位物,保持工作流一致

## 关键类型 / 函数 / 宏

### 主类

- `ANiagaraActor : public AActor`(L11):
  ```cpp
  UCLASS(MinimalAPI, hideCategories = (Activation, "Components|Activation", Input, Collision, "Game|Damage"), ComponentWrapperClass)
  ```
  - `ComponentWrapperClass`——元标志,告诉编辑器这是纯包装 Actor,某些 UI 简化
  - `MinimalAPI`——只导出必要符号,减少 DLL 体积
  - `hideCategories` 隐藏大量 AActor 默认面板(Activation/Input/Collision/Damage)——因为对纯特效 Actor 都没意义

### 唯一的核心字段

- `UNiagaraComponent* NiagaraComponent`(L26,`meta = (AllowPrivateAccess = "true")`):
  - **整个 Actor 的灵魂**——ANiagaraActor 就是这个 Component 的挂载壳
  - 私有 + `AllowPrivateAccess` = 对外提供 BP 只读访问,但类内保持封装

### 自销毁开关

- `uint32 bDestroyOnSystemFinish : 1`(L41):bitfield 存储
- `SetDestroyOnSystemFinish(bool)`(L21,`UFUNCTION(BlueprintCallable)`):BP 可调;注意它是 `NIAGARA_API` 导出

### 生命周期钩子

- `virtual void PostRegisterAllComponents() override`(L17):AActor 所有组件注册完时调,典型用法是绑定 `NiagaraComponent->OnSystemFinished`
- `void OnNiagaraSystemFinished(UNiagaraComponent* FinishedComponent)`(L45,`UFUNCTION(CallInEditor)`):**自销毁回调**。绑定在 Component 的 `OnSystemFinished` delegate 上;若 `bDestroyOnSystemFinish=true` 则 `Destroy()` 自己
  - `CallInEditor` 让这个函数也能在编辑器的 Actor 详情面板上一键触发,便于调试

### 编辑器可视化

- `#if WITH_EDITORONLY_DATA`(L28-L37):
  - `UBillboardComponent* SpriteComponent`(L31):场景里的小图标(粒子系统图标)
  - `UArrowComponent* ArrowComponent`(L35):方向指示箭头
  - 两个 getter `GetSpriteComponent / GetArrowComponent`(L52-L54)

### 编辑器接口

- `#if WITH_EDITOR`(L57-L64):
  - `virtual bool GetReferencedContentObjects(TArray<UObject*>& Objects) const override`:告诉 "Reference Viewer" 这个 Actor 引用了哪些资产(当然就是 NiagaraComponent->Asset)
  - `ResetInLevel()`(L63,`NIAGARA_API`):关卡里重置特效——通常是清掉已有模拟状态重跑

### 公共 getter

- `GetNiagaraComponent() const`(L49,inline):返回 Component 指针,`NIAGARA_API` 导出

## 依赖与被依赖

**上游(此文件 include / 使用):**
- `CoreMinimal.h`、`ObjectMacros.h`
- `GameFramework/Actor.h`(AActor 基类)

**下游(谁用它):**
- 关卡(.umap)里直接持有 ANiagaraActor 实例
- `UNiagaraFunctionLibrary::SpawnSystemAtLocation` 内部**不**走 ANiagaraActor——它直接 new UNiagaraComponent 并附在临时 Actor 上(Pool 用 `ANiagaraActor` 做池载体是另一种策略,见 Phase 9)
- 编辑器里 drag-drop `UNiagaraSystem` 到视口,UE 自动生成 `ANiagaraActor`

## 关键代码片段

> `NiagaraActor.h:L10-L13` @ `b6ab0dee9` — 类声明
> ```cpp
> UCLASS(MinimalAPI, hideCategories = (Activation, "Components|Activation", Input, Collision, "Game|Damage"), ComponentWrapperClass)
> class ANiagaraActor : public AActor
> {
>     GENERATED_UCLASS_BODY()
> ```

> `NiagaraActor.h:L23-L26` @ `b6ab0dee9` — 唯一的逻辑字段
> ```cpp
> private:
>     /** Pointer to System component */
>     UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category=NiagaraActor, meta = (AllowPrivateAccess = "true"))
>     class UNiagaraComponent* NiagaraComponent;
> ```

> `NiagaraActor.h:L43-L45` @ `b6ab0dee9` — 自销毁回调接口点
> ```cpp
> /** Callback when Niagara system finishes. */
> UFUNCTION(CallInEditor)
> void OnNiagaraSystemFinished(UNiagaraComponent* FinishedComponent);
> ```
> 这个函数是从 `UNiagaraComponent::OnSystemFinished` delegate 回调过来的——一个典型的 observer pattern 小而完整的示例。

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/Niagara/ANiagaraActor]] — 本文件定义的主类
- [[Wiki/Entities/Stock/Niagara/UNiagaraComponent]] — 唯一的逻辑字段
- [[Wiki/Concepts/UE/UE4-uobject-系统]] — ComponentWrapperClass meta 标志使用

## 与既有 wiki 的关系

- 与 [[Wiki/Concepts/UE/Niagara/Niagara-vs-cascade]] 的 "Cascade 的 AEmitter 对应 Niagara 的 ANiagaraActor" 隐含预期一致:都是极简包装
- 印证了 Niagara 的设计取向——**把所有逻辑都压到 Component 上**,Actor 只是挂载壳。与 UE4 通用 ActorComponent 哲学一致

## 开放问题 / 待深入

- Pool 的载体究竟是 Component 还是 Actor?`UNiagaraComponentPool` 的 API 看似直接管理 Component——意味着 ANiagaraActor 不是 pool 的主线,只是"静态放置"的载体?→ 确认留 Phase 9 [[Wiki/Sources/Stock/Niagara/NiagaraComponentPool]]
- `ResetInLevel()` 的实现细节(重置 SystemInstance 还是重新 Activate)?→ 查 .cpp 可;对本 Phase 无核心价值
- `OnNiagaraSystemFinished` 是在 `PostRegisterAllComponents` 里绑定的,但这是 Actor 级生命周期——如果 Component 在运行时被换掉(`SetAsset` 后 Reinitialize),delegate 还在吗?→ 留开放问题,非关键路径
