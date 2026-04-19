---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [lora, comfyui, composition, 生产力]
sources: 1
aliases: [Multi-LoRA, 多 LoRA 组合, LoRA 叠加]
---

# Multi-LoRA Composition（多 LoRA 组合）

> 单个 LoRA 有上限。**多 LoRA 组合**才是 AI 美术管线真正的生产力——把风格拆成独立可组合的模块化控件，这是 MidJourney 做不到的事。

---

## 概览

[[Wiki/Entities/AIArt/ComfyUI|ComfyUI]] 里可以同时挂载任意多个 [[Wiki/Concepts/AIArt/Lora|LoRA]]，每个独立调 strength、独立开关。这让风格从"一团黑盒"变成"一堆可调旋钮"。

---

## 典型 LoRA 栈

```
基座模型（如 Illustrious XL）
  ├─ 风格 LoRA: wwstyle_v1        (strength 0.8)   ← 打底风格
  ├─ 角色 LoRA: wwstyle_encore    (strength 0.9)   ← 特定角色
  ├─ 灯光 LoRA: dramatic_lighting (strength 0.5)   ← 光影增强
  └─ 细节 LoRA: detail_tweaker    (strength 0.3)   ← 细节细化
```

每层 LoRA 解决一个关注点，可独立迭代。这是**关注点分离（Separation of Concerns）** 在 AI 美术上的直接应用。

---

## 为什么 MidJourney 做不到

| 维度 | MidJourney | LoRA 组合 |
|---|---|---|
| 风格表达 | `--sref` 近似参考（黑盒） | LoRA 文件（白盒，可训可调） |
| 叠加能力 | 多个 sref 权重平均 | 每个 LoRA 独立挂、独立 strength |
| 可版本化 | ❌ | ✅（文件可提交 Git） |
| 可私有化 | ❌ | ✅（本地部署） |
| 可组合迭代 | 有限 | 每层独立更新 |

所以 LoRA 是**真正模块化的风格控件**。MidJourney 是"艺术家工具"，LoRA 是"工程师的风格库"。

---

## 调 strength 的经验

- **风格 LoRA**：0.7 - 0.9（太强会覆盖其他 LoRA）
- **角色 LoRA**：0.8 - 1.0（需要强特征锁定）
- **细节/辅助 LoRA**：0.2 - 0.5（只是微调）

多 LoRA 互相会**争抢权重**：如果角色 LoRA 训练时学了一些画风特征，会和风格 LoRA 冲突 → 调降角色 LoRA 或重训时用更纯的数据集。

---

## 和 Tag 库前端的衔接

在鸣潮流水线下的典型用法：

```
美术选 tag → 前端拼 prompt
    → 加 trigger: wwstyle_v1
    → ComfyUI workflow 里挂好风格+角色+灯光+细节 LoRA
    → 后端出图
    → 飞书回复
```

前端只暴露 tag 选择器，后端的 LoRA 配置 = **TA 管控的"风格版本"**。美术看不到底层变化。

---

## 相关

- [[Wiki/Concepts/AIArt/Lora]] — 单个 LoRA
- [[Wiki/Concepts/AIArt/Trigger-word]] — 每个 LoRA 各自的触发词
- [[Wiki/Entities/AIArt/ComfyUI]] — 提供多 LoRA 挂载的平台

## 引用来源

- [[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **LoRA 冲突检测**：当前没有自动化工具检测"A LoRA 和 B LoRA 学了冲突的特征"。人工盲评是唯一办法
- **LyCORIS 家族**（LoCon / LoHa）：对深层特征学习更强，和 LoRA 组合策略可能不同
- **LoRA 合并 vs 挂载**：可以用 `supermerger` 等工具把多 LoRA 合并成一个，是否更稳定？
- **自适应 strength**：根据 prompt 内容自动调各 LoRA strength，是研究方向但未成熟
