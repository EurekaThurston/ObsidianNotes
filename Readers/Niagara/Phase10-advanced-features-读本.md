---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, learning-path, phase-10, reader, sim-stages, fluid, grid]
sources: 6
aliases: [Phase 10 读本, Niagara 高级特性读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 10 读本 — Niagara 的高级特性:SimStages 与 Grid 模拟

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] **最后一个 Phase** 的主题读本。一次读完掌握 Simulation Stages 多 pass 架构 + Grid 2D/3D 场存储 + NeighborGrid 空间哈希 + Emitter 间数据交互。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 10 要回答的问题

> [!question] Phase 10 要回答
> - Simulation Stages 是什么?为什么需要多 pass?
> - Niagara 怎么做 2D/3D 流体模拟?
> - 粒子怎么查自己的邻居(SPH / boids)?
> - 不同 emitter 的数据怎么互通?

6 文件(944 行):

| # | 文件 | 行 | 角色 |
|---|---|---|---|
| 10.1 | `NiagaraSimulationStageBase.h` | 78 | SimStage Asset 侧 |
| 10.2 | `NiagaraDataInterfaceRW.h` | 246 | **RW DI 基础**(Grid2D/3D 基类) |
| 10.3 | `NiagaraDataInterfaceGrid2DCollection.h` | 268 | 2D 场网格(Texture2DArray) |
| 10.4 | `NiagaraDataInterfaceGrid2DCollectionReader.h` | 95 | 只读代理 |
| 10.5 | `NiagaraDataInterfaceGrid3DCollection.h` | 158 | 3D 场网格(RWTexture3D) |
| 10.6 | `NiagaraDataInterfaceNeighborGrid3D.h` | 99 | 空间哈希 |

---

## 1. Simulation Stages — 为什么需要多 pass

### 1.1 传统 Niagara 的局限

Phase 3 / 5 讲的 tick 流程:System Spawn → System Update → Emitter Spawn → Emitter Update → Renderer。这是**单 pass**:每帧每 emitter 脚本执行一次。

但流体模拟等算法要**多 pass**:
- Jacobi 压力求解 = 20-40 次迭代
- 扩散 / 平流 = 至少 2-3 个独立 pass
- Boids 群体 = 先写邻居,再读邻居计算

### 1.2 SimStages 解决方案

```cpp
UCLASS(meta = (DisplayName = "Generic Simulation Stage"))
class UNiagaraSimulationStageGeneric : public UNiagaraSimulationStageBase
{
    ENiagaraIterationSource IterationSource;       // Particles | DataInterface
    int32 Iterations = 1;
    uint32 bSpawnOnly : 1;
    uint32 bDisablePartialParticleUpdate : 1;
    FNiagaraVariableDataInterfaceBinding DataInterface;
};
```

一个 Emitter 可以有**多个 SimStage**。每帧 Emitter tick 依次跑所有 stage。每个 stage:
- **独立的 `UNiagaraScript`**(独立 VM/compute shader)
- **独立的 IterationSource**(粒子数 或 DI 元素数)
- **可重复 Iterations 次**(Jacobi)

### 1.3 Iterations × FSimulationStageMetaData::MinStage/MaxStage

`Iterations=40` 时 shader 侧不是 40 条独立 stage,而是用 **一条 `FSimulationStageMetaData` 带 MinStage/MaxStage=40**,shader 编译产物精简。Phase 8.5 已预告。

### 1.4 IterationSource 2 种

- **Particles**:传统每粒子一次
- **DataInterface**:每 DI 元素一次 —— thread 数 = `DI.GetElementCount()`,典型用 Grid(每 cell 一个 thread)

### 1.5 `bSpawnOnly` 与 `bDisablePartialParticleUpdate`

- `bSpawnOnly`(仅 DataInterface 模式):只在 emitter reset 后首 tick 跑 —— 用于初始化 grid
- `bDisablePartialParticleUpdate`(仅 Particles 模式):关闭 "读写同 buffer" 优化 —— debugging 工具

---

## 2. RW DI 基础(Phase 7 的延续)

Phase 7 末尾见过 `UNiagaraDataInterfaceRenderTarget2D : UNiagaraDataInterfaceRWBase`。Phase 10 是基类 + Grid 家族的完整展开。

### 2.1 `FNiagaraDataInterfaceProxyRW`

```cpp
struct FNiagaraDataInterfaceProxyRW : public FNiagaraDataInterfaceProxy {
    virtual FIntVector GetElementCount(FNiagaraSystemInstanceID) const = 0;    // thread 数
    virtual uint32 GetGPUInstanceCountOffset(FNiagaraSystemInstanceID) const;  // GPU count offset
    virtual void ClearBuffers(FRHICommandList&) {}
    virtual FNiagaraDataInterfaceProxyRW* AsIterationProxy() override { return this; }
};
```

**核心接口**:
- `GetElementCount` —— dispatch 时 "多少 threads"
- `AsIterationProxy = this` —— 让 SimStage 认可"本 DI 可作 iteration source"

### 2.2 `UNiagaraDataInterfaceRWBase`

Deprecated 字段 `OutputShaderStages / IterationShaderStages` —— 旧版,现走 `FSimulationStageMetaData`(Phase 8.5)。

### 2.3 Grid2D / Grid3D 基类

```cpp
// Grid3D
FIntVector NumCells;
float CellSize;
int32 NumCellsMaxAxis;
ESetResolutionMethod SetResolutionMethod;    // Independent / MaxAxis / CellSize
FVector WorldBBoxSize;
```

**`ESetResolutionMethod` 的意义**:对用户来说,想控制"cell 数"、"cell 大小"、"最大边 cell 数"三种之一更自然。勾一种,其他自动算。

### 2.4 共享 HLSL 函数名

```
NumCells / CellSize / WorldBBoxSize
SimulationToUnit / UnitToSimulation
UnitToIndex / UnitToFloatIndex / IndexToUnit
IndexToUnitStaggeredX / Y  ← 交错网格(MAC grid)
IndexToLinear / LinearToIndex
ExecutionIndexToGridIndex / ExecutionIndexToUnit
```

所有 Grid DI 共享这组 API。HLSL 里用 `NumCells_DI_0 / CellSize_DI_0 / ...`(后缀是 `DI_{index}`,编译器为每个 DI 实例生成唯一)。

---

## 3. Grid2DCollection — Texture2DArray 存场

### 3.1 存储策略

```cpp
class FGrid2DBuffer {
    FTexture2DArrayRHIRef GridTexture;   // ← slice = attribute
    FShaderResourceViewRHIRef GridSRV;
    FUnorderedAccessViewRHIRef GridUAV;
};
```

**N 个 attribute 用 N-slice Texture2DArray**。HLSL 采样:

```hlsl
float value = GridTexture.SampleLevel(sampler, uint3(x, y, attributeIdx), 0);
```

### 3.2 双缓冲

```cpp
struct FGrid2DCollectionRWInstanceData_RenderThread {
    TArray<TUniquePtr<FGrid2DBuffer>> Buffers;
    FGrid2DBuffer* CurrentData;
    FGrid2DBuffer* DestinationData;

    void BeginSimulate(FRHICommandList&) / EndSimulate(FRHICommandList&);
};
```

和 Phase 4 DataSet / Phase 6 Renderer 双缓冲同源。每 pass `BeginSimulate` → VM/shader 写 Destination → `EndSimulate` 切 Current ← Destination。

### 3.3 Proxy SimStage Hooks

```cpp
struct FNiagaraDataInterfaceProxyGrid2DCollectionProxy : public FNiagaraDataInterfaceProxyRW
{
    virtual void PreStage(...) override;        // 切 buffer
    virtual void PostStage(...) override;       // 收尾
    virtual void PostSimulate(...) override;    // 复制到 user RT
    virtual void ResetData(...) override;       // clear
    virtual FIntVector GetElementCount(...) const override;   // (NumCells.X, NumCells.Y, 1)
};
```

**4 个钩子** — RW DI 生命周期的四阶段。

### 3.4 VM / HLSL 函数家族

```
ClearCellFunctionName, CopyPreviousToCurrentForCellFunctionName
Set/Get/SampleGrid × Float / Vector2 / Vector3 / Vector4
GetXxxAttributeIndex
SetNumCells / GetNumCells / GetWorldBBoxSize / GetCellSize
```

**按 channel 数重载**是标准 HLSL 模式——1/2/3/4 channel 的 `Set/Get/Sample` 各一套,shader 里没有模板所以得展开。

### 3.5 Editor 自动 HLSL 生成

```cpp
virtual bool SupportsSetupAndTeardownHLSL() const override { return true; }
virtual bool SupportsIterationSourceNamespaceAttributesHLSL() const override { return true; }
// 生成相应 HLSL
```

让 Grid2DCollection 作 SimStage iteration source 时,**自动生成 StackContext parameter map 的读写 HLSL**——脚本写`Grid2D.Density = ...` 就够了,不用手写 shader 绑定。

### 3.6 Experimental + Deprecated BP 方法

标 Experimental。有几个 BP 方法(`FillTexture2D / FillRawTexture2D / GetRawTextureSize`)标 `DeprecatedFunction`,建议用 user param 提供 RT 的方式取代。

### 3.7 Editor Preview

```cpp
#if WITH_EDITORONLY_DATA
uint8 bPreviewGrid : 1;
FName PreviewAttribute = NAME_None;
#endif
```

Editor 里可开预览,调试 grid 可视化。

---

## 4. Grid2DCollectionReader — Emitter 间数据交互

### 4.1 为什么需要

示例:Emitter A 模拟烟雾密度到 Grid2D,Emitter B 根据这个密度 spawn 粒子。两者独立 emitter,但共享数据。

### 4.2 设计

```cpp
UCLASS(hidecategories = (Grid, RW))
class UNiagaraDataInterfaceGrid2DCollectionReader : public UNiagaraDataInterfaceGrid2D
{
    FString EmitterName;
    FString DIName;
    virtual void GetEmitterDependencies(UNiagaraSystem*, TArray<UNiagaraEmitter*>&) const override;
};
```

- 继承 Grid2D 基类(共享 API)
- 但 `hidecategories = (Grid, RW)` —— **尺寸继承自被读的 grid**,editor 隐藏这些参数
- `GetEmitterDependencies` 声明依赖:Niagara 调度器据此决定 tick 顺序——被读的 emitter 必须先 tick

### 4.3 Proxy

RT 侧存目标 proxy 的指针:

```cpp
struct FGrid2DCollectionReaderInstanceData_RenderThread {
    FNiagaraDataInterfaceProxyGrid2DCollectionProxy* ProxyToUse;
};
```

读时 follow 指针到目标 grid 的 buffer。

### 4.4 只有 GetValue / SampleGrid

```cpp
static const FName GetValueFunctionName;
static const FName SampleGridFunctionName;
```

不提供 Set——只读。防止读者污染被读 emitter 的数据。

---

## 5. Grid3DCollection — 3D 体积场

### 5.1 存储策略差异

```cpp
class FGrid3DBuffer {
    FTextureRWBuffer3D GridBuffer;   // ← 3D RW 纹理
};
```

和 2D 的 Texture2DArray 不同,3D 用 `RWTexture3D`。多属性用 **tile 打包**:

```
NumCells = (32, 32, 32), NumAttributes = 4
实际 GridBuffer = 3D 纹理 尺寸 (64, 32, 64)  例:NumTiles = (2, 1, 2)
attribute 0 在 tile (0,0,0)
attribute 1 在 tile (1,0,0)
attribute 2 在 tile (0,0,1)
attribute 3 在 tile (1,0,1)
```

### 5.2 为什么 2D 和 3D 策略不同

- 2D Texture2DArray:硬件直接支持 N-slice 采样,HLSL 写 `tex[uint3(x, y, attr)]`
- 3D RWTexture3D:UE4.26 RHI 没有 Texture3DArray,用 tile 打包变通

### 5.3 主字段

```cpp
int32 NumAttributes;
FNiagaraUserParameterBinding RenderTargetUserParameter;
ENiagaraGpuBufferFormat BufferFormat;
```

### 5.4 VM 函数(简化版)

```
SetValue / GetValue / SampleGrid
```

不像 2D 有 per-channel 重载——3D 常用 float/vec4。

---

## 6. NeighborGrid3D — 空间哈希

### 6.1 不同于 Collection

```
Grid3DCollection:    每 cell 存 1 值(场)
NeighborGrid3D:      每 cell 存 MaxNeighborsPerCell 个粒子索引(对象列表)
```

目的完全不同。

### 6.2 数据结构

```cpp
class NeighborGrid3DRWInstanceData {
    FIntVector NumCells;
    float CellSize;
    bool SetGridFromCellSize;               // true → 由 CellSize+WorldBBoxSize 反推 NumCells
    uint32 MaxNeighborsPerCell;            // ⭐
    FVector WorldBBoxSize;

    FRWBuffer NeighborhoodBuffer;           // 粒子索引数组,尺寸 CellCount × MaxNeighbors
    FRWBuffer NeighborhoodCountBuffer;       // 每 cell 当前数量
};
```

`SetGridFromCellSize` 是 4.26 特有的双驱动模式开关:两种配置 `NumCells` 的方式——**显式给 `NumCells`**(`SetGridFromCellSize=false`),或 **给 `CellSize`+`WorldBBoxSize` 反推 `NumCells`**(`SetGridFromCellSize=true`)。哪种都行,但 `PerInstanceTick` 里必须决定用哪条路径。

### 6.3 使用模式(SPH 典型)

```cpp
// SimStage 0 (IterationSource=Particles):
each particle:
    cellIdx = hash(Position)
    uint slotIdx;
    InterlockedAdd(Counts[cellIdx], 1, slotIdx);     // 原子占槽
    if (slotIdx < MaxNeighborsPerCell)
        Neighborhood[cellIdx * MaxNeighbors + slotIdx] = ParticleID;

// SimStage 1 (IterationSource=Particles):
each particle:
    cellIdx = hash(Position)
    density = 0;
    for (dx, dy, dz in [-1, 0, 1]^3):             // 27 个邻 cell
        ncell = cellIdx + offset(dx, dy, dz);
        for (k in [0, Counts[ncell])):
            neighborID = Neighborhood[ncell * MaxNeighbors + k];
            neighborPos = GetParticlePos(neighborID);
            density += kernel(Position - neighborPos);
```

### 6.4 `PreStage` 清空

每帧重建邻域关系:

```cpp
virtual void PreStage(FRHICommandList&, const FNiagaraDataInterfaceStageArgs&) override;
// 实现里清 NeighborhoodCountBuffer(set all to 0)
```

### 6.5 `MaxNeighborsPerCell` 容量陷阱

每 cell 溢出时**直接 drop**——不 resize 不 warn。`MaxNeighbors` 太小 → 粒子密集时丢邻居 → SPH 错误(密度偏低)。典型值 32-128,视粒子密度和 cell 大小调。

### 6.6 `PostInitProperties` 自动注册

```cpp
virtual void PostInitProperties() override {
    if (HasAnyFlags(RF_ClassDefaultObject))
        FNiagaraTypeRegistry::Register(FNiagaraTypeDefinition(GetClass()), true, false, false);
}
```

CDO 时自动把本 DI 类型加进 Niagara 类型系统——script 里能用 `NeighborGrid3D` 当参数类型。

---

## 7 条关键洞察

1. **SimStages = 多 pass + per-pass script + Iterations 收合** —— Jacobi 等迭代式可用一条 SimStage + Iterations=N 代替 N 条
2. **IterationSource 两种**:Particles(每粒子)/ DataInterface(每 cell)—— 后者让 "每 cell 跑一次 shader" 成可能
3. **RW DI 核心钩子 4 个**:`PreStage / PostStage / PostSimulate / ResetData` —— 每 SimStage 四时点回调
4. **Grid2D 用 Texture2DArray,Grid3D 用 RWTexture3D + tile 打包** —— RHI 限制导致不同策略,但共享 API
5. **Grid2DCollectionReader 是 Emitter 间数据交互的关键** —— 通过 `GetEmitterDependencies` 声明依赖,调度器据此排 tick 顺序
6. **NeighborGrid3D 和 Grid3DCollection 目的完全不同** —— 前者存对象列表(SPH/boids),后者存场(烟雾密度)
7. **`MaxNeighborsPerCell` 是 lossy 设计** —— 超容丢粒子不 warn,太小导致 SPH 精度问题

---

## Phase 10 留下的问题

- `ENiagaraIterationSource` 完整定义在其他头文件 → Phase 10 实际运行细节
- MAC grid(staggered)的 `IndexToUnitStaggeredX/Y` 使用场景 → fluid 模拟文档
- 2D/3D Collection 的 tile 布局算法(NumTiles 计算)→ cpp
- Reader 跨 emitter 的 tick group promotion → Phase 9 ScalabilityManager 或 cpp
- 用户自定义 `UNiagaraSimulationStage` 子类可能性 → `abstract` 标记表明目前只有 Generic

## 学习路径总结

**Phase 10 是最后一个**。从 Phase 0(心智模型)到 Phase 10(SimStages + Grid),10 个 Phase 覆盖了 Niagara 运行时全部核心代码(~286 文件中最重要的 ~69 个)。

---

## 深入阅读

### 本议题的原子页

- 源摘要(Source × 6)
  - SimStages 主干:[[Wiki/Sources/Stock/NiagaraSimulationStageBase]]
  - RW DI 基类:[[Wiki/Sources/Stock/NiagaraDataInterfaceRW]]
  - Grid 家族:[[Wiki/Sources/Stock/NiagaraDataInterfaceGrid2DCollection]] / [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid2DCollectionReader]] / [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid3DCollection]]
  - 空间哈希:[[Wiki/Sources/Stock/NiagaraDataInterfaceNeighborGrid3D]]
- 实体(Entity × 5,Reader 合并到 Grid2DCollection,Grid2D/3D 基类合并到 RWBase)
  - [[Wiki/Entities/Stock/UNiagaraSimulationStage]]
  - [[Wiki/Entities/Stock/UNiagaraDataInterfaceRWBase]]
  - [[Wiki/Entities/Stock/UNiagaraDataInterfaceGrid2DCollection]]
  - [[Wiki/Entities/Stock/UNiagaraDataInterfaceGrid3DCollection]]
  - [[Wiki/Entities/Stock/UNiagaraDataInterfaceNeighborGrid3D]]

### 前置议题

- [[Readers/Niagara/Phase4-data-model-读本]] — DataSet 双缓冲思路是 RW DI 四钩子的前置
- [[Readers/Niagara/Phase7-data-interface-读本]] — `UNiagaraDataInterfaceRenderTarget2D` 是首个 RW DI 范例
- [[Readers/Niagara/Phase8-gpu-simulation-读本]] §3 `FSimulationStageMetaData` — SimStages 的 shader 侧元数据

### 10 个 Phase 导航

- [[Readers/Niagara/Phase0-心智模型-读本]]
- [[Readers/Niagara/Phase1-asset-layer-读本]]
- [[Readers/Niagara/Phase2-component-layer-读本]]
- [[Readers/Niagara/Phase3-runtime-instance-读本]]
- [[Readers/Niagara/Phase4-data-model-读本]]
- [[Readers/Niagara/Phase5-cpu-script-execution-读本]]
- [[Readers/Niagara/Phase6-rendering-读本]]
- [[Readers/Niagara/Phase7-data-interface-读本]]
- [[Readers/Niagara/Phase8-gpu-simulation-读本]]
- [[Readers/Niagara/Phase9-world-management-读本]]
- **[[Readers/Niagara/Phase10-advanced-features-读本]]**(本页,终点)

---

*本读本由 [[Claudian]] 基于 Phase 10 的 6 个头文件(合计 944 行)综合生成,2026-04-20。commit `b6ab0dee9`。Niagara 学习路径至此完结。*
