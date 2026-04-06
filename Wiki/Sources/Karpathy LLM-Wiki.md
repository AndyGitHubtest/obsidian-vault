---
title: "Karpathy LLM-Wiki 理念"
created: 2026-04-06
updated: 2026-04-06
tags: [AI, 知识管理, 理念]
source: "https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f"
---

# Karpathy LLM-Wiki 理念

> 来源: Karpathy Gist (2026)

## 核心洞察

**传统 RAG 的缺陷**：每次查询时 LLM 都从零开始拼凑碎片，没有知识积累。

**LLM-Wiki 方案**：LLM 主动构建并维护一个持久的、结构化的 wiki。知识被编译一次，然后持续更新。

## 关键区别

| 维度 | RAG | LLM-Wiki |
|------|-----|----------|
| 知识状态 | 每次重新检索 | 持久累积 |
| 交叉引用 | 查询时临时构建 | 已存在 |
| 矛盾检测 | 不检测 | 主动标记 |
| 合成质量 | 依赖检索质量 | 持续优化 |

## 架构

```
Raw Sources (不可变) → Wiki (LLM 维护) ← Schema (LLM 规范)
```

## 三个操作

1. **Ingest**: 新来源 → LLM 读取 → 更新 10-15 个页面
2. **Query**: 提问 → LLM 回答 → 好答案也存为页面
3. **Lint**: 定期检查矛盾/过时/孤儿页面

## 两个关键文件

- **index.md**: 内容目录，按分类组织
- **log.md**: 时间线日志，追加式记录

## 对我们的启发

- 知识库应该是一个"活的系统"，不是静态文档堆
- LLM 负责所有整理工作，人负责提问和筛选
- Obsidian = IDE, LLM = 程序员, Wiki = 代码库
