---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [base-model, sdxl, anime, noobai]
sources: 1
aliases: [NoobAI, NoobAI XL, NoobAIXL]
---

# NoobAI XL

> [[Wiki/Entities/AIArt/Illustrious-XL|Illustrious XL]] 的社区继续训练版——tag 覆盖更广，开源许可更友好。对游戏公司项目常是**比 Illustrious 更稳妥的选择**。

---

## 概览

- **架构**：SDXL（继承自 Illustrious XL）
- **血缘**：Illustrious XL → 社区用更大数据集继续训 → NoobAI XL
- **Tag 系统**：扩展后的 Danbooru tag
- **生态**：开源社区主导，LoRA 生态和 Illustrious 互通

---

## 关键事实 / 属性

| 属性 | 值 |
|---|---|
| VRAM（训练） | 12-16GB |
| VRAM（推理） | 8-12GB |
| 最佳 Caption 格式 | Danbooru tag 风格（见 [[Wiki/Concepts/AIArt/Caption-strategy]]） |
| 商用许可 | **开源友好** |
| 适合 | 二次元 / stylized 游戏 concept art |
| 优势 | tag 覆盖比 Illustrious 更广 |

---

## 相对于 Illustrious XL 的优势

1. **许可更清晰**：开源友好，商用门槛低
2. **Tag 覆盖更广**：社区继续训练时用了更大的数据集
3. **持续迭代**：社区驱动，更新频率可能更高

**相对劣势**：
- 作为后训版本，在某些原版擅长的场景可能**细节略输**
- 社区版本治理相对松散，需要自己甄别版本质量

---

## 适用场景

对鸣潮这类**二次元风格化游戏**：
- 如果 Illustrious 的许可有顾虑 → **优先选 NoobAI XL**
- 如果需要最广 tag 覆盖 → NoobAI XL
- 如果已经在 Illustrious 上训好 LoRA，一般可直接用在 NoobAI 上（血缘同源，基本兼容）

---

## 训练 LoRA 的超参

和 [[Wiki/Entities/AIArt/Illustrious-XL]] 一致（SDXL 架构通用配置）：
- `network_dim: 32`（风格 LoRA）
- `learning_rate: 1e-4`
- Caption 用 Danbooru tag

详见 [[Wiki/Sources/AIArt/Lora-deep-dive]] 的超参模板。

---

## 相关

- [[Wiki/Entities/AIArt/Illustrious-XL]] — 母版
- [[Wiki/Concepts/AIArt/Base-model-selection]] — 基座选型分析
- [[Wiki/Concepts/AIArt/Lora]] — 挂载技术

## 引用来源

- [[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **具体版本**：2026-04 时的社区主流版本号是什么？
- **Illustrious 上训的 LoRA 直接用在 NoobAI 上**，效果实测差异多大？
- **NoobAI 训的 LoRA 兼容 Pony 吗**？
