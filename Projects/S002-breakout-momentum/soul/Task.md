---
project: S002-breakout-momentum
type: quant-strategy
---

# 📋 S002 抓爆点策略 - 任务追踪

> 当前状态：**DESIGN 完成，等待确认后进入 DEVELOP**

---

## 一、状态机总览

| 阶段 | 状态 | 开始时间 | 完成时间 | 产出 |
|------|------|----------|----------|------|
| INIT | ✅ | 15:30 | 15:35 | Task.md |
| PLANNING | ✅ | 15:35 | 15:40 | 策略定义.md |
| DESIGN | ✅ | 15:40 | 15:55 | 信号架构.md (5层体系) |
| DEVELOP | ⏳ | | | 策略代码 |
| TEST | ⏳ | | | 回测报告 |
| DEPLOY | ⏳ | | | 上线确认 |
| DONE | ⏳ | | | 项目交付 |

---

## 二、任务拆解（匹配 5 层模块体系）

### DEVELOP 阶段任务

#### 层 1: 标的过滤模块
| # | 任务 | 文件 | 优先级 | 状态 | 依赖 |
|---|------|------|--------|------|------|
| D1.1 | 流动性过滤（成交额/深度/价差） | data/filter.py | P0 | ⏳ | - |
| D1.2 | 数据质量过滤（缺失率/异常值） | data/filter.py | P0 | ⏳ | D1.1 |
| D1.3 | 垃圾币过滤（Meme/单字母/插针/滑点） | data/filter.py | P0 | ⏳ | D1.1 |
| D1.4 | 市场环境过滤（趋势/轮动/震荡/死水） | data/filter.py | P0 | ⏳ | D1.1 |

#### 层 2: 压缩检测模块
| # | 任务 | 文件 | 优先级 | 状态 | 依赖 |
|---|------|------|--------|------|------|
| D2.1 | 波动率压缩检测 | signals/compression.py | P0 | ⏳ | D1 |
| D2.2 | ATR 压缩检测 | signals/compression.py | P0 | ⏳ | D1 |
| D2.3 | 布林带宽压缩检测 | signals/compression.py | P0 | ⏳ | D1 |
| D2.4 | 价格区间压缩检测 | signals/compression.py | P0 | ⏳ | D1 |
| D2.5 | K线实体压缩检测 | signals/compression.py | P1 | ⏳ | D1 |
| D2.6 | 压缩质量过滤（缩量不死/结构健康/无插针） | signals/compression.py | P0 | ⏳ | D2.1-D2.5 |
| D2.7 | 压缩评分计算 | signals/compression.py | P0 | ⏳ | D2.6 |

#### 层 3: 爆发确认模块
| # | 任务 | 文件 | 优先级 | 状态 | 依赖 |
|---|------|------|--------|------|------|
| D3.1 | 价格突破确认（收盘突破/幅度/实体质量） | signals/breakout.py | P0 | ⏳ | D2 |
| D3.2 | 收益率异常确认（Z-score） | signals/breakout.py | P0 | ⏳ | D2 |
| D3.3 | 连续位移确认（3/5/8根累计） | signals/breakout.py | P0 | ⏳ | D2 |
| D3.4 | 成交量确认（单根/连续/价量同步/背离） | signals/breakout.py | P0 | ⏳ | D2 |
| D3.5 | 多周期联动（触发+确认+背景过滤） | signals/breakout.py | P0 | ⏳ | D3.1 |
| D3.6 | 爆发评分计算 | signals/breakout.py | P0 | ⏳ | D3.1-D3.5 |

#### 层 4: 延续评分模块
| # | 任务 | 文件 | 优先级 | 状态 | 依赖 |
|---|------|------|--------|------|------|
| D4.1 | 回撤深度计算 | signals/continuation.py | P0 | ⏳ | D3 |
| D4.2 | 价格效率计算 | signals/continuation.py | P0 | ⏳ | D3 |
| D4.3 | 突破位站稳能力 | signals/continuation.py | P0 | ⏳ | D3 |
| D4.4 | 后续量能维持 | signals/continuation.py | P0 | ⏳ | D3 |
| D4.5 | 执行评分（滑点/深度/点差） | signals/continuation.py | P0 | ⏳ | D1 |
| D4.6 | 综合评分汇总（压缩+爆发+延续+执行） | signals/continuation.py | P0 | ⏳ | D4.1-D4.5 |

#### 层 5: 交易执行模块
| # | 任务 | 文件 | 优先级 | 状态 | 依赖 |
|---|------|------|--------|------|------|
| D5.1 | 入场逻辑（评分阈值/急拉过滤/量能衰减/入场方式） | execution/entry.py | P0 | ⏳ | D4 |
| D5.2 | 仓位管理（风险固定/分数加权/环境调仓/连亏降档） | risk/position.py | P0 | ⏳ | D5.1 |
| D5.3 | 止损逻辑（突破位/ATR/时间/前低） | execution/exit.py | P0 | ⏳ | D5.1 |
| D5.4 | 分批止盈（4阶段：1:1.5/1:2.5/前高/移动） | execution/exit.py | P0 | ⏳ | D5.1 |
| D5.5 | 移动止盈（ATR 追踪） | execution/exit.py | P0 | ⏳ | D5.4 |
| D5.6 | 失效检测（5条件：假突破/量能/站稳/吞没/环境） | execution/exit.py | P0 | ⏳ | D5.1 |
| D5.7 | 日内强制平仓（UTC 23:50） | execution/exit.py | P0 | ⏳ | D5.1 |
| D5.8 | 执行前检查（价差/深度/滑点/消息/资金费） | execution/entry.py | P1 | ⏳ | D5.1 |
| D5.9 | 执行后记录（触发原因/滑点/手续费/退出原因） | monitor/logger.py | P1 | ⏳ | D5.1 |

#### 回测与复盘
| # | 任务 | 文件 | 优先级 | 状态 | 依赖 |
|---|------|------|--------|------|------|
| T1 | 回测引擎（事件驱动/撮合/成本计算） | backtest/engine.py | P0 | ⏳ | D5 |
| T2 | 绩效指标计算（Sharpe/PF/DD/WR/PnL） | backtest/metrics.py | P0 | ⏳ | T1 |
| T3 | 分组回测（环境/周期/流动性/信号强度） | backtest/group_test.py | P0 | ⏳ | T1 |
| T4 | 参数优化（粗网格→细网格→鲁棒性） | backtest/optimizer.py | P1 | ⏳ | T1 |
| T5 | 过拟合检测（训练/验证/测试集分离） | backtest/overfit.py | P0 | ⏳ | T3 |
| T6 | 复盘统计（触发原因分布/退出原因/滑点分析） | monitor/stats.py | P1 | ⏳ | T1 |

---

## 三、当前阻塞项
- 无（DESIGN 完成，等待确认后开始编码）

## 四、状态流转记录
| 时间 | 从 | 到 | 触发条件 | 操作人 |
|------|----|----|----------|--------|
| 15:30 | - | INIT | 任务接收 | AI |
| 15:35 | INIT | PLANNING | 任务拆解完成 | AI |
| 15:40 | PLANNING | DESIGN | 策略定义完成 | AI |
| 15:55 | DESIGN | DESIGN | 25层清单升级 | AI |

---
*最后更新：2026-04-07 15:55*
