---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [lora, diffusion, ai-art, pipeline, reader, kuro, wuthering-waves]
sources: 11
aliases: [AIArt 读本, LoRA 深度指南读本, 鸣潮美术流水线读本, Lora Deep Dive 读本]
---

# LoRA 深度指南

> 本页是"鸣潮美术向 LoRA 落地方案"这一议题的**主题读本**——详细、精确、满满当当,一次读完即完整掌握从战略(为什么离开 MJ)、技术(LoRA 原理、基座选型、caption 策略、触发词、多 LoRA 组合)、工具(Kohya-ss + ComfyUI)到工程落地(6 个月路线图、合规、数据、硬件)的全链路。
>
> 面向:TA / 技术背景的美术决策者(即本 vault 作者 Eureka 自身的决策参考)。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. 这个议题要回答的问题

鸣潮团队当前的 AI 美术管线是 **MidJourney + tag 库 + 飞书机器人** 的组合:美术选 tag,前端拼 prompt,后端调 MJ,出图贴回飞书。这套管线已经跑得动,但有天花板——**风格不可锁定、不可私有、不可版本化**。

本议题要回答:

1. **战略**:继续堆 MJ,还是另起炉灶搭自有管线?离开 MJ 的**真正动机**是什么,不是"换个工具",而是解决结构性短板
2. **技术**:LoRA 到底是什么?它能解决 MJ 解决不了的哪些问题?原理上有什么限制?
3. **选型**:选哪个基座?为什么不是 Flux?每 3-6 个月新基座出来怎么办?
4. **打标策略**:为什么 auto-tag 直接训**必然**失败?caption 的正确姿势是什么?
5. **部署**:如何不重做前端、不打扰美术,让后端从 MJ 无缝切到 LoRA?
6. **合规**:公司素材 + 开源基座 + 生成产物,法律边界在哪里?
7. **时间表**:6 个月从 0 到产线主力,每个月做什么?

读完本文你应当能:
- 独立判断基座/工具/超参的选型合理性,不被"最新论文推荐"牵着走
- 和团队解释"为什么要做 LoRA、做到什么程度、什么时候算完"
- 看懂任何 LoRA 训练失败的报告,定位问题(大概率是 caption / 基座 / rank 三者之一)
- 避开最致命的法律陷阱(Flux Dev 商用、Illustrious 条款漂移、Civitai 数据泄漏)

叙事主线:**战略 → 技术 → 选型 → 数据 → 工具 → 落地**,每层都在回答"为什么这么选,而不是别的"。

---

## 1. 战略层:为什么要离开 MidJourney

### 1.1 MidJourney 在做什么

MidJourney(简称 MJ)是通用 text-to-image 模型,以艺术感和易用性著称。鸣潮的现有管线用 `midjourney-proxy` 转接 API,配合一个 tag 库和飞书机器人:

```
美术在飞书 → 选 tag → 前端拼 prompt → midjourney-proxy → MJ API → 回图 → 飞书
```

这套管线**能用、跑得动、美术接受度高**。为什么要动它?

### 1.2 MidJourney 的三个结构性短板

对一个**长期运营的游戏项目**,MJ 有三个无法通过换 prompt、充更多钱来解决的根本限制:

#### 1.2.1 风格不可锁定(黑盒)

MJ 的风格由其训练集决定,**你无法确保"我三个月后出的图,和今天出的图,是同一个风格"**。MJ 每次版本升级(v5 → v6 → v6.1)都会带来风格漂移。对需要**一致美术风格**的项目资产,这意味着:

- 同一个世界观的两批 concept art,画风可能已经不是同一种
- 美术评审通过的风格参考,三个月后再追加资产,MJ 已经 drift
- 项目后期需要大量返工统一风格,成本巨大

MJ 的 `--sref`(Style Reference)参数能部分缓解,但它是**在黑盒基础上再叠一层黑盒**——参考图的哪些维度被提取、怎么融合,都不透明。

#### 1.2.2 不可私有化

**MJ 是 SaaS**,所有 prompt 和生成数据都经过 MJ 服务器。这意味着:

- 公司美术 DNA(独特的配色偏好、笔触风格、世界观视觉语言)**无法训练进模型**——只能靠 prompt 工程近似描述,结果不稳定
- 任何发送给 MJ 的参考图,可能被用于未来训练
- 竞争对手用同样的 prompt 和 sref,能得到**相似但不同**的输出——你没有可防御的视觉资产

#### 1.2.3 不可版本化

作为 SaaS,MJ 的"哪个版本在服务"由 MJ 官方决定。**你没有办法在项目发布后 2 年,还能用"和项目启动时一模一样"的模型**重新出一张图。对长线运营(MMO / 卡牌 / 长线续作)是战略风险。

### 1.3 LoRA 能做什么(预告)

[[Wiki/Concepts/AIArt/Lora|LoRA]](Low-Rank Adaptation)是另一条路。一句话概括:

> **用几十 MB 的文件,给通用大模型打一层补丁,让它学会你团队的美术 DNA。**

对应三个短板的解法:

| MJ 短板 | LoRA 的答案 |
|---|---|
| 风格不可锁定 | LoRA 文件固定,挂上去永远是那个画风 |
| 不可私有化 | 本地训练,素材不出公司,输出纳入版本控制 |
| 不可版本化 | `wwstyle_v1.safetensors` 进 Git,5 年后还能复现 |

### 1.4 短中长三阶段路线预览

不是"明天把 MJ 砍掉"。源文档推荐分阶段:

| 阶段 | 时间窗 | 主力 | 变化 |
|---|---|---|---|
| **现在** | 0-1 月 | MJ + tag 库 | 维持现状,同时开始准备 LoRA 训练环境 |
| **过渡期** | 3-6 月 | MJ 主力,LoRA 并行 | 训 MVP LoRA,A/B 对比,积累经验 |
| **目标态** | 6-12 月 | ComfyUI + 自训 LoRA 主力,MJ 作为探索兜底 | 飞书前端不变,后端已切 |

**关键原则**:tag 库前端保留,只切后端路由——美术的使用体验不变。

### 1.5 小结:战略判断

- MJ 的问题**不是工具质量**,是"SaaS + 通用模型"这个范式的结构性短板
- LoRA 不是换一个 MJ,而是**从"租用通用能力"转向"拥有专属能力"**
- 迁移是渐进的,不切断现有产能

下面 §2-§7 从浅到深展开 LoRA 的技术全景。

---

## 2. LoRA 的原理

### 2.1 问题:为什么不能直接全量微调?

Stable Diffusion XL(SDXL)基座有 **26 亿参数**。如果直接微调整个模型来适配团队画风:

- **训练硬件**:多卡 A100,一轮训练数天
- **存储**:改完的模型约 **7 GB**,挂载需要全部载入
- **部署**:想换风格 = 换整个 7GB 模型,冷启动慢
- **组合**:完全做不到(一次只能用一个模型)
- **成本**:每次迭代都是几千美元的级别

这对一个需要频繁迭代、多风格共存的游戏项目**不现实**。

### 2.2 LoRA 的核心思路:冻结 + 旁路

[[Wiki/Concepts/AIArt/Lora|LoRA]] 的全称是 **Low-Rank Adaptation**(低秩适配)。核心思路:

- **冻结**主模型所有 25 亿参数不动
- **在关键层(主要是 cross-attention 层)旁边插入一对低秩矩阵 A × B**
- **只训练这对小矩阵**

用公式讲:原模型的权重 `W` 保持不变,新的实际权重变成:

```
W' = W + A × B

其中 W  = 冻结的原权重,例如 (1024 × 1024)
     A  = 训练的矩阵 (1024 × r)
     B  = 训练的矩阵 (r × 1024)
     r  = 秩(rank),通常 8~64,远小于 1024
```

由于 `r` 很小,`A × B` 本身参数量极少,但它能在模型能力上叠加一个**修正**。这个修正就是"团队画风"。

### 2.3 工程红利

| 维度 | 全量微调 | LoRA |
|---|---|---|
| 参数量 | 25 亿 | 几百万 |
| 文件体积 | 约 7 GB | **30-200 MB** |
| 训练硬件 | 多卡 A100 | **一张 3090/4090** |
| 训练时长(示意) | 数天 | **1-4 小时** |
| 部署 | 换整模型,冷启动慢 | **挂上去就生效,不挂就是原模型** |
| 组合 | 不可(一次只能一个) | **多 LoRA 同时挂载,独立调强度** |

**"挂上去就生效"** 这一条非常关键——意味着一个 ComfyUI workflow 可以在运行时动态决定挂载哪些 LoRA、各自多强,完全 data-driven。

### 2.4 四种 LoRA 类型

按"想教模型什么"分类:

| 类型 | 典型用途 | 数据量 | 训练难度 | 备注 |
|---|---|---|---|---|
| **风格 LoRA** | 锁定整体画风 | 30-80 张 | 中 | **最重要**,团队 DNA |
| **角色 LoRA** | 复现特定角色 | 15-30 张 | 低 | 角色特征少,容量需求低 |
| **概念 LoRA** | 特定特效/道具/怪物品类 | 20-50 张 | 中 | 按"品类"训一个 |
| **构图/姿势 LoRA** | 镜头语言、动作 pose | 20-50 张 | 中高 | 构图类难度高 |

**优先级**:

1. 先做 **1 个核心风格 LoRA** 打底,把团队美术 DNA 固化
2. 后续按需叠加角色 / 概念 LoRA

这是关键工程策略——**不要一开始就想训 10 个 LoRA**,先把打底的 1 个训好、用起来、建立流程。

### 2.5 关键技术参数速览

#### 秩(rank / network_dim)

控制 LoRA 的容量:

- **风格 LoRA**:甜点 `32`;想更泛化用 `16`,想更像用 `64`
- **角色 LoRA**:`16` 甚至 `8`(角色特征不需要那么大容量)
- ⚠️ rank 太大 → 过拟合、文件大、泛化差

#### Alpha(network_alpha)

`network_alpha` 一般取 `network_dim` 的一半(如 `dim=32, alpha=16`)。控制实际学习率缩放。

#### 训练参数基线

- 学习率:`1e-4`(UNet)、`5e-5`(Text Encoder)
- 目标 steps:`2000 - 4000` 通常足够
- 优化器:`AdamW8bit`(省 VRAM)
- 精度:`bf16`(更稳)或 `fp16`

完整 `config.toml` 模板见 §7.2 或 [[Wiki/Entities/AIArt/Kohya-ss]]。

### 2.6 四条串联决策

LoRA 本身只是技术,要真正落地涉及 4 条串联决策,任何一条做错,LoRA 会失败:

1. **选基座** → §3
2. **数据集 + caption 策略** → §4
3. **触发词设计** → §5
4. **多 LoRA 组合部署** → §6

下面按顺序展开。

---

## 3. 基座选型(最关键的战略决策)

### 3.1 为什么基座是第一决策

LoRA 是**架在基座模型上的薄层**。它**无法凭空造出"基座没有的能力"**——它只能**偏置**基座已有的能力往特定方向。

这意味着基座决定了:

- **风格先验**(anime / 写实 / 通用)→ 决定训练是否收敛得顺
- **架构与 VRAM 要求**(SDXL / Flux)→ 决定硬件门槛
- **商业许可**(Dev / Schnell / Commercial)→ 决定能否用于游戏项目
- **生态成熟度**(LoRA 数量、工具支持)→ 决定可借力程度

**基座选错,后面全白干。**

### 3.2 2026 年初的候选全景

| 基座 | 风格倾向 | 参数规模 | VRAM (训练) | 商用许可 | 评价 |
|---|---|---|---|---|---|
| [[Wiki/Entities/AIArt/Illustrious-XL\|Illustrious XL]] | 二次元 / stylized | SDXL (2.6B) | 12-16 GB | **有限制,查最新条款** | 二次元 SOTA 之一,识别 Danbooru tag |
| [[Wiki/Entities/AIArt/NoobAI-XL\|NoobAI XL]] | 二次元 / anime | SDXL | 12-16 GB | **开源友好** | Illustrious 的社区继续训练版,tag 覆盖更广 |
| Pony Diffusion XL v6/v7 | 卡通 / 二次元 | SDXL | 12-16 GB | 开源 | tag 识别强,但对 NSFW 激进(团队用要过滤) |
| SDXL base 1.0 | 通用 | 2.6B | 12-16 GB | 开源 | 中立,anime 能力弱于前三位 |
| [[Wiki/Entities/AIArt/Flux\|Flux.1 Dev]] | 写实为主 | 12B | **24 GB+** | **非商用** ⚠️ | 写实 SOTA,**不能商用**,stylized 较弱 |
| [[Wiki/Entities/AIArt/Flux\|Flux.1 Schnell]] | 写实 | 12B | 24 GB+ | Apache 2.0 ✅ | 可商用但质量低于 Dev |
| SD 1.5 | 全风格 | 860M | 8 GB | 开源 | 生态最丰富但分辨率低,**不推荐新项目** |

### 3.3 对鸣潮(二次元风格化游戏)的推荐

**首选**:[[Wiki/Entities/AIArt/Illustrious-XL|Illustrious XL]] 或 [[Wiki/Entities/AIArt/NoobAI-XL|NoobAI XL]]

理由:

1. 画风先验和 anime 对齐,**训练收敛快**
2. SDXL 架构成熟,工具链完整,LoRA 生态繁荣
3. 原生 1024×1024,出图素质够做 concept reference
4. VRAM 12-16 GB,**一张 3090/4090 能搞定**

**Illustrious vs NoobAI**:

| | Illustrious XL | NoobAI XL |
|---|---|---|
| 血缘 | 原版 | Illustrious 社区继续训,tag 覆盖更广 |
| 许可 | **有限制,条款变动过** | **开源友好** |
| LoRA 兼容 | — | 一般能直接用 Illustrious 训的 LoRA(血缘同源) |
| 团队用建议 | 条款模糊时慎用 | **游戏公司首选,许可陷阱少** |

**结论**:对鸣潮这类游戏公司项目,**NoobAI XL 比 Illustrious 更稳妥**,除非 Illustrious 在某个具体场景有明显优势。

### 3.4 为什么不推荐 Flux(对风格化游戏)

`Flux.1` 是 Black Forest Labs 的 12B 参数新架构,写实 SOTA。但对二次元游戏:

1. **写实感过强**:画风化能力反而弱于 SDXL 系 anime fine-tune
2. ⚠️ **Dev 非商用**:游戏公司用就是法律风险(见 §3.6)
3. **Schnell 质量打折**:能用但不如 Illustrious/NoobAI 在 anime 场景
4. **LoRA 生态不成熟**:相对 SDXL 还在早期
5. **硬件门槛高**:24 GB+ VRAM,团队硬件成本翻倍

**Flux 真正该考虑的场景**:

- 写实 3D 人物 / 环境 concept
- 纪实风 keyart
- 需要**精细文字渲染**(Flux 对 prompt 里的文字支持比 SDXL 好)

即便如此,**商用版本必须用 Schnell 或 Pro API**,不能用 Dev。

### 3.5 基座选型的永恒矛盾

AI 美术领域迭代极快——每 **3-6 个月**会有新基座出现(HiDream、Chroma 等)。

**原则**:

- **实际动工前务必再查一次当前 state**,不要拿半年前的推荐直接用
- **方法论通用**,基座换了只是重选一次的事(LoRA 技术、caption 策略、工具链都一样)
- **尽量选 SDXL 架构**(目前生态最稳),不要刚出的新架构就 all-in

### 3.6 许可陷阱(最容易被忽略)

| 基座 | 陷阱 |
|---|---|
| **Flux.1 Dev** | **非商业许可**,游戏公司不能用;即使用 Dev 训出的 LoRA **也继承**此限制 |
| **Illustrious** | 条款有过变动,每次下载版本都应重新确认,**留存下载时的截图作为合规证据** |
| **Pony** | 开源但分叉多,每个 checkpoint 条款可能不同 |
| **SDXL base** | 开源,但**训练集里的版权内容**仍有法律灰色 |

对公司项目的通用原则:**优先用自有美术素材训练,基座选许可清晰的 SDXL 系**。

### 3.7 小结:基座选型

- 鸣潮首选:**NoobAI XL**(许可友好、生态好、VRAM 友好)
- 备选:Illustrious XL(若在具体场景测下来明显更好)
- **坚决排除**:Flux Dev(非商用),Pony(NSFW 激进)
- 条款每次下载必查,留证据

---

## 4. Caption 策略(反常识核心)

这一节是 LoRA 技术栈里**最反直觉也最决定成败**的部分。auto-tag 一把梭会失败,原因不是 tag 错了,是 caption 策略错了。

### 4.1 问题:auto-tag 直接训为什么失败

美术的直觉反应:"把图片丢给 WD14 Tagger 自动打标,最全最准,直接训就行"——这是**错的**,而且是**系统性错误**。

WD14 Tagger 打出来的 tag 可能长这样:

```
1girl, solo, long hair, temple, sunset, anime style, cel shading, soft lighting, looking at viewer
```

直接拿这个去训,LoRA 学出来**风格极弱**,甚至几乎没用。为什么?

### 4.2 机理:LoRA 学的是"映射差"

LoRA 学的是"**输入 caption → 输出图**"的**映射差**——图里有什么、caption 描述了什么,差异的部分被 LoRA 归因。

- 如果 caption 里**没有**写某个特征,但图里**有** → LoRA 把它归因到"**LoRA 本身**" → 这个特征变成 LoRA 的**默认效果**(永远自带)
- 如果 caption 里**写了**某个特征 → LoRA 把"这个词"对应到"这个视觉" → 这个特征变成 **caption 可触发的内容**(prompt 写了才出现)

对**风格 LoRA**,你希望"画风、笔触、色调倾向"永远自带,所以:

### 4.3 核心原则(反常识)

> **你想让 LoRA 永远带的特征,caption 里不要写;你想让它按需出现的内容,caption 里写清楚。**

具体到风格 LoRA:

| 图里有的东西 | caption 里 | 结果 |
|---|---|---|
| 画风、笔触、色调倾向 | **不写** | 学成 LoRA 的默认画风(永远自带)✅ |
| `anime style` / `painting style` / `cel shading` 等描述画风的词 | **必须删除** | 否则 LoRA 学成"这个词 = 那个效果",污染 prompt 空间 ❌ |
| 具体角色、动作、场景、物件 | **要写** | 学成可控内容(prompt 写了才出现)✅ |
| 触发词 `wwstyle_v1` | **每张图 caption 最前面必加** | 学成 LoRA 的激活开关(见 §5) |

### 4.4 两种 Caption 格式

选对齐基座的训练格式,**别跨格式用**。

#### 4.4.1 Danbooru tag 风格(anime 基座)

适用:Illustrious / NoobAI / Pony。

```
wwstyle_v1, 1girl, solo, long hair, standing, temple, sunset, looking_at_viewer
```

特点:短、逗号分隔、每个 tag 独立。

#### 4.4.2 自然语言风格(SDXL base / Flux)

适用:通用 SDXL base、Flux。

```
wwstyle_v1, a young woman with long hair standing in front of a temple at sunset, looking at the viewer
```

特点:完整英文句子,BLIP-2 等工具可生成。

### 4.5 实操工作流

#### 4.5.1 三步法

1. **用 WD14 Tagger**(anime,kohya_ss 自带)或 **BLIP-2**(自然语言)批量自动打标
2. **人工扫一遍,删除描述画风的 tag**:
   - ❌ `anime style` / `painting style` / `cel shading`
   - ❌ `soft lighting`(如果这是你想让 LoRA 学走的风格特征)
   - ✅ 保留 `1girl, standing, temple, sunset` 等具体内容
3. **在每条 caption 最前面加触发词** `wwstyle_v1,`

#### 4.5.2 工具

- **WD14 Tagger**:anime 向 auto-tag,kohya_ss 自带
- **BLIP-2**:自然语言描述生成
- **BooruDatasetTagManager**:**批量编辑 caption 的神器**,对删除画风词、加触发词都高效

### 4.6 常见错误速查

| 错误 | 后果 |
|---|---|
| 用 auto-tag 的结果直接训,不删风格词 | **LoRA 学不到画风,效果极弱** |
| 把画风形容词写进 caption | 学成"这个词 = 那个效果"而不是 LoRA 默认 |
| 不加触发词 | LoRA 强制生效,生产时不灵活 |
| 触发词用常用词(如 `anime`) | 污染基座,没挂 LoRA 也会发作 |

### 4.7 角色 LoRA 的 caption 策略是否一样

> [!warning] 原理相同,实践有差别
> - 角色 LoRA 数据少(15-30 张),caption 编辑成本低
> - 触发词设计更重要(`wwstyle_encore` 直接对应角色名)
> - 图里你希望"永远带"的是**角色的视觉特征**(发色、瞳色、标志装饰),这些**不写入 caption**

### 4.8 小结:Caption 策略

- 反常识原则:**永远带的不写,按需触发的写清楚**
- auto-tag 不能直接用,必须人工删画风词
- 触发词在 caption 最前面,每张图都有
- Danbooru tag vs 自然语言,对齐基座训练格式

---

## 5. Trigger Word(触发词)

### 5.1 触发词是什么

触发词是 LoRA 的"**开关**"——

- 训练时:每张图 caption 最前面放一个独特词(如 `wwstyle_v1`),LoRA 把这个词学成"整个 LoRA 效果的开关"
- 生产时:prompt 里写 `wwstyle_v1` → 激活 LoRA 效果;不写 → 基座原样输出

```
生产 prompt 示例:
  wwstyle_v1, 1girl, standing on a cliff, dramatic sunset, cinematic lighting
  └────────┘  └─────────────────────────────────────────────────────┘
    触发词                            主体 prompt(美术关心的部分)
```

### 5.2 设计原则(四条)

#### 5.2.1 必须是基座没学过的词

**不能用**:`anime` / `style` / `art` / `realistic` / `portrait` — 基座**已经理解**这些词,用它们做触发词会**污染基座**(没挂 LoRA 时这些词也会生效,和 LoRA 强相关时这些词的含义又被扭曲)。

**应该用**:
- 项目代号 + 版本:`wwstyle_v1`、`akiart_v2`(ww = Wuthering Waves 鸣潮)
- 随机短字串:`xy7st9_v1`

#### 5.2.2 足够独特以便识别

让你**6 个月后看到 prompt 还能认出"这是哪个 LoRA"**。`xy7st9_v1` 在这条上不如 `wwstyle_v1` 直观。

#### 5.2.3 版本化

- `wwstyle_v1` → 第一版
- `wwstyle_v2` → 改进后
- 这样一份 prompt 固定了版本,**可复现**。一年后重出图不需要记住"当时用的哪个 LoRA"

#### 5.2.4 一个 LoRA 一个触发词

除非明确想做多触发词切换(进阶用法,§12.3),**每个 LoRA 一个触发词**,简单清晰。

### 5.3 生产约定

```
基础 prompt 格式:
  <trigger>, <subject>, <composition>, <lighting>, ...

例:
  wwstyle_v1, 1girl, standing on a cliff, dramatic sunset, cinematic lighting
```

在 ComfyUI workflow 里把 `wwstyle_v1,` **作为 prompt 前缀自动拼接**,美术只关心后面的内容,完全不用知道 trigger 的存在。

### 5.4 常见错误

| 错误 | 后果 |
|---|---|
| 用 `style` / `art` / `anime` 做触发词 | 污染基座,没挂 LoRA 也会受影响 |
| 每版 LoRA 用相同触发词 | 不挂明确版本的 LoRA 就无法保证效果 |
| 训练时加、生产时忘 | LoRA 效果弱或无 |
| 触发词太长(一整句) | 记忆负担大,容易写错 |

### 5.5 小结:Trigger Word

- 触发词 = LoRA 的开关 + 版本 key
- 四原则:独特 / 可识别 / 版本化 / 一 LoRA 一触发词
- ComfyUI 自动拼前缀,对美术透明

---

## 6. Multi-LoRA 组合(真正的生产力)

这一节揭示 LoRA 相对 MJ 最大的结构性优势——**模块化组合**。单个 LoRA 是有上限的工具,多 LoRA 组合才是 **MJ 做不到的事**。

### 6.1 概览:多 LoRA 同时挂

[[Wiki/Entities/AIArt/ComfyUI|ComfyUI]] 里可以**同时挂载任意多个 LoRA**,每个独立调 strength、独立开关。这让风格从"一团黑盒"变成"**一堆可调旋钮**"。

### 6.2 典型 LoRA 栈

一个"鸣潮风格一号角色 + 戏剧灯光"场景的典型挂载:

```
基座模型(Illustrious XL 或 NoobAI XL)
  ├─ 风格 LoRA: wwstyle_v1        (strength 0.8)   ← 打底风格(团队 DNA)
  ├─ 角色 LoRA: wwstyle_encore    (strength 0.9)   ← 特定角色(安可/安柯)
  ├─ 灯光 LoRA: dramatic_lighting (strength 0.5)   ← 光影增强
  └─ 细节 LoRA: detail_tweaker    (strength 0.3)   ← 细节细化
```

每层 LoRA **解决一个关注点,独立迭代**。这是**关注点分离**(Separation of Concerns)在 AI 美术上的直接应用。

### 6.3 为什么 MidJourney 做不到

| 维度 | MidJourney | LoRA 组合 |
|---|---|---|
| 风格表达 | `--sref` 近似参考(黑盒) | **LoRA 文件**(白盒,可训可调) |
| 叠加能力 | 多个 sref 权重平均 | **每个 LoRA 独立挂、独立 strength** |
| 可版本化 | ❌ | ✅(文件可提交 Git) |
| 可私有化 | ❌ | ✅(本地部署) |
| 可组合迭代 | 有限 | **每层独立更新** |

所以 LoRA 不只是"另一个风格工具",而是**真正模块化的风格控件**。MJ 是"艺术家工具",LoRA 是"工程师的风格库"。

### 6.4 Strength 调节经验

- **风格 LoRA**:`0.7 - 0.9`(太强会覆盖其他 LoRA)
- **角色 LoRA**:`0.8 - 1.0`(需要强特征锁定)
- **细节/辅助 LoRA**:`0.2 - 0.5`(只是微调)

### 6.5 多 LoRA 互相争抢权重

> [!warning] 重叠特征会互相削弱
> 当多个 LoRA 学了**重叠的特征**,会互相削弱:
>
> - 如果角色 LoRA 训练时不小心学了一些画风特征(caption 没清干净),会和风格 LoRA 冲突
> - 表现:强度相加后画面崩坏、风格漂移
>
> **解法**:
> - 重训问题 LoRA,caption 清干净
> - 调降冲突 LoRA 的 strength
> - 更换基座重训(可能是基座对某些特征偏好不一致)

### 6.6 和 Tag 库前端的衔接

鸣潮的用户故事:

```
美术在飞书 → 选 tag(tag 库 UI 不变)
    ↓
前端拼 prompt
    ↓
后端路由 → ComfyUI API
    ↓
ComfyUI 里 workflow 挂好风格+角色+灯光+细节 LoRA(TA 管控)
    ↓
加 trigger: `wwstyle_v1,` 作为 prompt 前缀(自动)
    ↓
ComfyUI 生图 → 飞书 card 回复
```

**分工**:
- **前端**:tag 选择器,美术看到的界面,**不变**
- **TA 后端管控**:选哪些 LoRA、各自 strength、触发词、workflow 版本
- **美术感知**:只看到"风格变得更可控、迭代更快",不关心底层

这是**"后端替换,前端不变"**的典型增量改造。

### 6.7 小结:Multi-LoRA

- 风格分离成独立模块,组合得出最终画面
- 每个模块独立训、独立 strength、独立 Git 版本
- **MJ 永远做不到** —— 这是 LoRA 最硬的差异化
- TA 管控后端,美术体验无感

---

## 7. 工具链:Kohya-ss + ComfyUI

### 7.1 分工

| 环节 | 工具 | 理由 |
|---|---|---|
| **训练** | [[Wiki/Entities/AIArt/Kohya-ss\|kohya_ss]] | Windows 友好、文档多、WD14 Tagger 内置、DreamBooth/LyCORIS 全支持 |
| **推理/部署** | [[Wiki/Entities/AIArt/ComfyUI\|ComfyUI]] | 节点图、workflow-as-code、API mode、多 LoRA 原生支持 |
| **Caption 编辑** | BooruDatasetTagManager | 批量编辑的神器 |
| **Auto-tag** | WD14 Tagger(kohya 自带) / BLIP-2 | 批量打标 |

### 7.2 Kohya-ss 训练

#### 7.2.1 kohya_ss 是什么

- **本体**:`github.com/bmaltais/kohya_ss`(GUI)
- **底层**:`github.com/kohya-ss/sd-scripts`(CLI,更灵活)
- **平台**:Windows / Linux / macOS(Windows 支持最好)
- **最小显存**:12 GB(启用 gradient_checkpointing);**推荐**:24 GB(RTX 3090/4090)

#### 7.2.2 最小可行 config.toml

```toml
[model_arguments]
pretrained_model_name_or_path = "D:/models/NoobAI-XL-v1.safetensors"

[dataset_arguments]
train_data_dir = "D:/datasets/wwstyle_v1/img"
resolution = "1024,1024"
enable_bucket = true

[training_arguments]
output_name = "wwstyle_v1"
max_train_epochs = 10
train_batch_size = 2
mixed_precision = "bf16"
cache_latents = true
cache_text_encoder_outputs = true

[optimizer_arguments]
optimizer_type = "AdamW8bit"
learning_rate = 1e-4

[network_arguments]
network_module = "networks.lora"
network_dim = 32
network_alpha = 16
```

关键开关:

- `enable_bucket = true` — Aspect Ratio Bucketing,自动按比例分组训练(数据集多种长宽比时必须)
- `cache_latents` + `cache_text_encoder_outputs` — **省 VRAM 利器**,训练时不重复 encode
- `mixed_precision = "bf16"` — 比 fp16 更稳,4090 支持
- `AdamW8bit` — 比 AdamW 省 VRAM

#### 7.2.3 数据目录约定

```
D:/datasets/wwstyle_v1/
├── img/
│   └── 10_wwstyle/          ← "10_" 是 repeat 次数
│       ├── 001.png
│       ├── 001.txt          ← 同名 .txt 就是 caption
│       ├── 002.png
│       ├── 002.txt
│       └── ...
├── sample_prompts.txt        ← 训练中定时出 sample 用
└── config.toml
```

前缀 `10_` = 每张图训练 10 次。**一个 epoch 的 steps 数** = 数据集张数 × repeat 次数 ÷ batch_size。

例:50 张 × repeat 10 ÷ batch 2 = **250 steps/epoch**,10 epoch 共 2500 steps,落在目标范围 2000-4000。

#### 7.2.4 kohya 和同类工具对比

| 工具 | 场景 | 推荐度 |
|---|---|---|
| **kohya_ss GUI** | Windows 主力,风格 LoRA 训练 | ★★★★★ |
| sd-scripts | kohya 底层 CLI,灵活但门槛高 | ★★★(后期上 pipeline 用) |
| ai-toolkit (ostris) | 新锐,**Flux 支持最好** | ★★★★(Flux 场景) |
| OneTrainer | 纯 GUI,新手友好 | ★★★ |
| Civitai 在线训练 | 懒人方案 | ★ **(公司数据严禁用!)** |

> [!warning] Civitai 在线训练公司数据禁止
> 上传给第三方 = 数据泄漏。**任何公司美术素材都必须本地训练**。

### 7.3 ComfyUI 部署

#### 7.3.1 ComfyUI 是什么

- **类型**:节点图式 Stable Diffusion 推理 / 管线编排 UI
- **仓库**:`github.com/comfyanonymous/ComfyUI`
- **核心特点**:每个节点是一个操作(load model、encode prompt、sample、save),连起来成 workflow
- **许可**:GPLv3

#### 7.3.2 为什么生产管线首选 ComfyUI

1. **Workflow 即代码**:所有管线配置是 JSON,**可版本化、可审查、可复用**
2. **API mode 强**:HTTP 提交 workflow → 返回图片,**天然适合后端集成**
3. **多 LoRA 挂载**:实现 §6 的必备平台
4. **扩展性**:ControlNet / IP-Adapter / 各种自定义节点应有尽有
5. **无商业许可限制**:**自己部署,数据不出公司**

#### 7.3.3 典型 Workflow 节点栈

风格 LoRA 生产的最小 workflow:

```
CheckpointLoader (加载基座,如 NoobAI XL)
  ↓
LoraLoader (挂 wwstyle_v1)
  ↓
LoraLoader (挂角色 LoRA,可选)
  ↓
CLIPTextEncode (正向 prompt:"wwstyle_v1, 1girl, ...")
  ↓
CLIPTextEncode (负向 prompt:"worst quality, ...")
  ↓
KSampler (生成)
  ↓
VAEDecode (解码)
  ↓
SaveImage / PreviewImage
```

#### 7.3.4 飞书机器人后端集成(核心代码)

```python
def handle_tag_selection(tag_combo):
    # 1. 拼主体 prompt
    prompt = assemble_prompt(tag_combo)

    # 2. 加触发词前缀(美术无感)
    prompt = f"wwstyle_v1, {prompt}"

    # 3. 加载 workflow 模板
    workflow = load_workflow_template("ww_style_v1.json")

    # 4. 运行时参数注入
    workflow["6"]["inputs"]["text"] = prompt
    workflow["10"]["inputs"]["lora_name"] = "wwstyle_v1.safetensors"
    workflow["10"]["inputs"]["strength_model"] = 0.8

    # 5. 提交 → 等结果 → 回飞书
    prompt_id = comfyui_client.queue_prompt(workflow)
    images = comfyui_client.wait_for_images(prompt_id)
    send_feishu_card(images)
```

**架构**:`飞书前端(tag 选择器)→ 后端路由 → ComfyUI API → 图片 → 飞书 card`。

### 7.4 和同类推理工具对比

| 工具 | 定位 | 推荐度 |
|---|---|---|
| **ComfyUI** | **生产管线、工程化** | ★★★★★ |
| Forge | A1111 的性能优化分支 | ★★★★ |
| Automatic1111 webui | 老牌,插件多但迭代慢 | ★★★ |
| InvokeAI | 美术友好 UI,商用许可清晰 | ★★★ |

**对鸣潮管线**:ComfyUI 是**唯一合理选择**,因为要做飞书后端集成,workflow-as-code 必须。

### 7.5 硬件要求速查

| 场景 | VRAM |
|---|---|
| SDXL + 单 LoRA 推理 | 8 GB(紧),12 GB(舒适) |
| SDXL + 多 LoRA 推理 | 12-16 GB |
| Flux + LoRA 推理 | 24 GB+ |
| SDXL LoRA 训练(最小) | 12 GB(开 gradient_checkpointing) |
| SDXL LoRA 训练(推荐) | 24 GB(RTX 3090/4090) |
| Flux LoRA 训练 | 24 GB+ |

**云方案**:RunPod / vast.ai,A100 约 0.5-2 USD/h,短期试训合算。

### 7.6 小结:工具链

- 训练:**kohya_ss GUI**(Windows 首选)
- 推理/部署:**ComfyUI**(API mode + workflow-as-code,生产唯一选择)
- 硬件:4090 单卡打天下,云方案备胎

---

## 8. 落地路线图(6 个月)

基于鸣潮团队现状(Windows + 4090 + midjourney-proxy + 飞书机器人),源文档推荐的阶段划分:

| 阶段 | 时间 | 主要动作 | 产出 |
|---|---|---|---|
| **M1:准备** | 第 1 月 | kohya_ss + ComfyUI 装好;下 NoobAI XL(/Illustrious);组织 40-60 张测试数据集;试训第一个风格 LoRA **(不上生产)** | 工作流跑通,第一份 `wwstyle_v0.1.safetensors` |
| **M2:MVP LoRA** | 第 2 月 | 重新选图(严选,单一画风);训 v0.1;美术盲评;迭代到 v1.0 | `wwstyle_v1.safetensors` 美术认可 |
| **M3:管线搭建** | 第 3 月 | ComfyUI API + 飞书后端路由;前端多出一个"风格模式"切换(MJ / LoRA);MJ vs LoRA A/B 对比 | 飞书用户可切换 MJ / LoRA,积累数据 |
| **M4-M6:扩展** | 4-6 月 | 训 2-3 个角色 LoRA;训 1-2 个概念 LoRA;根据 A/B 数据决定替换策略 | 3-5 个 LoRA 文件,LoRA 栈模板化 |
| **6 月后** | - | ComfyUI 主力,MJ 作为快速探索兜底 | MJ 调用量 < 30% |

### 8.1 关键原则

1. **tag 库前端不重做**,只切后端路由——美术使用体验不变
2. **不切死 MJ**,保留作为"快速探索兜底"(新风格试水、灵感阶段)
3. **A/B 先行**,数据说话,不靠感觉判断切换时机
4. **LoRA 纳入 Git**——`wwstyle_v1.safetensors` 可被 Git LFS 管理,workflow JSON 直接 Git 管理

### 8.2 M1 准备期的具体动作(当前阶段)

截至 2026-04-20,本 vault 所属的鸣潮美术向 TA 处于 **M1 准备期**(尚未开始 MVP 训练):

- [x] ingest LoRA 深度指南,方案文档化
- [ ] 本机装 kohya_ss + ComfyUI
- [ ] 下 NoobAI XL 并验证推理
- [ ] 组织 40-60 张候选数据集(公司自有素材,单一画风)
- [ ] 试训第一个 `wwstyle_v0.1`(不上生产,只验证工具链)

进到 M2 的前提:M1 清单全部 done,`wwstyle_v0.1` 出图有风格(哪怕弱)。

---

## 9. 合规 / 数据 / 硬件

### 9.1 数据合规(最容易出事)

原则:**训练数据优先用公司自有素材,避免互联网抓图**。

| 情境 | 判定 |
|---|---|
| 全部用公司已有美术 output | ✅ 安全 |
| 用公司 licensed 素材库(Adobe Stock 等) | ✅(看 license 允许 AI 训练否) |
| 互联网抓图 / Pinterest / ArtStation | ❌ **法律风险** |
| Civitai 上传数据在线训练 | ❌ **绝对禁止**(数据出公司) |
| 混用:自有 + 少量公开素材参考 | 🔶 **需要法务介入判定** |

### 9.2 基座许可追踪

| 基座 | 许可状态 | 对鸣潮建议 |
|---|---|---|
| Illustrious | **条款变动过,每次下载重确认** | 下载时截图存合规证据 |
| NoobAI XL | **开源友好** | **首选** |
| Flux.1 Dev | **非商业许可** | **严禁** 用于商用产线 |
| Flux.1 Schnell | Apache 2.0 | 可用但质量打折 |
| Pony XL | 开源,NSFW 激进 | 需人工过滤,不推荐 |
| SDXL base 1.0 | 开源 | 通用,anime 弱 |

### 9.3 LoRA 继承基座的许可

> [!warning] 重要:LoRA 法律上继承基座许可
> 用某基座训的 LoRA 在**法律上继承基座的许可限制**。用 Flux Dev 训出的 LoRA 也**不能商用**。

### 9.4 硬件成本(参考)

| 资源 | 一次性 | 月均 |
|---|---|---|
| RTX 4090 工作站 | ¥15,000-20,000 | 电费 ¥200-400 |
| RTX 3090 工作站 | ¥8,000-12,000 | 同上 |
| 云 A100(RunPod) | 0 | ~$100-500(按使用量,试训阶段) |

对鸣潮:**本地 4090 本地训**,云只在高峰用。

---

## 10. 评估准则

怎么判断一个 LoRA 训好了?不是"美术觉得好看",而是四个**可独立验证的维度**:

| 准则 | 含义 | 测试方法 |
|---|---|---|
| **风格一致** | 同一 prompt 多次出图,风格方差小 | 跑 10 次同一 prompt,看统计分布 |
| **prompt 服从** | 换 prompt 内容(换角色、换场景),响应正确 | 用 6-10 个测试 prompt 跑一遍 |
| **可调 strength** | strength 0.3 ~ 1.0 输出可预期变化 | 固定 prompt 跑 5 档 strength |
| **不污染基座** | 挂 vs 不挂 LoRA,基座自身能力保持 | 不挂 LoRA 跑几个 prompt,对比 |

第四条最容易被忽略——一个"污染基座"的 LoRA,挂上去画风很像团队,**摘下来基座也变了(不再能输出通用内容)**,这种 LoRA 长期不可用。

---

## 11. 全景回看:一条完整管线

把前 10 节串起来,一次**从美术选 tag 到飞书出图**的完整路径:

```
┌───────────────────────────────────────────────────────────────┐
│  美术在飞书选 tag: "1girl + standing + sunset + temple"        │
│    │                                                          │
│    ▼                                                          │
│  前端(tag 库 UI,不变)拼出主体 prompt                          │
│    "1girl, standing, sunset, temple"                          │
│    │                                                          │
│    ▼                                                          │
│  后端路由(新加)判断:走 MJ 还是 LoRA?                          │
│    │(LoRA 模式)                                               │
│    ▼                                                          │
│  加触发词前缀:"wwstyle_v1, 1girl, standing, sunset, temple"    │
│    │                                                          │
│    ▼                                                          │
│  ComfyUI API 提交 workflow:                                    │
│    · 加载基座 NoobAI XL                                        │
│    · 挂风格 LoRA: wwstyle_v1 @ 0.8                             │
│    · 挂角色 LoRA: wwstyle_encore @ 0.9(若 tag 含角色)         │
│    · 挂灯光 LoRA: dramatic_lighting @ 0.5(若 tag 含戏剧灯光)  │
│    · CLIPTextEncode → KSampler → VAEDecode → SaveImage        │
│    │                                                          │
│    ▼                                                          │
│  ComfyUI 回 HTTP(图片 base64 或路径)                          │
│    │                                                          │
│    ▼                                                          │
│  后端构造飞书 card,回复美术                                    │
│    │                                                          │
│    ▼                                                          │
│  美术看到图 → 满意/不满意 → 调 tag 再来一次                    │
└───────────────────────────────────────────────────────────────┘
```

**关键观察**:
- 美术体验和现在用 MJ 几乎一样——还是选 tag,等出图
- 变化全在后端:**基座可控、风格可锁、可版本化、多 LoRA 组合**
- TA 在**后端 workflow + LoRA 栈** 上管控,不打扰美术

---

## 12. 进阶方向(中长期)

这些在 MVP 阶段**不要碰**,6 月后按实际需要再评估。

### 12.1 DoRA(Weight-Decomposed LoRA)

LoRA 的改进版,**小数据表现更好**。kohya_ss 已支持。对角色 LoRA(数据量本就少)可能明显受益。待实测。

### 12.2 LyCORIS 家族(LoCon / LoHa)

比 LoRA 更强的深层特征学习,**文件更大、训练更慢**。和 LoRA 组合策略可能不同。

### 12.3 IP-Adapter

**比 LoRA 更轻的"风格注入"方案**,不用训练,直接给参考图生效。是替代 LoRA 还是补充?目前看是**补充**(IP-Adapter 适合一次性参考,LoRA 适合团队 DNA 固化)。

### 12.4 ControlNet + LoRA

ControlNet 给构图控制(骨架、深度、边缘),LoRA 给风格。组合起来能做**"给定构图 → 出团队风格图"**,对原画转制有用。

### 12.5 训练数据回流

生产中被**美术评分高**的出图自动入训练池,**季度 retrain** LoRA。未验证的长期演进路径,理论上能让 LoRA 越训越"懂团队口味"。

### 12.6 多触发词 LoRA

一个 LoRA 学多个风格变体(如 `wwstyle_day` / `wwstyle_night`),用不同触发词切换。原理上可行,实践效果待验证。

---

## 13. 关键洞察(带走)

1. **离开 MJ 不是换工具,是换范式**——从"租用通用能力"到"拥有专属能力",解决风格锁定、私有化、版本化三个结构性短板
2. **LoRA 是架在基座上的薄层**,不能凭空造能力,只能偏置基座已有的能力。基座选型是**第一决策**,错了后面全白干
3. **Caption 反常识原则**:永远带的不写(画风),按需触发的写清楚(主体)。这是 auto-tag 一把梭失败的根本原因
4. **触发词是 LoRA 的开关 + 版本 key**,设计 4 原则:独特 / 可识别 / 版本化 / 一 LoRA 一触发词
5. **Multi-LoRA 组合才是真正的生产力**——单 LoRA 是工具,多 LoRA 栈是模块化风格控件系统,MJ 永远做不到
6. **工具选型已收敛**:kohya_ss 训 + ComfyUI 部署,其他是路线偏离
7. **前端不重做,只切后端路由**——增量改造路径,美术无感迁移
8. **Flux Dev 对游戏公司是法律红线**,Civitai 在线训练对公司数据是数据泄漏红线,这两条不要碰
9. **训好 LoRA 的四维评估**:风格一致 / prompt 服从 / strength 可调 / 不污染基座。第四条最易被忽视
10. **基座每 3-6 月重查一次**,方法论不变,基座换了只是重选一次的事

---

## 14. 议题遗留的问题(开放问题)

### 14.1 技术待验证

- **DoRA 在风格 LoRA 场景**的稳定性(源文档未展开,kohya 已支持)
- **LyCORIS vs LoRA 的实际取舍**(深层特征 vs 文件大小)
- **IP-Adapter 是否能部分替代临时风格 LoRA**(节省训练成本)
- **多触发词 LoRA 的实践效果**(一个 LoRA 多变体)
- **LoRA 合并 vs 挂载**:`supermerger` 把多 LoRA 合一,是否更稳定?
- **自适应 strength**:根据 prompt 自动调各 LoRA strength(研究方向未成熟)

### 14.2 工程待落实

- **数据集上限**:源文档给 100 张,大型团队做产业级 LoRA(上千张)经验缺
- **LoRA 训练的团队协作 SOP**:任务分配、版本管理、盲评流程未详述
- **Workflow JSON 多人协作**:如何 diff / review?
- **热重载 LoRA**:换 LoRA 不重启 ComfyUI 的最佳实践

### 14.3 合规待评估

- **"自有素材 + 少量公开参考"混用**的法律边界(需法务介入)
- **Illustrious 条款漂移**:已训 LoRA 是否受新条款追溯影响?
- **LoRA 文件本身的版权归属**(团队训的 LoRA 属于谁?)

### 14.4 业务待观察

- **tag 库前端的技术债评估**:如果前后端耦合严重,切后端工作量估算?
- **A/B 对比的指标定义**:美术主观评分?出图接受率?效率(单图时间)?

---

## 15. 下一步预告

**主线**:按 §8 路线图推进,M1 准备期已启动(ingest 本方案 = 准备期第一步),下一步本机装 kohya_ss + ComfyUI,试训 `wwstyle_v0.1`。

**本读本的后续更新时机**:

1. **M2 MVP LoRA 完成时**——回来更新 §10 评估准则的**实测数据**,补"第一版 LoRA 的经验"
2. **M3 管线搭建完成时**——补 §11 全景图的**实测延迟、吞吐、成本**
3. **基座选型有新拐点时**(如 HiDream / Chroma 成熟)——重写 §3,老内容移到"历史档案"
4. **发现更好的 caption 策略**(如 DoRA 对 caption 的要求不同)——补 §4

---

## 深入阅读

### AIArt 主题的原子页

**概念(5)**:
- [[Wiki/Concepts/AIArt/Lora]] — LoRA 技术原理 + 四种类型 + 秩/alpha 等关键参数
- [[Wiki/Concepts/AIArt/Base-model-selection]] — 基座选型的完整对比表 + 许可陷阱
- [[Wiki/Concepts/AIArt/Caption-strategy]] — Caption 反常识原则 + 两种格式 + 实操工作流
- [[Wiki/Concepts/AIArt/Trigger-word]] — 触发词 4 原则 + 常见错误
- [[Wiki/Concepts/AIArt/Multi-lora-composition]] — 多 LoRA 组合 + MJ 对比 + strength 经验

**实体(5)**:
- [[Wiki/Entities/AIArt/Illustrious-XL]] — Illustrious XL 基座详情
- [[Wiki/Entities/AIArt/NoobAI-XL]] — NoobAI XL 基座详情(推荐)
- [[Wiki/Entities/AIArt/Flux]] — Flux.1 的三个版本 + 许可陷阱
- [[Wiki/Entities/AIArt/Kohya-ss]] — 训练工具完整指南
- [[Wiki/Entities/AIArt/ComfyUI]] — 部署工具完整指南

**源(1)**:
- [[Wiki/Sources/AIArt/Lora-deep-dive]] — 源文档摘要(含 15 章 + 3 附录的全貌)

### 原始资料

- [[Raw/Notes/Lora_Deep_Dive]] — 原始 LoRA 深度指南全文

### 跨主题联系

- [[Readers/Methodology/从 Memex 到 LLM Wiki|方法论读本]] — 本 wiki 的组织原理,本读本是方法论的应用例
- [[Wiki/Concepts/AIApps/Llm|LLM 概念]] — AI 模型的基础
- [[Wiki/Concepts/AIApps/Hallucination|幻觉]] — AI 输出的通用问题,Diffusion 模型也有对应表现

### 外部参考

- kohya_ss GUI:`github.com/bmaltais/kohya_ss`
- ComfyUI:`github.com/comfyanonymous/ComfyUI`
- NoobAI XL:见 HuggingFace
- Illustrious XL:见 HuggingFace / Civitai(注意条款)

---

*本读本由 [[Claudian]] 基于 AIArt 议题的 11 个原子页 + LoRA 深度指南原文综合生成,2026-04-20。*
