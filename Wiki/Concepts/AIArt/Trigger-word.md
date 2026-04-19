---
type: concept
created: 2026-04-19
updated: 2026-04-19
tags: [lora, trigger-word, caption, 命名约定]
sources: 1
aliases: [Trigger Word, 触发词, 激活词]
---

# Trigger Word（触发词）

> LoRA 的"开关"——放在 caption 最前面的独特词，生产时在 prompt 里写它就激活 LoRA 的效果，不写就是基座原样。

---

## 概览

训练 [[Wiki/Concepts/AIArt/Lora|LoRA]] 时，在**每张图的 caption 最前面**加一个独特的触发词，例如：

```
wwstyle_v1, 1girl, solo, long hair, ...
```

LoRA 会把"这个词"学成"整个 LoRA 效果的开关"。

- 生产时 prompt 里写 `wwstyle_v1` → 触发风格
- 不写 → 基座原样输出

---

## 设计原则

### 1. 必须是基座没学过的词

**不能用**：
- `anime`、`style`、`art`（过于常见，会污染基座）
- `realistic`、`portrait`（基座理解它们，会冲突）

**应该用**：
- 项目代号 + 版本：`wwstyle_v1`、`akiart_v2`
- 随机短字串：`xy7st9_v1`

### 2. 足够独特以便识别

让你 6 个月后看到 prompt 还能认出"这是哪个 LoRA"。

### 3. 版本化

- `wwstyle_v1` → 第一版
- `wwstyle_v2` → 改进后
- 这样一份 prompt 固定了版本，可复现

### 4. 一个 LoRA 一个触发词

除非你明确想做多触发词切换（进阶用法），一般每个 LoRA 就一个触发词，简单清晰。

---

## 常见错误

| 错误 | 后果 |
|---|---|
| 用 `style` / `art` / `anime` 做触发词 | 污染基座，没挂 LoRA 也会受影响 |
| 每版用相同触发词 | 不挂明确版本的 LoRA 就无法保证效果 |
| 不加触发词（或训练时加、生产时忘） | LoRA 效果弱或无 |
| 触发词太长（一整句） | 记忆负担大，容易写错 |

---

## 生产约定示例

```
基础 prompt 格式：
  <trigger>, <subject>, <composition>, <lighting>, ...

例：
  wwstyle_v1, 1girl, standing on a cliff, dramatic sunset, cinematic lighting
```

在 ComfyUI 模板里把 `wwstyle_v1,` 作为 prompt 前缀自动拼接，美术只用关心后面的内容。

---

## 相关

- [[Wiki/Concepts/AIArt/Caption-strategy]] — 触发词是 caption 的首个 token
- [[Wiki/Concepts/AIArt/Lora]] — 触发词激活的就是 LoRA
- [[Wiki/Concepts/AIArt/Multi-lora-composition]] — 多 LoRA 时每个都有自己的触发词
- [[Wiki/Entities/AIArt/ComfyUI]] — 生产时自动拼接触发词的平台

## 引用来源

- 主题读本(推荐通读):[[Wiki/Readers/AIArt/Lora-深度指南-读本]]
- 原子 source:[[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **多触发词 LoRA**：一个 LoRA 学多种变体（如 `wwstyle_day` / `wwstyle_night`），原理上可行但实践效果待验证
- **触发词强度**：ComfyUI 可以给 prompt 里的 token 加权重（如 `(wwstyle_v1:1.2)`），实际效果是否等同于调 LoRA strength？
