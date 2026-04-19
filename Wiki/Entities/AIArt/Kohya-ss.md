---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [tool, lora, training, kohya]
sources: 1
aliases: [kohya, kohya_ss, Kohya SS, sd-scripts]
---

# Kohya-ss

> 当前主流的 [[Wiki/Concepts/AIArt/Lora|LoRA]] 训练工具，Windows 友好，生态最成熟。对鸣潮这类 Windows + RTX 显卡 + 公司素材保密的场景是**首选**。

---

## 概览

- **类型**：LoRA / DreamBooth / 全量微调训练工具
- **本体**：kohya_ss GUI（Windows 友好）
- **底层**：sd-scripts（CLI，更灵活）
- **平台**：Windows / Linux / macOS（Windows 支持最好）
- **仓库**：`github.com/bmaltais/kohya_ss`（GUI）、`github.com/kohya-ss/sd-scripts`（CLI）

---

## 关键事实 / 属性

| 属性 | 值 |
|---|---|
| 推荐度（风格 LoRA） | ★★★★★ |
| 最小显存 | 12GB（启用 gradient_checkpointing） |
| 推荐显存 | 24GB（RTX 3090/4090） |
| 自带工具 | WD14 Tagger（auto-caption） |
| 支持基座 | SDXL / SD 1.5 / Flux 等主流 |
| 学习曲线 | 中等（GUI 帮助大） |

---

## 核心能力

- **LoRA 训练**：network_dim / alpha / learning_rate 全可调（见 [[Wiki/Sources/AIArt/Lora-deep-dive]] 超参模板）
- **DreamBooth**：全量微调
- **LyCORIS**：支持 LoCon / LoHa / DoRA 等 LoRA 变体
- **Aspect Ratio Bucketing**：`enable_bucket` 自动多比例分组训练
- **WD14 Tagger**：内置，给 anime 图批量打 Danbooru tag
- **Sample Image**：训练中每 N steps 出 sample 图监控 loss

---

## 训练配置文件结构（示例）

```toml
[model_arguments]
pretrained_model_name_or_path = "D:/models/Illustrious-XL-v0.1.safetensors"

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

完整示例见 [[Wiki/Sources/AIArt/Lora-deep-dive]] 附录 A。

---

## 数据目录约定

```
D:/datasets/wwstyle_v1/
├── img/
│   └── 10_wwstyle/          ← "10_" 是 repeat 次数
│       ├── 001.png
│       ├── 001.txt          ← 同名 .txt 就是 caption
│       └── ...
├── sample_prompts.txt
└── config.toml
```

文件夹名前缀 `10_` = 每张图训练 10 次。数据集 50 张 × repeat 10 = 一个 epoch 500 steps。

---

## 和同类工具对比

| 工具 | 场景 | 推荐度 |
|---|---|---|
| **kohya_ss GUI** | Windows 主力，风格 LoRA 训练 | ★★★★★ |
| sd-scripts | kohya 的底层 CLI，灵活但门槛高 | ★★★（后期上 pipeline 用） |
| ai-toolkit (ostris) | 新锐，Flux 支持最好 | ★★★★（Flux 场景） |
| OneTrainer | 纯 GUI，新手友好 | ★★★ |
| Civitai 在线训练 | 懒人方案 | ★（**公司数据严禁用**） |

---

## 选它的理由（对鸣潮）

- Windows 环境原生支持
- 公司素材**本地训练**，不上传第三方
- 文档和教程多
- 和 [[Wiki/Entities/AIArt/ComfyUI]] 配合好（kohya 训、ComfyUI 推）

---

## 相关

- [[Wiki/Concepts/AIArt/Lora]] — 训练的技术
- [[Wiki/Concepts/AIArt/Caption-strategy]] — kohya 里 caption 规则的落地
- [[Wiki/Entities/AIArt/ComfyUI]] — 训完 LoRA 的下游推理工具
- [[Wiki/Entities/AIArt/Illustrious-XL]] / [[Wiki/Entities/AIArt/NoobAI-XL]] — 常用基座

## 引用来源

- [[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **kohya_ss 的 DoRA 支持稳定性**：已支持但最佳超参待积累
- **训练日志监控**：除 sample image 外，是否有更好的可视化（TensorBoard / wandb 集成）？
