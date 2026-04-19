---
type: source
created: 2026-04-19
updated: 2026-04-19
tags: [lora, diffusion, game-art, pipeline, kuro, wuthering-waves]
sources: 1
aliases: [LoRA 深度指南, Lora Deep Dive]
source_type: note
---

# LoRA 深度指南 — 给鸣潮美术向流水线的落地方案

- **原文**：[[Raw/Notes/Lora_Deep_Dive]]
- **作者**：Eureka（鸣潮美术向 TA,本 vault 主人）与 Claude 共同撰写
- **日期**：2026-04
- **面向**：TA / 技术背景的美术负责人（即作者自身的决策参考）
- **类型**：note（技术落地指南 / 方案建议）
- **方案阶段**：**准备期**——尚在评估 / 选型，未开始 MVP 训练

---

## 摘要

本文是一份面向鸣潮美术向 TA 的 **LoRA 落地完整方案**，覆盖从原理到生产管线的全链路。核心主张：在现有 MidJourney + tag 库方案之外，用 LoRA 建立团队**自己的可控风格管线**。因为 MJ 永远是通用模型，风格不可锁定；而 LoRA 能把团队美术 DNA 固化为 30-200MB 的文件，可版本化、可私有化、可组合。

文章分为 15 个章节 + 3 个附录，从 LoRA 原理（冻结主模型 + 低秩矩阵 A×B）、四种 LoRA 类型（风格/角色/概念/姿势）、基座选型（Illustrious XL / NoobAI XL 首选，Flux 因商用许可和风格偏差不推荐）、数据集准备（40-60 张、单一画风、统一到 1024+）、**Caption 反常识策略**（想让 LoRA 永远带的不写、想按需触发的写清楚）、训练超参（rank 32 / lr 1e-4 / bf16）、工具链（kohya_ss 训练 + ComfyUI 部署）、多 LoRA 组合、合规与法律，到 6 个月团队落地路线图。

对鸣潮管线的核心建议：**tag 库前端保留，后端从 MJ 逐步迁移到 ComfyUI + 自训 LoRA**，复用飞书机器人 UI，美术使用体验不变但风格可控性上一个量级。

---

## 关键要点

### 战略层

- **MJ 永远做不到的事**：MJ 是通用模型，风格不可锁定、不可私有化、不可版本化；LoRA 可以
- **短中长三阶段路线**：现在 MJ+tag → 3-6 月 MJ+并行训练 → 6-12 月 ComfyUI+自训 LoRA 主力，MJ 作为探索兜底
- **前端后端解耦**：tag 库前端不重做，只切后端路由（MJ → ComfyUI）

### 技术层

- **四种 LoRA**：风格、角色、概念、姿势。优先级 = 风格 LoRA 打底
- **基座选型**（2026 年初）：Illustrious XL / NoobAI XL 是二次元首选；Flux Dev **禁止商用**，对游戏公司是法律红线
- **Caption 反常识核心**：LoRA 学的是 caption→图的**映射差**。caption 没写但图里有的 → 学成默认；caption 写了的 → 学成可触发内容
- **Trigger Word 规则**：放在每条 caption 最前面，必须独特且版本化（`wwstyle_v1`），避免常见词（`anime/style/art`）污染基座
- **超参推荐**：`network_dim=32`（风格）、`network_alpha=16`、`lr=1e-4`、`AdamW8bit`、`bf16`、`cache_text_encoder_outputs=true`
- **评估准则**：风格一致 + prompt 服从 + 可调 strength + 不污染基座（挂 vs 不挂出图质量不变）

### 工具层

- **训练**：kohya_ss GUI（Windows 主流）
- **推理/部署**：ComfyUI（API mode + workflow-as-code）
- **素材管理**：BooruDatasetTagManager（批量 caption 编辑）
- **WD14 Tagger**（anime）/ BLIP-2（natural language）做自动打标

### 合规层

- 训练数据优先用公司自有素材，避免互联网抓图
- Illustrious 的许可条款历史变动过，每次下载重新确认
- Flux Dev 非商用、Schnell 可商用但质量打折
- Civitai 在线训练**公司数据严禁用**

### 硬件层

- 训练：RTX 3090/4090（24GB）舒适；3060（12GB）勉强
- 推理：SDXL+LoRA 8GB 可跑；Flux+LoRA 24GB 起
- 云方案：RunPod / vast.ai（0.5-2 USD/h A100）

---

## 涉及实体 / 概念

**概念**：
- [[Wiki/Concepts/AIArt/Lora|LoRA]] — 全文核心技术
- [[Wiki/Concepts/AIArt/Base-model-selection|基座模型选型]] — 最关键战略决策
- [[Wiki/Concepts/AIArt/Caption-strategy|Caption 策略]] — 反常识重点
- [[Wiki/Concepts/AIArt/Trigger-word|Trigger Word]] — 命名约定
- [[Wiki/Concepts/AIArt/Multi-lora-composition|Multi-LoRA 组合]] — 生产力倍增器

**实体**：
- [[Wiki/Entities/AIArt/Illustrious-XL|Illustrious XL]] — 首选基座
- [[Wiki/Entities/AIArt/NoobAI-XL|NoobAI XL]] — 首选基座（许可更友好）
- [[Wiki/Entities/AIArt/Flux|Flux]] — 写实向基座（对游戏公司有许可陷阱）
- [[Wiki/Entities/AIArt/Kohya-ss|Kohya-ss]] — 首选训练工具
- [[Wiki/Entities/AIArt/ComfyUI|ComfyUI]] — 首选推理/部署工具

---

## 落地路线图（原文核心输出）

基于鸣潮团队现状（Windows + 4090 + midjourney-proxy + 飞书机器人）：

| 阶段 | 时间 | 主力工具 |
|---|---|---|
| **M1：准备** | 1 月 | kohya_ss + ComfyUI 装好，下 Illustrious/NoobAI，试训第一个风格 LoRA（不上生产） |
| **M2：MVP LoRA** | 2 月 | 重新选图 → 训 v0.1 → 美术盲评 → 迭代到 v1.0 |
| **M3：管线搭建** | 3 月 | ComfyUI API + 飞书后端路由 + MJ vs LoRA A/B 对比 |
| **M4-M6：扩展** | 4-6 月 | 训角色/概念 LoRA，根据 A/B 数据决定替换策略 |
| **6 月后** | - | ComfyUI 主力，MJ 作为快速探索兜底 |

---

## 附录速查

- **附录 A**：kohya_ss 最小可行 `config.toml` 配置
- **附录 B**：ComfyUI 飞书后端集成伪代码
- **附录 C**：值得关注的中长期方向（DoRA、LyCORIS、IP-Adapter、ControlNet+LoRA、训练数据回流）

---

## 与既有 wiki 的关系

- **新增**：本 wiki 首次覆盖 AI 美术生成话题；新建 `AIArt` 主题目录下 5 概念 + 5 实体
- **印证**：暂无——这是 wiki 里该主题的首个源
- **矛盾**：暂无

---

## 开放问题（原文留白 + 需本地验证）

- **基座实时选择**：每 3-6 月重查一次，2026-04 此刻的实际选择需要自己实测
- **数据集上限**：文章给 100 张上限，但大型团队做产业级 LoRA（上千张数据）的经验未覆盖
- **LoRA 训练的团队协作流程**：训练任务分配、模型版本管理、美术盲评的 SOP 未详述
- **Tag 库前端的技术债评估**：如果现在前后端耦合严重，切换后端的工作量估算？
- **法律边界的具体 case**：用"自有美术+部分公开素材"训练是否安全？需要法务介入
