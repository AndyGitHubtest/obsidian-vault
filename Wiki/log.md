---
title: "Wiki 演化日志"
created: 2026-04-06
type: log
---

# Wiki 演化日志

> 追加式记录。每条以 `## [日期] 操作 | 标题` 开头。
> 查询最近 5 条: `grep "^## \[" log.md | tail -5`

---

## [2026-04-06] init | 知识库体系初始化

- 创建 Wiki 目录结构（Hermes / 量化交易 / _Templates / 项目）
- 创建 index.md（总索引）和 log.md（此文件）
- 创建 schema.md（LLM 维护规范）
- 录入初始页面：知识框架、S001-Pro 系列、Hermes 能力文档
- 融入 Karpathy LLM-Wiki 理念

## [2026-04-06] ingest | Karpathy LLM-Wiki 理念

- 来源: gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- 核心: 持久 wiki vs RAG，知识复利增长
- 产出: [[Karpathy LLM-Wiki]] 页面

## [2026-04-06 22:30] task | 建立知识库体系

- 创建完整目录结构（Wiki/量化交易/Hermes/_Templates/项目）
- 创建核心文档：index.md, log.md, schema.md, 知识框架.md, 策略规格说明.md, 能力全景图.md
- 创建模板：策略文档模板, 回测报告模板, Bug记录模板
- 创建 Karpathy LLM-Wiki 理念页面
- 初始化 Git 仓库并推送到 GitHub
- git: 1b4ea9e

## [2026-04-06 22:45] task | 建立静默工作记忆机制

- 更新 schema.md：新增"静默工作记忆"章节
- 更新 Memory：记录静默记录铁律
- 规则：每次任务完成后自动追加 log.md + 更新 Memory 关键事实
- 用户无需额外操作，自动记录工作轨迹

## [2026-04-06 23:00] task | 优化铁律体系

- 重构为三级体系：L0 安全 > L1 流程 > L2 偏好
- 新增 P0 最高流程铁律：回答前先思考与关联，给出建议并等待确认
- 统一真相源至 Wiki/schema.md，Memory 存引用
- git: 7afce88

## [2026-04-06 23:10] task | 新增 P1 项目开发规则（从 0 到 1）

- 定义六阶段开发流程：需求→数据→开发→回测→部署→运维
- 每个阶段 P0 确认后才进入下一阶段
- 关联现有铁律（L0/L1/L2）贯穿全流程
- git: 待提交

## [2026-04-06 23:20] task | 详细化 P1 项目开发规则（通用版）

- 扩展六阶段为详细版：每阶段含步骤/产出物/检查清单/常见陷阱
- 新增项目类型快速映射表（量化/Web/API/爬虫）
- 新增贯穿规则与铁律关联表
- schema.md 从 40 行扩展到 150+ 行
- git: 待提交
