---
type: entity
category: data-asset-family
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy, data-asset]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/EffectDataAsset
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
aliases: [UEffectModelBase, EffectDataAsset, UEffectModelXxx]
---

# UEffectModelBase 与 DataAsset 家族

> 特效的**配置数据层**——美术/策划在编辑器里创建 `.uasset` 资源配置每种特效的具体表现。C++ 运行时根据这些 DataAsset 构造对应的 [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] + `FEffectSpec`(Batch 3)。

## UObject 基础(背景知识)

先让你理解 UE 在干什么:
- `UDataAsset` / `UPrimaryDataAsset` 是 UE 提供的"**纯数据资源**"基类——对应一个 `.uasset` 文件,在编辑器里可打开编辑各个 `UPROPERTY` 字段。
- `UPrimaryDataAsset` 比 `UDataAsset` 多一点:能**按 Asset Registry 的 PrimaryAssetId 寻址**,支持按 type 批量扫描/预加载。Kuro 选 `UPrimaryDataAsset` 说明特效要参与异步加载管线。
- `UCLASS(Blueprintable)` —— 允许蓝图继承。但几乎所有子类都标了 `HideDropdown = true`,**意思是"允许继承但编辑器里不显示下拉",防止业务随便拖出来用**。

## UEffectModelBase(基类)

```cpp
UCLASS(Blueprintable)
class KUROGAMEPLAY_API UEffectModelBase : public UPrimaryDataAsset
{
    GENERATED_BODY()
public:
    // —— 时间三段式(核心!) ——
    UPROPERTY(EditAnywhere) float StartTime = -1.0f;  // 起始段
    UPROPERTY(EditAnywhere) float LoopTime = 0.0f;    // 循环段
    UPROPERTY(EditAnywhere) float EndTime = 0.0f;     // 结束段
    
    // —— 播放控制 ——
    UPROPERTY(EditAnywhere) bool AutoPlay = true;
    UPROPERTY(EditAnywhere) bool AutoDestroy = true;
    UPROPERTY(EditAnywhere) bool ContinuousLoop = false;
    //   ContinuousLoop:循环特效"变成持久特效",业务侧用法同循环
    
    // —— TimeDilation ——
    UPROPERTY(EditAnywhere) bool IgnoreTimeDilation;
    UPROPERTY(EditAnywhere) bool IgnoreGlobalTimeDilation;
    
    // —— 优先级 & 场景标志 ——
    UPROPERTY(EditAnywhere) bool UiScenePrimitive;       // 用于 UiScene3D 显示
    UPROPERTY(EditAnywhere) int  ImportanceLevel = 0;    // 重要性等级(影响剔除)
    
    // —— 手动回放默认值 ——
    UPROPERTY(EditAnywhere) float DefaultManualTime = 0.0;
    UPROPERTY(EditAnywhere) float DefaultManualSpeed = 1.0;
    
    // —— 启用条件开关 ——
    UPROPERTY(EditAnywhere) bool IgnoreDisable = false;
    UPROPERTY(EditAnywhere) bool DisableOnMobile = false;
    UPROPERTY(EditAnywhere) bool HideOnBurstSkill = false;  // 大招时隐藏
    UPROPERTY(EditAnywhere) bool NeedDisableWithActor = false;
    
    // —— 程序特标(注释:"只能程序修改") ——
    UPROPERTY(DisplayName="程序用超远距离标志位,只能程序修改")
    bool SuperFarProgramFlag = false;
    
    UPROPERTY(DisplayName="联机队友无法看见此特效")
    bool HideForProtoPlayer = false;
    
    UPROPERTY(DisplayName="DebugHidden")
    bool bDebugHidden = false;

    // —— 剔除用包围盒 ——
    virtual FBoxSphereBounds CalcBounds(const FTransform& LocalToWorld,
        TMap<UEffectModelBase*, int32>& CalculatedModels)
    {
        CalculatedModels.FindOrAdd(this, 0)++;
        return CalcBounds(LocalToWorld);
    }
protected:
    virtual FBoxSphereBounds CalcBounds(const FTransform& LocalToWorld) const { /* 零尺寸 */ }
};
```

### 为什么有两个 `CalcBounds` 重载

两层 virtual:
- `CalcBounds(LocalToWorld, CalculatedModels)` — 外层入口,传入一个**引用计数 map**(`TMap<UEffectModelBase*, int32>`)。
- `CalcBounds(LocalToWorld)` — 内层真正计算。

这是为了**防 Group 嵌套导致的死循环**——`UEffectModelGroup` 递归调用子 DA 的 `CalcBounds` 时,若 A→B→A 构成环,计数会>1,就跳过。见 [[#UEffectModelGroup|下面]]。

### 三段式时间 = 核心运行模型

`StartTime | LoopTime | EndTime` 是整个系统的节奏控制轴:
- `StartTime >= 0` 后才开始播 Loop
- `LoopTime = 0` 表示"不循环,播完走人"
- `LoopTime > 0` 循环,接收外部 Stop 才进入 End
- 总生命周期:`StartTime + LoopTime + EndTime`(循环特效的 `LoopTime` 是单次循环时长,不是总时长)

[[entities/project-game/kuro-effect-system/effect-life-time|FEffectLifeTime]] 就围绕这 3 个字段构建状态机。

---

## 18 个子类分类表

按 **作用层级** 分类:

### 粒子 / 主表现类(3)

| 子类 | 关键字段 | 说明 |
|---|---|---|
| `UEffectModelNiagara` | `UNiagaraSystem* NiagaraRef`、4 种参数 Map (Float/Color/Vector/Object)、`ExtraStates[]`、`GlobalScale/Color/Alpha`、`bPointCloudNiagara` | **主力**;绝大多数特效走这条路。Extra States 允许一个特效在"激活态"与"沉静态"间切参数 |
| `UEffectModelGpuParticle` | `UKuroGPUParticleDA* Data`、`Loop`、`EnablePingPong`、`ReversePlay` | Kuro 自研 GPU 粒子系统的接入;比 Niagara 轻量,用于大量小特效 |
| `UEffectModelNDC` | `UNiagaraDataChannelAsset* NiagaraDataChannel` | Niagara Data Channel,用于特效间数据共享;**注释里说:暂停/显隐不支持,需谨慎用**。把 NDC 包装成 DA 只是"让业务无感的使用 DA 形式" |

### 网格 / 模型类(3)

| 子类 | 关键字段 | 说明 |
|---|---|---|
| `UEffectModelStaticMesh` | `UStaticMesh* StaticMeshRef`、单/多 `UMaterialInstance*` override、`MaterialFloat/ColorParameters`、`EnableCollision`、`CastShadow`、`ScreenSizeCullRatio` | 场景静态网格特效(如浮空符文、数学几何体) |
| `UEffectModelSkeletalMesh` | `USkeletalMesh*`、`UAnimationAsset*`、`ReplaceMaterials[]`、`LodBias`、`HideFrames` | 骨骼特效(带动画) |
| `UEffectModelSequencePose` | `SequenceTimeRange`、`DefaultMaterialToUse`、`MaxPose`(clamp 1-8)、`bIncludeOutline` | **残影/姿态序列** — 记录角色过去 N 帧的 pose 并复用材质渲染。刀光剑影的"挥刀拖尾"常用 |

### 贴花类(3)

| 子类 | 关键字段 | 说明 |
|---|---|---|
| `UEffectModelDecal` | `UMaterialInterface* DecalMaterialRef`、`SortOrder`、`ZfadingFactor/Power`、`ForceRenderScene/Charactor` | 标准贴花(地面符文、血污) |
| `UEffectModelCurveTrailDecal` | `UKuroCurveTrailDecalConfig* Config` | 沿曲线生成的贴花(车辙、拖尾印迹);绝大多数字段藏在 Config 里 |
| `UEffectModelGhost` | `UMaterialInstance*`、`USkeletalMesh*`、23 个 `EEffectModelGhostCppComponent` 插槽枚举(Body/Weapon×5/Hulu/Other×17)、`AlphaCurve`、`FloatParameters`、`ColorParameters` | **骨骼残影**(ghost trail);将角色的骨骼网格复制一份,带 Alpha 衰减消失。枚举的 23 个槽对应角色身上能拉残影的组件 |

### 轨迹类(2)

| 子类 | 关键字段 | 说明 |
|---|---|---|
| `UEffectModelTrail` | `AttachToBones`、`AttachBoneNames[]`、`RelativeLocations[]`、`LocationsCurve`、`UMaterialInstance*`、`UnitLength=10`、`DissipateSpeed`、`Alpha`、`DestroyAtOnce`、`DissipateSpeedAfterDead` | 贝塞尔刀光类型,绑到骨骼 |
| (上面的 `UEffectModelCurveTrailDecal` 也属轨迹类) | | |

### 光照类(1,但超级复杂)

| 子类 | 关键字段 | 说明 |
|---|---|---|
| `UEffectModelLight` | 基础 `Intensity/Color/Radius/FalloffExponent` + 聚光灯 + `ELightQualityType` + **大量"角色专用"字段**(CharacterLightAlpha、FalloffExponent、BlendAtten、BlendIntensity、`EToonLightType`、Priority、Intensity、Color、ShadowIntensity、RealTimeShadowIntensity、DesaturationIntensity、硬边光 Intensity/Color/ShadowColor/Blend)、`bCastShadowPc`(PC 投影)、`DriveEffectByWeatherCustomDataName` | 点/聚光灯;角色专用字段反映 Aki 的**卡通光照系统**有专属 light 路径 |

### UI / 屏幕类(2)

| 子类 | 关键字段 | 说明 |
|---|---|---|
| `UEffectModelBillboard` | `EBillboardMode OrientAxis`、`IsFixSize`、`ScaleSize`、`MaxDistance`、`MinSize` | 始终朝向相机的面片;距离/固定尺寸两种模式 |
| `UEffectModelPostProcess` | **700+ 行!!** 覆盖:体积(Volume)参数、天气优先级、材质层覆盖、径向模糊、主灯/天光、雾效、Bloom、色差、色调、黑白闪、灰度渐变、RetroFuturistic(CRT/VHS 复古)、屏幕扰动、自动曝光、BurstDissolve、SkyBox、ColorGrading | **最复杂的 DA**;嵌套 `FKuroEffectPostProcessSkyBoxSetting`(~170 字段) |

### 组合 / 容器类(2)

| 子类 | 关键字段 | 说明 |
|---|---|---|
| `UEffectModelGroup` | `TMap<UEffectModelBase*, float> EffectData` + Location/Rotation/Scale 曲线 | **多个子特效的容器**;`float` 值用途待 Batch 3(推测是启动延迟) |
| `UEffectModelMultiEffect` | 单个 `UEffectModelBase*` + `EMultiEffectType::BuffBall` + `BaseNum/SpinSpeed/Radius` | **程序化多实例**:一个特效按公式生成 N 个围绕某点旋转的副本(目前只有 BuffBall 一种类型,就是角色身上的 Buff 球) |

### 音频 / 材质控制(2)

| 子类 | 关键字段 | 说明 |
|---|---|---|
| `UEffectModelAudio` | `UAkAudioEvent*`(Wwise)、`LocationOffsets[]`、`FadeOutTime=200ms`、`EAudioFadeCurve`、`KeepAlive`、`EnableOcclusion`、`TrailingAudioEvent` | 音效特效;注意:**`EffectSystem` 同时管音效**。`TrailingAudioEvent` 用于尾音 |
| `UEffectModelMaterialController` | **2 个** `UKuroMaterialControllerDataAsset*`(普通 + Group) | 最简单的 DA;具体字段都外置到 MaterialController 资产中 |

## 共通模式(读完家族后的观察)

### 1. `FKuroCurveFloat/Vector/LinearColor` 反复出现
大多数"时变参数"字段类型不是 `float`,而是 `FKuroCurveFloat`。这是 Kuro 自定义的曲线类型(Curve + Constant fallback),**编辑器可以切换到曲线模式**(`bUseCurve = true`),让字段随特效时间变化。这让美术不用代码就能做"起始很亮、中间变暗、结束淡出"的效果。

### 2. 每种表现类型都带 `FloatParameters` / `ColorParameters` / `VectorParameters`
`TMap<FName, FKuroCurveFloat>` 是通用的"动态材质参数集合"——key 是材质参数名,value 是随时间变化的曲线。运行时,`FEffectSpec` 会 tick 这些曲线、把当前值写到 Dynamic Material Instance。

### 3. 继承树只有一层
所有子类直接继承 `UEffectModelBase`,没有中间层。换句话说:特效类型是"**平等的变种**",不存在"我是 X 的 Y 版本"。这让 Init 的 `switch-by-type` 更清晰,也简化了 [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] 的重载方式。

### 4. 绝大多数子类 `HideDropdown = true`
配置用的 DA 不应该手动创建,而是通过工具链批量导入/转换。这是一个工程效率的讲究。

## 与其他实体的关系

- **被 DA 直接持有的运行时**:`FEffectHandleInfo::EffectActor`(Old)、`FEffectHandle::EffectActor`(New)——Actor 上挂相应组件(`UNiagaraComponent`、`UStaticMeshComponent` 等)
- **DA 的字段被 Spec 读**:`FEffectModelNiagaraSpec`、`FEffectModelStaticMeshSpec` 等(Batch 3)
- **DA 的参数被 Parameters 实例收集**:[[entities/project-game/kuro-effect-system/effect-parameters|KuroEffectParameters]]
- **New 里 DA 仍被引用**:`FEffectHandle::GetEffectData() -> UEffectModelBase*`(兼容保留);但主数据路径改成 `FEffectSpecData` + `FEffectInitModel`

## Twin(New 版对应)

- [[entities/project-game/kuro-effect-system-new/effect-spec-data|FEffectSpecData]] — New 把"配置数据"改写成轻量结构体(但仍然保留 `UEffectModelBase` 作为具体类型表达)

实际上 New **没有废弃 `UEffectModelBase` 家族**,而是在其上加了一层 `FEffectSpecData` 元信息(Id/SpecType/LifeTime 这些从 db 来的字段)。编辑器里美术还是配 `UEffectModelNiagara` 等 DA,运行时 `FEffectSpecData` 引用这些 DA 的 Path。**DA 是数据,SpecData 是 DA 的元数据索引**。

## 引用来源

- 原始代码:`F:\Aki\dev\Source\Client\Plugins\Kuro\KuroGameplay\Source\KuroGameplay\Public\KuroEffectSystem\EffectDataAsset\` 目录 18 个 .h 文件
- [[sources/project-game/kuro-effect-system/overview|Old 系统 overview]]

## 开放问题

- `UEffectModelGroup::EffectData` 的 `float` 值是启动延迟还是播放速度?(Batch 3 读 `EffectModelGroupSpec.hpp` 时回答)
- `UEffectModelNiagara::ExtraStates[]` 在运行时如何切换?(有 `SetNiagaraEffectExtraState(ExtraState)` API,但具体实现在 Spec 里)
- `UEffectModelGhost::MeshComponentsToUse` 这 23 个槽枚举和角色骨骼哪个部位匹配?(游戏侧角色 Actor 的组件名约定)
- `UEffectModelPostProcess` 这么大的 DA 运行时怎么下发?是一次性 Apply 还是逐字段 Tick?(Batch 3 PostProcessSpec)
- `UEffectModelNiagara::bPointCloudNiagara` 走的是 New `EEffectSpecDataType_PointCloudNiagara` 独立分支,而非 Niagara 普通分支;这个细分的运行时差异在哪?
