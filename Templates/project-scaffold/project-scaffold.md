---
file: project-scaffold.md
purpose: 新项目初始化时的目录结构和文件清单
---

# 🏗️ 项目脚手架 - 新项目初始化标准

## 初始化流程

当收到新项目任务时，执行以下步骤：

### Step 1: 确认项目类型
- 开发项目 → 使用 dev-templates
- 量化策略 → 使用 quant-templates

### Step 2: 创建项目目录
```
projects/
└── [项目名]/
    ├── soul/                    # SOUL 核心文件
    │   ├── status.md            # 状态机当前状态
    │   ├── Task.md              # 任务追踪
    │   ├── LOG.md               # 执行日志
    │   ├── CHECKLIST.md         # 自检清单
    │   ├── PRD.md               # 需求文档 (PLANNING 产出)
    │   ├── ARCH.md              # 架构文档 (DESIGN 产出)
    │   └── TEST.md              # 测试报告 (TEST 产出)
    ├── docs/                    # 项目文档
    │   ├── requirements/        # 需求相关
    │   ├── design/              # 设计相关
    │   └── decisions/           # ADR 决策记录
    ├── src/                     # 源代码 (开发项目)
    │   └── ...
    ├── strategy/                # 策略代码 (量化策略)
    │   └── ...
    ├── tests/                   # 测试代码
    │   └── ...
    └── data/                    # 数据 (量化策略)
        └── ...
```

### Step 3: 复制 SOUL 核心模板
从 `Templates/soul-core/` 复制到 `projects/[项目名]/soul/`：
- status.md
- Task.md
- LOG.md
- CHECKLIST.md

### Step 4: 初始化状态
- status.md → STATUS: INIT
- Task.md → 填写项目信息
- LOG.md → 记录初始化操作

### Step 5: 初始化 Git
```bash
cd projects/[项目名]
git init
git add .
git commit -m "init: [项目名] 项目初始化 - SOUL INIT"
```

---

## 项目类型对应模板映射

| 项目类型 | 需求模板 | 架构模板 | 测试模板 |
|----------|----------|----------|----------|
| 开发项目 | dev-templates/01-产品需求-PRD.md | dev-templates/02-架构设计.md | dev-templates/05-测试验证.md |
| 量化策略 | quant-templates/01-策略定义.md | quant-templates/02-信号架构.md | quant-templates/05-回测验证.md |

---

## 初始化检查清单
- [ ] 项目目录结构已创建
- [ ] SOUL 核心文件已复制
- [ ] status.md 状态设置为 INIT
- [ ] Task.md 已填写项目信息
- [ ] LOG.md 已记录初始化
- [ ] Git 仓库已初始化
- [ ] 初始 commit 已完成
