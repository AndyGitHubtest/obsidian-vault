---
file: git-sync-rules.md
purpose: Git 同步自动化规则 - 每次变更自动提交
version: 1.0
---

# 🔗 Git 同步自动化规则

> 任何变更必须可追溯、可回滚、可审计

---

## 核心原则

1. **小步提交** — 每个有意义的变更单独提交
2. **信息完整** — commit message 说明做了什么、为什么
3. **自动推送** — 提交后自动 push 到 GitHub
4. **禁止本地堆积** — 不允许有未提交的变更

---

## 自动提交规则表

### 触发事件 → Git 动作
| 事件 | 提交频率 | Commit 格式 |
|------|----------|-------------|
| 创建项目 | 立即 | `init: [项目名] 项目初始化` |
| 新增文档 | 立即 | `docs: 创建 [文档名] - [阶段] 阶段产出` |
| 更新文档 | 立即 | `docs: 更新 [文档名] - [变更说明]` |
| 新增代码文件 | 立即 | `feat: 新增 [模块名] - [功能说明]` |
| 修改代码文件 | 立即 | `fix/feat: [模块] - [变更说明]` |
| 修复 Bug | 立即 | `fix: [模块] - 修复 [问题描述]` |
| 测试变更 | 立即 | `test: 更新测试 - [变更说明]` |
| 状态流转 | 立即 | `chore: 状态 [旧]→[新] - [原因]` |

### Commit Message 规范
```
类型: [模块] 简要说明

类型枚举:
  init    - 项目初始化
  feat    - 新功能
  fix     - Bug 修复
  docs    - 文档变更
  test    - 测试变更
  chore   - 状态流转、配置变更
  refactor - 代码重构
  revert  - 回滚
```

---

## 自动化流程

### 每次变更后自动执行
```
1. git add .
2. git status --short  # 确认变更
3. git commit -m "[格式化的 message]"
4. git push origin main
5. 确认 push 成功
6. 记录到 LOG.md
```

### 推送失败自动处理
```
1. 重试 push（最多 3 次）
2. 仍失败 → 记录到 LOG.md
3. 继续执行，不阻塞流程
4. 下次操作时重新 push
```

---

## 项目结构 Git 规则

### 忽略文件 (.gitignore)
```
# Python
__pycache__/
*.pyc
*.pyo
.venv/
venv/
*.egg-info/

# 数据库 (大文件)
*.db
*.sqlite

# 敏感信息
.env
*.key
*.pem
config/secrets.*

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# 日志 (可选)
*.log
```

### 必须提交的文件
| 文件类型 | 必须提交 | 说明 |
|----------|----------|------|
| SOUL 核心文件 | ✅ | status.md, Task.md, LOG.md 等 |
| 代码文件 | ✅ | .py, .js, .ts 等 |
| 配置文件 | ✅ | .yaml, .json, .toml (不含敏感信息) |
| 文档 | ✅ | .md |
| 测试文件 | ✅ | test_*.py |
| 数据库 | ❌ | 太大，单独管理 |
| 密钥文件 | ❌ | 安全原因 |

---

## 分支策略

### 标准流程
```
main (保护分支)
  ↑
  └── develop (开发分支)
        ↑
        └── feature/[功能名] (功能分支)
```

### 自动化规则
| 动作 | 规则 |
|------|------|
| 创建功能分支 | `git checkout -b feature/[名]` |
| 功能完成 | 合并到 develop → 删除功能分支 |
| 开发完成 | develop 合并到 main → tag 版本 |
| 紧急修复 | 从 main 创建 hotfix/ → 合并回 main 和 develop |

### 简化模式 (单文件项目)
- 直接使用 main 分支
- 每次变更直接 commit + push
- 使用 tag 标记重要版本

---

## 回滚规则

### 代码回滚
```
1. git log --oneline  # 找到目标版本
2. git revert <commit>  # 安全回滚（保留历史）
3. 或 git reset --hard <commit>  # 强制回滚（谨慎）
4. git push --force (如用 reset)
5. 记录到 LOG.md
```

### 回滚触发条件
- 新代码引入严重 Bug
- 部署失败
- 测试不通过
- 性能严重下降

---
*版本：1.0 | 最后更新：{{date}}*
