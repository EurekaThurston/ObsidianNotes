---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, learning-path, phase-6, reader, renderer, rendering]
sources: 10
aliases: [Phase 6 读本, Niagara 渲染系统读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 6 - Niagara 粒子如何变成屏幕像素

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 6 的**主题读本**。一次读完掌握 Niagara Renderer 家族:[[Wiki/Concepts/UE4/UE4-资产与实例|Asset/Instance]] 对偶、GT→RT 数据流、4 种具体类型(Sprite/Ribbon/Mesh/Light)的能力边界与陷阱。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 6 要回答的问题

> [!question] Phase 6 要回答
> - 粒子数据(Phase 4 的 DataSet)怎么变成屏幕上的像素?
> - 一个 Emitter 可以有几种渲染方式?它们的机制有多大差异?
> - Properties 和 Renderer 为什么分两个类?
> - CPU sim 和 GPU sim 在渲染路径上有什么不同?
> - 为什么 Ribbon 和 Light 是 CPU only?

10 文件按对偶关系分组:

| 类型 | Properties(Asset) | Renderer(运行时) |
|---|---|---|
| 基类 | `NiagaraRendererProperties.h` (259) | `NiagaraRenderer.h` (144) |
| Sprite | `NiagaraSpriteRendererProperties.h` (316) | `NiagaraRendererSprites.h` (102) |
| Ribbon | `NiagaraRibbonRendererProperties.h` (361) | `NiagaraRendererRibbons.h` (103) |
| Mesh | `NiagaraMeshRendererProperties.h` (288) | `NiagaraRendererMeshes.h` (74) |
| Light | `NiagaraLightRendererProperties.h` (98) | `NiagaraRendererLights.h` (31) |

---

## 1. Asset/Instance 对偶

Phase 0 的心智模型贯穿始终:Asset 层是**配置**,Runtime 层是**执行**。

```
Asset 层(UObject,住磁盘)          Runtime 层(非 UObject,住内存)
────────────────────────────────    ────────────────────────────────
UNiagaraRendererProperties          FNiagaraRenderer
    ↓ 被 UNiagaraEmitter 持有           ↓ 被 FNiagaraSceneProxy 持有
    ↓ CreateEmitterRenderer() 工厂    ↓ 每 EmitterInstance 一个
    ↓                                  ↓
具体子类对:                          具体子类对:
  USpriteRendererProperties     ←→    FNiagaraRendererSprites
  URibbonRendererProperties     ←→    FNiagaraRendererRibbons
  UMeshRendererProperties       ←→    FNiagaraRendererMeshes
  ULightRendererProperties      ←→    FNiagaraRendererLights
```

**一个 Emitter 可以有多个 Renderer**。比如火焰 emitter 可能同时有 Sprite(火苗)+ Light(照明)+ Mesh(火星碎片)三个 renderer,每帧各自独立渲染。

Properties 和 Renderer 分开的**三个理由**:
1. **Asset 层要序列化**(UObject + UPROPERTY),Runtime 层要性能(non-UObject)
2. **1-N 关系**:一个 Asset Properties 对应 N 个 EmitterInstance 的 Renderer
3. **GT/RT 隔离**:Runtime Renderer 在 GT 构造,但 `DynamicDataRender` 在 RT 改,需要专门的 `SetDynamicData_RenderThread` 跨线程设置

---

## 2. Properties 基类的职责

`UNiagaraRendererProperties`(ABSTRACT UCLASS)四个核心字段:

```cpp
FNiagaraPlatformSet Platforms;                             // Scalability 启用平台
int32 SortOrderHint;                                       // 同 Emitter 内渲染顺序
bool bIsEnabled;                                           // 运行时开关
bool bMotionBlurEnabled;                                   // 运动模糊

TArray<const FNiagaraVariableAttributeBinding*> AttributeBindings;  // 绑定哪些粒子属性
```

四个纯虚方法:

```cpp
FNiagaraRenderer* CreateEmitterRenderer(FeatureLevel, Emitter, Component);  // 工厂
FNiagaraBoundsCalculator* CreateBoundsCalculator();                          // 算 bounds
void GetUsedMaterials(...);
void GetRendererWidgets(...);  // editor UI
```

### 2.1 `FNiagaraRendererLayout` 的 GT+RT 双 copy

```cpp
struct FNiagaraRendererLayout {
    TArray<FNiagaraRendererVariableInfo> VFVariables_GT;  // GT 可改
    TArray<FNiagaraRendererVariableInfo> VFVariables_RT;  // RT 只读

    void Finalize();  // GT → RT 拷贝
    TConstArrayView<FNiagaraRendererVariableInfo> GetVFVariables_RenderThread() const;
};
```

Finalize 时拷贝——GT 改不影响 RT 当前帧。经典的"双缓冲 layout"模式。

### 2.2 `FNiagaraRendererVariableInfo` 的编码

```cpp
int32 GetGPUOffset() const {
    int32 Offset = GbEnableMinimalGPUBuffers ? GPUBufferOffset : DatasetOffset;
    if (bHalfType) Offset |= 1 << 31;  // 最高位标记 half
    return Offset;
}
```

`GbEnableMinimalGPUBuffers` CVar:开启后只上传 VF 需要的变量(不是整个 DataSet),省带宽。`bHalfType` 压到 offset 最高位——shader 侧按这个位判断数据类型。

---

## 3. Renderer 运行时基类

`FNiagaraRenderer`(非 UObject)核心流程三阶段:

```
GT 阶段:
    GenerateDynamicData(SceneProxy, Properties, Emitter)
        → 读 EmitterInstance 的 DataSet + RendererBindings
        → 快照本帧数据到 FNiagaraDynamicDataBase 子类
        → 返回 dynamic data 指针

GT→RT 桥:
    SetDynamicData_RenderThread(DynamicData)
        → ENQUEUE_RENDER_COMMAND 把 DynamicData 送到 RT
        → RT 存到 DynamicDataRender 字段

RT 阶段:
    GetDynamicMeshElements(Views, ViewFamily, VisMap, Collector, SceneProxy)
        → 从 DynamicDataRender 构 FMeshBatch
        → 提交 FMeshElementCollector

    (或 Light 专有)
    GatherSimpleLights(OutParticleLights)
        → 从 DynamicDataRender 填 FSimpleLightArray
```

### 3.1 `FNiagaraDynamicDataBase` 的 union 技巧

```cpp
struct FNiagaraDynamicDataBase {
    FMaterialRelevance MaterialRelevance;
    ENiagaraSimTarget SimTarget;

    union {
        FNiagaraDataBuffer* CPUParticleData;
        FNiagaraComputeExecutionContext* GPUExecContext;
    } Data;
};
```

CPU sim → union 存 DataBuffer 指针;GPU sim → union 存 ComputeContext 指针。`GetParticleDataToRender` 按 SimTarget 选择读哪个。

具体子类扩展这个基类加 renderer-specific 数据(如 `FNiagaraDynamicDataSprites` 加 cutout 数据,`FNiagaraDynamicDataRibbon` 加排序后的 segment 数组)。

### 3.2 空 SRV 池

Niagara VF 有多个 SRV slot(Float / Float2 / Float4 / Int / UInt / Texture2D / Half 等)。某次不用但 shader 必须绑某个 slot 时,绑到**空 SRV**:

```cpp
NIAGARA_API static FRHIShaderResourceView* GetDummyFloatBuffer();
NIAGARA_API static FRHIShaderResourceView* GetDummyFloat2Buffer();
NIAGARA_API static FRHIShaderResourceView* GetDummyFloat4Buffer();
// ... 9 种 dummy buffer
```

共享池,进程生命周期。

---

## 4. Sprite Renderer — 最复杂的通用型

### 4.1 17 个 VF slot

```cpp
enum ENiagaraSpriteVFLayout::Type {
    Position, Color, Velocity, Rotation, Size, Facing, Alignment, SubImage,
    MaterialParam0, MaterialParam1, MaterialParam2, MaterialParam3,
    CameraOffset, UVScale, MaterialRandom, CustomSorting, NormalizedAge,
    Num  // = 17
};
```

Properties 有对应 15+ 个 `FNiagaraVariableAttributeBinding` 字段。Runtime Renderer 缓存 `VFBoundOffsetsInParamStore[17]` —— 每个 slot 在 emitter 参数存储里的 offset。

### 4.2 5 种 FacingMode

| Mode | 说明 |
|---|---|
| `FaceCamera` | 朝摄像机原点,UP 对齐 |
| `FaceCameraPlane` | 完全平行摄像机平面 |
| `CustomFacingVector` | 朝 Particles.SpriteFacing |
| `FaceCameraPosition` | 朝摄像机位置,不依赖旋转(VR 稳定) |
| `FaceCameraDistanceBlend` | 远距离 FaceCameraPosition,近距离 FaceCamera |

**VR 建议 FaceCameraPosition**:摄像机每帧抖动少量,`FaceCamera` 会让粒子跟着抖;`FaceCameraPosition` 只看位置不看旋转,粒子稳定。

### 4.3 Cutout:半透明 overdraw 优化

半透明材质的 sprite 默认是矩形,中间 alpha=0 的区域仍然参与 overdraw。Cutout 系统生成贴合不透明区域的 4/8 多边形:

```cpp
#if WITH_EDITORONLY_DATA
UTexture2D* CutoutTexture;
TEnumAsByte<ESubUVBoundingVertexCount> BoundingMode;  // 4 or 8
float AlphaThreshold;
void CacheDerivedData();  // 离线生成 bounding geometry
#endif

// Runtime:
FNiagaraCutoutVertexBuffer CutoutVertexBuffer;
int32 NumCutoutVertexPerSubImage;
```

Editor 保存时 `CacheDerivedData` 计算几何,存到 `FSubUVDerivedData DerivedData`。Runtime 只读。

### 4.4 `bGpuLowLatencyTranslucency`

GPU sim 情况下,tick 可能和渲染存在一帧 latency(tick 在 PostOpaqueRender 阶段的话)。半透明粒子如果需要**当前帧**数据(与不透明物体完美共存),开这个开关走低延迟通路——`FNiagaraDynamicDataBase::GetParticleDataToRender(bIsLowLatencyTranslucent=true)` 返回 `TranslucentDataToRender`(如果有),否则回退到 `DataToRender`。

---

## 5. Ribbon Renderer — 有序连接 + tessellation

### 5.1 为什么 CPU only

Ribbon 依赖**跨粒子连接**——给定 `RibbonId + LinkOrder`,按顺序把粒子连成条带。CPU sim 里 VM 能访问整个 DataSet,排序+索引生成直接做;GPU sim 里每个 thread 只处理一个粒子,跨粒子访问难。

UE 4.26 Niagara Ribbon 没实现 GPU 路径,所以 `IsSimTargetSupported(CPUSim)` only。

### 5.2 Multi-ribbon per emitter

```cpp
FNiagaraVariableAttributeBinding RibbonIdBinding;          // 哪条 ribbon
FNiagaraVariableAttributeBinding RibbonLinkOrderBinding;   // 同 ribbon 内顺序
```

两个绑定让一个 emitter 同时画多条独立 ribbon(例:多条闪电同时)。

### 5.3 自适应 Tessellation

头文件里只有配置字段——实际的"每帧统计→下帧 tessellation factor"逻辑在 `NiagaraRendererRibbons.cpp` 内部私有,不对外暴露:

```cpp
// NiagaraRibbonRendererProperties.h L237-263
UPROPERTY(EditAnywhere, Category = "Tessellation")
float CurveTension;                                // 0-1,曲线张力
ENiagaraRibbonTessellationMode TessellationMode;   // Automatic / Custom / Disabled
int32 TessellationFactor;                          // 1-16,最大加细倍数
bool bUseConstantFactor;                           // 勾上则不根据曲率自适应
float TessellationAngle;                           // 0-180 度,触发加细的角度阈值
bool bScreenSpaceTessellation;                     // 屏幕空间自适应
```

Automatic 模式下引擎每帧基于粒子曲率+twist 动态选择实际 factor(≤ `TessellationFactor`);Custom 模式则固定为 `TessellationFactor`。> [!warning] 之前版本的读本在此列过 5 个 `mutable float` 运行时统计字段(TessellationCurvature/TwistAngle/…),那是幻觉——**4.26 头文件里没有这些字段**,相关统计在 `.cpp` 的匿名结构里。

### 5.4 双 UV channel

```cpp
FNiagaraRibbonUVSettings UV0Settings / UV1Settings;
```

每 UV 独立:
- `LeadingEdgeMode / TrailingEdgeMode`(SmoothTransition / Locked)
- `DistributionMode`(ScaledUniformly / ScaledUsingRibbonSegmentLength / TiledOverRibbonLength)
- `TilingLength / Offset / Scale`
- `bEnablePerParticleUOverride / bEnablePerParticleVRangeOverride` —— 每粒子 UV 覆盖

---

## 6. Mesh Renderer — StaticMesh Instancing

### 6.1 核心:UStaticMesh + 每粒子 Transform

```cpp
UStaticMesh* ParticleMesh;                 // 必须 "Niagara Mesh Particles"
bool bOverrideMaterials;
TArray<FNiagaraMeshMaterialOverride> OverrideMaterials;
```

**14 个 VF slot**:比 Sprite 少 3 个(没有 Rotation / Alignment / CustomSorting?但有 Transform)。`Transform` slot 让每粒子有完整 4x4 矩阵的自由度。

### 6.2 Facing 4 模式 + Axis 锁定

```cpp
enum ENiagaraMeshFacingMode {
    Default,           // 粒子 local X 轴 + Particles.Transform
    Velocity,          // X 轴对齐 Velocity
    CameraPosition,    // X 轴指向摄像机位置
    CameraPlane,       // X 轴指向视平面
};

bool bLockedAxisEnable;
FVector LockedAxis;
ENiagaraMeshLockedAxisSpace LockedAxisSpace;  // Simulation / World / Local
```

### 6.3 LOD 所有粒子共享

```cpp
int32 MeshMinimumLOD = 0;
int32 GetLODIndex() const;  // 基于相机距离
```

⚠️ **per-instance LOD 不支持**——所有粒子走同 LOD。场景里粒子分布深度大时可能出现"远处全高 LOD"问题。

### 6.4 `FNiagaraMeshMaterialOverride`

```cpp
USTRUCT()
struct FNiagaraMeshMaterialOverride {
    UMaterialInterface* ExplicitMat;
    FNiagaraUserParameterBinding UserParamBinding;  // User 参数优先
};
```

两种覆盖方式,User binding 优先。Phase 2 `UNiagaraFunctionLibrary::OverrideSystemUserVariableStaticMesh` 设置的就是 `UserParamBinding` 路径。

---

## 7. Light Renderer — 不产 mesh batch 的特殊类

### 7.1 为什么不产 mesh batch

Light 不是"画出来的几何体",它是**场景光源系统的一项注册**。UE 的 `FSimpleLightArray` 是场景渲染器在 deferred pass 处理的 simple light 集合,只需要 `FSimpleLightEntry`(颜色/强度)+ `FSimpleLightPerViewEntry`(位置/半径)即可。

FNiagaraRendererLights 的 override 非常少:

```cpp
virtual FPrimitiveViewRelevance GetViewRelevance(...) const override;  // 标记有 light
virtual FNiagaraDynamicDataBase* GenerateDynamicData(...) override;    // 填 light 数据
virtual void GatherSimpleLights(FSimpleLightArray& OutLights) const override;  // 提交
// 不 override GetDynamicMeshElements !
```

### 7.2 开销警告

```cpp
UPROPERTY(EditAnywhere, Category = "Light Rendering")
uint32 bAffectsTranslucency : 1;
```

代码注释直接警告:"create only a few particle lights at most, and the smaller they are, the less they will cost"。半透明+多光源对 fill rate 毁灭性打击。

### 7.3 CPU only 的原因

GPU 粒子数据在 GPU 侧,`GatherSimpleLights` 需要 CPU readback——要么 stall,要么 latency。4.26 时代设计选择:不支持 GPU light renderer。

---

## 8 条关键洞察

1. **Renderer 是 Properties + Runtime 的对偶**——Asset 侧可序列化,Runtime 侧非 UObject 追求性能
2. **一个 Emitter 可多个 Renderer**。火焰可以是 Sprite + Light + Mesh 叠加
3. **4 种 Renderer 的能力差异**:
   - Sprite:通用,CPU+GPU,17 个 VF slot,支持 Cutout 和 VR 稳定 facing
   - Ribbon:CPU only,多条 per emitter,自适应 tessellation
   - Mesh:CPU+GPU,StaticMesh 实例化,14 个 slot,per-instance LOD 不支持
   - Light:CPU only,不产 mesh batch,走 simple light 系统,半透明开销大
4. **`FNiagaraDynamicDataBase` 的 union** 优雅地处理 CPU/GPU 双路径的数据源
5. **GT+RT 双 copy layout**(`FNiagaraRendererLayout::VFVariables_GT/_RT`)+ `SetDynamicData_RenderThread` 跨线程钩子 = Niagara 渲染 GT/RT 隔离的核心
6. **Half type 编码到 offset 最高位**(`Offset |= 1 << 31`)—— shader 按位判断数据类型
7. **`bGpuLowLatencyTranslucency`** 解决 GPU sim 半透明粒子"和不透明物体同步"的帧延迟问题,走 TranslucentDataToRender 通路
8. **Cutout 优化**只在 editor 离线生成(`CacheDerivedData`),runtime 直接读 derived data——生产环境零 cost

---

## 自检问题(读完回答)

下面这些题需要把"对偶模式 + GT/RT 隔离 + 4 种 Renderer 的能力矩阵"全部串起来。

1. **Properties + Renderer 一定要分开吗?**:三个理由(性能 / 1-N 关系 / GT-RT 隔离)里如果合一,1-N 怎么破?GT/RT 怎么破?——把"如果合一"想清楚就懂了为什么必须拆。
2. **Ribbon 和 Light 都 CPU only,但原因不同**:Ribbon 是因为算法依赖跨粒子访问;Light 是因为什么?如果 UE 未来支持 GPU readback 零延迟,Light 能转 GPU 吗?Ribbon 能吗?——能区分"算法限制"和"集成限制"才算理解。
3. **`bGpuLowLatencyTranslucency` 关闭的视觉症状**:GPU sim 关掉低延迟通路,半透明粒子会和不透明物体之间出现什么具体异常?(提示:粒子位置滞后一帧 / 与物理同步)为什么不透明粒子完全不需要这个开关?
4. **`FNiagaraDynamicDataBase` union 的并发隐患**:union 同时存 CPU `FNiagaraDataBuffer*` 或 GPU `FNiagaraComputeExecutionContext*` 是因为同 Emitter SimTarget 不会变。但如果未来支持运行时切 SimTarget(Asset 编辑后热重载),union 切换会撞到什么具体的 GT/RT 竞态?
5. **Half offset MSB 编码的扩展极限**:`Offset |= 1 << 31` 隐含了 "DataSet component 不超过 2^31" 的假设。如果未来粒子数据结构规模翻倍,这个编码方式如何演进才能不破坏 shader 兼容?(直接换 64 位 offset / 加 prefix bit / 全部 half / 还有别的方案吗?)
6. **Cutout 在 runtime 的"零成本"边界**:Cutout 几何在 editor `CacheDerivedData` 时离线生成,runtime 直接读 derived data。这个"零成本"在哪种情形下会被打破?(提示:动态修改 SubUV 贴图 / 运行时改 BoundingMode)
7. **17 个 Sprite VF slot 的取舍**:如果你做一个新 Renderer 类型,只需要 5 个属性(Position / Color / Size / Rotation / NormalizedAge),应该继承 Sprite 改通过 attribute binding 关掉 12 个 slot,还是新写一个 Renderer 类?判断标准是什么?

---

## Phase 6 留下的问题

- `FNiagaraSpriteVertexFactory / RibbonVertexFactory / MeshVertexFactory` shader 绑定细节 → **Phase 8.9-8.12**
- GPU sim 情况下 Renderer 如何从 `FNiagaraComputeExecutionContext` 获取数据(通过 `DataToRender / TranslucentDataToRender`)→ Phase 8
- `NiagaraGPUSortInfo` + `SortIndices` 如何接入 GPU 排序 → **Phase 8.7** `NiagaraGPUSortInfo` / `NiagaraSortingGPU`
- Component Renderer(`FNiagaraComponentRenderPool`,Phase 3 出现)——本 Phase 未覆盖,可能不在学习路径,按需专题
- Ribbon tessellation shader 细节 → shader 代码(不在学习路径)

## 下一步预告

**Phase 7**:数据接口系统。Niagara 脚本最强扩展点——让脚本读外部数据(相机、mesh、曲线、RenderTarget、碰撞查询)。10 文件:DI 基类 × 2 + 典型 DI × 8。

---

## 深入阅读

### 本议题的原子页

- 源摘要(Source × 10)
  - 基类:[[Wiki/Sources/Stock/NiagaraRendererProperties]] / [[Wiki/Sources/Stock/NiagaraRenderer]]
  - Sprite:[[Wiki/Sources/Stock/NiagaraSpriteRendererProperties]] / [[Wiki/Sources/Stock/NiagaraRendererSprites]]
  - Ribbon:[[Wiki/Sources/Stock/NiagaraRibbonRendererProperties]] / [[Wiki/Sources/Stock/NiagaraRendererRibbons]]
  - Mesh:[[Wiki/Sources/Stock/NiagaraMeshRendererProperties]] / [[Wiki/Sources/Stock/NiagaraRendererMeshes]]
  - Light:[[Wiki/Sources/Stock/NiagaraLightRendererProperties]] / [[Wiki/Sources/Stock/NiagaraRendererLights]]
- 实体(Entity × 6,Properties/Renderer 合并)
  - [[Wiki/Entities/Stock/UNiagaraRendererProperties]] / [[Wiki/Entities/Stock/FNiagaraRenderer]]
  - [[Wiki/Entities/Stock/UNiagaraSpriteRendererProperties]]
  - [[Wiki/Entities/Stock/UNiagaraRibbonRendererProperties]]
  - [[Wiki/Entities/Stock/UNiagaraMeshRendererProperties]]
  - [[Wiki/Entities/Stock/UNiagaraLightRendererProperties]]

### 前置议题

- [[Readers/Niagara/Phase 2 - Component 层的五职责]] — Component Render Pool、Activate/Deactivate 对 Renderer 的影响
- [[Readers/Niagara/Phase 3 - Niagara 的心脏]] — Emitter Instance 的 `RendererBindings` 参数存储
- [[Readers/Niagara/Phase 4 - Niagara 的数据语言]] — `FNiagaraDataSet` + `FNiagaraDataSetAccessor` 是本 Phase 所有 `GetParticleData*` 的源
- [[Readers/Niagara/Phase 5 - Niagara 脚本如何跑起来]] — CPU 脚本写入粒子后才轮到 Renderer 读

### 相关概念

- [[Wiki/Concepts/UE4/UE4-资产与实例]] — Properties/Renderer 正是 Asset/Instance 对偶的又一实例
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — Light/Ribbon 限 CPU 的根源

### 下一步 / 导航

- 下一阶段:[[Readers/Niagara/Phase 7 - 最强扩展点 Data Interface]] — DI 让脚本吃外部数据
- 学习路径总图:[[Wiki/Syntheses/Niagara/Niagara-learning-path]]
- 仓综合视图:[[Wiki/Overview]]

---

*本读本由 [[Claudian]] 基于 Phase 6 的 10 个头文件(合计 1776 行)综合生成,2026-04-20。commit `b6ab0dee9`。*
