---
file: status.md
purpose: SOUL 状态机当前状态 - 唯一状态源
---

# 🔄 [项目名称] - 状态追踪

## 当前状态
**STATUS:** INIT

## 状态历史
| 时间 | 状态 | 变更原因 |
|------|------|----------|
| {{date}} | INIT | 项目初始化 |

## 状态转换规则
```
INIT → PLANNING    (Task.md 任务拆解完成)
PLANNING → DESIGN  (PRD.md 完成并确认)
DESIGN → DEVELOP   (ARCH.md 完成并确认)
DEVELOP → TEST     (代码完成，无编译错误)
TEST → DEPLOY      (测试全通过)
DEPLOY → DONE      (上线确认，文档完整)

回滚:
TEST → DEVELOP     (测试失败)
DEPLOY → TEST      (部署失败)
任何阶段 → 上阶段   (发现上游文档缺失)
```

## 准入检查 (每次状态变更前执行)
- [ ] 当前阶段产出物完整
- [ ] 当前阶段 LOG.md 已更新
- [ ] 无未解决的 ERROR
- [ ] 下一阶段前置文档已就绪

## 转出检查 (每次状态变更时执行)
- [ ] 产出物已提交 Git
- [ ] LOG.md 已记录本次变更
- [ ] Task.md 已更新任务状态
- [ ] 下一阶段 Task.md 任务已准备

---
*最后检查：{{date}}*
