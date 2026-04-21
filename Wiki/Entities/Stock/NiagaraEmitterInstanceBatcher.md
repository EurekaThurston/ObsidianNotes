---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, batcher, gpu, fx-system]
sources: 1
aliases: [NiagaraEmitterInstanceBatcher, GPU Batcher]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterInstanceBatcher.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraEmitterInstanceBatcher

> **GPU Niagara 的渲染线程总调度器**。名字有误导——不是"CPU 批处理",而是 GPU Compute 的批处理 + dispatch。继承 `FFXSystemInterface`。**Phase 8 真正展开**。

## 一句话角色

`class NiagaraEmitterInstanceBatcher : public FFXSystemInterface`。RT 驻留对象,按渲染 pipeline stage 接收 GT 推来的 `FNiagaraGPUSystemTick`,dispatch compute shader,管理 GPU 实例计数 + GPU 排序 + Dummy UAV 池。

## 三阶段 Tick Stage

```cpp
enum class ETickStage { PreInitViews, PostInitViews, PostOpaqueRender };
```

每个 tick 根据依赖(GlobalDistanceField / DepthBuffer / EarlyViewData)分到对应 stage。

## 核心 RT 入口(FFXSystemInterface 合约)

| 方法 | 对应 Stage |
|---|---|
| `PreInitViews` | PreInitViews |
| `PostInitViews(ViewUniformBuffer)` | PostInitViews |
| `PreRender(GlobalDistanceFieldParameterData)` | (继续处理) |
| `PostRenderOpaque(ViewUniformBuffer, SceneTextures)` | PostOpaqueRender |

## 核心字段

- `GPUInstanceCounterManager`(`FNiagaraGPUInstanceCountManager`)— Phase 8.6
- `GPUSortManager`(`TRefCountPtr<FGPUSortManager>`)+ `SimulationsToSort` — Phase 8.7
- `GlobalCBufferLayout / SystemCBufferLayout / OwnerCBufferLayout / EmitterCBufferLayout` — 4 类 cbuffer 布局缓存
- `Ticks_RT` — RT 队列
- `DummyBufferPool / DummyTexturePool` — 空 UAV 池

## 关键 API

```cpp
void GiveSystemTick_RenderThread(FNiagaraGPUSystemTick&);      // GT→RT 入口
void InstanceDeallocated_RenderThread(FNiagaraSystemInstanceID);
void Run(const FNiagaraGPUSystemTick&, ..., const FNiagaraShaderRef&, ...);  // 实际 dispatch
bool AddSortedGPUSimulation(FNiagaraGPUSortInfo&);            // 注册排序任务
```

## 相关

- [[Wiki/Entities/Stock/FNiagaraGPUSystemTick]] — 输入的 tick 打包
- [[Wiki/Entities/Stock/FNiagaraComputeExecutionContext]] — tick 里引用
- [[Wiki/Entities/Stock/FNiagaraSystemSimulation]] — GT 侧 batcher 对位
- `FNiagaraGPUInstanceCountManager` / `FGPUSortManager` — Phase 8 深入

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraEmitterInstanceBatcher]]
- 主题读本:[[Readers/Niagara/Phase 5 - Niagara 脚本如何跑起来]] § 6(Phase 8 主战场)

## 开放问题(大部分延迟到 Phase 8)

- 全部 GPU 资源管理细节 → Phase 8
