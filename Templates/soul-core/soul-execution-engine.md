---
file: soul-execution-engine.md
purpose: AI 执行引擎 - 收到任务后的自动执行流程
version: 1.0
---

# ⚡ AI 执行引擎 - 任务处理流程

> 收到用户任务后，AI 自动执行的完整流程

---

## 入口：接收任务

### 用户输入示例
- "做个 XXX 网站"
- "开发一个 XXX 策略"
- "实现 XXX 功能"

### AI 自动动作（无需用户确认）
```
1. 识别任务意图
2. 判断项目类型（开发/量化）
3. 读取 soul-core/SOUL-MASTER.md → 加载系统规则
4. 读取 soul-core/soul-automation-rules.md → 加载执行规则
5. 开始执行 INIT 阶段
```

---

## 执行引擎循环

```python
while status != "DONE":
    # 1. 读取当前状态
    current_status = read("soul/status.md").status
    
    # 2. 加载对应阶段规则
    rules = load_rules(current_status)
    
    # 3. 执行阶段任务
    for action in rules.actions:
        execute(action)
        log_to("soul/LOG.md", action)
    
    # 4. 自检
    checklist = load_checklist(current_status)
    if not check_all(checklist):
        # 自动修复
        fix_issues()
        continue
    
    # 5. 状态流转
    next_status = rules.next_status
    transition(current_status, next_status)
    log_to("soul/LOG.md", f"状态 {current_status}→{next_status}")
    
    # 6. Git 提交
    git_commit_and_push()
```

---

## 各阶段执行详情

### INIT 阶段自动执行
```
输入：用户一句话需求
处理：
  1. 解析需求 → 提取 {项目名, 类型, 目标}
  2. 创建目录 → projects/{项目名}/soul/
  3. 复制文件 → status.md, Task.md, LOG.md, CHECKLIST.md
  4. 填写 Task.md → 自动拆解任务
  5. 设置 status.md → STATUS: INIT
  6. Git init + commit
输出：项目结构就绪
流转：自动进入 PLANNING
```

### PLANNING 阶段自动执行
```
输入：Task.md
处理：
  1. 选择模板 → dev 或 quant 的 01 阶段模板
  2. 填充 PRD.md → 自动分析需求并填写
  3. 补充细节 → AI 自动完善模糊项
  4. 识别风险 → 自动列出
  5. 检查完整性 → 按 CHECKLIST
输出：PRD.md 完整
流转：自动进入 DESIGN
```

### DESIGN 阶段自动执行
```
输入：PRD.md
处理：
  1. 选择模板 → dev 或 quant 的 02 阶段模板
  2. 设计架构 → 自动描述系统结构
  3. 模块划分 → 自动拆分
  4. 技术选型 → 自动推荐
  5. 覆盖性检查 → 确保覆盖所有 PRD 需求
输出：ARCH.md 完整
流转：自动进入 DEVELOP
```

### DEVELOP 阶段自动执行
```
输入：ARCH.md + Task.md
处理：
  1. 按依赖顺序执行任务
  2. 每个任务：编码 → 检查 → 文档同步 → Git
  3. 全部完成后：完整性检查
输出：可运行代码
流转：自动进入 TEST
```

### TEST 阶段自动执行
```
输入：代码
处理：
  1. 生成测试用例
  2. 执行测试
  3. 分析结果
  4. 如失败 → 回滚 DEVELOP
输出：TEST.md 测试报告
流转：通过 → DEPLOY / 失败 → DEVELOP
```

### DEPLOY 阶段自动执行
```
输入：测试通过的代码
处理：
  1. 准备环境
  2. 执行部署
  3. 验证功能
  4. 配置监控
  5. 如失败 → 回滚 TEST
输出：上线确认
流转：成功 → DONE / 失败 → TEST
```

### DONE 阶段自动执行
```
输入：部署成功的系统
处理：
  1. 最终自检
  2. 生成总结
  3. 通知用户
输出：项目交付报告
```

---

## 异常处理自动化

### 场景 1：用户需求模糊
```
检测：PRD 无法填写完整
处理：AI 基于常见模式生成合理假设
标注：在 PRD 中标注"AI 假设"
继续：不阻塞流程
```

### 场景 2：代码编译失败
```
检测：语法检查报错
处理：AI 自动分析错误并修复
重试：修复后重新检查
回滚：3 次修复失败 → 记录问题，暂停
```

### 场景 3：测试不通过
```
检测：测试用例失败
处理：
  1. 记录失败原因
  2. 回滚到 DEVELOP
  3. 自动修复代码
  4. 重新测试
```

### 场景 4：部署失败
```
检测：服务启动失败或功能异常
处理：
  1. 执行回滚
  2. 记录失败原因
  3. 回到 TEST 重新验证
```

### 场景 5：文档不一致
```
检测：代码变更但文档未更新
处理：
  1. 自动识别变更
  2. 更新对应文档
  3. 提交 Git
```

---

## 通知机制

### 自动通知用户
| 事件 | 通知方式 |
|------|----------|
| 阶段完成 | LOG.md 记录 + 简要告知 |
| 项目完成 | 完整报告 |
| 遇到阻塞 | 说明问题，等待决策 |
| 需要用户输入 | 明确列出需要的信息 |

### 不通知用户
| 事件 | 原因 |
|------|------|
| 正常流程推进 | 自动执行，无需确认 |
| 小 Bug 自动修复 | 已解决，不需用户关注 |
| 文档自动同步 | 后台操作 |

---
*版本：1.0 | 最后更新：{{date}}*
