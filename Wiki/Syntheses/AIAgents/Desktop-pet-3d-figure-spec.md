---
type: synthesis
created: 2026-04-29
updated: 2026-04-29
tags: [aiagents, desktop-pet, solaris-3, 3d, vrm, pmx, mmd, asset-pipeline]
sources: 1
aliases: [桌宠 3D 规范, 3D figure spec, PMX 转 VRM]
---

# 桌宠 3D 形象规范 + PMX 入资产管线

> 本页是 [[Readers/AIAgents/桌宠 AI 入口的从零方案]] 的延伸专题——**当桌宠从 Live2D-only 升级到"2D / 3D toggle"形态时**,3D 资产长什么样、动作怎么管、形象层架构怎么改,以及一个常见入口场景:从 aplaybox / bowlroll 等 MMD 站下载的 .pmx 模型怎么进我们的资产管线。
>
> Solaris-3 当前(v0.3.1)只走 Live2D;3D 是 P5+ 的事。本页是规范储备,不是 P0-P3 施工手册。

---

## 0. 议题与边界

主读本明确了**形象层与 Hub 严格解耦**:"今天 Live2D 明天 VRM 后天像素 sprite,Hub 一行不改"。这个抽象在 v0.3.1 还没真兑现——`pixi-live2d-display` 是直接挂在前端的,`IFigureRenderer` 接口不存在。

升级 3D 形象等于**把这个抽象第一次落地**。本页回答四件事:

1. **3D 格式选什么**(VRM 几乎是唯一答案,§1)
2. **模型 + 动作的资产规范**(§2-§3)
3. **2D/3D toggle 在代码里怎么落地**(IFigureRenderer 抽象,§4)
4. **PMX(MMD 模型)入此管线的具体操作**(§5)

**不在本页范围**:Live2D 资产规范(已在 [[Wiki/Syntheses/AIAgents/Desktop-pet-team-distribution]] §1 间接覆盖)、TTS lipsync 实现(P5+ 议题,本页只到 viseme expression 命名)、3D 角色 IP 设计(美术议题,非工程)。

---

## 1. 格式选 VRM(几乎不需要犹豫)

### 1.1 VRM 是什么

**VRM = glTF 2.0 + 角色专用元数据扩展**(humanoid 骨骼标准化、Expression 表情语义、LookAt 视线、Spring Bone 简化物理、License meta)。原生面向 VTuber / 数字人场景,Pixiv 主导,Khronos glTF 之上的 vendor extension,生态彻底开源。

### 1.2 选它的硬理由

| 理由 | 替代方案的代价 |
|---|---|
| `@pixiv/three-vrm` 是事实标准库,免费 + 持续维护 | 自写 Three.js 角色 loader = 3-6 月 |
| Humanoid 骨骼**标准化**(56 标准骨,严格 enum) | 裸 glTF 每加一个动作要写一份骨骼映射 |
| Expression **enum 标准化**(happy/aa/blink/...) | 自己定语义 → LLM 输出"happy=0.8"你不知道映射到哪 |
| LookAt + Spring Bone 自带 | 自己实现 = 1-2 月 |
| 单文件分发(贴图、骨骼、表情、物理参数全打进 .vrm) | Live2D 那种"一个 model3.json + 8 个 png + 5 个 json"的零碎管线 |
| VRoid Studio 可视化捏角色一键导出 | 美术友好门槛极低 |
| Mixamo / mocap 动作能 retarget(humanoid 通用) | 自定义骨骼 = 反生态 |

### 1.3 不选 VRM 的反例

| 候选 | 为什么不 |
|---|---|
| **裸 glTF / glb** | 等于自实现 VRM 那套语义,无理由 |
| **FBX** | 不进 web,要先转,徒增管线 |
| **MMD PMX** 直接上 | 见 §5 — 可以,但 humanoid / Expression / 物理三套语义全靠自桥接 |
| **自定义骨骼** | 每加一个动作写一份 map,毁掉 Mixamo 这条免费动作管道 |

> 版本目标:**VRM 1.0**(`@pixiv/three-vrm` v3+ 默认)。0.x 资产用 `VRMUtils.migrate` 迁移,不要混存。

---

## 2. 模型规范(.vrm)

### 2.1 几何 + 拓扑

| 项 | 规范 | 备注 |
|---|---|---|
| 三角面 | 桌面 30-50k,移动 < 20k | 桌宠半身露出,通常 < 30k 够 |
| 骨骼 | VRM Humanoid 标准 56 骨 | 严格映射,错一个 retarget 全废 |
| 蒙皮权重 | 每顶点 ≤ 4 影响 | glTF 约束 |
| 文件大小 | 目标 < 10 MB | 团队版分发要走 Git LFS,见 [[Wiki/Syntheses/AIAgents/Desktop-pet-team-distribution]] §2 |

### 2.2 贴图 + Shader

| 项 | 规范 |
|---|---|
| 贴图集 | 1× albedo + 1× normal +(可选)1× emissive |
| 分辨率 | **1024×1024**(桌宠不需要 2k+) |
| 内嵌 | 全部嵌入 .vrm,不外挂 |
| Shader | **二次元用 MToon**(VRM 原生 toon ramp);写实用 Standard PBR |

### 2.3 Expression(强制 enum,不能瞎起名)

VRM 1.0 spec 固定 enum,模型导出时**必须**配齐:

| 类别 | 必须 | 用途 |
|---|---|---|
| 情绪 | `happy / angry / sad / relaxed / surprised` | LLM 情绪驱动 |
| 口型 viseme | `aa / ih / ou / ee / oh` | TTS lipsync(P5+ 才接,但模型阶段就要做齐) |
| 视线/眨眼 | `blink / blinkLeft / blinkRight / lookUp / lookDown / lookLeft / lookRight` | LookAt + 自动 blink |

> ⚠️ Expression 半残的 .vrm = 半残角色。**模型阶段一次做齐,运行时不要再补 blendshape**——补一次等于重新过整个导出/加密/分发管线。

### 2.4 LookAt + Spring Bone

- **LookAt**:VRM 自带,有 Bone 模式(转头骨)和 Expression 模式(纯 blendshape 眼球),桌宠两种都行,Bone 模式更自然
- **Spring Bone**:头发、裙摆、领带等次级动作,**在导出时配好**(stiffness / drag / hitRadius / collider),运行时**不要**调参,留 magic number 是雷

---

## 3. 动作规范

### 3.1 容器:.glb + AnimationMixer

VRM 是模型容器,**动作单独以 .glb 保存**,运行时用 `three.js` 的 `AnimationMixer` 播,不要内嵌进 .vrm(不通用,工具支持参差)。

### 3.2 必备动作集

| 类 | 必备 | 时长 | 来源 |
|---|---|---|---|
| **Idle**(loop) | ✅ | 5-10s | VRoid 自带 / 手 K / Mixamo idle |
| **Blink** | ✅ | — | 用 VRMExpression `blink`,**不写 .glb**(自动定时驱动) |
| **LookAt** | ✅ | — | VRM 自带 LookAt 系统,**不写 .glb** |
| **Lipsync** | ✅(P5 接 TTS 时) | — | viseme expression 实时驱动,**不写 .glb** |
| **Emote**(one-shot) | 推荐 | 1-3s | Mixamo retarget:wave / nod / shake / think / bow / clap |
| **Pose preset** | 可选 | 静态 + blend | 站 / 坐 / 抱臂(轻量队伍 ID 加成) |

### 3.3 桩动画约束(桌宠特有)

> ⚠️ **hips 不能位移**——桌宠是定点角色,Mixamo 的"走路 / 跑步"会让人飘出屏幕。

导出动作时:

- root motion 关掉,只保留 hips 旋转
- locomotion 类(走/跑/跳)一律剔除
- 上肢 + 头 + 表情主导

### 3.4 资产目录约定

沿用 [[Wiki/Syntheses/AIAgents/Desktop-pet-team-distribution]] §1 的 `config/effects/` 风格:

```
config/character3d/
├── models/
│   ├── solaris.vrm                  # 默认主角
│   └── (团队/用户自有角色)
├── animations/
│   ├── idle/
│   │   └── idle_breath.glb          # 必有
│   ├── emotes/
│   │   ├── wave.glb / nod.glb / shake.glb / think.glb / bow.glb
│   └── poses/
│       └── standing.glb
└── manifest.json                    # 动作/表情 ↔ 事件映射
```

`manifest.json` 把动作绑事件,沿用 Flipbook 同形态:

```json
{
  "model": "models/solaris.vrm",
  "credit": {
    "author": "原模型作者名",
    "source": "https://...",
    "license": "见 readme.txt"
  },
  "idle": "animations/idle/idle_breath.glb",
  "events": {
    "tool_call_start": { "anim": "animations/emotes/think.glb", "expr": { "relaxed": 0.6 } },
    "tool_call_done":  { "anim": "animations/emotes/nod.glb",   "expr": { "happy": 0.8 } },
    "greet":           { "anim": "animations/emotes/wave.glb" },
    "error":           { "anim": "animations/emotes/shake.glb", "expr": { "sad": 0.5 } }
  }
}
```

> **复利体现**:Hub 事件总线一份,2D Flipbook(`config/effects/`)和 3D anim(`config/character3d/`)各自订阅同一组事件 ID。**不重写事件路由**。

---

## 4. 切换架构(2D/3D toggle)

### 4.1 IFigureRenderer 抽象

形象层抽统一接口,Live2D / VRM 各做一份实现:

```typescript
// hub/figure/IFigureRenderer.ts
interface IFigureRenderer {
  mount(canvas: HTMLCanvasElement): Promise<void>;
  load(modelPath: string): Promise<void>;
  setExpression(name: string, weight: number): void;          // happy/aa/blink/...
  playAnimation(path: string, opts?: { loop?: boolean }): void;
  setLookAt(targetX: number, targetY: number): void;
  setLipsyncViseme(v: 'aa'|'ih'|'ou'|'ee'|'oh', w: number): void;
  destroy(): void;
}

class Live2DRenderer implements IFigureRenderer { /* PixiJS + pixi-live2d-display */ }
class VRMRenderer    implements IFigureRenderer { /* Three.js + @pixiv/three-vrm */ }
```

切换 = **销毁旧 + new 新 + reload**。Hub / MCP / 对话层 0 改动。这正是主读本说的"形象层与 Hub 严格解耦"具体落地。

### 4.2 不要做的两件事

| 反模式 | 为什么不 |
|---|---|
| **2D + 3D 同时叠加** | GPU 浪费 + 形象重影,toggle 不是 stack |
| **Three.js 塞进 PixiJS 同 canvas** | 共享 WebGL 上下文坑深(blend mode / depth 互冲),**用两块独立 canvas**,toggle 时换显示哪块 |

### 4.3 配置层

`config/user.ts` 加 `character.mode: '2d' | '3d'`,设置页"角色"tab 加 toggle。三层配置(defaults/team/user)模型不动——这正是 [[Wiki/Syntheses/AIAgents/Desktop-pet-team-distribution]] §2 配置层架构在多形态资产上的复利。

---

## 5. PMX(MMD 模型)进资产管线

aplaybox / bowlroll / niconi 这一脉的二次元角色资源,默认是 PMX(MikuMikuDance 原生格式)+ VMD 动作。是个常见入口。

### 5.0 ⚠️ 许可证(早于一切)

MMD 模型几乎都附 `readme.txt` / `利用規約.txt`,**许可证比模型本身重要**:

| 条款 | 出现率 | 对桌宠的影响 |
|---|---|---|
| **二次配布禁止** | 极高 | 不能装进团队 build 分发 |
| **改变形改造的可否** | 高 | PMX→VRM 算改造,可能被禁 |
| **商用禁止** | 高 | 个人项目可,公司内部工具有争议 |
| **作者署名** | 必有 | 设置页"关于"tab 必须显示 credit |
| **用途限定**(如 "VTuber 禁") | 中 | 桌宠一般不在限定里,但要确认 |

**单人自用**:大概率没事(自用范畴宽)。
**团队版 build**:**先读 readme,有"二次配布禁止"就直接放弃,不要赌**。

### 5.1 两条技术路线

| 路线 | 做法 | 优点 | 代价 |
|---|---|---|---|
| **A. PMX → VRM**(推荐) ⭐ | Blender 转换,沿用本页全部规范 | 不污染技术栈;LookAt/Expression/Spring Bone 全自动 | 一次性 0.5-2 天工作量 |
| **B. Three.js MMDLoader** 直接吃 | Three.js 自带 `MMDLoader` + `MMDAnimationHelper`,VMD 直接跑 | 不转换;MMD 圈 .vmd 数量远超 VRM | 单独写 `MMDRenderer`;Ammo.js 物理 ~2MB;morph 名映射 VRMExpression 要写表 |

> **判断**:除非重度消费 MMD 舞蹈生态(VTuber 直播向),桌宠场景**几乎肯定走 A**。MMD 动作 90% 是全身 locomotion 舞蹈,桌宠是站桩半身用不上。

### 5.2 PMX → VRM 工具链

| 工具 | 角色 |
|---|---|
| Blender 4.x | 主转换环境 |
| **mmd_tools** addon([powroupi/blender_mmd_tools](https://github.com/powroupi/blender_mmd_tools)) | PMX/VMD 导入 |
| **VRM Add-on for Blender**(saturday06 官方) | VRM 导出 + humanoid 映射 + Expression 配置 |
| **CATS Blender Plugin** | 自动化清理(bone 合并 / weight 修复 / 减面)|

### 5.3 转换步骤

1. **导入 PMX**:File → Import → MikuMikuDance Model (.pmx),贴图同目录自动加载
2. **CATS 一键 Fix Model**:合并骨骼、修法线、清空冗余 vertex group
3. **Bone 映射 VRM Humanoid**:VRM Add-on → Create VRM Model,把 PMX 日语骨骼(`頭` `首` `上半身` `下半身` `右腕` `左ひじ` ...)拖到 humanoid slot;**IK 骨**(`右足ＩＫ` 类)不拖,留作 IK 不影响导出
4. **Expression remap**(关键):

   | PMX morph | → VRMExpression | 强制度 |
   |---|---|---|
   | `まばたき` / `ウィンク` | `blink` / `blinkLeft` | ✅ |
   | `笑い` / `にこり` | `happy` | ✅ |
   | `怒り` | `angry` | ✅ |
   | `悲しい` / `困る` | `sad` | ✅ |
   | `びっくり` / `驚き` | `surprised` | ✅ |
   | `あ` `い` `う` `え` `お` | `aa` `ih` `ou` `ee` `oh` | ✅(lipsync 必须) |
   | 其余装饰性 morph | customExpression 或丢弃 | 可选 |

5. **Spring Bone 重配**:PMX rigid-body+joint → VRM Spring Bone(简化弹性)。头发 / 裙摆骨骼链拖进去,设 stiffness/drag/hitRadius;PMX 原物理参数对不齐,**凭感觉调,接近就行**(桌宠物理影响小,不要花太多时间)
6. **减面 / 贴图压缩**:CATS Decimation 到 30-50k 三角面;贴图统一 1024×1024;目标 .vrm < 10 MB
7. **Export VRM 1.0**:File → Export → VRM,**填 metadata**(作者用原 PMX 作者名、license URL、allowedUserName)——把许可证写进文件头,长期可追溯

### 5.4 检验导出质量

把 .vrm 拖进 [VRM Viewer](https://vrm-viewer.modelviewer.dev/) 或 `@pixiv/three-vrm` 官方 demo:

- ✅ 角色站姿正常,无骨骼穿模
- ✅ Expression 下拉切换,happy/sad/aa 都正确触发
- ✅ Blink 自动循环
- ✅ LookAt 跟鼠标
- ✅ 头发飘动不疯

任何一项失败回 Blender 修。

### 5.5 动作怎么办(.vmd → .glb)

PMX 自带的不是动作,VMD 是单独文件。两条路:

**a) 离线 VMD → glb 烘焙**(推荐)
1. Blender mmd_tools 导入 .pmx
2. 加载 .vmd → 烘成 keyframe
3. **裁掉 hips 位移**(§3.3 桩动画约束)
4. 导出 .glb 进 `config/character3d/animations/`

**b) Mixamo retarget**
- 模型已经是 VRM humanoid,Mixamo 直接 retarget
- 注意:Mixamo 是欧美写实比例,二次元 PMX 比例偏移会有姿态怪(肩塌 / 手腕长),emote 类小动作问题不大,大幅度动作会怪

> 桌宠实战:idle/blink/lookat/lipsync 四件 VRM 自带不需要 .vmd;emote(招手/点头/思考)用 Mixamo 几个就够。**不上 .vmd 舞蹈**——桌宠不是 VTuber 直播。

---

## 6. 常见坑(综合)

| 坑 | 表现 | 修 |
|---|---|---|
| **PMX 导入通体粉色** | 贴图丢 | .pmx 文件移动后没带 textures 目录;回原压缩包整体导 |
| **导出 VRM 后 T-pose** | humanoid bone 没全映射 | VRM addon 报红逐项检查 |
| **嘴巴 lipsync 抖** | viseme morph 强度跟 0-1 不一致 | 在 .vrm preview 里测,过强就在 Blender 调 morph 上限 |
| **VRM 1.0 vs 0.x 选错** | `@pixiv/three-vrm` v3+ 部分 features 失效 | 必选 1.0 |
| **物理炸**(头发飞天) | Spring Bone 极端参数 | 逐链调或关掉物理(set spring bone count 0) |
| **桌宠摄像机不对** | 不同 .vrm 角色身高不同 | 统一 `cameraPreset`(scale + Y offset + fov),换角色不调相机 |
| **透明窗口 + Three.js 抖** | Win11 集显 alpha 三层串联问题 | `renderer = new WebGLRenderer({ alpha: true })` + `setClearColor(0,0)` + Tauri 主窗口透明,**单独验** |
| **MToon 变塑料** | toon shader 参数掉 | 在 Blender material 面板调 outline width / shading shift |

---

## 7. 时机:为什么 3D 是 P5+

主读本路线 P0/P1/P1.5/P2 不动,3D 是 **P5+**:

| 何时 | 理由 |
|---|---|
| **P0-P4 不要做** | Live2D 已支撑核心价值;加 3D 把渲染层调试翻倍,**会拖死 MCP 主线** |
| **P5+** | 当作"v2.0 角色升级"卖点,跟 distribution / TTS 同期上 |
| **触发条件** | P4 跑稳 + 团队/IP 有 3D 美术产能 |

> Live2D 美术不一定会出 3D。**别在没有 3D 美术产能时立项 P5 的 3D 模式**——形象层抽象建好就行,VRMRenderer 实现先空着或 stub。

### 7.1 工时估算(单模型转换)

| Blender 熟练度 | PMX → VRM 工时 |
|---|---|
| 熟手(每周用) | 2-4 小时 |
| 用过几次 | 6-10 小时(humanoid 映射 + expression remap 占大头) |
| 第一次摸 | 1-2 天,跟教程走 |

> 转换卡住超过 2 天 → 退回路线 B(MMDLoader 直接吃 PMX),长期再补 VRM 抽象。**别让美术资产管线卡死主线**。

---

## 8. 关键洞察

1. **VRM = humanoid 标准化 + Expression enum + LookAt + Spring Bone 四件套打包**——这四件每件自己实现都是 1-2 月,合计就是不选 VRM 的全部代价
2. **Expression enum 标准化是 LLM ↔ 形象的桥**——LLM 输出 `happy=0.8` 必须有标准接收端,自定义命名等于把这条桥拆了
3. **2D/3D 是 toggle 不是 stack**——叠加会重影 + GPU 浪费;两块独立 canvas + 销毁/重建是干净路径
4. **桌宠桩动画约束 = root motion 必关**——所有外来动作(Mixamo / VMD)都要过这道滤
5. **PMX → VRM 是 90% 项目唯一合理路径**,MMDLoader 直吃只在重度消费 .vmd 舞蹈生态时成立
6. **MMD 模型许可证早于技术评估**——团队版 build 必读 readme,二次配布禁止就放弃,不赌
7. **manifest.json 事件名跟 Flipbook 同 schema**——Hub 事件总线一份,2D/3D 各自订阅;复利体现 = 不重写路由

---

## 相关

- [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]] — 桌宠议题概念,本页是其 3D 形象延伸
- [[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]] — 选型矩阵(本页 §1 VRM 选择的上游)
- [[Wiki/Syntheses/AIAgents/Desktop-pet-team-distribution]] — 团队分发架构(§1 Flipbook、§2 配置层是本页 §3.4 / §4.3 的同形参考)
- [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]] — MCP host(本页事件总线对接的下游)
- [[Readers/AIAgents/桌宠 AI 入口的从零方案]] — 主题读本,本页是 P5+ 的延伸专题

## 深入阅读(未来若展开)

- VRM 1.0 spec(Khronos glTF 扩展)逐字段精读
- VRM Spring Bone 物理参数调参指南(stiffness/drag/colliderGroup 取值经验)
- Mixamo → VRM retarget 在 `@pixiv/three-vrm` 运行时方案 vs 离线 Blender 方案对比
- MMDLoader 直接吃 PMX 的实施手册(若 Solaris-3 未来真要并行支持)
- Lipsync 实现(TTS 输出 → viseme 时间线 → setLipsyncViseme 实时驱动)详细架构

---

*本页 2026-04-29 由 [[Wiki/Entities/Claudian|Claudian]] 综合两轮桌宠 3D 形象议题对话(规范定义 + PMX 入管线)产出。*
