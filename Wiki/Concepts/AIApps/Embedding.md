---
type: concept
created: 2026-04-20
updated: 2026-04-20
tags: [ai, foundation, retrieval, representation, embedding]
sources: 1
aliases: [Embedding, 向量嵌入, 词向量, 语义向量, 文本向量]
---

# Embedding (向量嵌入)

> 把文字(或图片、音频)转成一串数字——一个几百到几千维的**向量**,让"意思"变成"几何距离"。意思相近 → 向量相近,意思无关 → 向量远。是 [[Wiki/Concepts/Methodology/Rag|RAG]] 和所有"语义检索"产品的底层数学。

## 一句话角色

Embedding 是连接"人类语言"和"计算可操作的数字空间"的桥梁。一旦语义变成了向量,所有"找相近的东西""分类""聚类""推荐"都能通过**向量之间的距离计算**(通常是余弦相似度)完成——这是一套彻底不同于关键词匹配的检索范式。

## 核心事实 / 字段速查

| 维度 | 说明 |
|---|---|
| **输入** | 一段文字(或图像/音频,多模态 embedding) |
| **输出** | 一个定长向量,典型维度 384 / 768 / 1024 / 1536 / 3072 |
| **模型** | 由专门的 embedding model 产出,独立于对话 LLM |
| **核心性质** | 语义相近 → 向量距离近(cosine 相似度高) |
| **距离度量** | 通常余弦相似度(`cos(A, B) = A·B / |A||B|`)、有时 L2 距离 |
| **不可逆** | 向量无法还原回原文(与哈希类似) |
| **相同输入 → 相同输出** | 同一模型下确定性,可缓存 |

## 为什么要有 Embedding

传统关键词搜索只能匹配字面相同的词。用户问"帮我找关于猫的段落":

- 关键词搜索:命中"猫",miss "喵星人"、"tabby"、"家养宠物"
- Embedding 语义搜索:"猫"向量 ↔ 这几个词向量都近 → 都能找到

**这个"近"是训练得来的**。Embedding 模型在海量"相近文本 vs 无关文本"的样本上学出一个映射,使相近文本在高维空间里聚在一起。

## 典型工作流(RAG 场景)

```
文档库 ─┐
        │  [文档] ─embedding─► [向量] ──► 存入向量数据库(FAISS/Pinecone/Milvus/pgvector)
        │
用户提问 ─┐
         │  [问题] ─embedding─► [查询向量] ──► 在向量库里找 topK 最近的文档向量
         │                                    └─► 取对应原文片段,拼进 prompt
         │
        LLM 基于拼入的片段生成回答
```

**关键点**:Embedding 只负责检索,不负责回答;回答仍然是 LLM 的事。Embedding 把"语义相关的文本找出来"这一步从"不可能靠 LLM 做"(context 装不下全库)变成"可行"。

## ⚠️ 常见陷阱

- **Embedding 模型选型影响检索质量**:OpenAI `text-embedding-3-large`、BGE、Jina、Cohere 等差异很大,中文场景选型和英文不一样。换模型 = 重新 embed 全库
- **维度越高不一定越好**:高维 embedding 检索更精但存储和计算贵;384-768 维对大多数场景已够
- **相同文本在不同模型下的 embedding 完全不兼容**——不能跨模型比较
- **Embedding 是定长**:长文档要先切块(chunking),切块策略(按段落 / 按 token 数 / overlap)直接影响命中率
- **Embedding 不理解时间、数值、否定**——"上周"/"本月"这种语义它能懂,但"订单金额 > 1000"这种条件它完全不行。所以 RAG 里一般要**混合检索**:embedding + 关键词 + metadata filter

## 不止 RAG

Embedding 的应用远不止检索:

- **聚类 / 分类**:给文本打标签(情感分析、主题分类)的前置步骤
- **推荐系统**:用户行为 → embedding,找相似用户 / 相似商品
- **去重**:两段文本 embedding 接近 → 很可能重复内容
- **异常检测**:与历史样本 embedding 距离远 → 潜在异常
- **多模态对齐**:CLIP 把图片和文字映射到**同一个**向量空间 → 能用文字搜图片(AI 美术生成里的 text-to-image 基础)

## 相关

- [[Wiki/Concepts/Methodology/Rag|RAG]] — Embedding 最主流的消费者
- [[Wiki/Concepts/AIApps/Llm|LLM]] — 和 embedding 模型是**不同**模型,但都属 AI foundation
- [[Wiki/Concepts/AIApps/Context-window|上下文窗口]] — 如果 context 够大能装下全库,就不需要 embedding 检索;embedding 是为"装不下"而生
- [[Wiki/Concepts/AIApps/Hallucination|幻觉]] — Embedding-based RAG 是对抗幻觉的主要机制之一
- [[Wiki/Concepts/AIArt/Base-model-selection|AI 美术基座]] — 图像生成里的 CLIP / T5 编码器本质也是多模态 embedding

## 深入阅读

- 主题读本:[[Readers/AIApps/AI 应用生态全景 2026]](第四章 RAG / Embedding)
- 相关概念:[[Wiki/Concepts/Methodology/Rag]] 的"Embedding 向量嵌入"一节

## 引用来源

- [[Wiki/Sources/AIApps/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]]) — 第 4 章

## 开放问题 / 待深入

- **Embedding 模型选型**本 wiki 尚未评估 — OpenAI `text-embedding-3`、BGE、Jina、Cohere 各自场景和开销
- **向量数据库选型**(FAISS 单机、Pinecone 云、Milvus 自托管、pgvector 关系库内)尚未展开
- **多模态 embedding**(CLIP、SigLIP)与本仓 AI 美术主题的交叉:图像搜索 / 自动标注数据集 / 风格相似度评估——未 ingest
- **Embedding-based vs BM25 / 混合检索**的 benchmark 也未入 wiki;真实生产 RAG 多半混用
