---
type: concept
created: 2026-04-18
updated: 2026-04-18
tags: [niagara, cascade, UE4, 粒子系统, 设计哲学]
sources: 0
aliases: [Cascade, Niagara 对比 Cascade, 粒子系统演进]
---

# Niagara vs Cascade

> Cascade 是 UE4 的旧粒子系统，Niagara 是 UE4.20+ 的新一代替代品。理解它们的哲学差异，能帮你理解 Niagara 源码里许多"为什么这么设计"的决策。

---

## 概览

Cascade（UE3 时代延续）和 Niagara（UE4.20 引入）解决的是同一个问题：**让引擎产生粒子特效**。但它们的思路截然不同：

| 维度 | Cascade | Niagara |
|---|---|---|
| **设计哲学** | 黑盒模块组合 | 完全可编程数据流 |
| **行为定义方式** | 固定模块（无法修改内部） | 可视化脚本图（完全自定义） |
| **数据可见性** | 粒子数据对用户不透明 | 所有粒子属性完全可见可读写 |
| **CPU/GPU** | 主要 CPU（GPU 有限支持） | CPU + GPU 一等公民 |
| **性能模型** | 每粒子逐个处理 | SIMD 批量处理（VectorVM） |
| **源码位置** | `Engine/Source/Runtime/Engine/Classes/Particles/` | `Engine/Plugins/FX/Niagara/` |
| **当前状态** | 维护模式（不再新增功能） | 主力开发（持续迭代） |

---

## Cascade 的设计模型

### 黑盒模块

Cascade 的特效由**固定模块**组合而成：Initial Velocity、Lifetime、Size by Life、Color over Life……每个模块是一个黑盒，你只能填参数，不能修改它"怎么工作"。

```
Cascade Emitter:
  ┌──────────────┐
  │ Spawn Rate   │  ← 固定逻辑，只改数值
  │ Initial Vel  │  ← 固定逻辑，只改数值
  │ Color/Life   │  ← 固定逻辑，只改数值
  └──────────────┘
```

- **优点**：上手快，艺术家友好
- **缺点**：需要程序员实现新模块才能做新效果；不同项目重复实现类似功能

### 数据不透明

Cascade 的粒子数据（位置、速度、颜色等）被内部管理，**你无法在一个模块里直接读另一个模块写的数据**（除非有专门的接口）。这导致复杂交互很难实现。

### 源码结构

Cascade 的核心类在引擎本体（不是插件）：
- `UParticleSystem` → 资产（对应 Niagara 的 `UNiagaraSystem`）
- `UParticleEmitter` → Emitter 资产
- `FParticleEmitterInstance` → 运行时实例
- `UParticleModule` → 模块基类（黑盒的那些模块）

---

## Niagara 的设计模型

### 完全可编程的数据流

Niagara 的核心转变：**粒子行为由可视化脚本图定义**，图里可以任意连接节点，每个节点对应一个数学/逻辑操作。

```
Niagara Emitter Script（图示）:
  [Particle.Position]  →  [Add]  →  [Particle.Position]
                               ↑
                    [Particle.Velocity * DeltaTime]
```

这意味着：
- **任意属性可读写**：你可以在 Update 脚本里读任意粒子属性，写入任意属性
- **自定义属性**：可以给粒子添加任意自定义属性（如 `Particle.MyCustomFloat`）
- **数据接口（Data Interface）**：通过 DI 读取外部数据（相机位置、骨骼动画、音频频谱……），详见 [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]]

### 统一的脚本阶段

Niagara Emitter 的行为被分为几个固定**执行阶段**（Script），每帧按顺序运行：

| 阶段 | 触发时机 | 作用 |
|---|---|---|
| `EmitterSpawnScript` | Emitter 首次激活时 | 初始化 Emitter 级别变量 |
| `EmitterUpdateScript` | 每帧 | 更新 Emitter 级别变量（如发射速率） |
| `ParticleSpawnScript` | 每个新粒子诞生时 | 初始化粒子属性（位置、速度、颜色…） |
| `ParticleUpdateScript` | 每帧，对所有活跃粒子 | 更新粒子属性（物理、生命值减少…） |
| `EventHandlerScript` | 收到事件时 | 响应粒子事件（碰撞、死亡等） |
| `SimulationStageScript` | （可选，GPU 专用） | 多 pass 计算（流体模拟等） |

这些脚本类型对应 `UNiagaraScript` 里的 `ENiagaraScriptUsage` 枚举——Phase 1 读 `NiagaraScript.h` 时会直接看到。

### SIMD 批量执行

Cascade 的每粒子处理模型：
```
for particle in particles:
    process(particle)   ← 逐个处理，CPU cache unfriendly
```

Niagara 的 VectorVM 模型：
```
process(particles[0..3])  ← 一次处理 4 个粒子（SIMD 4-wide）
process(particles[4..7])
...
```

VectorVM 是 Niagara 的 CPU 端"虚拟机"，专为 SIMD 指令集（SSE/AVX）优化。大量粒子时性能差异显著。

---

## 关键哲学转变：显式 vs 隐式

这是理解 Niagara 源码复杂性的根本原因。

**Cascade 的隐式**：
- 粒子有哪些属性？固定的，写死在代码里
- 粒子数据怎么存？引擎内部黑盒，不对外暴露
- 模块怎么执行？按固定顺序，不可更改

**Niagara 的显式**：
- 粒子有哪些属性？**完全动态**，脚本图里用到哪些，编译时确定（`FNiagaraTypeDefinition` + `FNiagaraVariable`）
- 粒子数据怎么存？**显式的 DataSet**（`FNiagaraDataSet`），SoA 布局，每个属性是一个连续数组
- 模块怎么执行？**编译成字节码**，由 VectorVM / GPU Compute Shader 执行

这种显式设计带来了：
- ✅ 极高的灵活性和可扩展性
- ✅ 完全的 GPU 原生支持
- ❌ 更复杂的源码（编译、绑定、反射…全是你要学的东西）

---

## 迁移与共存

UE4 允许同时存在 Cascade 和 Niagara 资产，但官方的态度是：

> "All new projects should use Niagara. Cascade receives bugfixes only."

Cascade 的资产扩展名是 `.uasset`（`UParticleSystem` 类），Niagara 也是 `.uasset`（`UNiagaraSystem` 类）——文件扩展名一样，但类型完全不同，不可互换。

---

## 和源码学习的关系

读 Niagara 源码时，有时候你会在代码里看到 `// Legacy` 或类似 Cascade 风格的注释——那是历史遗留。Niagara 在 4.20 引入时并非完全重写，部分设计受 Cascade 影响。

理解 Cascade 的限制 → 理解 Niagara 为什么要那样设计 → 读源码时不会困惑"这个复杂结构到底解决了什么问题"。

---

## 相关
- [[Wiki/Concepts/UE/UE4-资产与实例]] — 两者都遵循资产/实例二元模型
- [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]] — Niagara 的 GPU 能力是与 Cascade 的关键差异之一
- [[Wiki/Syntheses/UE/Niagara/Niagara-learning-path]] — 学习路径（Phase 1 开始读 Niagara 资产类）

## 开放问题 / 待深入
- `FNiagaraEmitter` 里有一个 `bInterpolatedSpawning` 选项，这是 Cascade 时代就存在的概念（插值生成防止帧率抖动），值得在 Phase 1 时留意
- Niagara 4.26 里仍有部分功能（如某些粒子事件类型）功能上不完整，一些项目仍保留 Cascade 资产——这不是 bug，是功能未完全追平的历史遗留
