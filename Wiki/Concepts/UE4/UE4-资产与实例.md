---
type: concept
created: 2026-04-18
updated: 2026-04-18
tags: [UE4, asset, instance, 基础概念]
sources: 0
aliases: [Asset, Instance, 资产实例模型, UAsset]
---

# UE4 资产与实例

> UE4 最核心的二元模型：**Asset（资产）** 是存在磁盘上的"描述"，**Instance（实例）** 是运行时的"活体"。理解这个分离，是读懂 Niagara 源码的第一把钥匙。

---

## 概览

想象一份建筑蓝图（Blueprint）和一栋按图建造的楼：

- **蓝图**：画在纸上，描述楼长什么样。可以被复印多份，可以存档，可以修改后再建。
- **楼**：实际存在的建筑，占用现实空间，有自己的状态（住了哪些人、灯是否开着）。

UE4 里完全是同样的模式：

| 概念 | 类比 | 技术形式 | 存在周期 |
|---|---|---|---|
| **Asset（资产）** | 蓝图/设计图 | UObject 子类，`.uasset` 文件 | 持久，存磁盘 |
| **Instance（实例）** | 实际建造的楼 | C++ 对象（F 类或 UObject） | 临时，运行时 |

---

## 资产（Asset）

### 什么是资产

资产就是 Content Browser 里看到的那些文件：`.uasset`、`.umap`。每个资产在代码里对应一个 **UObject 子类的对象**（序列化到磁盘的那个）。

打开编辑器 → Content Browser → 双击一个 Niagara System → 你编辑的就是 `UNiagaraSystem` 这个资产对象。

### 资产的特征

- **只描述，不执行**：资产只保存"应该是什么样"，不保存运行时状态
- **可共享**：同一个资产可以在场景里同时存在 100 个实例，它们共用同一份资产数据
- **可序列化**：能保存/加载，这是 UObject 系统提供的能力（见 [[Wiki/Concepts/UE4/UE4-uobject-系统]]）

### Niagara 里的资产

| 资产类 | 对应文件 | 说明 |
|---|---|---|
| `UNiagaraSystem` | `.uasset` | 整个特效系统的定义（含若干 Emitter） |
| `UNiagaraEmitter` | `.uasset` | 一个 Emitter 的定义（含脚本、渲染器） |
| `UNiagaraScript` | 嵌在 Emitter 内 | 编译后的粒子行为脚本（字节码） |
| `UNiagaraRendererProperties` 子类 | 嵌在 Emitter 内 | 渲染器配置（Sprite/Ribbon/Mesh/Light） |
| `UNiagaraParameterCollection` | `.uasset` | 全局参数集，可以跨 System 共享 |

---

## 实例（Instance）

### 什么是实例

实例是资产在**运行时**的"化身"。当你在场景里放一个 `NiagaraComponent`、按下 Play，引擎就会根据 `UNiagaraSystem` 资产**创建**一个 `FNiagaraSystemInstance`，这就是实例。

实例：
- **持有运行时状态**：粒子的当前位置、速度、生命值、当前时间
- **不持久**：游戏退出或特效播放完毕后销毁
- **不可共享**：同一场景里的 100 个特效，有 100 个独立的实例（各自状态独立）

### Niagara 里的实例

| 实例类 | 对应资产 | 说明 |
|---|---|---|
| `FNiagaraSystemInstance` | `UNiagaraSystem` | 运行中的特效系统，持有所有 Emitter 实例 |
| `FNiagaraEmitterInstance` | `UNiagaraEmitter` | 运行中的单个 Emitter，持有粒子数据 |
| `FNiagaraDataSet` | — | 运行时粒子属性的实际内存块 |

注意 `F` 前缀——这些都是普通 C++ 对象，**不是 UObject**，不被 GC 管理，由 `TSharedPtr` / `TUniquePtr` 手动管理生命周期。

---

## 资产和实例的关系图

```
Content Browser（磁盘）              运行时（内存）
┌──────────────────────┐             ┌──────────────────────────┐
│  UNiagaraSystem      │──创建──────▶│  FNiagaraSystemInstance  │
│  (NS_Fire.uasset)    │             │  - 状态机                 │
│  - EmitterHandles[]  │   引用（只读）│  - 持有 EmitterInstances  │
│  - ParameterDefs     │◀────────────│  - 当前时间/帧计数        │
└──────────────────────┘             └──────────────────────────┘
         │                                      │
         │ 包含                                 │ 包含
         ▼                                      ▼
┌──────────────────────┐             ┌──────────────────────────┐
│  UNiagaraEmitter     │──创建──────▶│  FNiagaraEmitterInstance  │
│  - Scripts           │             │  - 粒子 DataSet           │
│  - RendererProps     │   引用（只读）│  - 当前粒子数量           │
└──────────────────────┘◀────────────│  - 运行时 DI 绑定         │
                                     └──────────────────────────┘
```

**关键点**：实例**引用**资产（读取参数），但**不修改**资产。资产是只读模板。

---

## Component：连接资产和实例的桥梁

资产不会自己跑起来。游戏世界里，是 `UNiagaraComponent` 把资产和实例连在一起：

```
场景里的 Actor
  └── UNiagaraComponent（组件）
         ├── 持有 UNiagaraSystem* 指针（资产引用）
         └── 持有 FNiagaraSystemInstance（实例，运行时创建）
```

当你在蓝图里调用 `Spawn System At Location`，背后：
1. 创建一个临时 Actor
2. 挂上一个 `UNiagaraComponent`
3. Component 根据你传入的 `UNiagaraSystem` 资产，创建 `FNiagaraSystemInstance`
4. 开始 Tick

---

## 为什么要分离？

这个设计回答了一个核心问题：**同一个特效在场景里同时出现 500 次，内存怎么办？**

- **不分离的方案**：每个特效都完整复制一份资产数据 → 500 份 Emitter 脚本字节码 → 内存爆炸
- **分离的方案**：500 个实例**共享**同一份资产 → 资产数据只有 1 份 → 500 个实例只保存各自的运行时状态（粒子位置/速度等）

同时：
- 资产可以在编辑器里修改后**热重载**，运行中的实例自动同步
- 资产可以被多个不同的 Component 引用，实现"同一个特效在不同场景中复用"

---

## 快速记忆口诀

> **资产 = 食谱（怎么做这道菜的描述）**
> **实例 = 盘子里的菜（正在吃的那份）**
>
> 一份食谱可以同时做 100 盘菜；每盘菜的状态（吃了多少）是独立的。

---

## 相关
- [[Wiki/Concepts/UE4/UE4-uobject-系统]] — UObject 是资产的技术基础
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — Cascade 也有类似的资产/实例模型，但更不透明
- [[Wiki/Syntheses/Niagara/Niagara-learning-path]] — 学习路径总览（Phase 1 开始读具体资产类）

## 开放问题 / 待深入
- `UNiagaraSystem` 内部有一个"编译"步骤：资产保存的不是原始图，而是编译后的数据（`FNiagaraSystemCompiledData`）。这个编译/缓存机制在 Phase 1 时再展开
- `CDO`（Class Default Object）：UClass 上有一个"默认实例"，有时候你取到的不是真正运行中的对象，而是 CDO——这是读编辑器代码时常见的陷阱
