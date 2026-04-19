---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [base-model, flux, realistic, black-forest-labs]
sources: 1
aliases: [Flux, Flux.1, FLUX, Black Forest Labs Flux]
---

# Flux.1

> Black Forest Labs 的 12B 参数扩散模型，2026 年初**写实 SOTA**。但风格化能力较弱，Dev 版本**非商用许可**，对游戏项目需警惕。

---

## 概览

- **架构**：新架构（非 SDXL 家族），12B 参数
- **原生分辨率**：1024×1024 及更高
- **风格倾向**：**写实为主**
- **三个版本**：Dev / Schnell / Pro，许可和用途各不相同

---

## 三个版本的关键区别

| 版本 | 许可 | 速度 | 质量 | 商用？ |
|---|---|---|---|---|
| **Flux.1 Dev** | **非商业许可** | 中等 | **SOTA** | ❌ **不能商用** |
| **Flux.1 Schnell** | Apache 2.0 | 快 | 中（低于 Dev） | ✅ 可商用 |
| **Flux.1 Pro** | 仅 API 访问 | - | SOTA | ✅（通过 API） |

⚠️ **游戏公司必看**：Flux.1 Dev 权重**禁止商业使用**，即使训出的 LoRA 也继承这一限制。商用只能走 Schnell 或 Pro API。

---

## 关键事实 / 属性

| 属性 | 值 |
|---|---|
| 参数量 | 12B（比 SDXL 2.6B 大近 5 倍） |
| VRAM（训练） | **24GB+** |
| VRAM（推理） | 24GB+（或启用 quantization） |
| 架构 | 非 SDXL（不同的 transformer 设计） |
| Caption 格式 | 自然语言（不是 Danbooru tag） |

---

## 对风格化游戏的评价

**不推荐 Flux**（对二次元/卡通游戏）：

1. **写实感过强**：画风化能力反而弱于 SDXL 系的 anime fine-tune
2. **Dev 非商用**：游戏公司用就是法律风险
3. **Schnell 质量打折**：能用但不如 Illustrious/NoobAI 在 anime 场景
4. **LoRA 生态不成熟**：相对 SDXL 还在早期
5. **硬件门槛高**：24GB+ VRAM，团队硬件成本翻倍

对**写实向项目**才考虑。

---

## 什么场景选 Flux

- 写实 3D 人物 / 环境 concept
- 纪实风 keyart
- 需要精细文字渲染（Flux 对 prompt 里的文字支持比 SDXL 好）

即使如此，**商用务必确认版本**：
- 内部迭代/探索 → Dev（不对外）
- 正式产出对外用 → 必须 Schnell 或 Pro API

---

## 相关

- [[Wiki/Concepts/AIArt/Base-model-selection]] — 基座选型对比（写实 vs 风格化）
- [[Wiki/Entities/AIArt/Illustrious-XL]] — 二次元向的对立面选择
- [[Wiki/Concepts/AIArt/Lora]] — Flux 也支持 LoRA，但生态弱

## 引用来源

- [[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **Schnell 在 anime LoRA 场景的实测效果**（文章只给了定性评价）
- **Flux 后续版本**（Flux.2 / 新迭代）的商用许可变化
- **Quantization（FP8/INT4）**：能否把 VRAM 压到 12GB 以下可用？
