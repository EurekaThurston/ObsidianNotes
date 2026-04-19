---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [lora, caption, training, 反常识]
sources: 1
aliases: [Caption 策略, 打标策略, Reverse Caption Principle]
---

# Caption 策略（风格 LoRA 的反常识核心）

> **核心原则：你想让 LoRA "永远带"的特征，caption 里不要写；你想让它"按需出现"的内容，caption 里写清楚。**
>
> 这条规则反直觉，但决定了 [[Wiki/Concepts/AIArt/Lora|LoRA]] 能不能用。

---

## 机理：为什么反着来

LoRA 学的是"输入 caption → 输出图"的**映射差**。

- caption 里**没有**描述但**图里有**的东西 → 归因到"LoRA 本身"，变成 LoRA 的**默认效果**
- caption 里**写了**的东西 → LoRA 把"这个词"对应到"这个视觉"，变成**可触发的内容**

所以对**风格 LoRA**：

| 图里有的东西 | caption 里 | 结果 |
|---|---|---|
| 画风、笔触、色调倾向 | **不写** | 学成 LoRA 的默认画风（永远自带） |
| 具体角色、动作、场景、物件 | **要写** | 学成可控内容（prompt 里写了才出现） |

---

## 两种 Caption 格式

### Danbooru tag 风格（用于 anime 基座）

适用：[[Wiki/Entities/AIArt/Illustrious-XL|Illustrious]]、[[Wiki/Entities/AIArt/NoobAI-XL|NoobAI]]、Pony

```
wwstyle_v1, 1girl, solo, long hair, standing, temple, sunset, looking_at_viewer
```

### 自然语言风格（用于 SDXL base 或 Flux）

```
wwstyle_v1, a young woman with long hair standing in front of a temple at sunset, looking at the viewer
```

选对齐基座的训练格式，别跨格式用。

---

## 触发词（Trigger Word）

每张图的 caption **最前面**放一个独特的触发词，如 `wwstyle_v1`。详见 [[Wiki/Concepts/AIArt/Trigger-word]]。

---

## 实操工作流

### 自动打标 + 人工修剪

1. 用 **WD14 Tagger**（anime）或 **BLIP-2**（natural language）批量自动打标
2. 人工扫一遍，**删除描述画风的 tag**（如 `anime style`、`painting style`、`cel shading`）
   ↑ **这些要让 LoRA 学走**，不能留在 caption 里
3. 补充描述**主体内容**的 tag（如果 tagger 遗漏）
4. 在每条 caption 最前面加触发词 `wwstyle_v1,`

### 常见错误

| 错误 | 后果 |
|---|---|
| 用 auto-tag 的结果直接训，不删风格词 | LoRA 学不到画风，效果极弱 |
| 把画风形容词写进 caption | 学成"这个词 = 那个效果"而不是 LoRA 默认 |
| 不加触发词 | LoRA 强制生效，生产时不灵活 |
| 触发词用常用词（如 `anime`） | 污染基座，没挂 LoRA 也会发作 |

---

## 工具链

- **WD14 Tagger**：anime 向 auto-tag，kohya_ss 自带
- **BLIP-2**：通用自然语言描述生成
- **BooruDatasetTagManager**：批量编辑 caption 的神器

---

## 相关

- [[Wiki/Concepts/AIArt/Lora]] — 这个策略服务的训练技术
- [[Wiki/Concepts/AIArt/Trigger-word]] — caption 的首个 token
- [[Wiki/Concepts/AIArt/Base-model-selection]] — 决定 caption 用什么格式
- [[Wiki/Entities/AIArt/Kohya-ss]] — 配 WD14 Tagger 的训练工具

## 引用来源

- [[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **角色 LoRA 的 caption 策略是否一样？** 原理相同，但实践中数据量少、trigger 设计更重要
- **BLIP-2 vs WD14 的质量对比**：没有量化评估，凭经验选
- **多触发词 LoRA**：能否一个 LoRA 学多个风格变体（如白天/夜晚），用不同触发词切换？
