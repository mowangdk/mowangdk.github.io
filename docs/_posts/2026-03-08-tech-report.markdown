---
layout: post
title:  "weekly report"
date:   2026-03-08 22:30:08 +0800
categories: weeklyreport
---

一个多月了， 2月又是去欧洲出差，又是过年的，结果休了很长时间，今天开始恢复了

# 读书

### agentic filesystem

openclaw 在一月到目前为止的两个月内火的一塌糊涂，AI agent 本来也是我们在投入的方向， 不过没想到突然一大堆的需求都过来了，突然被搞的措手不及

#### Agentic Retrieval

Agentic Retrieval (智能体检索) 已经从传统的“输入关键词 -> 搜索文档 -> 返回片段”演变为一个动态、具备推理能力、多步循环的决策过程。

第一步：意图识别与 Query 重写 (Query Transformation)
Agent 不会直接把用户的原始问题扔进数据库。它会利用 LLM 的推理能力：

分解任务：如果用户问“公司去年的财务状况和行业平均水平对比如何？”，Agent 会将其拆分为“查公司财务”和“查行业报告”两个子查询。
重写 Query：Agent 将口语化的提问转化为更适合搜索引擎的专业表达（例如加上特定领域术语、补全上下文）。

第二步：工具与源的选择 (Tool/Source Selection)
Agent 有权决定去哪里找资料：

***多源检索***：不仅查向量数据库 (Vector DB)，还可以决定调用搜索 API (Google/Bing)、查询内部 SQL 数据库、甚至调用代码去分析一个 CSV 文件。
***选择工具***：Agent 会评估：“这个问题适合用关键词检索（BM25）还是语义检索（Embedding）？”
第三步：自主递归检索 (Recursive/Iterative Retrieval)
这是“Agentic”最关键的特征：
***链式思考*** (Chain of Thought)：检索到第一轮资料后，Agent 先读一遍。如果发现证据不足或有矛盾，它会自主产生新的搜索 Query 进行下一轮搜索。
Self-Correction：如果搜索结果全是噪音，Agent 会反馈：“这次查询太宽泛了，我需要缩小范围”，并自动修正检索词。

第四步：信息评估与过滤 (Verification & Reranking)
Agent 对检索回来的大量碎片信息进行“去噪”：

相关性评分：并不是所有检索到的文档都是有用的，Agent 会剔除无关信息。
事实一致性校验：检查文档中是否存在相互冲突的观点，如果存在，Agent 会标注出来并进行核实。

第五步：记忆与上下文管理
Working Memory：Agent 将检索到的关键事实存入“工作记忆”，在生成答案时参考。
长短期记忆整合：检索到的知识不仅用于回答当前问题，还会被 Agent 存入长效记忆（如在你的 AgenticFilesystem 存储中），以便下次同类任务直接调用。


特性	传统 RAG	Agentic Retrieval
主动性	被动检索 (一次性)	主动决策 (多次迭代)
查询生成	固定模式	智能拆解与重写
工具使用	仅限 Vector DB	任意工具 (SQL, 网页, 代码, API)
复杂性	适合简单查询	适合处理多跳、复杂逻辑的任务
自我反馈	无反馈	具备“觉得不够，继续搜”的自我修正能力



# 工作
