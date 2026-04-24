---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, learning-path, phase-7, reader, data-interface, di]
sources: 9
aliases: [Phase 7 读本, Niagara 数据接口系统读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 7 - 最强扩展点 Data Interface

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/UE/Niagara/Niagara-learning-path]] Phase 7 的**主题读本**。一次读完掌握 DI 生态:主基类机制 + 7 种典型 DI 的能力与陷阱 + [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟|CPU/GPU 双路径]] + per-instance data 生命周期。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 7 要回答的问题

> [!question] Phase 7 要回答
> - Niagara 脚本里 `SkeletalMesh.SamplePosition()` 这种调用,运行时如何找到 C++ 实现?
> - 一个 DI 是如何**同时**支持 CPU VM 和 GPU compute shader 的?
> - 为什么 DI 有 per-instance data?它存在哪?
> - DI 如何影响 System Instance 的 TickGroup 和 Stage?

9 文件(7.1 与 Phase 5.2 同文件):

| # | 文件 | 行 | 角色 |
|---|---|---|---|
| 7.2 | `NiagaraDataInterface.h` | 890 | **主基类** + VM 绑定模板族 + Proxy 基类 |
| 7.3 | `NiagaraDataInterfaceCurve.h` | 58 | 最简单示例(float 曲线) |
| 7.4 | `NiagaraDataInterfaceCurveBase.h` | 251 | Curve 公共基类(LUT 机制) |
| 7.5 | `NiagaraDataInterfaceCamera.h` | 107 | 相机查询 + 距离排序 |
| 7.6 | `NiagaraDataInterfaceCollisionQuery.h` | 93 | 4 种碰撞查询方式 |
| 7.7 | `NiagaraDataInterfaceStaticMesh.h` | 491 | Mesh 采样(面积加权、section 过滤) |
| 7.8 | `NiagaraDataInterfaceSkeletalMesh.h` | 976 | **最复杂**:共享 skinning 缓存 |
| 7.9 | `NiagaraDataInterfaceTexture.h` | 63 | GPU only 2D 纹理采样 |
| 7.10 | `NiagaraDataInterfaceRenderTarget2D.h` | 138 | RW 类型,Phase 10 基础 |

---

## 1. DI 的三条路径

DI 机制最核心的事是**同一个用户声明的函数**在三处被实现/使用:

```
编译期:GetFunctions() → 注册可用函数签名(FNiagaraFunctionSignature)
    ↓ 编译器生成字节码,函数调用变成 DI index + 函数 index
运行时 CPU:GetVMExternalFunction(BindingInfo) → 返回 lambda 封装
    ↓ VM 遇 DI 调用,查 FunctionTable,执行 lambda
运行时 GPU:GetParameterDefinitionHLSL + GetFunctionHLSL → 拼进 compute shader
    ↓ compute shader 直接调用生成的 HLSL 函数
```

这是 DI 同时支持 CPU 和 GPU 的核心——**两条完全独立的代码生成路径**,共享同一个用户 API(`FNiagaraFunctionSignature`)。

## 2. Per-Instance Data 机制

不是所有 DI 都需要 per-instance 数据。Curve DI 是无状态的(LUT 在 Asset 级共享),Camera DI 需要缓存本帧相机位置(per SystemInstance),SkeletalMesh 需要缓存 skinning 结果(可跨 SystemInstance 共享)。

```
FNiagaraSystemInstance {
    TArray<uint8, TAlignedHeapAllocator<16>> DataInterfaceInstanceData;  // 大 blob
    TArray<TPair<TWeakObjectPtr<UNiagaraDataInterface>, int32>> DataInterfaceInstanceDataOffsets;
}
```

初始化时:
1. SystemInstance 遍历所有 DI,调 `PerInstanceDataSize()` 算出总大小
2. 分配 blob(16-byte 对齐,DI 可能用 SIMD)
3. 每个 DI 按 offset 调 `InitPerInstanceData(PerInstanceData, this)` 写自己那块

每帧:
- `PerInstanceTick` / `PerInstanceTickPostSimulate`(如有 `HasPreSimulateTick / HasPostSimulateTick`)
- GPU 需要的话 `ProvidePerInstanceDataForRenderThread` 打包送 RT → `Proxy::ConsumePerInstanceDataFromGameThread`

GT→RT 的数据大小通过 `PerInstanceDataPassedToRenderThreadSize`(**必须 16-byte 对齐**,代码注释明确)声明。

---

## 3. Curve DI — 最简单的示例

### 3.1 LUT 的意义

`FRichCurve` 是 UE 的样条曲线,运行时 `Eval(X)` 是 O(log N) 的段查找 + 三次插值。每粒子每帧都调太贵。

**LUT 烘焙**:编辑器调 `UpdateLUT(128)` 把曲线采样 128 次存 `TArray<float> ShaderLUT`。运行时 `SampleCurve(X)`:

```cpp
template<typename UseLUT>
float SampleCurveInternal(float X) {
    if constexpr (UseLUT::Value) {
        float T = NormalizeTime(X);          // 0-1 索引
        int idx = T * LUTNumSamplesMinusOne;
        float frac = ...;
        return Lerp(LUT[idx], LUT[idx+1], frac);  // 线性插值
    } else {
        return Curve.Eval(X);                 // 慢但精确
    }
}
```

### 3.2 TCurveUseLUTBinder 的编译期分派

```cpp
template<typename NextBinder>
struct TCurveUseLUTBinder {
    template<typename... ParamTypes>
    static void Bind(UNiagaraDataInterface* Interface, ...) {
        if (CurveInterface->bUseLUT)
            NextBinder::Bind<ParamTypes..., TIntegralConstant<bool, true>>(...);
        else
            NextBinder::Bind<ParamTypes..., TIntegralConstant<bool, false>>(...);
    }
};
```

运行时**一次**判 `bUseLUT`,生成对应模板特化的 lambda。之后 VM 调用**零分支**。

### 3.3 Expose to Materials

```cpp
UPROPERTY() uint32 bExposeCurve : 1;
UPROPERTY() UTexture2D* ExposedTexture;
```

Niagara 给材质提供"通用曲线"——`UpdateExposedTexture` 把 LUT 写成 `UTexture2D`,材质采样器可绑定。

---

## 4. Camera DI — Per-Instance 样例

```cpp
struct FCameraDataInterface_InstanceData {
    FVector CameraLocation;
    FRotator CameraRotation;
    float CameraFOV;

    TQueue<FDistanceData, EQueueMode::Mpsc> DistanceSortQueue;
    TArray<FDistanceData> ParticlesSortedByDistance;
};
```

**MPSC 队列**:多个 VM worker 并行写(Multi-Producer),GT tick 时单线程消费(Single-Consumer)排序——最常见的 producer-consumer 模式。

### TickGroup 约束

```cpp
HasTickGroupPrereqs() → true;
CalculateTickGroup(const void* PerInstanceData) → ETickingGroup;
RequiresEarlyViewData() → true;
```

这些返回让 `FNiagaraSystemInstance::UpdatePrereqs` 把 System 移到合适的 TickGroup(view 数据就绪之后)。

### `bRequireCurrentFrameData = false` 的取舍

默认 true——保证脚本看到的相机是本帧的。关掉换性能:用上一帧数据 → 可以提前 tick(不等 view) → GT 总负担降低,但相机快速切换时特效和相机差一帧。

---

## 5. CollisionQuery DI — 4 种路径

| 方法 | 平台 | 性能 | 精度 | 适用 |
|---|---|---|---|---|
| `PerformQuerySyncCPU` | CPU | ⚠️ 阻塞 | 精确 | 少量粒子 |
| `PerformQueryAsyncCPU` | CPU | 高吞吐 | 1 帧延迟 | 大量粒子 |
| `QuerySceneDepth` | GPU | 快 | 只屏幕内 | 屏幕特效 |
| `QueryMeshDistanceField` | GPU | 较快 | 全场景 DF | 精确需求 + DF 开启 |

`HasPreSimulateTick` + `HasPostSimulateTick` 双开——发起异步查询 + 读回结果。

---

## 6. StaticMesh DI — 面积加权采样

### 6.1 为什么要面积加权

Mesh 里不同三角形面积差异大。均匀按"三角形索引"随机选会让小三角形和大三角形概率相同——小三角形密度大。面积加权让粒子在 mesh 表面**均匀分布**。

### 6.2 Alias 方法

`FStaticMeshFilteredAreaWeightedSectionSampler : FWeightedRandomSampler` —— UE 自带 **Walker's Alias Method**,O(1) 面积加权采样。

GPU 侧 `FStaticMeshGpuSpawnBuffer::SectionInfo`:

```cpp
struct SectionInfo {
    uint32 FirstIndex;
    uint32 NumTriangles;
    float Prob;      // 本 section 的概率
    uint32 Alias;    // alias 的 section 索引
};
```

上传到 GPU SRV,compute shader 按 alias 方法采样。

### 6.3 Section 过滤

```cpp
struct FNDIStaticMeshSectionFilter {
    TArray<int32> AllowedMaterialSlots;  // 白名单
};
```

只在指定 material slot 对应的 sections 里采样——用于"只从岩石部分 spawn,不从植被部分"。

### 6.4 3 种 Source 模式 + 1 个 Fallback

```cpp
enum ENDIStaticMesh_SourceMode : uint8 {
    Default,           // Source → AttachParent → DefaultMesh(降级)
    Source,            // 只用显式 Source
    AttachParent,      // 只用 parent actor/component
    DefaultMeshOnly,   // 只用 DI 配置的 DefaultMesh
};
```

Phase 2 `UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMesh` 就是设置 Source 路径。

### 6.5 CPU Access 陷阱

Mesh 没开 CPU Access(UE Asset 属性)→ `bMeshAllowsCpuAccess=false` → CPU 只能做 socket、bounds 之类,**三角形/顶点采样 only GPU**。这是常见坑。

---

## 7. SkeletalMesh DI — 最复杂的骨骼蒙皮

### 7.1 Skinning 的三模式

```cpp
enum class ENDISkeletalMesh_SkinningMode : uint8 {
    None,          // 引用 pose,按需算
    SkinOnTheFly,  // 骨骼前置,顶点按需
    PreSkin,       // 全部前置
};
```

- **None**:不动骨骼。粒子只需"静态 mesh"一样用。省到极致
- **SkinOnTheFly**:骨骼矩阵前置算,顶点读到才 skin。**默认选**
- **PreSkin**:全顶点前置算。访问大量顶点时最快,但内存最高

### 7.2 共享机制

```
FNDI_SkeletalMesh_GeneratedData (世界级全局,住 WorldManager)
    ↓
TMap<USkeletalMeshComponent, TSharedPtr<FSkeletalMeshSkinningData>>
    ↓
FSkeletalMeshSkinningData (引用计数共享)
    ├─ BoneRefToLocals[2]                           ← 双缓冲(速度计算)
    ├─ ComponentTransforms[2]
    ├─ TArray<FLODData> LODData
    │       └─ FLODData { SkinnedCPUPositions[2], SkinnedTangentBasis, PreSkinnedVertsUsers }
    ├─ BoneMatrixUsers / TotalPreSkinnedVertsUsers
    └─ FRWLock RWGuard
```

**多个 DI 指向同一 MeshComponent → 共享 SkinningData**。引用计数 + RW lock。一个 mesh 被 N 个 DI 用,skinning 只做一次。

### 7.3 Usage Handle

```cpp
struct FSkeletalMeshSkinningDataUsage {
    int32 LODIndex;
    bool bUsesBoneMatrices;
    bool bUsesPreSkinnedVerts;
};
```

每个 DI 注册一个 Usage(要用哪 LOD,要骨骼/顶点)→ GeneratedData 判断要真正算什么——**按最大需求决策**。

### 7.4 速度计算靠双缓冲

`BoneRefToLocals[2]` 和 `SkinnedCPUPositions[2]` 是 `[CurrIndex]` 和 `[CurrIndex^1]` —— `CurrIndex` 每 tick 翻转。VM 读当前帧,可同时读前一帧的同一数据算速度。

和 Phase 3 `FNiagaraSystemInstance::GlobalParameters[2]` 是**不同的双缓冲**(前者服务 skinning 速度,后者服务 GT/RT 竞态)。

### 7.5 Filter Mode

```cpp
enum class ENDISkeletalMesh_FilterMode : uint8 {
    None, SingleRegion, MultiRegion
};
```

UE 的 `FSkeletalMeshSamplingRegion` 是资产级 "一组骨骼附近三角形" 的预定义。Multi region 用 `FSkeletalMeshSamplingRegionAreaWeightedSampler` 跨区域加权。

典型用途:"只在头部 spawn 血液粒子"——定义 head region,DI 只从那里采样。

---

## 8. Texture DI — GPU-only 的极简示范

整个 DI 63 行,单字段 `UTexture* Texture`,3 个 VM 函数。`CanExecuteOnTarget` 只返回 `GPUComputeSim`。

**架构启示**:DI 可以非对称——只支持一种 SimTarget。

### `GetParameterDefinitionHLSL` 生成形如:

```hlsl
Texture2D {TextureName}_{DIName};
SamplerState {SamplerName}_{DIName};
uint2 {DimensionsBaseName}_{DIName};
```

`{DIName}` 是编译器为这个 DI 实例生成的唯一后缀——不同 emitter 同名 DI 区分开。

---

## 9. RenderTarget2D DI — RW 的第一个示范

### 9.1 为什么继承 `UNiagaraDataInterfaceRWBase` 不是 `UNiagaraDataInterface`

RW DI(Phase 10 Simulation Stages 基础)需要**额外接口**:
- `GetElementCount(SystemInstanceID) → FIntVector` —— SimStage dispatch 的 thread 数
- `AsIterationProxy()` —— iteration proxy 转换

普通 DI 基类没有这些。`UNiagaraDataInterfaceRWBase` 是 Phase 10 会详展的中间类。

### 9.2 RT 的双缓冲 InstanceData

```cpp
struct FRenderTarget2DRWInstanceData_GameThread {
    FIntPoint Size;
    ETextureRenderTargetFormat Format;
    UTextureRenderTarget2D* TargetTexture;
    FNiagaraParameterDirectBinding<UObject*> RTUserParamBinding;
};

struct FRenderTarget2DRWInstanceData_RenderThread {
    FIntPoint Size;
    FTextureReferenceRHIRef TextureReferenceRHI;
    FUnorderedAccessViewRHIRef UAV;   // ← 关键
};
```

GT 结构里是 `UTextureRenderTarget2D*`,RT 结构里是 `RHI + UAV`——跨线程必要的形态转换。

### 9.3 两种 RT 来源

1. **User 参数提供**:`RenderTargetUserParameter` 绑定 User.RTRef,外部设置
2. **DI 自管理**:`ManagedRenderTargets TMap<uint64, UTextureRenderTarget2D*>`,DI 按 SystemInstanceID 内部创建

后者让 DI "自举"——没 BP 提供 RT 也能工作。

### 9.4 Phase 10 预告

`PostSimulate` 在 Proxy 里覆盖,SimStages 完成后回调。Grid 模拟典型流程:

```
Stage 0:Advect(粒子推进)
Stage 1:Projection(压力投影)
Stage 2:Render(写入 RT)
PostSimulate:清 UAV / swap
```

Phase 10 完整展开。

---

## 9 条关键洞察

1. **DI 的本质是"脚本 API 扩展"**。想让脚本读/写某种外部数据,就定义一个 DI 子类
2. **三路代码生成**(VM register / C++ lambda / HLSL)让同一 API 跨 CPU/GPU 透明
3. **Per-Instance Data blob** 在 SystemInstance 里 16-byte 对齐分配,DI 按 offset 取
4. **VM 绑定模板族**(TNDIBinder 系列 + `DEFINE_NDI_FUNC_BINDER` 宏)让子类用声明式语法注册函数,编译器生成 dispatch 代码
5. **Curve DI 的 LUT + bExposeCurve** 体现 "预计算 + 跨系统复用" 思路——不止给脚本,也给材质
6. **SkeletalMesh DI 的共享 SkinningData** 是典型的 "世界级缓存 + 引用计数 + RWLock" 三件套,跨 SystemInstance 复用
7. **CPU Access 是 Mesh DI 的常见陷阱**——没开就只能 GPU 采样三角形/顶点
8. **TickGroup Prereqs** 让 DI 能把 System Instance 移到正确的 tick 阶段(Camera 要晚 tick,Physics 查询要早)
9. **RW DI 是 Phase 10 的桥**——RenderTarget2D 已经展示模式:双缓冲 InstanceData + GetElementCount + PostSimulate 钩子

---

## 自检问题(读完回答)

下面这些题需要把"DI 三路代码生成 + per-instance 数据 + 7 种典型 DI 的能力矩阵"全部串起来。

1. **CPU/GPU 实现对齐的"隐患"**:DI 同时支持 CPU(C++ lambda)和 GPU(HLSL 代码生成),共享同一个 `FNiagaraFunctionSignature`。如果某天 CPU 实现里有 bug,导致同一函数 CPU/GPU 结果不一致,Niagara 架构上提供了什么自动检测机制?(提示: 实际上**没有**——这恰恰是开放问题)。这个事实如何影响你写一个新 DI 时的测试策略?
2. **LUT 特化的反例**:Curve DI 用编译期模板分派(`TCurveUseLUTBinder`)生成 `bUseLUT=true/false` 两种特化,运行时零分支。但代价是函数生成翻倍。在什么场景下你**应该**保留运行时分支(不开 LUT 特化),反而更好?
3. **SkeletalMesh 共享 SkinningData 的回收链**:`BoneRefToLocals[2]` 双缓冲 + 引用计数。50 个 instance 共享同一 mesh,某个 instance 中途销毁,会触发什么具体的资源回收链?如果在 instance 销毁的同一帧又有 instance 创建并复用 SkinningData,RWLock 怎么保证不出错?
4. **CollisionQuery 4 路径的取舍**:为"霰弹枪散弹击中地形"特效选哪种碰撞查询?——CPU Sync 的"延迟成本"具体是多少?CPU Async 的"1 帧延迟"会让散弹的命中视觉上滞后吗?为什么 GPU 路径(SceneDepth / DistanceField)不能完全替代 CPU?
5. **`UNiagaraDataInterfaceRWBase` 的多余接口**:RT2D DI 不继承 `UNiagaraDataInterface` 而继承 `UNiagaraDataInterfaceRWBase`,多了 `GetElementCount` / `AsIterationProxy()`。这两个接口对"被 SimStage 当迭代源"为什么是必需的?(提示:回想 `IterationSource = DataInterface` 模式下 thread 数怎么定)
6. **CPU Access 关闭 + Scalability LOD 降级**:StaticMesh 没开 CPU Access → 三角形采样只 GPU。如果某低端机进一步降级到完全没 GPU 模拟(全部 CPUSim),这个 mesh 还能用吗?Niagara 怎么响应——静默失败、报错、降级用 socket?
7. **MPSC 队列在 Camera DI 的存在**:Camera DI 的 `DistanceSortQueue` 用 MPSC——为什么是 MPSC 而不是 SPSC?谁在并发 push?谁单线程消费?如果改成普通 `TArray` + lock,在 50 实例场景下吞吐会降多少量级?
8. **三路代码生成的实现负担对比**:写一个新 DI,CPU 路径(GetVMExternalFunction 返 lambda)和 GPU 路径(GetParameterDefinitionHLSL + GetFunctionHLSL 拼字符串)各需要写多少代码?为什么 GPU 路径"看起来更原始"(手拼 HLSL 字符串)的设计 Niagara 没改进——技术债还是有意?

---

## Phase 7 留下的问题

- `UNiagaraDataInterface` 后 490 行(本次未扒)→ 按需 offset 读
- `UNiagaraDataInterfaceRWBase` 完整接口 → **Phase 10**
- `FNiagaraDataInterfaceProxyRW` + SimStages 交互 → **Phase 10**
- Grid2D / Grid3D / NeighborGrid 具体实现 → **Phase 10**
- `FStaticMeshGpuSpawnBuffer` 的 GPU resource 生命周期 → 按需扒
- GPU 侧 shader 绑定机制(`FNiagaraDataInterfaceParametersCS` 特化)的 LAYOUT_FIELD 模式 → 已在 Phase 5.2 覆盖,具体实例化靠子类

## 下一步预告

**Phase 8**:GPU 模拟。14 文件的最大 Phase —— shader 编译 + 排序 + GPU 实例计数 + vertex factory。难度 ⭐⭐⭐⭐⭐。

---

## 深入阅读

### 本议题的原子页

- 源摘要(Source × 9;7.1 `NiagaraDataInterfaceBase` 已在 Phase 5.2 覆盖)
  - 运行时基类:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterface]]
  - Curve 家族:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceCurveBase]] / [[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceCurve]]
  - 场景数据:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceCamera]] / [[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceCollisionQuery]]
  - Mesh 家族:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceStaticMesh]] / [[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceSkeletalMesh]]
  - 纹理家族:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceTexture]] / [[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceRenderTarget2D]]
  - (编译期基类 [[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceBase]] 在 Phase 5)
- 实体(Entity × 8,Curve 合并 Base)
  - [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterface]]
  - [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceCurve]]
  - [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceCamera]]
  - [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceCollisionQuery]]
  - [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceStaticMesh]]
  - [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceSkeletalMesh]]
  - [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceTexture]]
  - [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceRenderTarget2D]]

### 前置议题

- [[Readers/UE/Niagara/Phase 2 - Component 层的五职责]] — FunctionLibrary 里 Component 覆盖 DI 的重型 API
- [[Readers/UE/Niagara/Phase 3 - Niagara 的心脏]] — SystemInstance 的 `DataInterfaceInstanceData` blob 布局
- [[Readers/UE/Niagara/Phase 4 - Niagara 的数据语言]] — ParameterStore 里 DI 的存储
- [[Readers/UE/Niagara/Phase 5 - Niagara 脚本如何跑起来]] — DI 基类 + VM Dispatch + GPU Shader Parameters 首次出场

### 相关概念

- [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]] — CPU DI(C++ lambda)vs GPU DI(HLSL 代码生成)的双路径根源

### 下一步 / 导航

- 下一阶段:[[Readers/UE/Niagara/Phase 8 - Niagara 的 GPU 模拟管线]] — Shader 编译 + GPU 实例计数 + VertexFactory
- 后续深入:[[Readers/UE/Niagara/Phase 10 - Niagara 的高级特性]] — RW DI 四钩子 + Grid2D/3D/NeighborGrid3D
- 学习路径总图:[[Wiki/Syntheses/UE/Niagara/Niagara-learning-path]]
- 仓综合视图:[[Wiki/Overview]]

---

*本读本由 [[Claudian]] 基于 Phase 7 的 9 个头文件(合计 2998 行,大文件扒头部)综合生成,2026-04-20。commit `b6ab0dee9`。*
