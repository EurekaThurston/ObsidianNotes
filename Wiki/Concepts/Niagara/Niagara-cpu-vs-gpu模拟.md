---
type: concept
created: 2026-04-18
updated: 2026-04-18
tags: [niagara, gpu, cpu, 模拟, 计算着色器, vectorvm]
sources: 0
aliases: [CPU Simulation, GPU Simulation, Niagara GPU, VectorVM, Compute Shader]
---

# Niagara CPU vs GPU 模拟

> Niagara 的每个 Emitter 可以选择在 CPU 或 GPU 上运行。这个选择深刻影响了它的能力边界、性能特征和源码实现方式。

---

## 概览

粒子模拟的本质是：**每帧对大量粒子执行相同的计算**（更新位置、速度、颜色……）。这种"大量相同计算"天然适合并行，而 CPU 和 GPU 的并行能力天差地别：

| | CPU | GPU |
|---|---|---|
| 核心数 | 8 ~ 32 核 | 数千个计算单元 |
| 适合的任务 | 复杂逻辑、分支多、需要游戏状态 | 简单重复、大规模并行 |
| Niagara 中的执行器 | **VectorVM**（SIMD 软件虚拟机） | **Compute Shader**（HLSL） |
| 粒子上限（典型） | 数万 | 数十万 ~ 百万 |
| 调试难度 | 容易 | 困难 |

---

## CPU 模拟

### 执行者：VectorVM

Niagara 的 CPU 模拟不是逐粒子跑普通 C++ 代码，而是跑在一个叫 **VectorVM** 的自定义虚拟机上。

VectorVM 是 `Engine/Source/Runtime/VectorVM/` 下的引擎内置模块（不在 Niagara 插件里），它：

- 把 Niagara 脚本图**编译成字节码**
- 字节码的指令集专为 SIMD 设计：每条指令同时操作 **4 或 8 个粒子**
- 在 CPU 上，利用 SSE/AVX 指令集并行执行

**类比**：VectorVM 就像一个专门为粒子计算设计的"微型 CPU"，每次处理一小批（4/8个）粒子，循环直到处理完所有粒子。

### CPU 模拟的执行流程

```
UNiagaraScript (字节码)
      │
      ▼
FNiagaraScriptExecutionContext  ← 绑定：字节码 + DataSet + DataInterface
      │
      ▼
VectorVM::Exec()  ← 实际执行字节码，每次 4/8 粒子
      │
      ▼
FNiagaraDataSet  ← 粒子属性数组（Position, Velocity, Color...）被更新
```

### CPU 模拟的优势

- ✅ **可访问游戏状态**：可以读 Actor 位置、物理结果、游戏逻辑变量
- ✅ **精确碰撞**：`NiagaraDataInterfaceCollisionQuery` 可以做完整的物理 line trace
- ✅ **事件系统**：粒子间可以发送/接收事件（死亡事件、碰撞事件）
- ✅ **调试友好**：可以在 Niagara Debugger 里逐粒子检查属性值
- ✅ **DataInterface 全支持**：所有 DI（骨骼网格采样、相机、音频等）都支持 CPU

### CPU 模拟的局限

- ❌ **粒子数量上限低**：CPU 核心少，超过几万粒子帧率明显下降
- ❌ **主线程/任务线程开销**：大量 CPU 粒子会吃 CPU 预算
- ❌ **无法利用 GPU 的天然并行**

---

## GPU 模拟

### 执行者：Compute Shader

GPU 模拟把粒子脚本**编译成 HLSL Compute Shader**，在 GPU 上执行。每个 GPU 线程处理一个粒子，成千上万线程同时运行。

```
UNiagaraScript (GPU 字节码 / HLSL)
      │
      ▼
NiagaraShader (FNiagaraShader)  ← 编译为 GPU Compute Shader
      │
      ▼
FNiagaraEmitterInstanceBatcher  ← 每帧把 Dispatch 命令提交给渲染线程
      │
      ▼
GPU 执行  ← 数万个线程并行处理所有粒子
      │
      ▼
GPU Buffer（粒子数据，留在 GPU 显存）  ─────────────▶ 渲染（Vertex Factory 直接读）
```

注意：GPU 模拟的粒子数据**留在 GPU 显存里**，不拷回 CPU。渲染阶段 Vertex Factory 直接从 GPU Buffer 读数据绘制——这是 GPU 粒子高效的关键。

### GPU 模拟的优势

- ✅ **海量粒子**：百万级粒子也能跑，因为 GPU 有数千个并行计算单元
- ✅ **零 CPU 开销**（模拟阶段）：CPU 只提交命令，不做实际计算
- ✅ **渲染零拷贝**：数据不用 CPU↔GPU 拷贝，直接在 GPU 上模拟+渲染

### GPU 模拟的限制

- ❌ **访问游戏状态受限**：GPU 上无法直接读 Actor 位置——必须通过 DI 把数据**提前上传**到 GPU（每帧一次 CPU→GPU 拷贝）
- ❌ **不支持所有 DataInterface**：只有实现了 `GetParameterDefinitionHLSL()` 和 `GetFunctionHLSL()` 的 DI 才能在 GPU 上用
- ❌ **Fixed Bounds 要求**：因为粒子在 GPU 上，CPU 不知道粒子在哪，所以 GPU Emitter **必须手动设置 Bounds**（编辑器里勾选 Fixed Bounds），否则遮挡剔除会出错
- ❌ **调试极难**：GPU 上的粒子状态几乎无法实时检查
- ❌ **粒子数量是预分配的**：GPU Emitter 需要预先分配 Buffer 大小（`MaxParticleCount`），运行时不能动态扩容

---

## 如何选择 CPU vs GPU

在 Niagara Editor 里，每个 Emitter 右上角有一个 **"Sim Target"** 下拉：
- `CPU Sim`
- `GPU Compute Sim`

**选 CPU 当：**
- 粒子数量少（< ~5000）
- 需要精确碰撞
- 需要粒子事件（死亡/碰撞触发生成）
- 需要读复杂游戏状态（NPC 骨骼、实时音频）

**选 GPU 当：**
- 需要大量粒子（烟雾、液体、群体效果）
- 视觉效果为主，不需要复杂游戏交互
- 可以接受 Fixed Bounds

---

## 在源码里如何区分

### 枚举值

`UNiagaraEmitter` 里有一个字段标记模拟目标：
```cpp
// NiagaraEmitter.h
ENiagaraSimTarget SimTarget;  // CPUSim 或 GPUComputeSim
```

`ENiagaraSimTarget` 是一个枚举，定义在 `NiagaraCommon.h`（Phase 4 学习）。

### 运行时分叉

`FNiagaraEmitterInstance` 在 Tick 时，根据 `SimTarget` 走不同路径：
- CPU → 调用 `VectorVM::Exec()`
- GPU → 提交 Dispatch 命令到 `FNiagaraEmitterInstanceBatcher`

`FNiagaraEmitterInstanceBatcher`（Phase 5/8 学习）是两条路的调度中枢，是理解 CPU/GPU 分叉的关键文件。

### Shader 编译

GPU 模拟在编辑器里会触发 Shader 编译（右下角转圈）。这个流程：
- `UNiagaraScript` 保存 GPU 字节码
- `NiagaraShader` 模块把字节码编译成真正的 HLSL → DXBC/SPIRV
- 编译结果缓存在 `NiagaraShaderMap`（Phase 8 学习）

---

## 混合使用

一个 `UNiagaraSystem` 可以**同时包含 CPU Emitter 和 GPU Emitter**。例如：
- GPU Emitter：发射数万个火花（大量轻量粒子）
- CPU Emitter：发射 5 个大火球（少量但需要碰撞和物理）

两种 Emitter 通过 `FNiagaraSystemInstance` 统一管理，各自走自己的执行路径，最终一起渲染。

---

## 相关
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — Cascade 基本不支持真正的 GPU 模拟；这是 Niagara 的核心优势之一
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — CPU/GPU 的选择体现在 Asset（Emitter 的配置）上，运行时实例才真正执行
- [[Wiki/Syntheses/Niagara/Niagara-learning-path]] — Phase 5（CPU VM）和 Phase 8（GPU 模拟）分别深入两条路径

## 开放问题 / 待深入
- **Async GPU Readback**：有些情况下需要把 GPU 粒子数量读回 CPU（如判断特效是否结束）。`NiagaraGPUInstanceCountManager` 处理这个异步读回——Phase 8 展开
- **Simulation Stages（Phase 10）**：GPU 专属功能，允许一个 Emitter 在一帧内多次 Dispatch，实现迭代式计算（流体、布料）
- 4.26 里 GPU 模拟和光线追踪（Ray Tracing）的支持还在早期阶段，`NiagaraGPURayTracingTransformsShader.h` 可以看到相关入口
