---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [lora, diffusion, sdxl, fine-tuning, 核心概念]
sources: 1
aliases: [LoRA, Low-Rank Adaptation, 低秩适配]
---

# LoRA (Low-Rank Adaptation)

> 在冻结的主扩散模型旁边插入一对低秩矩阵 A×B，只训练这对小矩阵——用极低的成本给大模型打补丁。对游戏美术来说：**用几 MB 的文件让通用大模型学会团队自己的美术 DNA**。

---

## 概览

全量微调一个 SDXL 模型要改 25 亿参数，训练要大卡、要大数据、出来还要存几个 GB。LoRA 做的是另一件事：

- **冻结主模型**，参数不动
- **在关键层（主要是 cross-attention）旁边插入 A×B 低秩矩阵**
- 只训练这对小矩阵

由此换来的工程红利：

| 维度 | 全量微调 | LoRA |
|---|---|---|
| 参数量 | 25 亿 | 几百万 |
| 文件体积 | 约 7GB | 30-200MB |
| 训练硬件 | 多卡 | 一张 3090/4090 |
| 部署 | 换整模型 | 挂上去就生效，不挂就是原模型 |
| 组合 | 不可 | 多 LoRA 同时挂载，独立调强度 |

---

## 对游戏美术的独特意义

LoRA 是 **MidJourney 永远做不到的事**：

- MidJourney 是通用模型，风格不可锁定、不可私有化、不可版本化
- LoRA 可以把团队的美术 DNA **固化成文件**，纳入版本控制，像代码一样迭代
- 多 LoRA 可组合（见 [[Wiki/Concepts/AIArt/Multi-lora-composition]]）→ 模块化风格控件

---

## 四种 LoRA 类型

| 类型 | 典型用途 | 数据量 | 训练难度 |
|---|---|---|---|
| **风格 LoRA** | 锁定整体画风 | 30-80 张 | 中（最常用） |
| **角色 LoRA** | 复现特定角色 | 15-30 张 | 低 |
| **概念 LoRA** | 特定特效/道具/怪物品类 | 20-50 张 | 中 |
| **构图/姿势 LoRA** | 镜头语言、动作 pose | 20-50 张 | 中高 |

**优先级建议**：先做 1 个**核心风格 LoRA** 打底，把团队美术 DNA 固化。后续按需做角色/概念 LoRA 叠加。

---

## 关键技术细节

### 秩（rank/network_dim）

- **风格 LoRA**：甜点 32；想更泛化用 16，想更像用 64
- **角色 LoRA**：16 甚至 8（角色特征不需要那么大容量）
- rank 太大 → 过拟合、文件大、泛化差

### Alpha

`network_alpha` 一般取 `network_dim` 的一半。控制实际学习率缩放。

### 训练参数

- CPU 基础学习率：`1e-4`（UNet）、`5e-5`（Text Encoder）
- 目标 steps：2000-4000 通常足够
- 优化器：`AdamW8bit`（省 VRAM）

详见 [[Wiki/Sources/AIArt/Lora-deep-dive]] 的超参模板。

---

## 依赖的其他决策

LoRA 本身只是技术，要落地还涉及几个串联决策：

1. **选基座** → [[Wiki/Concepts/AIArt/Base-model-selection]]
2. **准备数据集 + 打标签** → [[Wiki/Concepts/AIArt/Caption-strategy]]
3. **设计触发词** → [[Wiki/Concepts/AIArt/Trigger-word]]
4. **生产部署时组合多个 LoRA** → [[Wiki/Concepts/AIArt/Multi-lora-composition]]

---

## 相关

- [[Wiki/Concepts/AIArt/Base-model-selection]] — LoRA 必须挂在某个基座上
- [[Wiki/Concepts/AIArt/Caption-strategy]] — 决定 LoRA 学到什么、不学到什么
- [[Wiki/Concepts/AIArt/Trigger-word]] — 激活 LoRA 的命名约定
- [[Wiki/Concepts/AIArt/Multi-lora-composition]] — LoRA 的组合生产力
- [[Wiki/Entities/AIArt/Kohya-ss]] — 训练工具
- [[Wiki/Entities/AIArt/ComfyUI]] — 推理/组合部署工具

## 引用来源

- 主题读本(推荐通读):[[Readers/AIArt/LoRA 深度指南]]
- 原子 source:[[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **DoRA**（Weight-Decomposed LoRA）：LoRA 的改进版，小数据表现更好，kohya 已支持——尚未系统评估
- **LyCORIS 家族**（LoCon / LoHa）：对深层特征学习更强，和 LoRA 的取舍待研究
- **IP-Adapter**：比 LoRA 更轻的"风格注入"方案，不用训练——是替代还是补充？
- **训练数据回流**：生产中被美术评分高的出图自动入训练池，季度 retrain——未验证的长期演进路径
