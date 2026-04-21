---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, learning-path, phase-8, reader, gpu, compute-shader]
sources: 13
aliases: [Phase 8 读本, Niagara GPU 模拟读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 8 - Niagara 的 GPU 模拟管线

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 8 的**主题读本**。一次读完掌握 Niagara GPU 流水线:脚本 → compute shader → instance count → sort → draw indirect 的完整链路。本 Phase 是 [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟|CPU/GPU 双路径]] 里 GPU 侧的全貌。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 8 要回答的问题

> [!question] Phase 8 要回答
> - Niagara 脚本如何变成 compute shader?
> - 粒子数量是 GPU 算出来的,CPU 怎么发 draw call?
> - GPU 模拟的粒子数据怎么传给 vertex shader 渲染?
> - 半透明粒子的 GPU 排序怎么实现?

14 文件(8.8 Batcher 已 Phase 5 覆盖),实际新 13 文件。分 4 个子系统:

| 子系统 | 文件 |
|---|---|
| Shader 编译 | NiagaraShared, Shader, ShaderType, ShaderMap, ScriptBase |
| GPU Instance Count + Draw Indirect | GPUInstanceCountManager, DrawIndirect |
| GPU Sort | GPUSortInfo, SortingGPU |
| Vertex Factory | VertexFactory + Sprite/Ribbon/Mesh 三种 |

---

## 1. Niagara GPU 的整体流水线

```
GT:UNiagaraScript 编辑 → 编译产物(VM bytecode + HLSL)
         ↓
GT:FNiagaraShaderScript 包装,FNiagaraShaderMap 缓存
         ↓ (compile queue,worker thread)
GT:FNiagaraShader 可用
         ↓

每帧 GT:
    FNiagaraSystemInstance::Tick_Concurrent
        → 为每个 GPU emitter 构 FNiagaraComputeInstanceData(Phase 5)
        → 打包 FNiagaraGPUSystemTick(Phase 5)
        → GiveSystemTick_RenderThread → Batcher

每帧 RT:
    Batcher 按 ETickStage 处理(Phase 5)
        → 绑定 FNiagaraShader 参数
        → Dispatch compute shader
        → 粒子数据更新 GPUBufferFloat/Int/Half(Phase 4)
        → count 更新到 FNiagaraGPUInstanceCountManager.CountBuffer

PreRender(RT):
    FGPUSortManager 回调 → Batcher::GenerateSortKeys
        → FNiagaraSortKeyGenCS → 生成 (key, particleIndex) 对
        → radix sort → SortedIndices SRV

一帧末(RT):
    InstanceCountMgr.UpdateDrawIndirectBuffer
        → FNiagaraDrawIndirectArgsGenCS → DrawIndirectBuffer

渲染 pass:
    Phase 6 Renderer::GetDynamicMeshElements
        → 构 FMeshBatch(IndirectArgs = DrawIndirectBuffer, offset)
        → FNiagaraVertexFactory 绑 UBO + SRV(particle data + sort indices)
        → UE 渲染器 RHIDrawIndexedPrimitiveIndirect
```

---

## 2. Shader 编译基础

### 2.1 模块分层

Shader 相关全在 `NiagaraShader` 模块(不在 Niagara 主模块),依赖仅 `NiagaraCore`(Phase 5.1)。让 RHI/Shader 代码不拉 Niagara 主模块。

### 2.2 `FNiagaraShader` 参数矩阵

一个 GPU Emitter compile 出的 `FNiagaraShader` 持**海量参数**(粗分 8 类):

1. **粒子 Buffer**:3 SRV(Float/Int/Half input)+ 3 UAV(output)
2. **Instance Count**:共享 count buffer UAV + read/write offset
3. **Persistent IDs**:Free ID SRV + IDToIndex UAV
4. **Constant Buffers**:5 种(Global/System/Owner/Emitter/External)× 2 帧 = 10 个 + View UBO
5. **Spawn/Tick**:SimStart / TickCounter / SpawnInfoOffsets / SpawnInfoParams / NumSpawned / UpdateStartInstance / CopyBeforeStart
6. **SimStages**(Phase 10):Default/CurrentStageIndex / IterationInfo / NormalizedIterationIndex / DispatchThreadIdToLinear
7. **Events**:4 × Float/Int SRV + UAV + read/write stride(MAX_CONCURRENT_EVENT_DATASETS=4)
8. **DI 参数**:按 DI 类型可变

### 2.3 ThreadGroup Size 平台分化

```cpp
return Platform == SP_PS4 ? 64 : 32;
```

PS4 wave 是 64,其他(GCN/RDNA/NV)是 32。HLSL 里 `[numthreads(THREADGROUP_SIZE, 1, 1)]` define 化。

### 2.4 ShaderMap / ShaderMapId / 缓存

注意:`FNiagaraShaderMapId` 的**物理定义在 `NiagaraShared.h`**(不是 `NiagaraShaderMap.h`)——本节归在 ShaderMap 叙事线下,只因它是 ShaderMap 的 key。

```cpp
// 定义:NiagaraShared.h L229-256
class FNiagaraShaderMapId {
    FGuid CompilerVersionID;
    ERHIFeatureLevel::Type FeatureLevel;
    TMemoryImageArray<FMemoryImageString> AdditionalDefines;
    FSHAHash BaseCompileHash;
    TMemoryImageArray<FSHAHash> ReferencedCompileHashes;
    FPlatformTypeLayoutParameters LayoutParams;
    bool bUsesRapidIterationParams;
};
```

任何一项变 → id 变 → 重编。DDC(Derived Data Cache)用这个做 key,命中率决定编译开销。

### 2.5 编译流程

```
UNiagaraScript 编辑 →
FNiagaraCompilationQueue 加 item →
FNiagaraShaderQueueTickable 每帧处理 →
FNiagaraShaderType::BeginCompileShader(Script) →
    → worker thread 编译 HLSL →
FinishCompileShader 创建 FShader →
加入 FNiagaraShaderMap 缓存
```

### 2.6 `FNiagaraDataInterfaceParamRef`

DI 的 shader 参数绑定用 `LAYOUT_FIELD` + `TMemoryImagePtr`——**支持 binary memory image** 序列化进 shader cache。这是 Phase 5 `FNiagaraDataInterfaceParametersCS` 为什么 non-virtual 的原因——虚表指针无法序列化。

---

## 3. `UNiagaraScriptBase` + `FSimulationStageMetaData`(Phase 10 预告)

`NiagaraScriptBase.h` 里的两个类是 Phase 10 的桥。简要看下:

### `FSimulationStageMetaData`

```cpp
USTRUCT() struct FSimulationStageMetaData {
    FName SimulationStageName;
    FName IterationSource;                  // 迭代目标 DI(None=particles)
    uint32 bSpawnOnly : 1;
    uint32 bWritesParticles : 1;
    uint32 bPartialParticleUpdate : 1;
    TArray<FName> OutputDestinations;
    int32 MinStage, MaxStage;               // 一条元数据可覆盖 N 次迭代
};
```

Phase 10 SimStages 的元数据单元——每 stage 描述 "读哪个迭代源 / 是否写粒子 / 输出到哪里"。

### `UNiagaraScriptBase`

抽象基类(位于 NiagaraShader 模块),纯虚 `GetSimulationStageMetaData` 在 `UNiagaraScript`(Phase 1)实现。让 Shader 模块能依赖 script "我有几个 stages" 不拉 Niagara 主模块。

---

## 4. GPU Instance Count Manager — 解决"count 只 GPU 知道"

### 4.1 为什么需要

GPU 粒子数量在 compute shader 里可能每帧变化(粒子死亡/新生)。传统做法:CPU readback 回来,下帧发 draw call。**但 readback 会 stall**——等 GPU 完成才能读。

Niagara 方案:**不读回,直接用 draw indirect**。粒子数存在 GPU buffer 里,发 draw call 时用 `RHIDrawIndexedPrimitiveIndirect`,GPU 自己查 count。

### 4.2 共享 count buffer

```cpp
FRWBuffer CountBuffer;     // 一整块 buffer,所有 emitter 的 count 按 offset 共享
```

每个 emitter 的 `FNiagaraDataBuffer`(Phase 4)持 `GPUInstanceCountBufferOffset`——指向 CountBuffer 中它的槽位。共享比"每 emitter 一个 buffer"省 memory / descriptor。

### 4.3 `AcquireEntry / FreeEntry` 池化

```cpp
uint32 AcquireEntry();          // 分配 offset,若有 FreeEntries 复用
void FreeEntry(uint32& Offset); // 释放到 FreeEntries 池,参数置 INDEX_NONE
```

`FreeEntries: TArray<uint32>` 避免碎片——entry 数稳定时不增长 buffer。

### 4.4 Draw Indirect 生成流程

```
Phase 6 Renderer::CreateRenderThreadResources:
    AddDrawIndirect(countOffset, numIndicesPerInstance, startIndex, bStereo, bCulled)
        → 登记一条 FNiagaraDrawIndirectArgGenTaskInfo
        → 返回 entry offset(in DrawIndirectBuffer)

一帧末 UpdateDrawIndirectBuffer:
    → Dispatch FNiagaraDrawIndirectArgsGenCS
    → CS 读 TaskInfos + CountBuffer → 写 DrawIndirectBuffer

Renderer 绘制:
    MeshBatch.IndirectArgsBuffer = DrawIndirectBuffer
    MeshBatch.IndirectArgsOffset = entry offset * 5 * 4 bytes
    → RHIDrawIndexedPrimitiveIndirect
```

### 4.5 Culled counts 分支

```cpp
uint32 AcquireCulledEntry();
FRWBuffer* AcquireCulledCountsBuffer(...);
```

单独一组 count,存 view-culled 后的粒子数。Flags 里 `Flag_UseCulledCounts` 决定 draw 用哪个。

---

## 5. GPU 排序

### 5.1 为什么需要

半透明粒子必须按深度排序(near→far)才能正确混合。Mesh/Sprite emitter 数量动辄几千,每帧排序。

### 5.2 `FNiagaraGPUSortInfo` 任务描述

```cpp
struct FNiagaraGPUSortInfo {
    int32 ParticleCount;
    ENiagaraSortMode SortMode;               // ViewDepth / ViewDistance / Custom ...
    int32 SortAttributeOffset;               // 自定义属性 offset
    FShaderResourceViewRHIRef ParticleDataFloat/Half/Int SRV;
    uint32 GPUParticleCountOffset;            // 实际 count 在 CountBuffer 的 offset
    FVector ViewOrigin / ViewDirection;

    // Culling
    bool bEnableCulling;
    int32 CullPositionAttributeOffset / ...;
    TArray<FPlane, TFixedAllocator<10>> CullPlanes;

    // GPUSortManager bindings
    FGPUSortManager::FAllocationInfo AllocationInfo;
    EGPUSortFlags SortFlags;
};
```

### 5.3 `SetSortFlags` 三轴

```cpp
SortFlags = KeyGenAfterPreRender | ValuesAsInt32 |
    (bHighPrecisionKeys ? HighPrecisionKeys : LowPrecisionKeys) |
    (bTranslucentMaterial ? AnySortLocation : SortAfterPreRender);
```

- **High vs Low precision**:32-bit 或 16-bit sort key(trade-off 精度 vs 带宽)
- **AnySortLocation vs SortAfterPreRender**:半透明任意取,不透明强制 pre-render 后

### 5.4 `FNiagaraSortKeyGenCS`

4 permutation(ENABLE_CULLING × SORT_MAX_PRECISION)。输入粒子数据 SRV + view 参数 + culling 参数,输出 `(key, particleIndex)` 对 UAV + culled count。`NIAGARA_KEY_GEN_THREAD_COUNT=64`。

### 5.5 完整流水

```
Phase 6 Renderer::CreateRenderThreadResources:
    构 FNiagaraGPUSortInfo
        → Batcher::AddSortedGPUSimulation(SortInfo)
        → 注册到 FGPUSortManager

PreRender 阶段:
    GPUSortManager 回调 Batcher::GenerateSortKeys(BatchId, NumElements, Flags, KeysUAV, ValuesUAV)
        → Batcher 为每个 SortInfo dispatch FNiagaraSortKeyGenCS
        → CS 写 (key, index) 到 UAV
    GPUSortManager 自己做 radix sort
    结果 SRV 回馈给 Renderer

Renderer:
    VF.SetSortedIndices(sortedSRV, offset)
    VS 从 SortedIndices 按 SV_VertexID → 粒子索引 → 粒子数据
```

---

## 6. Vertex Factory — 粒子到像素的最后一步

### 6.1 VF 在 UE 渲染里的角色

VertexFactory 是 UE 渲染器的抽象:**把不同 vertex 数据源(mesh、粒子、样条)统一成 material shader 能读的 vertex interpolants**。材质编译时不知道 VF 细节,VS 里 `GetVertexFactoryIntermediates` 从 VF 专属 shader code 里读数据。

### 6.2 三种 Niagara VF

```
                     Sprite       Ribbon      Mesh
──────────────────── ─────────── ─────────── ───────────
UBO 数               2(stable+loose) 2        1
Mesh 顶点数据        无          无          FStaticMeshDataType
Tessellation 支持     ❌          ❌          ✅
Instancing 方式       隐式展开     索引展开      显式 SV_InstanceID
特有 SRV             CutoutGeometry TangentsAndDistances / MultiRibbonIndices / PackedPerRibbonDataByIndex —
```

### 6.3 共同数据流

所有 Niagara VF 的 VS 逻辑:

```
顶点 = particle index(SV_VertexID / SV_InstanceID)
    → SortedIndices[index](如启用排序)
    → NiagaraParticleDataFloat/Half[computed offsets]   ← 读粒子属性
    → 用 Offset(PositionOffset / VelocityOffset...)从粒子数据取属性
    → 配合 VF 专有逻辑(billboard expand / ribbon tessellation / mesh transform)
    → 输出 vertex interpolants
```

### 6.4 Half 类型 offset MSB 编码

Phase 6 讲过:`Offset |= 1 << 31` 标记 half。VF VS 里按 MSB 选择从 `NiagaraParticleDataHalf` SRV 读还是 `Float` SRV 读。零运行时分支代价。

### 6.5 `FNiagaraNullSortedIndicesVertexBuffer`

全局单例的空 SRV(1 个 int32=0)。没排序时绑它,VS 始终可以用 `SortedIndices[i]`(返回 0)—— 配合简单的"没排序就等于 identity"shader 逻辑,避免特殊分支。

### 6.6 `CheckAndUpdateLastFrame`

多 view 时(stereo、split screen、反射/折射),同一 VF 可能一帧被多次绑定。这个方法返回 true 表示本帧首次访问本 view,需要刷新 per-view 数据;false 跳过重做。性能优化。

---

## 7. Draw Indirect Args 生成

### 7.1 问题

`RHIDrawIndexedPrimitiveIndirect(IndirectArgsBuffer, offset)` 需要 indirect args buffer 里每条:

```cpp
{ uint32 IndexCountPerInstance, InstanceCount, StartIndex, BaseVertex, StartInstance };  // 5 uint32
```

但 `InstanceCount`(粒子数)在 GPU。CPU 不能直接写。

### 7.2 `FNiagaraDrawIndirectArgsGenCS` 解决

```cpp
// TaskInfos SRV: 每条 (countOffset, numIndicesPerInstance, startIndex, flags)
// InstanceCounts UAV: 共享 count buffer
// DrawIndirectArgs UAV: 输出

[numthreads(64,1,1)]
void Main(uint tid : SV_DispatchThreadID) {
    if (tid >= TaskCount) return;
    uint4 task = TaskInfos[tid];  // (countOffset, numIndices, startIndex, flags)
    uint count = InstanceCounts[task.x];  // 读粒子数
    if (task.y == -1) {  // clear task
        InstanceCounts[task.x] = 0;
    } else {
        DrawIndirectArgs[tid * 5 + 0] = task.y;      // IndexCountPerInstance
        DrawIndirectArgs[tid * 5 + 1] = count;        // InstanceCount
        DrawIndirectArgs[tid * 5 + 2] = task.z;       // StartIndex
        DrawIndirectArgs[tid * 5 + 3] = 0;            // BaseVertex
        DrawIndirectArgs[tid * 5 + 4] = 0;            // StartInstance
    }
}
```

**GPU 全自动**——CPU 不介入数字转换。

### 7.3 Task double-use

`NumIndicesPerInstance = -1` 表示"只清 counter,不生成 arg"。InstanceCountClearTasks 和 ArgGenTasks 合并到一个 dispatch,省调度开销。

### 7.4 `FNiagaraDrawIndirectResetCountsCS`

Fallback 路径,不支持 RW texture buffer 的平台(`FSupportsTextureRW permutation=0`)用纯 buffer 版。

---

## 9 条关键洞察

1. **Shader 代码在独立模块**(NiagaraShader)—— 接口/实现分离让 RHI 不拉 Niagara 主模块
2. **`FNiagaraShader` 是单一 compute shader 体系** —— 每 GPU emitter 一份,持海量 LAYOUT_FIELD 参数(~30 类)
3. **`FNiagaraDataInterfaceParamRef` 用 binary memory image 序列化**,non-virtual 设计 —— shader cache 持久化
4. **共享 Instance Count buffer** 解决"GPU count CPU 不知"—— `RHIDrawIndexedPrimitiveIndirect` + `DrawIndirectArgsGenCS` 全 GPU 自动
5. **`FNiagaraDrawIndirectArgGenTaskInfo` 双用** —— `NumIndicesPerInstance = -1` 既 clear 又 gen,同一 dispatch 处理
6. **GPU Sort 走 `FGPUSortManager`** —— Niagara 只做 key 生成(`FNiagaraSortKeyGenCS`),radix sort 由通用 manager 做
7. **`FSimulationStageMetaData` 里 MinStage/MaxStage 区间**让一条元数据覆盖 N 次迭代 —— Phase 10 Jacobi 等迭代式计算
8. **3 种 VF** 能力矩阵明确:Mesh 独占 tessellation + 需 StaticMesh 顶点数据,Sprite/Ribbon 全 SRV-based
9. **Half type offset 最高位编码** —— VS 编译期无需知道粒子数据类型,运行时按 MSB 选 SRV

---

## 自检问题(读完回答)

下面这些题需要把"shader 编译流水线 + Instance Count + Sort + VertexFactory + Draw Indirect"全部串起来。

1. **`bUsesRapidIterationParams` 进 ShaderMapId 的必要性**:这只是 1 个 bit,但任何位变就重编。回到 Phase 1 的 RapidIteration 烘焙开关——为什么烘不烘这一位会让生成的 shader 字节码不同?能不能把它放到 push constant 而不进 shader id?
2. **共享 count buffer vs per-emitter 独立 count**:Niagara 选共享 buffer + per-emitter offset。如果改用每 emitter 独立 buffer,GPU 性能(memory / descriptor / dispatch)和 CPU 性能(管理代价)各会怎么变?为什么共享是更优解?这个权衡在哪种规模下会反转?
3. **不支持 compute shader 的平台,GPU sim 怎么办**:`RHIDrawIndexedPrimitiveIndirect` + `DrawIndirectArgsGenCS` 都依赖 compute。在没有 compute 能力的平台,Niagara 是完全禁用 GPU sim,还是有降级路径?(提示:CVar / EmitterPlatformSet / `CanExecuteOnTarget`)
4. **DI 参数的"双声明"模式**:`UNiagaraDataInterface` 是 UCLASS 有虚函数,但 `FNiagaraDataInterfaceParametersCS` 必须 non-virtual + LAYOUT_FIELD 才能进 shader cache。你想给一个新 DI 加一个 GPU 参数,需要遵循什么严格的"双声明"流程?(`DECLARE_NIAGARA_DI_PARAMETER` / `IMPLEMENT_NIAGARA_DI_PARAMETER` 各做什么)
5. **`FGPUSortManager` 分工的代价**:Niagara 只生成 (key, particleIndex) 对,radix sort 由通用 manager 做。这个分工的代价是什么?(提示: 排序时机限制 / culling 集成 / 精度选择灵活度)。这个权衡为什么对 Niagara 是值得的?
6. **"零分支"的 VS 实际成本转移**:Half offset MSB 编码 + Dummy SRV 池让 VertexFactory shader 零运行时分支。这个"零"成本实际转移到了哪里?(shader 编译时间 / 运行时 SRV 数量 / 描述符堆大小 / 内存)——能算清这笔帐才算明白"零分支"的真正含义。
7. **Draw Indirect 任务双用的精巧**:`NumIndicesPerInstance = -1` 表示 clear task,合并到同一 dispatch。如果不合并,每帧需要多少额外 dispatch?这个 "用一个 magic value 复用 task slot" 在工程上有什么风险?
8. **GPU sim 的"零拷贝"边界**:数据留在 GPU 显存到渲染——但什么时候必须 readback?(提示:Light renderer / debugger / SkeletalMesh DI 的某些查询)。零拷贝是 Niagara GPU 的核心优势,但理解它的边界才能知道何时该选 CPU sim。

---

## Phase 8 留下的问题

- Phase 5 `FNiagaraGPUSystemTick` 的 `InstanceData_ParamData_Packed` 具体打包格式 → 看 cpp
- VS 具体怎么从 NiagaraParticleDataFloat SRV 按 offset 读 —— shader (.ush) 文件
- SimStages 完整机制 → **Phase 10**(本 Phase 已预告 `FSimulationStageMetaData`)
- Grid DI 的 iteration source + output destination 与 SimStages 的交互 → **Phase 10**
- Overlap compute 机制(`UseOverlapCompute`)→ Phase 5 Batcher cpp
- Readback manager 的非阻塞读回细节 → Phase 5 Batcher `GpuReadbackManagerPtr`

## 下一步预告

**Phase 9**:世界管理。`FNiagaraWorldManager` / `FNiagaraScalabilityManager` / `UNiagaraComponentPool` / `UNiagaraSettings` / `UNiagaraEffectType` / `FNiagaraPlatformSet`。偏系统架构,⭐⭐⭐ 难度。

---

## 深入阅读

### 本议题的原子页

- 源摘要(Source × 13;8.8 `NiagaraEmitterInstanceBatcher` GPU 侧已在 Phase 5.5 覆盖)
  - Shader 编译链:[[Wiki/Sources/Stock/NiagaraShared]] / [[Wiki/Sources/Stock/NiagaraShader]] / [[Wiki/Sources/Stock/NiagaraShaderType]] / [[Wiki/Sources/Stock/NiagaraShaderMap]] / [[Wiki/Sources/Stock/NiagaraScriptBase]]
  - GPU 运行时基础:[[Wiki/Sources/Stock/NiagaraGPUInstanceCountManager]] / [[Wiki/Sources/Stock/NiagaraGPUSortInfo]]
  - VertexFactory 家族:[[Wiki/Sources/Stock/NiagaraVertexFactory]] / [[Wiki/Sources/Stock/NiagaraSpriteVertexFactory]] / [[Wiki/Sources/Stock/NiagaraRibbonVertexFactory]] / [[Wiki/Sources/Stock/NiagaraMeshVertexFactory]]
  - 排序与间接绘制:[[Wiki/Sources/Stock/NiagaraSortingGPU]] / [[Wiki/Sources/Stock/NiagaraDrawIndirect]]
  - (CPU 侧 Batcher/Tick:[[Wiki/Sources/Stock/NiagaraEmitterInstanceBatcher]] 在 Phase 5)
- 实体(Entity × 5,部分类型做了合并)
  - [[Wiki/Entities/Stock/FNiagaraShader]](合并 ShaderType/ShaderMap/ScriptBase/Shared 的 4 个头文件)
  - [[Wiki/Entities/Stock/FNiagaraGPUInstanceCountManager]]
  - [[Wiki/Entities/Stock/FNiagaraGPUSort]](合并 GPUSortInfo + SortingGPU 2 文件)
  - [[Wiki/Entities/Stock/FNiagaraVertexFactory]](合并 Sprite/Ribbon/Mesh 4 文件)
  - [[Wiki/Entities/Stock/FNiagaraDrawIndirect]]

### 前置议题

- [[Readers/Niagara/Phase 4 - Niagara 的数据语言]] — `FNiagaraDataBuffer` 的 `GPUBufferFloat/Int/Half`、ParameterStore padding 映射
- [[Readers/Niagara/Phase 5 - Niagara 脚本如何跑起来]] — CPU 脚本执行、GPU Tick 打包、Batcher 主循环
- [[Readers/Niagara/Phase 6 - Niagara 粒子如何变成屏幕像素]] — Renderer 如何持 VertexFactory、Dummy SRV、GPU DynamicData
- [[Readers/Niagara/Phase 7 - 最强扩展点 Data Interface]] — DI 的 GPU shader 参数生成(`FNiagaraDataInterfaceParametersCS`)

### 相关概念

- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — GPU 侧的全貌就是本 Phase

### 下一步 / 导航

- 下一阶段:[[Readers/Niagara/Phase 9 - Niagara 的世界管理与可扩展性]] — World 级全局调度、Scalability、Pool
- 选修进阶:[[Readers/Niagara/Phase 10 - Niagara 的高级特性]] — SimStages 多 pass 和 Grid DI
- 学习路径总图:[[Wiki/Syntheses/Niagara/Niagara-learning-path]]
- 仓综合视图:[[Wiki/Overview]]

---

*本读本由 [[Claudian]] 基于 Phase 8 的 13 个头文件(合计 2500 行)综合生成,2026-04-20。commit `b6ab0dee9`。*
