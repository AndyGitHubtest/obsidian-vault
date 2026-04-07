# S001-Pro 统计套利系统 - 核心开发规格说明书

> **版本**: V2.1-Enhanced  
> **最后更新**: 2025-04-06  
> **状态**: 开发中 → 待实施  
> **铁律**: 代码为真相，文档为规格，所有修改必须基于实际代码验证

---

## 📐 一、系统架构总览

### 1.1 核心模块关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                        S001-Pro 运行时                          │
│                         (main.py)                               │
│                                                                 │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    │
│  │  ScanEngine  │───▶│ pairs_v2.json│◀───│ SignalEngine    │    │
│  │ (scanner.py) │    │ (扫描输出)   │    │ (signal_engine) │    │
│  └──────┬──────┘    └──────┬───────┘    └────────┬────────┘    │
│         │                  │                     │              │
│         ▼                  ▼                     ▼              │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    │
│  │  klines.db  │    │ StateManager │    │ IsolatedExecutor│    │
│  │ (SQLite DB) │    │ (state.py)   │    │ (executor.py)   │    │
│  └─────────────┘    └──────┬───────┘    └────────┬────────┘    │
│                            │                     │              │
│                            ▼                     ▼              │
│                     ┌──────────────┐    ┌─────────────────┐    │
│                     │ state.json   │    │  Binance API    │    │
│                     │ (状态持久化) │    │  (交易所)       │    │
│                     └──────────────┘    └─────────────────┘    │
│                            │                     │              │
│                            └──────────┬──────────┘              │
│                                       ▼                         │
│                              ┌─────────────────┐                │
│                              │ Telegram 推送   │                │
│                              │ (notify模块)    │                │
│                              └─────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 模块职责矩阵

| 模块 | 文件 | 核心职责 | 输入 | 输出 | 依赖 |
|------|------|----------|------|------|------|
| **Runtime** | `main.py` | 系统启动、主循环、异常恢复 | config/strategy.yaml | Telegram通知 | 所有模块 |
| **Scanner** | `scanner.py` | 全市场扫描、配对筛选、打分排名 | klines.db | pairs_v2.json | SQLite |
| **SignalEngine** | `signal_engine.py` | 实时Z-Score计算、信号判断 | 价格数据+pairs_v2.json | 交易信号 | numpy |
| **Executor** | `executor.py` | 逐仓下单、双腿同步、订单管理 | 交易信号 | 订单结果 | ccxt, Binance |
| **StateManager** | `state.py` | 状态持久化、重启对账、日度重置 | state.json | 状态更新 | JSON文件 |
| **Optimizer** | `optimizer.py` | Optuna参数优化、回测验证 | klines.db | 最优参数 | optuna, SQLite |
| **Notifier** | main.py内嵌 | Telegram消息推送 | 事件数据 | TG消息 | requests |

### 1.3 三层时间窗口架构

```yaml
统计层 (扫描阶段):
  数据窗口: 90天 1m K线 (129,600根)
  计算指标: beta, half_life, hurst, corr_mean, corr_std
  更新频率: 每6小时 (可配置)
  输出: pairs_v2.json (Top 30配对+参数)

稳定性层 (参数验证):
  数据窗口: 90天历史回测
  验证内容: 回归次数≥30, spread_cv<0.3, 参数稳定性
  输出: 参数可信度评分

执行层 (实时交易):
  数据窗口: 实时1m K线流
  计算指标: 实时z_score, spread, 持仓PnL
  更新频率: 每分钟 (心跳检查)
  输出: 交易指令 (开/平/止损)
```

---

## 🔄 二、文件流转与数据流转

### 2.1 完整数据流转图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           数据生命周期                                       │
│                                                                             │
│  [数据源]                    [处理层]                    [应用层]            │
│                                                                             │
│  Binance API                                                                │
│      │                                                                      │
│      ▼                                                                      │
│  ┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐      │
│  │ K线下载脚本 │────▶│   klines.db      │────▶│  ScanEngine         │      │
│  │ (独立进程)  │     │ (SQLite, 1m/5m)  │     │ (每6小时扫描)       │      │
│  └─────────────┘     └──────────────────┘     └──────────┬──────────┘      │
│                                                          │                 │
│                                                          ▼                 │
│                                               ┌─────────────────────┐      │
│                                               │ pairs_v2.json.tmp   │      │
│                                               │ (临时文件, 原子写)  │      │
│                                               └──────────┬──────────┘      │
│                                                          │                 │
│                                                          ▼                 │
│                                               ┌─────────────────────┐      │
│                                               │ pairs_v2.json       │      │
│                                               │ (最终配对文件)      │      │
│                                               │ 含: 配对列表+参数   │      │
│                                               └──────────┬──────────┘      │
│                                                          │                 │
│                    ┌─────────────────────────────────────┼─────────────┐   │
│                    │                                     │             │   │
│                    ▼                                     ▼             ▼   │
│         ┌──────────────────┐              ┌──────────────────┐           │
│         │ SignalEngine     │              │ StateManager     │           │
│         │ (实时信号计算)   │              │ (状态持久化)     │           │
│         └────────┬─────────┘              └────────┬─────────┘           │
│                  │                                 │                     │
│                  ▼                                 ▼                     │
│         ┌──────────────────┐              ┌──────────────────┐           │
│         │ 交易信号         │              │ state.json       │           │
│         │ (z_score, action)│              │ (持仓+PnL+日期)  │           │
│         └────────┬─────────┘              └──────────────────┘           │
│                  │                                                       │
│                  ▼                                                       │
│         ┌──────────────────┐                                            │
│         │ IsolatedExecutor │                                            │
│         │ (逐仓下单)       │                                            │
│         └────────┬─────────┘                                            │
│                  │                                                       │
│                  ▼                                                       │
│         ┌──────────────────┐                                            │
│         │ Binance API      │                                            │
│         │ (真实订单)       │                                            │
│         └──────────────────┘                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 关键文件规范

#### pairs_v2.json (扫描输出)

```json
{
  "version": 2,
  "scan_time": "2025-04-06T12:00:00Z",
  "scan_id": "abc123def456",
  "pairs": [
    {
      "pair_id": "BTC_ETH",
      "symbol_a": "BTC/USDT",
      "symbol_b": "ETH/USDT",
      "beta_1m": 1.42,
      "beta_5m": 1.45,
      "half_life_5m": 72.5,
      "hurst_5m": 0.38,
      "corr_mean": 0.85,
      "corr_std": 0.08,
      "spread_robust_std": 0.0012,
      "regression_count": 45,
      "score": 8.7,
      "params": {
        "z_entry": 2.5,
        "z_exit": 0.5,
        "z_stop": 3.5,
        "param_version": 1,
        "param_set_at": "2025-04-06T12:00:00Z",
        "param_method": "optuna"
      },
      "risk": {
        "max_hold_minutes": 120,
        "position_size_usd": 100,
        "funding_diff": 0.0002
      }
    }
  ]
}
```

**原子更新机制**:
```python
# 错误做法 (可能读到半写入文件)
json.dump(data, open('pairs_v2.json', 'w'))

# 正确做法 (原子更新)
import tempfile
import os

tmp_path = 'pairs_v2.json.tmp'
final_path = 'pairs_v2.json'

# 1. 写入临时文件
with open(tmp_path, 'w') as f:
    json.dump(data, f, indent=2)
    f.flush()
    os.fsync(f.fileno())

# 2. 语法校验
with open(tmp_path, 'r') as f:
    json.load(f)  # 验证JSON有效性

# 3. 原子替换
os.rename(tmp_path, final_path)

# 4. 备份旧版本
import shutil
backup_path = f'pairs_v2.json.bak.{int(time.time())}'
shutil.copy2(final_path, backup_path)
# 只保留最近3个备份
```

#### state.json (状态持久化)

```json
{
  "version": 1,
  "last_updated": "2025-04-06T12:30:00Z",
  "positions": {
    "BTC_ETH": {
      "symbol_a": "BTC/USDT",
      "symbol_b": "ETH/USDT",
      "side_a": "sell",
      "side_b": "buy",
      "entry_z": 2.5,
      "entry_time": "2025-04-06T10:15:00Z",
      "entry_price_a": 85000.0,
      "entry_price_b": 2000.0,
      "amount_a": 0.001176,
      "amount_b": 0.0708,
      "margin_usd": 100,
      "layer_status": {
        "layer1": {"filled": true, "pct": 30},
        "layer2": {"filled": true, "pct": 30},
        "layer3": {"filled": false, "pct": 40}
      },
      "stop_loss_z": 3.5,
      "max_hold_until": "2025-04-06T12:15:00Z"
    }
  },
  "daily_pnl": -12.50,
  "daily_trades": 3,
  "loss_streak": 1,
  "last_reset_date": "2025-04-06",
  "kill_switch": {
    "soft_stop": false,
    "hard_stop": false,
    "emergency_stop": false
  }
}
```

**读写安全**:
```python
# 读取时加锁 (防止读写冲突)
import fcntl

with open('state.json', 'r') as f:
    fcntl.flock(f, fcntl.LOCK_SH)  # 共享锁
    data = json.load(f)
    fcntl.flock(f, fcntl.LOCK_UN)

# 写入时加排他锁
with open('state.json.tmp', 'w') as f:
    fcntl.flock(f, fcntl.LOCK_EX)  # 排他锁
    json.dump(data, f)
    f.flush()
    os.fsync(f.fileno())
    os.rename('state.json.tmp', 'state.json')
    fcntl.flock(f, fcntl.LOCK_UN)
```

### 2.3 数据库流转

```
klines.db (SQLite)
├── 表: klines
│   ├── symbol: "BTC/USDT"
│   ├── interval: "1m" | "5m"
│   ├── ts: 时间戳 (UTC)
│   ├── open, high, low, close, volume
│   └── 唯一索引: (symbol, interval, ts)
│
├── 查询模式:
│   ├── Scanner: SELECT DISTINCT symbol FROM klines WHERE interval='1m'
│   ├── Scanner: SELECT * FROM klines WHERE symbol=? AND interval='1m' 
│   │            ORDER BY ts DESC LIMIT 129600 (90天)
│   └── SignalEngine: 实时价格从API获取，不查DB
│
└── 更新模式:
    └── 独立下载进程 (非策略代码)
        ├── 增量更新: INSERT OR IGNORE
        └── 完整性校验: COUNT(*) 验证
```

**重要**: 策略代码对 `klines.db` 只有**只读权限**，数据更新由独立的下载进程负责。

---

## 🔍 三、扫描阶段强化规格

### 3.1 分级配对流水线

```yaml
L1 轻量筛选 (全部候选对):
  输入: 542个币种 → C(542,2) = 146,881对
  过滤:
    - Spearman相关 > 0.5
    - 同板块/同赛道优先 (DeFi对DeFi, L1对L1)
    - 剔除单字母币和Meme币
    - 日成交额 > 200万U (铁律)
    - 数据完整率 > 99%
    - 上市 > 90天
  输出: ~5,000对

L2 统计筛选 (5,000对):
  计算:
    - rolling_corr_mean (288根5m滚动)
    - half_life (均值回复半衰期)
    - beta (OLS回归)
  过滤:
    - rolling_corr_mean > 0.6
    - corr_std < 0.15
    - half_life ∈ (12, 120) 根
  输出: ~500对

L3 深度验证 (500对):
  计算:
    - Engle-Granger 协整检验 (p < 0.1)
    - ADF 平稳性检验 (p < 0.1)
    - Hurst 指数 ∈ (0.2, 0.55)
    - Kalman Filter 残差平稳性
    - spread_robust_std (替代CV)
    - 回归次数 (90天内 ≥ 30次)
  输出: 100-200对候选池

L4 回测排名 (候选池):
  方法: Optuna 100 trials, TPE+CMA-ES
  窗口: 滚动3段×30天 (覆盖不同行情)
  目标: 0.5×Sharpe + 0.2×Return - 0.2×Drawdown + 0.1×Calmar
  验证: Walk-Forward (训练60天 → 验证30天)
  输出: Top 30 → pairs_v2.json
```

### 3.2 异常清洗修复

```python
# 旧: |return|>10% → clip (人为制造假平稳)
# 新: |return|>8% → 标记异常窗口，统计计算时排除

def clean_returns(prices, threshold=0.08):
    """标记异常窗口，不clip"""
    returns = np.diff(np.log(prices))
    abnormal_mask = np.abs(returns) > threshold
    
    # 返回清洗后的价格和异常窗口索引
    clean_prices = prices[~abnormal_mask[:-1]]
    abnormal_indices = np.where(abnormal_mask)[0]
    
    return clean_prices, abnormal_indices

# 在统计计算时排除异常窗口
def calculate_rolling_stats(prices, abnormal_indices, window=288):
    """滚动统计，排除异常窗口"""
    stats = []
    for i in range(window, len(prices)):
        window_prices = prices[i-window:i]
        window_returns = np.diff(np.log(window_prices))
        
        # 排除异常窗口内的数据
        clean_returns = [r for j, r in enumerate(window_returns) 
                        if (i-window+j) not in abnormal_indices]
        
        if len(clean_returns) > window * 0.8:  # 至少80%数据有效
            stats.append({
                'mean': np.mean(clean_returns),
                'std': np.std(clean_returns),
                'valid_pct': len(clean_returns) / len(window_returns)
            })
    
    return stats
```

### 3.3 Spread计算修复

```python
# 旧: spread_cv = std / mean (除零风险)
# 新: spread_robust_metric = rolling_std / (p95 - p5)

def calculate_spread_robust(prices_a, prices_b, beta, window=288):
    """稳健的spread计算，避免CV除零"""
    spread = np.log(prices_a) - beta * np.log(prices_b)
    
    rolling_std = pd.Series(spread).rolling(window).std()
    rolling_range = pd.Series(spread).rolling(window).quantile(0.95) - \
                    pd.Series(spread).rolling(window).quantile(0.05)
    
    # 避免除零
    rolling_range = rolling_range.replace(0, np.nan).fillna(0.0001)
    
    robust_metric = rolling_std / rolling_range
    
    return {
        'spread': spread,
        'rolling_std': rolling_std,
        'robust_metric': robust_metric,
        'mean': np.mean(spread),
        'std': np.std(spread)
    }
```

### 3.4 打分归一化

```python
def normalize_score(factors, weights):
    """所有因子min-max归一化到[0,1]后再加权"""
    normalized = {}
    
    for key, values in factors.items():
        min_val = np.min(values)
        max_val = np.max(values)
        
        if max_val - min_val < 1e-10:
            normalized[key] = np.zeros_like(values)
        else:
            # 注意: 有些因子是反向的 (越小越好)
            if key in ['corr_std', 'half_life', 'vol_ratio']:
                normalized[key] = 1.0 - (values - min_val) / (max_val - min_val)
            else:
                normalized[key] = (values - min_val) / (max_val - min_val)
    
    # 加权求和
    score = np.zeros(len(values))
    for key, weight in weights.items():
        score += weight * normalized[key]
    
    return score
```

---

## ⚡ 四、交易阶段强化规格

### 4.1 双确认+分层统一流程

```
状态机: WAITING → OBSERVING → TRIGGERED → CONFIRMED → EXECUTING → FILLED

WAITING (空闲):
  条件: 无信号
  动作: 持续监控5m z_score

OBSERVING (预触发):
  条件: 5m z >= E - 0.3
  动作: 
    - 进入观察状态
    - 准备资金
    - 启动假突破检测计时器
  超时: 2分钟内z回落到<1.5 → 标记假突破 → 回到WAITING

TRIGGERED (触发):
  条件: 5m z >= E
  动作:
    - 启动双确认计时器 (最多3分钟)
    - 监控1m z_score

CONFIRMED (确认):
  条件: 1m z >= E + 0.2
  动作:
    - 确认通过
    - 进入分层执行

EXECUTING (分层执行):
  状态: {layer1: pending, layer2: pending, layer3: pending}
  
  确认通过瞬间:
    - 如果当前z >= E+0.5: 直接开满 (30%+30%+40%)
    - 如果当前z >= E: 开60% (30%+30%)，剩余40%挂单在E+0.5
    - 如果当前z >= E-0.3: 开30%，其余挂单等待
  
  每层独立状态追踪:
    layer1: {threshold: E-0.3, pct: 30%, status: pending}
    layer2: {threshold: E, pct: 30%, status: pending}
    layer3: {threshold: E+0.5, pct: 40%, status: pending}

FILLED (完成):
  条件: 所有层状态均为filled
  动作: 更新state.json，启动持仓管理
```

### 4.2 出场状态机

```python
class ExitStateMachine:
    def __init__(self, entry_z, exit_x):
        self.entry_z = entry_z
        self.exit_x = exit_x
        
        # 三层出场阈值
        self.layers = {
            'layer1': {'threshold': exit_x + 0.5 * (entry_z - exit_x), 'pct': 0.3, 'executed': False},
            'layer2': {'threshold': exit_x + 0.25 * (entry_z - exit_x), 'pct': 0.3, 'executed': False},
            'layer3': {'threshold': exit_x, 'pct': 0.4, 'executed': False}
        }
    
    def check_exit(self, current_z):
        """检查出场条件，返回应平仓的百分比"""
        total_close_pct = 0.0
        
        for layer_name, layer in self.layers.items():
            if layer['executed']:
                continue
            
            # 注意: z是从高位往下回归
            if current_z <= layer['threshold']:
                layer['executed'] = True
                total_close_pct += layer['pct']
        
        return total_close_pct
    
    def handle_gap(self, prev_z, current_z):
        """处理跳空情况"""
        # 如果z跳空穿越多个层级，累计执行
        return self.check_exit(current_z)
```

### 4.3 双腿同步安全修复

```python
async def execute_pair_trade(self, symbol_a, symbol_b, side_a, side_b, 
                             amount_a, amount_b, max_wait_sec=3):
    """
    安全的 legs 同步执行
    
    方案: OCO变体 + 追单保护
    """
    # 1. 同时发IOC单 (Immediate-Or-Cancel)
    order_a = await self.place_ioc_order(symbol_a, side_a, amount_a)
    order_b = await self.place_ioc_order(symbol_b, side_b, amount_b)
    
    filled_a = order_a.get('filled', False)
    filled_b = order_b.get('filled', False)
    
    # 2. 双成 → 成功
    if filled_a and filled_b:
        return {'success': True, 'orders': [order_a, order_b]}
    
    # 3. A成B不成 → 追单B
    if filled_a and not filled_b:
        logger.warning(f"Leg A filled, B not filled. Chasing B...")
        
        # 立即发B的taker追单 (放宽到3秒)
        order_b_chase = await self.place_taker_order(symbol_b, side_b, amount_b, 
                                                     timeout=max_wait_sec)
        
        if order_b_chase.get('filled'):
            return {'success': True, 'orders': [order_a, order_b_chase]}
        else:
            # 追单失败 → A腿挂止损单保护 (不主动平A腿!)
            logger.warning(f"Chase failed for B. Placing protective stop on A.")
            await self.place_protective_stop(symbol_a, side_a, order_a['price'])
            
            # 标记为"待处理"状态，由持仓管理器后续处理
            return {
                'success': False, 
                'status': 'protective_stop_placed',
                'orders': [order_a],
                'max_hold_until': datetime.utcnow() + timedelta(minutes=5)
            }
    
    # 4. B成A不成 → 追单A (对称逻辑)
    if not filled_a and filled_b:
        # ... 对称处理 ...
        pass
    
    # 5. 双不成 → 返回失败
    return {'success': False, 'status': 'both_failed'}

async def place_protective_stop(self, symbol, side, entry_price):
    """挂保护性止损单，不主动平仓"""
    # 止损价 = 入场价 ± 0.5%
    if side == 'buy':
        stop_price = entry_price * 0.995
    else:
        stop_price = entry_price * 1.005
    
    await self.exchange.create_order(
        symbol=symbol,
        type='STOP_MARKET',
        side='sell' if side == 'buy' else 'buy',
        amount=...,  # 与原仓位相同
        params={'stopPrice': stop_price}
    )
```

**关键原则**: 
- **绝对不主动平已成交腿** (避免制造裸仓)
- 追单失败 → 挂保护性止损 → 等待max_hold超时处理
- 最大容忍时间: 5分钟未配对成功 → 强制平仓 (最后手段)

### 4.4 止损规范

```yaml
止损规则:
  固定止损: S = 3.5 (Z_STOP铁律)
  动态止损: S = max(3.5, E + 0.8)  # 保底3.5
  
  硬约束:
    - 止损单必须在开仓同时以stop-market方式挂出
    - 止损执行不经过信号引擎，直接API平仓
    - 止损后记录到state.json的loss_streak
  
  验证:
    - 每次开仓前检查: S >= 3.5
    - 代码路径验证: 所有异常分支都不能跳过止损
```

---

## 🛡️ 五、风控体系强化规格

### 5.1 全局止损明确定义

```yaml
软停 (日亏5% = $50):
  触发条件: daily_pnl <= -50
  动作: 
    - 停止开新仓
    - 已有持仓继续持有
    - 已有持仓止损收紧20% (S = S × 0.8)
  恢复: 次日UTC 00:00自动重置

硬停 (日亏10% = $100):
  触发条件: daily_pnl <= -100
  动作:
    - 所有持仓以市价平仓
    - 停止交易4小时
    - 发送告警通知
  恢复: 人工检查后手动重置

紧急停止 (日亏20% = $200):
  触发条件: daily_pnl <= -200
  动作:
    - 市价全平所有持仓
    - 停止所有API调用
    - 发送紧急告警
    - 断开交易循环
  恢复: 必须人工干预，重置kill_switch
```

### 5.2 连亏定义

```yaml
连亏统计:
  单配对: 24小时内同一配对亏损次数 ≥ 5次 → 冷却2小时
  全局: 24小时内所有配对亏损次数 ≥ 10次 → 冷却4小时
  
  冷却期行为:
    - 不撤已有持仓
    - 不开新仓
    - 继续监控已有持仓的止损/出场
  
  计数器重置:
    - 每日UTC 00:00重置
    - state.json持久化
```

### 5.3 相关性分级处理

```yaml
相关性监控 (48根1m K线滚动):
  corr ∈ [0.4, 1.0]: 正常
  corr ∈ [0.3, 0.4): 
    - 警告
    - 停止加仓
    - 止损收紧0.5 (S = S - 0.5)
  
  corr ∈ [0.2, 0.3):
    - 减仓50%
    - 准备平仓
  
  corr < 0.2:
    - 全部平仓
    - 标记该配对为"失效"
    - 下次扫描时排除
```

### 5.4 API/网络风控

```yaml
心跳检测:
  频率: 每30秒ping Binance API
  超时阈值: 5秒
  
  连续3次超时 → 暂停开仓
  连续10次超时 → 触发紧急停止
  
  恢复后:
    1. 先对账 (检查实际持仓与state.json是否一致)
    2. 处理幽灵仓位/孤儿仓位
    3. 确认一致后恢复交易
```

### 5.5 重启对账机制

```python
async def reconcile_positions(self):
    """
    启动时/网络恢复后的对账流程
    
    三类差异:
    1. 幽灵仓位 (API有, 记录无) → 立即平仓
    2. 孤儿仓位 (记录有, API无) → 从记录清除
    3. 数量不一致 → 以API为准，更新记录
    """
    # 1. 读取本地记录
    local_positions = self.state.positions
    
    # 2. 查询API实际持仓
    api_positions = await self.exchange.fetch_positions()
    api_map = self._normalize_positions(api_positions)
    
    changes = []
    
    # 3. 检查幽灵仓位
    for sym in api_map:
        if sym not in local_positions:
            logger.warning(f"👻 Ghost position detected: {sym}")
            # 立即平仓 (不属于我们的仓位)
            await self.close_position(sym)
            changes.append({'type': 'ghost_closed', 'symbol': sym})
    
    # 4. 检查孤儿仓位
    for sym in local_positions:
        if sym not in api_map:
            logger.warning(f"👤 Orphan position detected: {sym}")
            # 从记录清除 (可能已成交或已取消)
            del self.state.positions[sym]
            changes.append({'type': 'orphan_removed', 'symbol': sym})
    
    # 5. 检查数量不一致
    for sym in local_positions:
        if sym in api_map:
            local_amt = local_positions[sym].get('amount', 0)
            api_amt = api_map[sym].get('amount', 0)
            
            if abs(local_amt - api_amt) > 0.01:
                logger.warning(f"⚠️ Amount mismatch for {sym}: local={local_amt}, api={api_amt}")
                # 以API为准
                local_positions[sym]['amount'] = api_amt
                changes.append({'type': 'amount_fixed', 'symbol': sym})
    
    # 6. 保存更新后的状态
    if changes:
        self.state.save()
        await self.notify(f"🔧 Reconciliation: {len(changes)} changes applied")
    
    return changes
```

---

## 💰 六、资金与仓位强化规格

### 6.1 保证金定义

```yaml
资金结构:
  总资金: $1,000
  保留金: $200 (不动用)
  可用资金: $800
  
  每对保证金: $100 (固定)
  杠杆: 5x (逐仓)
  
  单对名义价值: $100 × 5x = $500
  
  资金分配:
    A腿保证金 = $100 / (1 + beta)
    B腿保证金 = $100 × beta / (1 + beta)
    
    示例 (beta=1.5):
      A腿: $100 / 2.5 = $40
      B腿: $100 × 1.5 / 2.5 = $60
  
  最大持仓对数: floor($800 / $100) = 8对
  但受限于:
    - 总仓位上限60% ($600) → 最多6对
    - 实际可开对数动态计算
```

### 6.2 动态资金分配

```python
def calculate_available_pairs(self, state, config):
    """动态计算可开仓对数"""
    total_capital = config['capital']
    reserve = config['reserve']
    max_position_pct = config['max_position_pct']
    margin_per_pair = config['margin_per_pair']
    
    available = total_capital - reserve
    max_by_capital = available / margin_per_pair
    max_by_position = (total_capital * max_position_pct) / margin_per_pair
    
    current_positions = len(state.positions)
    remaining_slots = min(max_by_capital, max_by_position) - current_positions
    
    return max(0, int(remaining_slots))
```

### 6.3 资金费处理

```yaml
资金费监控:
  频率: 每8小时 (与Binance结算周期一致)
  
  不利方向 (付资金费):
    - 每日资金费 > 预期利润的20% → 提前平仓
    - 预期利润 = (E - X) × spread_std - 4腿成本
  
  有利方向 (收资金费):
    - 可适当放宽max_hold (额外+30分钟)
    - 但不超过120分钟上限
```

---

## 📱 七、消息推送与统计分析模块

### 7.1 推送类型与频率控制

```yaml
推送分类:
  
  🚨 紧急类 (实时):
    - API异常告警 (连续超时/错误)
    - 重启对账结果 (幽灵仓位/孤儿仓位处理)
    - 程序异常 (未捕获错误/crash)
    - 紧急停止触发
    
  💹 交易类 (实时):
    - 开仓成功 (含分层信息)
    - 平仓/止损
    - 风控触发
    
  📊 信号类 (阈值跨越时):
    - 5m z跨过关键阈值 (E-0.3, E, E+0.5)
    - 仅推送首次跨越，不连续推送
    
  📈 状态类 (每小时):
    - 持仓摘要
    - 日度PnL
    - 配对健康度
    
  📋 报表类 (每日UTC 00:05):
    - 每日交易汇总
    - 扫描结果
    - 参数变更记录
```

### 7.2 推送消息格式规范

#### 开仓成功

```
🟢 开仓成功 | BTC_ETH
━━━━━━━━━━━━━━━━━━━━
📊 配对: BTC/USDT ↔ ETH/USDT
📈 方向: 空BTC + 多ETH
🎯 入场Z: 2.50
💰 保证金: $100 (A腿$40, B腿$60)
⏱️ 时间: 2025-04-06 12:30:00 UTC
📐 分层: 第1层30% ✅ | 第2层30% ⏳ | 第3层40% ⏳
🛑 止损: Z=3.50
⏰ 最大持仓: 120分钟 (至14:30)
━━━━━━━━━━━━━━━━━━━━
```

#### 平仓/止损

```
🔴 平仓 | BTC_ETH
━━━━━━━━━━━━━━━━━━━━
📊 配对: BTC/USDT ↔ ETH/USDT
💸 盈亏: +$12.50 (+12.5%)
🎯 出场Z: 0.45 (目标0.50)
⏱️ 持仓: 45分钟
📊 交易统计: 今日第3笔 | 胜率67%
━━━━━━━━━━━━━━━━━━━━

或 (止损):

❌ 止损 | BTC_ETH
━━━━━━━━━━━━━━━━━━━━
📊 配对: BTC/USDT ↔ ETH/USDT
💸 亏损: -$18.00 (-18.0%)
🎯 止损Z: 3.50 (触发)
⏱️ 持仓: 15分钟
📊 连亏计数: 2/5
━━━━━━━━━━━━━━━━━━━━
```

#### 风控触发

```
⚠️ 风控触发 | 软停
━━━━━━━━━━━━━━━━━━━━
📉 日亏: -$50.00 (-5.0%)
🚫 动作: 停止开新仓
📊 已有持仓: 继续持有 (止损收紧20%)
🔄 恢复: 次日UTC 00:00自动重置
━━━━━━━━━━━━━━━━━━━━
```

#### 扫描结果

```
🔍 扫描完成 | Top 30配对
━━━━━━━━━━━━━━━━━━━━
⏱️ 扫描时间: 2025-04-06 12:00:00 UTC
📊 有效配对: 146,881 → 筛选后 30对
🏆 Top 3:
  1. BTC_ETH (Score: 8.7, Z=2.5)
  2. SOL_AVAX (Score: 7.9, Z=2.3)
  3. MATIC_LINK (Score: 7.2, Z=2.1)
📈 平均相关性: 0.82
📉 平均half_life: 72根 (6小时)
━━━━━━━━━━━━━━━━━━━━
```

#### 每日报表

```
📊 每日报表 | 2025-04-06
━━━━━━━━━━━━━━━━━━━━
💰 日度PnL: +$45.00 (+4.5%)
📈 交易次数: 12笔
✅ 胜率: 58.3% (7胜5负)
📊 平均持仓: 38分钟
💸 总手续费: $8.40
🔄 最大回撤: 3.2%
📋 当前持仓: 3对 (BTC_ETH, SOL_AVAX, MATIC_LINK)
━━━━━━━━━━━━━━━━━━━━
```

### 7.3 推送实现

```python
class TelegramNotifier:
    def __init__(self, token, chat_id):
        self.token = token
        self.chat_id = chat_id
        self.base_url = f"https://api.telegram.org/bot{token}"
        
        # 频率控制
        self.last_send_time = {}
        self.min_interval = {
            'emergency': 0,      # 立即发送
            'trade': 10,         # 10秒间隔
            'signal': 60,        # 1分钟间隔
            'status': 3600,      # 1小时间隔
            'report': 86400      # 1天间隔
        }
    
    async def send(self, category, message, parse_mode='HTML'):
        """带频率控制的推送"""
        now = time.time()
        last = self.last_send_time.get(category, 0)
        min_interval = self.min_interval.get(category, 60)
        
        if now - last < min_interval:
            logger.debug(f"Rate limited: {category}")
            return
        
        try:
            url = f"{self.base_url}/sendMessage"
            payload = {
                'chat_id': self.chat_id,
                'text': message,
                'parse_mode': parse_mode,
                'disable_web_page_preview': True
            }
            
            async with aiohttp.ClientSession() as session:
                async with session.post(url, json=payload, timeout=10) as resp:
                    if resp.status == 200:
                        self.last_send_time[category] = now
                        logger.info(f"📱 Telegram sent: {category}")
                    else:
                        logger.error(f"Telegram error: {resp.status}")
                        
        except Exception as e:
            logger.error(f"Telegram send failed: {e}")
```

### 7.4 统计分析模块

```python
class PerformanceAnalyzer:
    """
    统计分析模块
    
    职责:
    1. 交易绩效统计
    2. 配对健康度监控
    3. 参数有效性分析
    4. 风险指标计算
    """
    
    def __init__(self, state_file='data/state.json', trade_log='data/trades.jsonl'):
        self.state_file = state_file
        self.trade_log = trade_log
    
    def calculate_metrics(self):
        """计算核心绩效指标"""
        trades = self.load_trades()
        
        if not trades:
            return {}
        
        pnls = [t['pnl_usd'] for t in trades]
        
        return {
            'total_trades': len(trades),
            'winning_trades': sum(1 for p in pnls if p > 0),
            'losing_trades': sum(1 for p in pnls if p <= 0),
            'win_rate': sum(1 for p in pnls if p > 0) / len(pnls),
            'total_pnl': sum(pnls),
            'avg_pnl': np.mean(pnls),
            'std_pnl': np.std(pnls),
            'max_win': max(pnls),
            'max_loss': min(pnls),
            'sharpe_ratio': np.mean(pnls) / np.std(pnls) if np.std(pnls) > 0 else 0,
            'max_drawdown': self.calculate_max_drawdown(pnls),
            'avg_hold_time': np.mean([t['hold_minutes'] for t in trades]),
            'profit_factor': self.calculate_profit_factor(pnls),
            'expectancy': np.mean(pnls),
            'per_pair_stats': self.calculate_per_pair_stats(trades),
            'daily_stats': self.calculate_daily_stats(trades)
        }
    
    def calculate_per_pair_stats(self, trades):
        """按配对统计"""
        pair_stats = {}
        
        for t in trades:
            pair = t['pair']
            if pair not in pair_stats:
                pair_stats[pair] = {
                    'trades': 0,
                    'wins': 0,
                    'total_pnl': 0,
                    'avg_hold': 0
                }
            
            pair_stats[pair]['trades'] += 1
            if t['pnl_usd'] > 0:
                pair_stats[pair]['wins'] += 1
            pair_stats[pair]['total_pnl'] += t['pnl_usd']
            pair_stats[pair]['avg_hold'] += t['hold_minutes']
        
        # 计算平均
        for pair in pair_stats:
            stats = pair_stats[pair]
            stats['win_rate'] = stats['wins'] / stats['trades']
            stats['avg_hold'] /= stats['trades']
        
        return pair_stats
    
    def calculate_daily_stats(self, trades):
        """按日统计"""
        daily = {}
        
        for t in trades:
            date = t['timestamp'][:10]  # YYYY-MM-DD
            if date not in daily:
                daily[date] = {
                    'trades': 0,
                    'pnl': 0,
                    'wins': 0
                }
            
            daily[date]['trades'] += 1
            daily[date]['pnl'] += t['pnl_usd']
            if t['pnl_usd'] > 0:
                daily[date]['wins'] += 1
        
        return daily
    
    def generate_report(self):
        """生成完整报告"""
        metrics = self.calculate_metrics()
        
        report = f"""
📊 S001-Pro 绩效报告
━━━━━━━━━━━━━━━━━━━━

💰 总体表现:
  总交易: {metrics['total_trades']}笔
  总盈亏: ${metrics['total_pnl']:.2f}
  胜率: {metrics['win_rate']*100:.1f}%
  夏普比率: {metrics['sharpe_ratio']:.2f}
  最大回撤: {metrics['max_drawdown']*100:.1f}%
  盈利因子: {metrics['profit_factor']:.2f}
  期望值: ${metrics['expectancy']:.2f}/笔

⏱️ 持仓特征:
  平均持仓: {metrics['avg_hold_time']:.0f}分钟
  最大盈利: ${metrics['max_win']:.2f}
  最大亏损: ${metrics['max_loss']:.2f}

📈 配对表现 Top 5:
{self.format_top_pairs(metrics.get('per_pair_stats', {}))}

📅 近7日表现:
{self.format_recent_days(metrics.get('daily_stats', {}))}

━━━━━━━━━━━━━━━━━━━━
        """
        
        return report
```

---

## 🔧 八、动态参数体系

### 8.1 参数约束

```yaml
E (进场阈值):
  范围: 1.5 ≤ E ≤ 4.0
  优化步长: 0.1
  说明: 过低→频繁交易，过高→错过机会

X (出场阈值):
  范围: 0.3 ≤ X ≤ E - 0.3
  优化步长: 0.1
  说明: 保证利润空间覆盖成本

S (止损阈值):
  规则: S = max(3.5, E + 0.8)
  说明: 对齐Z_STOP铁律，保底3.5

固定参数 (不优化):
  - 分层偏移: [-0.3, 0, +0.5]
  - 分层权重: [30%, 30%, 40%]
  - 双确认窗口: 3分钟
  - 统计窗口: 288根 (5m)
  - 打分权重: 固定8因子
  - 筛选阈值: 二筛10条件固定
```

### 8.2 在线自适应 (可选)

```yaml
自适应规则:
  评估频率: 每30分钟
  评估窗口: 最近2小时交易表现
  
  调整逻辑:
    胜率 > 60%: E = E - 0.2 (放宽入场)
    胜率 < 40%: E = E + 0.2 (收紧入场)
    
  约束:
    单次调整幅度: ≤ 0.2
    累计调整幅度: ≤ 0.5 (相对Optuna优化值)
    下次扫描时重置为Optuna优化值
  
  版本管理:
    param_version: 递增
    param_set_at: 时间戳
    param_method: "optuna" | "adaptive"
```

### 8.3 参数稳定性验证

```python
def validate_parameter_stability(self, best_params, db_path):
    """
    验证最优参数的稳定性
    
    检查 E*±0.2, X*±0.1, S*±0.2 的Sharpe变化
    如果变化>20% → 标记"不稳定"
    """
    E_star = best_params['z_entry']
    X_star = best_params['z_exit']
    S_star = best_params['z_stop']
    
    baseline_sharpe = self.backtest(E_star, X_star, S_star, db_path)
    
    perturbations = [
        (E_star + 0.2, X_star, S_star),
        (E_star - 0.2, X_star, S_star),
        (E_star, X_star + 0.1, S_star),
        (E_star, X_star - 0.1, S_star),
        (E_star, X_star, S_star + 0.2),
        (E_star, X_star, S_star - 0.2),
    ]
    
    sensitivities = []
    for E, X, S in perturbations:
        sharpe = self.backtest(E, X, S, db_path)
        change = abs(sharpe - baseline_sharpe) / abs(baseline_sharpe)
        sensitivities.append(change)
    
    max_sensitivity = max(sensitivities)
    
    return {
        'stable': max_sensitivity < 0.2,
        'max_sensitivity': max_sensitivity,
        'baseline_sharpe': baseline_sharpe
    }
```

---

## 📋 九、回测强化规格

### 9.1 回测配置

```yaml
回测设置:
  窗口: 滚动3段×30天 (覆盖不同行情)
  Trials: 100 (TPE + CMA-ES混合)
  成本模型:
    手续费: 0.0005 (maker+可能taker)
    滑点: 0.0007 (极端行情放大)
    资金费: 每8小时计一次
    四腿成本: 4 × (0.0005 + 0.0007) = 0.0048 = 0.48%
  延迟: 300ms (保守估计)

Walk-Forward验证:
  训练集: 前60天
  验证集: 后30天
  要求: 验证集Sharpe ≥ 训练集Sharpe × 0.6

多目标优化:
  主目标: 0.5×Sharpe + 0.2×Return - 0.2×Drawdown + 0.1×Calmar
  额外要求:
    - 胜率 > 45%
    - 最大回撤 < 15%
    - 交易次数 ≥ 20 (样本量够)
    - per_leg PnL > 0 (不能一条腿亏一条腿赚)
```

### 9.2 成本模型校准

```python
def calculate_real_cost(self, notional_a, notional_b, hold_hours):
    """
    计算真实交易成本
    
    四腿: 开仓2腿 + 平仓2腿
    """
    # 手续费 (双边)
    fee_rate = 0.0005
    total_notional = notional_a + notional_b
    fee_cost = 4 * fee_rate * total_notional  # 4腿
    
    # 滑点 (保守估计)
    slip_rate = 0.0007
    slip_cost = 4 * slip_rate * total_notional
    
    # 资金费 (每8小时)
    funding_periods = hold_hours / 8
    funding_cost = funding_periods * 0.0001 * total_notional  # 假设平均资金费率
    
    total_cost = fee_cost + slip_cost + funding_cost
    
    return {
        'fee': fee_cost,
        'slippage': slip_cost,
        'funding': funding_cost,
        'total': total_cost,
        'cost_pct': total_cost / total_notional
    }
```

---

## 🚨 十、P0/P1/P2 问题修复清单

### 10.1 P0 问题 (立即修复)

| # | 问题 | 影响 | 修复方案 | 状态 |
|---|------|------|----------|------|
| 1 | 双腿同步失败时"平已成交腿" = 制造裸仓 | 直接违反铁律 | 改为追单+保护性止损 | ❌ 待修复 |
| 2 | spread_cv除零风险 | 统计量崩溃 | 改用spread_robust_metric | ❌ 待修复 |
| 3 | S参数约束与Z_STOP=3.5铁律矛盾 | 止损可能失效 | 统一为S=max(3.5, E+0.8) | ❌ 待修复 |

### 10.2 P1 问题 (优先修复)

| # | 问题 | 影响 | 修复方案 | 状态 |
|---|------|------|----------|------|
| 4 | pairs_v2.json无原子更新 | 可能读到损坏文件 | write temp → rename | ❌ 待修复 |
| 5 | 双确认与分层入场逻辑冲突 | 入场行为不可预测 | 统一状态机流程 | ❌ 待修复 |
| 6 | 成本模型低估 (0.48%实际 vs 文档值) | 回测虚高 | 更新成本参数 | ❌ 待修复 |
| 7 | 重启对账机制缺失 | 可能重复开仓或丢失仓位 | 实现reconcile_positions | ❌ 待修复 |

### 10.3 P2 问题 (计划修复)

| # | 问题 | 影响 | 修复方案 | 状态 |
|---|------|------|----------|------|
| 8 | 回测窗口30天过短 | 参数过拟合 | 改为滚动3段×30天 | ❌ 待修复 |
| 9 | Optuna 50 trials不足 | 参数次优 | 提高到100 trials | ❌ 待修复 |
| 10 | 缺少Walk-Forward验证 | 无法验证泛化性 | 增加验证集 | ❌ 待修复 |

---

## 📁 十一、项目文件结构

```
S001-Pro/
├── config/
│   └── strategy.yaml          # 系统配置
├── src/
│   ├── main.py                # Runtime (主循环)
│   ├── scanner.py             # ScanEngine (扫描引擎)
│   ├── signal_engine.py       # SignalEngine (信号计算)
│   ├── executor.py            # IsolatedExecutor (逐仓执行器)
│   ├── state.py               # StateManager (状态管理)
│   ├── optimizer.py           # ParamOptimizer (参数优化)
│   ├── notifier.py            # TelegramNotifier (消息推送) [新增]
│   └── analyzer.py            # PerformanceAnalyzer (统计分析) [新增]
├── data/
│   ├── klines.db              # K线数据库 (只读)
│   ├── pairs_v2.json          # 扫描输出 (Top 30配对)
│   ├── state.json             # 状态持久化 (持仓+PnL)
│   └── trades.jsonl           # 交易日志 (逐笔记录)
├── docs/
│   └── SPEC_MASTER.md         # 本文档
├── tests/
│   ├── test_scanner.py
│   ├── test_signal.py
│   ├── test_executor.py
│   └── test_state.py
└── scripts/
    ├── deploy.sh              # 部署脚本
    └── download_klines.py     # K线下载脚本
```

---

## 📝 十二、开发检查清单

### 12.1 代码提交前检查

```yaml
P0 铁律检查:
  - [ ] 不产生裸仓 (所有异常路径都处理)
  - [ ] 基于实际代码 (ssh grep验证)
  - [ ] 实盘改动先备份
  
P1 流程检查:
  - [ ] git commit + git push (自动)
  - [ ] 语法检查通过
  - [ ] 单元测试通过
  - [ ] 真实回测验证
  
P2 偏好检查:
  - [ ] 中文注释
  - [ ] 代码挑刺审查
  - [ ] Mac本地优先开发
```

### 12.2 部署检查清单

```yaml
部署前:
  - [ ] 代码已推送到GitHub
  - [ ] 服务器拉取最新代码
  - [ ] 运行语法检查
  - [ ] 检查配置文件
  
部署中:
  - [ ] 备份当前运行版本
  - [ ] 停止旧服务
  - [ ] 启动新服务
  - [ ] 检查日志输出
  
部署后:
  - [ ] 验证服务正常运行
  - [ ] 检查Telegram通知
  - [ ] 确认持仓对账
  - [ ] 监控15分钟无异常
```

---

## 🔮 十三、未来优化方向

1. **多时间框架融合**: 5m信号 + 1m确认 + 15m趋势过滤
2. **机器学习增强**: 用XGBoost替代线性打分
3. **动态配对池**: 根据市场状态自动调整扫描频率
4. **风险平价**: 按波动率分配仓位，而非固定保证金
5. **多交易所套利**: 跨交易所统计套利
6. **实时风控面板**: Web UI展示实时风险指标

---

> **文档维护规则**:
> - 每次代码改动后同步更新本文档
> - P0/P1修复后立即更新对应章节
> - 版本号与代码版本号保持一致
> - 所有修改必须经过代码验证
