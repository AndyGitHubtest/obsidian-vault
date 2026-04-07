---
file: SOUL-MASTER.md
purpose: SOUL 系统总纲 - 所有规则的索引和入口
version: 1.0
---

# 🧠 Hermes SOUL 系统总纲

> State-Driven Operating Unified Loop
> 版本：1.0 | 最后更新：{{date}}

---

## 系统概述

Hermes SOUL 是一个 **无需人工干预的自动开发执行系统**。

接收任务后，系统自动完成：拆解 → 文档 → 设计 → 开发 → 测试 → 部署 → 交付

---

## 核心文件索引

| 文件 | 用途 | 路径 |
|------|------|------|
| status.md | 状态机当前状态 | soul-core/status.md |
| Task.md | 任务拆解与追踪 | soul-core/Task.md |
| LOG.md | 全流程执行日志 | soul-core/LOG.md |
| CHECKLIST.md | 各阶段自检清单 | soul-core/CHECKLIST.md |
| PRD.md | 需求文档 (PLANNING 产出) | soul-core/PRD.md |
| ARCH.md | 架构文档 (DESIGN 产出) | soul-core/ARCH.md |
| TEST.md | 测试报告 (TEST 产出) | soul-core/TEST.md |

## 规则文件索引

| 文件 | 用途 | 路径 |
|------|------|------|
| soul-automation-rules.md | AI 每阶段自动做什么 | soul-core/soul-automation-rules.md |
| state-transition-rules.md | 状态转换条件/回滚 | soul-core/state-transition-rules.md |
| doc-sync-rules.md | 文档与代码同步规则 | soul-core/doc-sync-rules.md |
| git-sync-rules.md | Git 同步自动化规则 | soul-core/git-sync-rules.md |
| project-scaffold.md | 项目脚手架标准 | project-scaffold/project-scaffold.md |

## 模板索引

| 类型 | 模板目录 | 数量 |
|------|----------|------|
| 开发项目 | dev-templates/ | 8 个 |
| 量化策略 | quant-templates/ | 8 个 |

---

## 状态机

```
INIT → PLANNING → DESIGN → DEVELOP → TEST → DEPLOY → DONE
```

### 阶段对应文档
| 阶段 | 核心产出 | 规则文件 |
|------|----------|----------|
| INIT | Task.md | soul-automation-rules.md §INIT |
| PLANNING | PRD.md | soul-automation-rules.md §PLANNING |
| DESIGN | ARCH.md | soul-automation-rules.md §DESIGN |
| DEVELOP | 代码 | soul-automation-rules.md §DEVELOP |
| TEST | TEST.md | soul-automation-rules.md §TEST |
| DEPLOY | 上线确认 | soul-automation-rules.md §DEPLOY |
| DONE | 项目交付 | soul-automation-rules.md §DONE |

---

## 核心铁律

1. **文档优先** — 无文档不执行
2. **状态锁定** — 禁止跳过状态
3. **改码必更文** — Code ≠ Doc is a bug
4. **失败必回滚** — 测试失败 → DEVELOP
5. **全程可审计** — 所有操作记录到 LOG.md
6. **自动提交** — 每次变更自动 Git commit + push
7. **自检必执行** — 每次状态变更执行 CHECKLIST

---

## 快速使用

### 启动新项目
1. 告诉 AI 项目目标（一句话）
2. AI 自动：INIT → 创建项目结构 → 填写 Task.md
3. 自动进入 PLANNING → 生成 PRD.md
4. 按状态机自动推进到 DONE

### 查看当前进度
- 读取 `status.md` → 当前状态
- 读取 `Task.md` → 任务进度
- 读取 `LOG.md` → 执行历史

### 手动干预
- 任何阶段可以暂停
- 可以要求 AI 解释当前操作
- 可以修改 PRD/ARCH 后继续

---

## 完成定义 (DONE)

必须全部满足：
- [ ] 所有功能完成
- [ ] 所有测试通过
- [ ] 成功部署
- [ ] 文档完整 (PRD/ARCH/TEST/LOG/Task)
- [ ] 日志完整 (全流程记录)
- [ ] 代码已 push

---

*系统版本：1.0*
*创建：{{date}}*
