---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [base-model, sdxl, anime, illustrious]
sources: 1
aliases: [Illustrious, Illustrious XL, IllustriousXL]
---

# Illustrious XL

> SDXL 架构的二次元 / stylized 基座模型，2026 年初二次元 SOTA 之一，理解 Danbooru tag，是风格化游戏美术训练 [[Wiki/Concepts/AIArt/Lora|LoRA]] 的首选基座。

---

## 概览

- **架构**：SDXL（2.6B 参数基座 + 风格化二次元继续训练）
- **原生分辨率**：1024×1024
- **Tag 系统**：Danbooru tag
- **生态**：2026 初社区主流选择之一，LoRA 数量丰富

---

## 关键事实 / 属性

| 属性 | 值 |
|---|---|
| VRAM（训练） | 12-16GB（一张 3090/4090 够用） |
| VRAM（推理） | 8-12GB |
| 最佳 Caption 格式 | Danbooru tag 风格（见 [[Wiki/Concepts/AIArt/Caption-strategy]]） |
| 商用许可 | **有限制，需查最新条款**（历史上有过变动） |
| 适合 | 二次元 / stylized 游戏 concept art |
| 不适合 | 写实风格、纪实摄影 |

---

## 适用场景

**适合**：
- 二次元风格化游戏（如鸣潮）的美术 LoRA 训练基座
- 需要精细 tag 控制的 concept art 管线
- 想要快速收敛的 anime 画风（和基座先验对齐）

**不适合**：
- 写实向项目（选 [[Wiki/Entities/AIArt/Flux|Flux]]）
- 通用中立基座（选 SDXL base 1.0）

---

## 和同类的对比

| | Illustrious XL | [[Wiki/Entities/AIArt/NoobAI-XL\|NoobAI XL]] | Pony Diffusion XL |
|---|---|---|---|
| 血缘 | 原版 | Illustrious 社区继续训 | 独立训练 |
| Tag 覆盖 | 中等 | 更广 | 激进（NSFW 倾向明显） |
| 许可 | 有限制 | 开源友好 | 开源 |
| 团队用适配 | 看条款 | 较安全 | NSFW 要过滤 |

---

## 许可陷阱

**Illustrious 的条款历史有过变动**。每次下载新版本，必须：
1. 重新阅读当前许可
2. 确认商用条款
3. 留存下载时的条款截图作为合规证据

对游戏公司：**如果许可存疑，用 NoobAI XL 作为继任**。

---

## 相关

- [[Wiki/Entities/AIArt/NoobAI-XL]] — 社区继续训练版（许可更友好）
- [[Wiki/Entities/AIArt/Flux]] — 写实向替代
- [[Wiki/Concepts/AIArt/Base-model-selection]] — 基座选型分析
- [[Wiki/Concepts/AIArt/Lora]] — 挂载在它上面的技术

## 引用来源

- [[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **最新版本的具体许可条款**（2026-04 版）
- **和 NoobAI XL 的训练差异**：具体多了什么 tag？
