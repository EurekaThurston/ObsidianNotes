---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [base-model, diffusion, sdxl, flux, 选型]
sources: 1
aliases: [基座模型选型, Base Model, 基座选择]
---

# 基座模型选型（Base Model Selection）

> LoRA 是架在基座模型上的薄层。**基座选错，后面全白干**。这是整个 AI 美术流水线的最关键战略决策。

---

## 概览

训练 [[Wiki/Concepts/AIArt/Lora|LoRA]] 之前，必须先选一个基座（base model）。基座决定了：

- **风格先验**（anime / 写实 / 通用）——决定训练是否收敛得顺
- **架构与 VRAM 要求**——决定硬件门槛
- **商业许可**——决定能否用于游戏项目
- **生态成熟度**——决定可用 LoRA/工具链/插件的规模

---

## 2026 年初的主流候选

| 基座 | 风格倾向 | 参数规模 | VRAM (训练) | 商用许可 | 评价 |
|---|---|---|---|---|---|
| [[Wiki/Entities/AIArt/Illustrious-XL\|Illustrious XL]] | 二次元 / stylized | SDXL 架构 | 12-16GB | 有限制（查最新条款） | 二次元 SOTA 之一，识别 Danbooru tag |
| [[Wiki/Entities/AIArt/NoobAI-XL\|NoobAI XL]] | 二次元 / anime | SDXL 架构 | 12-16GB | 开源友好 | Illustrious 的社区继续训练版，tag 覆盖更广 |
| Pony Diffusion XL v6/v7 | 卡通 / 二次元 | SDXL 架构 | 12-16GB | 开源 | tag 识别强，但对 NSFW 激进（团队用要过滤） |
| SDXL base 1.0 | 通用 | 2.6B | 12-16GB | 开源 | 中立基座，anime 能力弱于上面三位 |
| [[Wiki/Entities/AIArt/Flux\|Flux.1 Dev]] | 写实为主 | 12B | 24GB+ | **非商用** | 写实 SOTA，**不能商用**，stylized 较弱 |
| [[Wiki/Entities/AIArt/Flux\|Flux.1 Schnell]] | 写实 | 12B | 24GB+ | Apache 2.0 | 可商用但质量低于 Dev |
| SD 1.5 | 全风格 | 860M | 8GB | 开源 | 生态最丰富但分辨率低，不推荐新项目 |

---

## 风格化游戏（二次元/卡通）的推荐

**首选**：Illustrious XL 或 NoobAI XL

理由：
- 画风和 anime 基座的先验对得上，训练收敛快
- SDXL 架构成熟，工具链完整，LoRA 生态繁荣
- 1024×1024 原生分辨率，出图素质够做 concept reference
- VRAM 要求一张 3090/4090 能搞定

**不推荐 Flux**（对二次元游戏）：
- 写实感过强，画风化能力反而弱
- Dev 版本**不允许商业使用**，游戏公司用就是法律风险
- 训练要求高（24GB+），LoRA 生态相对不成熟

---

## 选型的永恒矛盾

这个领域迭代极快——每 3-6 个月会有新基座出现（HiDream、Chroma 等都在迭代）。

**原则**：
- **实际动工前务必再查一次当前 state**，不要拿半年前的推荐直接用
- **方法论通用**，基座换了只是重选一次的事
- **尽量选 SDXL 架构**（目前生态最稳），不要刚出的新架构就all-in

---

## 许可陷阱（最容易被忽略）

| 基座 | 陷阱 |
|---|---|
| Flux.1 Dev | **非商业许可**，游戏公司不能用 |
| Illustrious | 条款有过变动，每次下载版本都应重新确认 |
| Pony | 开源但分叉多，每个 checkpoint 条款可能不同 |
| SDXL base | 开源，但**训练集里的版权内容**仍有法律灰色 |

对公司项目：**优先用自有美术素材训练，基座选许可清晰的 SDXL 系**。

---

## 相关

- [[Wiki/Concepts/AIArt/Lora]] — 基座的用途：承载 LoRA
- [[Wiki/Entities/AIArt/Illustrious-XL]] — 首选之一
- [[Wiki/Entities/AIArt/NoobAI-XL]] — 首选之一（社区继续训）
- [[Wiki/Entities/AIArt/Flux]] — 写实场景才考虑，注意 Dev 非商用

## 引用来源

- 主题读本(推荐通读):[[Readers/AIArt/Lora-深度指南-读本]]
- 原子 source:[[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **新架构可能窗口**：2025 下半年的 HiDream、Chroma 进展如何？何时是从 SDXL 迁移的拐点？
- **许可追溯**：Illustrious 的条款历史变更是否影响已训 LoRA 的使用？
- **混用基座**：同一项目里不同场景用不同基座（如角色用 Illustrious、写实道具用 Flux Schnell）是否可行？
