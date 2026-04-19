# LoRA 深度指南 —— 给鸣潮美术向流水线的落地方案

> 面向：TA / 技术背景的美术负责人
> 目标：在 tag 库方案之外，用 LoRA 建立**团队自己**的可控风格管线

---

## 一、LoRA 是什么（30 秒版）

全量微调一个 SDXL 模型要改 25 亿参数，训练要大卡、要大数据、出来还要存几个 GB。LoRA（Low-Rank Adaptation）做的是另一件事：**冻结主模型，只在关键层（cross-attention 为主）旁边插一对低秩矩阵 A×B**，只训这对小矩阵。

- 参数量从 25 亿降到几百万
- 训练文件从 7GB 变成 30-200MB
- 推理时挂上去就生效，不挂就是原模型
- 可以**多个 LoRA 同时挂载**，每个独立调强度

对游戏美术来说，LoRA 的意义在于：**用极低的训练成本，让通用大模型学会团队自己的美术 DNA**。这是 MidJourney 永远做不到的事——MJ 再强也是通用模型，风格不可锁定，不能私有化，不能版本化。

---

## 二、游戏美术场景下的四种 LoRA

| 类型 | 典型用途 | 数据量 | 训练难度 |
|---|---|---|---|
| **风格 LoRA** | 锁定鸣潮整体画风 | 30-80 张 | 中（最常用，最重要） |
| **角色 LoRA** | 复现特定角色 | 15-30 张 | 低 |
| **概念 LoRA** | 特定特效/道具/怪物品类 | 20-50 张 | 中 |
| **构图/姿势 LoRA** | 镜头语言、动作 pose | 20-50 张 | 中高 |

**优先级建议**：先做 1 个**核心风格 LoRA** 打底，把团队美术 DNA 固化。后续按需做角色/概念 LoRA 叠加。

---

## 三、基础模型选型（最关键的战略决策）

LoRA 是架在基座模型上的薄层。基座选错，后面全白干。

### 2026 年初的主流候选

| 基座 | 风格倾向 | 参数规模 | VRAM (训练) | 商用许可 | 评价 |
|---|---|---|---|---|---|
| **Illustrious XL** | 二次元 / stylized | SDXL 架构 | 12-16GB | 有限制（需查最新条款） | 当前二次元 SOTA 之一，理解 Danbooru tag |
| **NoobAI XL** | 二次元 / anime | SDXL 架构 | 12-16GB | 开源友好 | Illustrious 的社区继续训练版，tag 覆盖更广 |
| **Pony Diffusion XL v6/v7** | 卡通 / 二次元 | SDXL 架构 | 12-16GB | 开源 | tag 识别强，对 NSFW 处理激进（团队用要注意） |
| **SDXL base 1.0** | 通用 | 2.6B | 12-16GB | 开源 | 中立基座，anime 能力弱于上面三位 |
| **Flux.1 Dev** | 写实为主 | 12B | 24GB+ | **非商用** | 写实 SOTA，stylized 较弱，**不能商用** |
| **Flux.1 Schnell** | 写实 | 12B | 24GB+ | Apache 2.0 | 可商用但质量低于 Dev |
| **SD 1.5** | 全风格 | 860M | 8GB | 开源 | 生态最丰富但分辨率低，不推荐新项目 |

### 对鸣潮的推荐

**首选：Illustrious XL 或 NoobAI XL**

理由：
- 二次元风格化游戏的画风，和 anime 基座的先验对得上，训练收敛快
- SDXL 架构成熟，工具链完整，LoRA 生态繁荣
- 1024×1024 原生分辨率，出图素质够做 concept reference
- VRAM 要求一张 3090/4090 能搞定

**不推荐 Flux**：
- 写实感过强，画风化能力反而弱
- Dev 版本**不允许商业使用**，游戏公司用就是法律风险
- 训练要求高，LoRA 生态相对不成熟

**注意**：这个领域每 3-6 个月会有新基座出现（2025 下半年的 HiDream、Chroma 等都在迭代）。**实际动工前务必再查一次当前 state**——但方法论通用，基座换了只是重选的事。

---

## 四、数据集准备（成败关键）

数据集的质量决定了 LoRA 能不能用。这一步不能偷懒。

### 风格 LoRA 的数据集构建

**4.1 数量**
- 下限：20 张
- 甜点：40-60 张
- 上限：100 张（再多容易过拟合或稀释风格特征）

**4.2 选图原则**
- **全部出自同一画风/同一画师/同一作品集**——风格 LoRA 最怕"平均风格"
- 覆盖多样的**主体和构图**（人像、全身、场景、物件各种都要有），但**画风保持一致**
- 不同色调都要有样本（白天/夜晚/室内/逆光）
- **去掉明显水印、签名、UI、对话框**
- **分辨率至少 1024×1024**（SDXL），越高越好（支持 bucket）

**4.3 清洗流程**
```
原始图 → 裁剪去 UI → 去水印（有的话）→ 超分到 1024+ → 按 aspect ratio 分组
```
工具：Photoshop 批处理 / Topaz Gigapixel / upscayl

**4.4 Aspect Ratio Bucketing**
SDXL 训练支持多比例分组，不用裁成正方形。常见 bucket：
```
1024×1024 (1:1)
896×1152 (3:4)
1152×896 (4:3)
832×1216 (2:3)
1216×832 (3:2)
768×1344 (竖)
1344×768 (横)
```
kohya_ss 开启 `enable_bucket` 即可。

---

## 五、Caption 策略（反常识重点）

> **核心原则：你想让 LoRA "永远带"的特征，caption 里不要写；你想让它"按需出现"的内容，caption 里写清楚。**

### 为什么？

LoRA 学的是"输入 caption → 输出图"的映射差。caption 里**没有**描述但**图里有**的东西，就会被归因到"LoRA 本身"。caption 里写了的东西，LoRA 会学成"这个词对应这个视觉"。

所以对**风格 LoRA**：
- 图里的画风、笔触、色调倾向 → caption 里**不写**（学成 LoRA 的默认效果）
- 图里具体的角色、动作、场景、物件 → caption 里**要写**（学成可控内容）

### Caption 格式选择

**对 anime 基座（Illustrious/NoobAI/Pony）**：用 **Danbooru tag 风格**
```
wwstyle, 1girl, solo, long hair, standing, temple, sunset, looking_at_viewer
```

**对 SDXL base / Flux**：用**自然语言**
```
wwstyle, a young woman with long hair standing in front of a temple at sunset, looking at the viewer
```

### Trigger Word

在每张图的 caption **最前面**加一个独特的触发词，例如 `wwstyle_v1`。原则：
- 必须是**基座模型没学过的词**（别用 anime / style / art 这种常见词，会污染基座）
- 足够独特以便识别
- 版本化（v1/v2 便于迭代）
- 生产时 prompt 里写 `wwstyle_v1` 触发风格，不写就是基座原样

### 自动打标 + 人工修剪工作流

1. 用 **WD14 Tagger** (anime) 或 **BLIP-2** (natural language) 批量自动打标
2. 人工扫一遍，**删除描述画风的 tag**（如 `anime style`、`painting style`、`cel shading`）——这些要让 LoRA 学走
3. 补充描述**主体内容**的 tag
4. 在每条 caption 最前面加 `wwstyle_v1,`

工具链：
- **kohya_ss** 自带 WD14 Tagger
- **BooruDatasetTagManager** 批量编辑 caption（神器）

---

## 六、训练超参推荐（SDXL 风格 LoRA）

这套是在 Illustrious XL / NoobAI XL / SDXL base 上训练**风格 LoRA** 的可用起点：

```yaml
# 架构
network_module: networks.lora
network_dim: 32           # 风格 LoRA 甜点；想要更泛化用 16，想要更像用 64
network_alpha: 16         # 一般取 dim 的一半

# 训练
train_batch_size: 2       # 24GB VRAM 可以开到 4
num_train_epochs: 10      # 先跑 10，看 loss 和 sample 再决定要不要加
learning_rate: 1e-4       # 风格 LoRA 的常用起点
unet_lr: 1e-4
text_encoder_lr: 5e-5     # TE 学习率一般比 UNet 低一半
lr_scheduler: cosine_with_restarts
lr_warmup_steps: 100

# 优化器
optimizer_type: AdamW8bit # 省 VRAM 的标配

# 精度 / 内存
mixed_precision: bf16     # 30 系以上支持；20 系用 fp16
gradient_checkpointing: true
xformers: true
cache_latents: true
cache_text_encoder_outputs: true  # SDXL 推荐开，省很多 VRAM

# 数据
resolution: 1024,1024
enable_bucket: true
bucket_reso_steps: 64
min_bucket_reso: 512
max_bucket_reso: 2048

# 正则化 / 抗过拟合
noise_offset: 0.0357      # 提升暗部对比度，SDXL 推荐开
min_snr_gamma: 5          # 加速收敛且更稳
prior_loss_weight: 1.0

# 保存
save_every_n_epochs: 1    # 每个 epoch 存一版，方便挑
save_precision: fp16
```

**Repetition / epoch 换算**：
- 数据集 50 张，每张 repeat 10 次 → 一个 epoch 500 steps
- 10 epoch 共 5000 steps
- 目标 steps：2000-4000 通常足够，超过容易过拟合

### 角色 LoRA 的超参调整

- `network_dim`: 降到 16 甚至 8（角色特征不需要那么大容量）
- `learning_rate`: 调低到 5e-5
- 数据量 20-30 张，repeat 更高（15-20）

---

## 七、工具链选型

### 训练

| 工具 | 评价 | 推荐度（给你） |
|---|---|---|
| **kohya_ss GUI** | Windows 友好，主流标准，生态最成熟 | ★★★★★ |
| **sd-scripts** | kohya 的底层 CLI，灵活但门槛高 | ★★★（后期上 pipeline 用） |
| **ai-toolkit** (ostris) | 新锐，Flux 支持最好 | ★★★★（如果用 Flux） |
| **OneTrainer** | 纯 GUI，适合完全新手 | ★★★ |
| **Civitai 在线训练** | 懒人方案，但素材要上传到第三方 | ★（**公司数据严禁用**） |

**你的场景**：Windows + RTX 显卡 + 公司素材保密 → **kohya_ss GUI**

### 推理 / 部署

| 工具 | 评价 | 推荐度 |
|---|---|---|
| **ComfyUI** | 节点式，工程化最强，可做复杂管线 | ★★★★★ |
| **Forge** | A1111 的性能优化分支 | ★★★★ |
| **Automatic1111 webui** | 老牌，插件多但迭代慢 | ★★★ |
| **InvokeAI** | 美术友好 UI，商用许可清晰 | ★★★ |

**对接飞书机器人**：**ComfyUI 跑 API mode**，后端用 HTTP 提交 workflow JSON。工程化最稳。

---

## 八、硬件要求

### 训练（SDXL LoRA）

| 显卡 | VRAM | 是否可行 | 备注 |
|---|---|---|---|
| RTX 3060 12GB | 12GB | ⚠️ 勉强 | 开 gradient_checkpointing + batch=1 |
| RTX 3090 / 4090 | 24GB | ✅ 舒适 | 首选 |
| RTX 5090 | 32GB | ✅ 奢侈 | 大 batch 快 |
| A6000 / A100 | 48/80GB | ✅ | 工作站级 |

一个风格 LoRA 在 4090 上跑 4000 steps ≈ 2-3 小时。

### 推理

- SDXL + LoRA：8GB 显存可跑，12GB 舒适
- Flux + LoRA：24GB 起步

### 云方案（省硬件成本）

- **RunPod / vast.ai**：按小时租 A100/H100，0.5-2 USD/h
- **阿里云 A10 / 腾讯云 T4**：国内访问稳，价格略高
- **本地出租 GPU 池**：预算够的话配 2-4 张 4090 的机器做团队共用服务器，长期最划算

---

## 九、训练流程

### 标准流程（首次训风格 LoRA）

```
1. 收集素材（40-60 张）
   ↓
2. 清洗 + 超分（统一到 1024+）
   ↓
3. 用 WD14 Tagger 自动打标
   ↓
4. 人工清 caption（删画风词，加 trigger）
   ↓
5. 配置 kohya（见超参模板）
   ↓
6. 跑 5 epoch 先看趋势
   ↓
7. 每 epoch 出 sample image 监控
   ↓
8. 挑最好的 epoch checkpoint
   ↓
9. XYZ grid 测强度（0.5 / 0.75 / 1.0）
   ↓
10. 美术盲评，选最终版本
```

### Sample 配置（关键监控手段）

kohya 训练时每 N steps 生成几张 sample 图，看趋势：

```
sample_every_n_epochs: 1
sample_prompts: /path/to/prompts.txt
sample_sampler: euler_a
```

prompts.txt 内容：
```
wwstyle_v1, 1girl, standing in a forest --w 768 --h 1024 --s 24 --l 7.5
wwstyle_v1, a mountain temple at dawn --w 1024 --h 768 --s 24 --l 7.5
wwstyle_v1, portrait of a warrior --w 768 --h 1024 --s 24 --l 7.5
```

这三张图每个 epoch 出一次，连起来看就能知道 LoRA 学得怎么样。

---

## 十、评估标准

**风格 LoRA 合格线**：

1. **风格一致性**：画几种完全不同主体（女孩、机械、风景、物件），画风看起来都像"同一个画师"
2. **Prompt 服从性**：加了 LoRA 还能画出训练集里没有的东西（训练集全是白天，强度 0.75 下要能画出夜晚）
3. **可调强度**：LoRA weight 从 0.3 到 1.0，风格强度单调增加，不跳变不崩
4. **不污染基座**：不挂 LoRA 时的出图质量不变（LoRA 文件本身不破坏基座）

**过拟合的信号**：
- 不管写什么 prompt，构图都一样
- 训练集里出现过的元素频繁复现
- 背景或服装细节高度雷同
- Prompt 里的新元素画不出来

**欠拟合的信号**：
- strength 拉到 1.0 画风还是不像
- 基座的默认风格占上风
- 加不加 LoRA 看起来差不多

---

## 十一、Multi-LoRA 组合（ComfyUI 的真正威力）

单个 LoRA 有上限。**多 LoRA 组合**才是真正的生产力。典型栈：

```
基座模型 (Illustrious XL)
  ├─ 风格 LoRA: wwstyle_v1   (strength 0.8)   ← 打底风格
  ├─ 角色 LoRA: wwstyle_encore (strength 0.9) ← 特定角色
  ├─ 灯光 LoRA: dramatic_lighting (strength 0.5) ← 光影增强
  └─ 细节 LoRA: detail_tweaker (strength 0.3)   ← 细节细化
```

ComfyUI 可以挂任意多个 LoRA，每个独立 strength，独立开关。

**这是 MidJourney 永远做不到的事**。MJ 的 `--sref` 只是近似"风格参考"，不是可组合的模块化控件。LoRA 才是真正的**模块化风格控件**。

---

## 十二、和 Tag 库方案的衔接

### 短中长三阶段

| 阶段 | 时间 | 主力工具 | 负责人 |
|---|---|---|---|
| **短期（0-3 个月）** | 立即 | MJ + tag 库 | TA + 美术评审 |
| **中期（3-6 个月）** | 并行启动 | MJ + tag 库生产；**开始训练风格 LoRA** | TA + 美术 + 可选 ML 顾问 |
| **长期（6-12 个月）** | 迁移 | ComfyUI + 自训 LoRA + tag 前端（飞书） | TA 主导 |

### 复用关系

- tag 库验证阶段产出的**高质量 prompt-图对**，未来可以直接作为 LoRA 训练集的 caption 候选
- 飞书前端 tag 选择器**不用重做**，只换后端：tag 拼出来的 prompt 从打到 MJ，改成打到 ComfyUI API
- 美术的使用体验**完全一致**，但风格可控性上一个量级

**设计前提**：tag 库的前端架构要一开始就和后端解耦。别把 MJ API 写死在前端里。

---

## 十三、常见陷阱（花钱买的教训）

| 陷阱 | 后果 | 避坑 |
|---|---|---|
| 数据集混画风 | LoRA 学出"糊糊的平均风格" | 严格单一画风 |
| Caption 写了画风描述 | LoRA 学不到画风 | 画风描述全部删掉 |
| Trigger word 用常用词 | 污染基座，没 trigger 也发作 | 用独特且版本化的词 |
| rank 开太大 | 过拟合，文件大，泛化差 | 风格 32，角色 16 |
| 只看最后一个 epoch | 错过最佳 checkpoint | 每 epoch 存档 + XYZ 测试 |
| 训练集有水印/UI | LoRA 会稳定画出水印/UI | 清洗必须彻底 |
| 分辨率混乱 | 效果飘忽 | 统一到 1024+ 并启用 bucket |
| 训完没做强度测试 | 生产时不知道怎么配 | XYZ grid 是必做 |
| 美术没参与数据筛选 | TA 以为好看，美术说不对 | **美术全程在场**，不可替代 |

---

## 十四、法律与合规

这对公司项目不是可选项。

- **训练数据版权**：最安全是用公司**自有美术素材**。如果用外部 concept art，必须有明确授权。用互联网抓图训练商业项目有法律风险。
- **基座模型许可**：
  - SDXL / Illustrious 系：一般开源可商用（但需查具体许可条款，Illustrious 的条款有过变动）
  - Flux.1 **Dev**：**非商业许可**，商业项目必须用 Schnell 或 Pro
  - Pony：开源但社区分叉多，用具体 checkpoint 前查清楚
- **LoRA 本身的产出**：你训的 LoRA 是你们的。**但不能单独分发未经授权的基座权重**，只能分发 LoRA 文件。
- **生成图的版权**：中国法律近期判例趋向"AI 生成物人类有创作贡献即可主张版权"，但基座的授权链要干净。

---

## 十五、给你团队的落地路线图

基于你现在的情况（鸣潮特效向 TA、Windows 环境、已有 midjourney-proxy 和飞书机器人），建议：

**第 1 个月：准备期（不影响现有 tag 库工作）**
- 一张 4090 机（或租 RunPod A100）
- 装 kohya_ss GUI
- 装 ComfyUI
- 下载 Illustrious XL 和 NoobAI XL 基座（先都留着）
- 从内部 concept art 里挑 50 张试训第一个风格 LoRA（纯实验，不上生产）

**第 2 个月：MVP LoRA**
- 基于第一次试训经验，重新选图、清洗、打标
- 训出 v0.1 风格 LoRA
- 让美术组盲评（LoRA on vs off 各出几张让大家猜哪边更像鸣潮）
- 迭代到 v1.0

**第 3 个月：管线搭建**
- ComfyUI 起 API 服务，写标准 workflow
- 飞书机器人后端加 ComfyUI 路由（和 MJ 路由共存，前端 tag 选择器多一个"后端选择"开关）
- 做 A/B 对比：同一套 tag 组合，MJ vs LoRA 出图，美术评审
- 积累评估数据

**第 4-6 个月：扩展与替换**
- 训 2-3 个角色 LoRA
- 训 1-2 个概念 LoRA（特定特效、特定建筑）
- 根据 A/B 数据决定什么场景用 MJ、什么场景用本地 LoRA
- 逐步把 tag 库后端迁到 ComfyUI

**6 个月后**：有机会完全替代 MJ 的大部分场景，保留 MJ 作为"快速探索"兜底。

---

## 附录 A：kohya_ss 最小可行配置示例

`config.toml`（风格 LoRA，SDXL 基座）：

```toml
[model_arguments]
pretrained_model_name_or_path = "D:/models/Illustrious-XL-v0.1.safetensors"

[dataset_arguments]
train_data_dir = "D:/datasets/wwstyle_v1/img"
resolution = "1024,1024"
enable_bucket = true
min_bucket_reso = 512
max_bucket_reso = 2048
bucket_reso_steps = 64

[training_arguments]
output_dir = "D:/loras/wwstyle_v1"
output_name = "wwstyle_v1"
save_model_as = "safetensors"
save_every_n_epochs = 1
max_train_epochs = 10
train_batch_size = 2
gradient_checkpointing = true
gradient_accumulation_steps = 1
mixed_precision = "bf16"
save_precision = "fp16"
seed = 42
cache_latents = true
cache_text_encoder_outputs = true
xformers = true

[optimizer_arguments]
optimizer_type = "AdamW8bit"
learning_rate = 1e-4
unet_lr = 1e-4
text_encoder_lr = 5e-5
lr_scheduler = "cosine_with_restarts"
lr_warmup_steps = 100
lr_scheduler_num_cycles = 3

[network_arguments]
network_module = "networks.lora"
network_dim = 32
network_alpha = 16
network_train_unet_only = false

[advanced_arguments]
noise_offset = 0.0357
min_snr_gamma = 5

[sample_prompt_arguments]
sample_every_n_epochs = 1
sample_sampler = "euler_a"
sample_prompts = "D:/datasets/wwstyle_v1/sample_prompts.txt"
```

目录结构：
```
D:/datasets/wwstyle_v1/
├── img/
│   └── 10_wwstyle/          ← repeat 次数在文件夹名前缀
│       ├── 001.png
│       ├── 001.txt          ← caption
│       ├── 002.png
│       ├── 002.txt
│       └── ...
├── sample_prompts.txt
└── config.toml
```

---

## 附录 B：ComfyUI 飞书后端集成伪代码

```python
# 飞书机器人收到 tag 选择 → 后端处理
def handle_tag_selection(tag_combo):
    # 拼 prompt
    prompt = assemble_prompt(tag_combo)  # 沿用 tag 库的拼接逻辑

    # 加 LoRA trigger
    prompt = f"wwstyle_v1, {prompt}"

    # 提交 ComfyUI workflow
    workflow = load_workflow_template("ww_style_v1.json")
    workflow["6"]["inputs"]["text"] = prompt       # 正向 prompt 节点
    workflow["10"]["inputs"]["lora_name"] = "wwstyle_v1.safetensors"
    workflow["10"]["inputs"]["strength_model"] = 0.8

    # 提交并等待
    prompt_id = comfyui_client.queue_prompt(workflow)
    images = comfyui_client.wait_for_images(prompt_id)

    # 返回飞书
    send_feishu_card(images)
```

---

## 附录 C：值得关注的几个方向（中长期）

- **DoRA**（Weight-Decomposed Low-Rank Adaptation）：LoRA 的改进版，小数据上表现更好，kohya 已支持
- **LoCon / LoHa / LyCORIS**：LoRA 变体，对深层特征学习更强
- **IP-Adapter**：比 LoRA 更轻量的"风格注入"方案，不用训练
- **ControlNet + LoRA 组合**：线稿约束 + 风格 LoRA，是 concept art 流水线的王炸组合
- **训练数据回流 LoRA**：生产中被美术评分高的出图自动入训练池，季度 retrain——长期美术风格自动进化
