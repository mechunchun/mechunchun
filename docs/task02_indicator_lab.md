# 技术指标计算实验规范 (Indicator Lab Spec)

> 版本: 1.0 | 更新日期: 2026-07-04
> 任务编号: task02 | 关联规范: stock_data_spec.md v1.0
> 本文件定义了基于中芯国际港股价格数据计算四项技术指标的标准化流程，以 Jupyter Notebook 为载体展现完整计算过程。

---

## 1. 任务概述

### 1.1 目标
以中芯国际港股（00981.HK）日线行情数据为输入，计算 RSI、MACD、布林带、ATR 四项技术指标，通过 Jupyter Notebook 逐单元格展示从数据加载、预处理、公式推导到可视化输出的完整过程，形成可复现、可教学的技术指标实验记录。

### 1.2 设计原则
- **可读性优先**: 每个 Notebook 单元格聚焦一个逻辑步骤，Markdown 说明与 Code 交替排列
- **从零实现**: 指标计算不依赖 `talib` 等黑盒库，使用 pandas/numpy 手写公式，便于理解原理
- **中间变量保留**: 计算过程中的中间变量（如 EMA、平均涨幅等）保留在 DataFrame 中，不丢弃
- **可视化即验证**: 每个指标计算完成后立即配图，用图形确认计算结果正确

### 1.3 技术栈
| 组件 | 版本/说明 |
|------|-----------|
| Python | 3.13 |
| pandas | ≥ 2.0 |
| numpy | ≥ 1.24 |
| matplotlib | ≥ 3.7 |
| Jupyter Notebook | ≥ 7.0 |
| 字体 | Microsoft YaHei / SimHei |

---

## 2. 数据输入

### 2.1 数据源
遵循 `stock_data_spec.md` 规范，使用 Tushare `hk_daily` 接口获取的港股日线行情。

### 2.2 标的信息

| 项目 | 值 |
|------|-----|
| 股票名称 | 中芯国际 |
| 股票代码 | 00981.HK |
| 交易所 | 香港交易所（港股通标的） |
| 行业 | 半导体 |
| 币种 | 港元 (HKD) |

### 2.3 输入字段
仅使用日线行情 (`daily_quotes`) 中的以下字段：

| 字段 | 类型 | 用途 |
|------|------|------|
| trade_date | str | 日期索引 (YYYYMMDD) |
| open | float | 开盘价 |
| high | float | 最高价（ATR 需要） |
| low | float | 最低价（ATR 需要） |
| close | float | 收盘价（RSI/MACD/布林带核心输入） |
| pre_close | float | 昨收价（ATR 需要） |
| vol | float | 成交量（辅助参考） |

### 2.4 数据范围
```yaml
date_range:
  start_date: "20250101"
  end_date: "20260704"
  expected_records: "~250 个交易日"
  rationale: "约1年数据，足够计算26日EMA等长周期指标，同时图表不过于拥挤"
```

### 2.5 数据预处理要求
1. 将 `trade_date` 从 `YYYYMMDD` 字符串转为 `datetime` 类型并设为索引
2. 按日期**升序排列**（指标计算需要时间正序）
3. 检查缺失值：`close` 字段不得有 NaN，如有则前向填充 (`ffill`)
4. 检查价格合理性：`high >= low`，`close > 0`

---

## 3. 指标定义与公式

### 3.1 RSI — 相对强弱指标

**类别**: 动量类指标
**默认参数**: 14 日
**回答的问题**: 近期涨势强还是跌势强？是否超买/超卖？

#### 公式
```
步骤 1: 计算每日价格变动
    ΔP_t = Close_t - Close_{t-1}

步骤 2: 分离涨幅和跌幅
    Gain_t  = max(ΔP_t, 0)
    Loss_t  = max(-ΔP_t, 0)

步骤 3: 计算平均涨幅和平均跌幅（Wilder 平滑法）
    第一个平均值 = 前14日的简单算术平均
    AvgGain_t = (AvgGain_{t-1} × 13 + Gain_t) / 14
    AvgLoss_t = (AvgLoss_{t-1} × 13 + Loss_t) / 14

步骤 4: 计算相对强弱值
    RS_t = AvgGain_t / AvgLoss_t    （AvgLoss=0 时 RS=∞, RSI=100）

步骤 5: 计算RSI
    RSI_t = 100 - 100 / (1 + RS_t)
```

#### 输出字段
| 字段名 | 说明 |
|--------|------|
| price_change | 当日价格变动 |
| gain | 涨幅（≥0） |
| loss | 跌幅（≥0） |
| avg_gain | 14日平均涨幅（Wilder平滑） |
| avg_loss | 14日平均跌幅（Wilder平滑） |
| rs | 相对强弱值 |
| rsi_14 | RSI 指标值 (0~100) |

#### 信号阈值
- RSI > 70 → 超买区
- RSI < 30 → 超卖区
- 30 ≤ RSI ≤ 70 → 正常区

---

### 3.2 MACD — 指数平滑异同移动平均线

**类别**: 趋势类指标
**默认参数**: 快线 12 日，慢线 26 日，信号线 9 日
**回答的问题**: 趋势方向是什么？动量在加速还是减速？

#### 公式
```
步骤 1: 计算指数移动平均（EMA）
    EMA_t = α × Price_t + (1-α) × EMA_{t-1}
    其中 α = 2 / (N+1)
    EMA12: α = 2/13 ≈ 0.1538
    EMA26: α = 2/27 ≈ 0.0741

    初始值: EMA_0 = 前 N 日的简单移动平均 (SMA)

步骤 2: 计算离差值 DIF（快线）
    DIF_t = EMA12_t - EMA26_t

步骤 3: 计算信号线 DEA（慢线）
    DEA_t = EMA9(DIF_t)
    即对 DIF 再做 9 日 EMA

步骤 4: 计算MACD柱状图
    MACD_hist_t = (DIF_t - DEA_t) × 2
```

#### 输出字段
| 字段名 | 说明 |
|--------|------|
| ema_12 | 12日指数移动平均 |
| ema_26 | 26日指数移动平均 |
| dif | 离差值（快线）= EMA12 - EMA26 |
| dea | 信号线（慢线）= DIF 的 9日EMA |
| macd_hist | MACD 柱状值 = (DIF - DEA) × 2 |

#### 信号规则
- DIF 上穿 DEA → **金叉**（买入信号）
- DIF 下穿 DEA → **死叉**（卖出信号）
- 柱状图由绿转红 → 动量由弱转强
- 柱状图由红转绿 → 动量由强转弱

---

### 3.3 Bollinger Bands — 布林带

**类别**: 统计类指标
**默认参数**: 周期 20 日，标准差倍数 2
**回答的问题**: 价格相对均值偏离多远？波动率在扩大还是收缩？

#### 公式
```
步骤 1: 计算中轨（20日简单移动平均）
    MA_t = mean(Close_{t-19}, ..., Close_t)

步骤 2: 计算20日标准差（总体标准差，ddof=0）
    σ_t = std(Close_{t-19}, ..., Close_t, ddof=0)

步骤 3: 计算上下轨
    Upper_t = MA_t + 2 × σ_t
    Lower_t = MA_t - 2 × σ_t

步骤 4: 计算 %B（价格在通道中的位置）
    %B_t = (Close_t - Lower_t) / (Upper_t - Lower_t)

步骤 5: 计算带宽（通道宽度占比）
    Bandwidth_t = (Upper_t - Lower_t) / MA_t × 100
```

#### 输出字段
| 字段名 | 说明 |
|--------|------|
| bb_ma | 中轨 = 20日SMA |
| bb_std | 20日标准差 |
| bb_upper | 上轨 = MA + 2σ |
| bb_lower | 下轨 = MA - 2σ |
| bb_pct_b | %B 指标 (0=下轨, 1=上轨) |
| bb_bandwidth | 带宽 (%) |

#### 信号规则
- 价格触及/突破上轨 → 短期偏强，警惕回落
- 价格触及/突破下轨 → 短期偏弱，关注反弹
- 带宽收窄至低位 → 波动率极低，可能酝酿突破
- 带宽放大 → 波动率上升，趋势加速

---

### 3.4 ATR — 真实波幅指标

**类别**: 波动率类指标
**默认参数**: 14 日
**回答的问题**: 每天平均波动多少港元？止损应该设多宽？

#### 公式
```
步骤 1: 计算真实波幅 (True Range)
    TR_t = max(
        High_t - Low_t,                    # 当日振幅
        |High_t - PreClose_t|,             # 跳空高开
        |Low_t  - PreClose_t|              # 跳空低开
    )

    注: 第一行无 PreClose, TR = High - Low

步骤 2: 计算 ATR（Wilder 平滑法）
    第一个 ATR = 前14日 TR 的简单算术平均
    ATR_t = (ATR_{t-1} × 13 + TR_t) / 14

步骤 3: 计算 ATR 止损线（可选）
    ATR_stop_long  = Close_t - 2 × ATR_t    # 多头止损
    ATR_stop_short = Close_t + 2 × ATR_t    # 空头止损
```

#### 输出字段
| 字段名 | 说明 |
|--------|------|
| tr | 真实波幅 (True Range) |
| atr_14 | 14日 ATR（Wilder平滑） |
| atr_pct | ATR占收盘价百分比 (%) = ATR / Close × 100 |
| atr_stop_long | 多头止损线 = Close - 2×ATR |

#### 信号规则
- ATR 上升 → 波动率放大，市场活跃
- ATR 下降 → 波动率收敛，市场平淡
- 止损设置: 止损价 = 买入价 - N × ATR（N 常取 2~3）

---

## 4. 指标参数配置

```yaml
indicators:
  rsi:
    period: 14                # 计算周期
    overbought: 70            # 超买阈值
    oversold: 30              # 超卖阈值

  macd:
    fast_period: 12           # 快线EMA周期
    slow_period: 26           # 慢线EMA周期
    signal_period: 9          # 信号线EMA周期
    multiplier: 2             # 柱状图乘数

  bollinger:
    period: 20                # 中轨SMA周期
    std_multiplier: 2         # 标准差倍数
    std_ddof: 0               # 总体标准差（ddof=0）

  atr:
    period: 14                # 计算周期
    stop_multiplier: 2        # 止损倍数
```

---

## 5. Notebook 逐单元格设计

Notebook 文件名: `indicator_lab_smic_00981HK.ipynb`

### 5.1 单元格规划总览

| 序号 | 类型 | 标题 | 内容摘要 |
|------|------|------|----------|
| 1 | Markdown | 标题与简介 | 任务说明、标的信息、指标清单 |
| 2 | Code | 环境准备 | 导入依赖库、全局样式设置 |
| 3 | Markdown | 数据加载说明 | 数据来源、字段说明、预处理步骤 |
| 4 | Code | 加载数据 | 读取CSV、日期转换、排序、缺失值检查 |
| 5 | Code | 数据概览 | head/describe/shape、价格走势图 |
| 6 | Markdown | RSI 计算说明 | 原理简述、公式、参数 |
| 7 | Code | RSI 计算 | 逐步实现 Wilder 平滑法 |
| 8 | Code | RSI 可视化 | 价格 + RSI 双面板图，标注超买超卖区 |
| 9 | Markdown | MACD 计算说明 | 原理简述、公式、参数 |
| 10 | Code | EMA 辅助函数 | 定义 `calc_ema(series, period)` |
| 11 | Code | MACD 计算 | DIF / DEA / 柱状图 |
| 12 | Code | MACD 可视化 | 价格 + DIF/DEA + 柱状图三面板 |
| 13 | Markdown | 布林带计算说明 | 原理简述、公式、参数 |
| 14 | Code | 布林带计算 | 中轨/上下轨/%B/带宽 |
| 15 | Code | 布林带可视化 | 价格 + 通道填充 + %B 面板 |
| 16 | Markdown | ATR 计算说明 | 原理简述、公式、参数 |
| 17 | Code | ATR 计算 | TR / ATR / 止损线 |
| 18 | Code | ATR 可视化 | 价格 + ATR + ATR% 面板 |
| 19 | Markdown | 综合分析 | 四指标交叉解读 |
| 20 | Code | 综合面板 | 4合1 子图 dashboard |
| 21 | Code | 导出结果 | 合并指标列输出 CSV |
| 22 | Markdown | 结论与延伸 | 指标解读、局限性、后续方向 |

### 5.2 各单元格详细规格

---

#### Cell 1 — Markdown: 标题与简介
```markdown
# 中芯国际 (00981.HK) 技术指标计算实验

> 本 Notebook 基于 2025-01-01 至 2026-07-04 的港股日线数据，
> 从零计算 RSI、MACD、布林带、ATR 四项技术指标，展现完整推导过程。

**标的信息**: 中芯国际 00981.HK | 港交所 | 半导体 | 港元
**数据范围**: ~250 个交易日
**指标清单**: RSI(14) / MACD(12,26,9) / BOLL(20,2) / ATR(14)
```

---

#### Cell 2 — Code: 环境准备
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
from pathlib import Path

# 中文字体 & 全局样式
plt.rcParams['font.sans-serif'] = ['Microsoft YaHei', 'SimHei']
plt.rcParams['axes.unicode_minus'] = False
plt.rcParams['figure.dpi'] = 120

# 配色（中国股市惯例：红涨绿跌）
COLOR_UP    = '#e74c3c'
COLOR_DOWN  = '#27ae60'
COLOR_FLAT  = '#95a5a6'
COLOR_BG    = '#f8f9fa'

# 数据路径
DATA_FILE = Path("../data/00981_HK/daily_quotes.csv")
OUTPUT_DIR = Path("../output")
OUTPUT_DIR.mkdir(exist_ok=True)
```

---

#### Cell 3 — Markdown: 数据加载说明
简述数据来源（Tushare hk_daily）、字段定义、预处理步骤（日期转换→升序→缺失值检查）。

---

#### Cell 4 — Code: 加载数据
```python
df = pd.read_csv(DATA_FILE, encoding='utf-8-sig', parse_dates=['trade_date'])
df = df.set_index('trade_date').sort_index()  # 升序排列

# 缺失值检查
assert df['close'].isna().sum() == 0, "close 字段存在缺失值"
# 价格合理性检查
assert (df['high'] >= df['low']).all(), "存在 high < low 的异常记录"

print(f"数据范围: {df.index[0].date()} ~ {df.index[-1].date()}")
print(f"记录数: {len(df)}")
df[['open','high','low','close','vol']].head()
```

---

#### Cell 5 — Code: 数据概览 + 价格走势图
```python
fig, ax = plt.subplots(figsize=(14, 5))
ax.plot(df.index, df['close'], color='#2c3e50', linewidth=1.2, label='收盘价')
ax.fill_between(df.index, df['low'], df['high'], alpha=0.1, color='#2c3e50', label='日内振幅')
ax.set_title(f'中芯国际 (00981.HK) 收盘价走势  {df.index[0].date()} ~ {df.index[-1].date()}',
             fontsize=14)
ax.set_ylabel('价格 (港元)', fontsize=12)
ax.grid(True, alpha=0.3, linestyle='--')
ax.legend(loc='upper left')
plt.tight_layout()
plt.show()

df[['open','high','low','close','vol']].describe().round(2)
```

---

#### Cell 6 — Markdown: RSI 计算说明
> **RSI (相对强弱指标)** 衡量价格变动的动量强度。  
> 核心：N日内平均涨幅占"涨幅+跌幅"的比例。  
> 使用 Wilder 平滑法（指数加权，权重 1/N）。  
> 超买区 > 70，超卖区 < 30。

---

#### Cell 7 — Code: RSI 计算（逐步）
```python
# 步骤1: 价格变动
df['price_change'] = df['close'].diff()

# 步骤2: 分离涨跌
df['gain'] = df['price_change'].clip(lower=0)
df['loss'] = (-df['price_change']).clip(lower=0)

# 步骤3: Wilder 平滑（等价于 EMA with alpha=1/period）
period = 14
df['avg_gain'] = df['gain'].ewm(alpha=1/period, adjust=False).mean()
df['avg_loss'] = df['loss'].ewm(alpha=1/period, adjust=False).mean()

# 步骤4: RS
df['rs'] = df['avg_gain'] / df['avg_loss']

# 步骤5: RSI
df['rsi_14'] = 100 - (100 / (1 + df['rs']))
# avg_loss=0 时 RSI=100
df.loc[df['avg_loss'] == 0, 'rsi_14'] = 100

df[['close','price_change','gain','loss','avg_gain','avg_loss','rs','rsi_14']].tail(10).round(3)
```

---

#### Cell 8 — Code: RSI 可视化
- 上方面板：收盘价折线
- 下方面板：RSI 折线，70/30 水平虚线，超买超卖区填色

---

#### Cell 9 — Markdown: MACD 计算说明
> **MACD** 通过快慢 EMA 的离差捕捉趋势方向。  
> DIF = EMA(12) - EMA(26)，DEA = EMA(DIF, 9)，柱状图 = (DIF - DEA) × 2。  
> 金叉/死叉是核心信号。

---

#### Cell 10 — Code: EMA 辅助函数
```python
def calc_ema(series: pd.Series, period: int) -> pd.Series:
    """计算指数移动平均 (EMA)
    alpha = 2/(period+1), 初始值 = 前 period 日的 SMA
    """
    return series.ewm(span=period, adjust=False).mean()
```

---

#### Cell 11 — Code: MACD 计算
```python
fast, slow, signal = 12, 26, 9

df['ema_12'] = calc_ema(df['close'], fast)
df['ema_26'] = calc_ema(df['close'], slow)
df['dif']    = df['ema_12'] - df['ema_26']
df['dea']    = calc_ema(df['dif'], signal)
df['macd_hist'] = (df['dif'] - df['dea']) * 2

# 识别金叉/死叉
df['dif_above'] = df['dif'] > df['dea']
df['cross'] = df['dif_above'].diff()
golden_cross = df[df['cross'] == True].index   # 金叉
death_cross  = df[df['cross'] == False].index  # 死叉

df[['close','ema_12','ema_26','dif','dea','macd_hist']].tail(10).round(3)
```

---

#### Cell 12 — Code: MACD 可视化
- 上面板：收盘价 + EMA12/EMA26 叠加
- 中面板：DIF / DEA 折线，标注金叉/死叉点
- 下面板：MACD 柱状图（红涨绿跌）

---

#### Cell 13 — Markdown: 布林带计算说明
> **布林带** 用 20日SMA ± 2倍标准差构建价格通道。  
> ~95% 的价格落在通道内。通道收窄 = 波动率低，可能酝酿突破。

---

#### Cell 14 — Code: 布林带计算
```python
period, std_mult = 20, 2

df['bb_ma']      = df['close'].rolling(period).mean()
df['bb_std']     = df['close'].rolling(period).std(ddof=0)
df['bb_upper']   = df['bb_ma'] + std_mult * df['bb_std']
df['bb_lower']   = df['bb_ma'] - std_mult * df['bb_std']
df['bb_pct_b']   = (df['close'] - df['bb_lower']) / (df['bb_upper'] - df['bb_lower'])
df['bb_bandwidth'] = (df['bb_upper'] - df['bb_lower']) / df['bb_ma'] * 100

df[['close','bb_ma','bb_upper','bb_lower','bb_pct_b','bb_bandwidth']].tail(10).round(3)
```

---

#### Cell 15 — Code: 布林带可视化
- 上面板：收盘价 + 通道上下轨（半透明填充）+ 中轨虚线
- 下面板：%B 指标（0~1 范围，标注1.0和0.0水平线）

---

#### Cell 16 — Markdown: ATR 计算说明
> **ATR** 衡量价格波动的绝对幅度（单位：港元）。  
> TR = max(当日振幅, |最高-昨收|, |最低-昨收|)  
> ATR = TR 的 14日 Wilder 平滑  
> 主要用途：止损设置（止损 = 买入价 - 2×ATR）

---

#### Cell 17 — Code: ATR 计算
```python
period = 14

# 步骤1: True Range
df['h_l']  = df['high'] - df['low']
df['h_pc'] = (df['high'] - df['close'].shift(1)).abs()
df['l_pc'] = (df['low']  - df['close'].shift(1)).abs()
df['tr']   = df[['h_l','h_pc','l_pc']].max(axis=1)

# 步骤2: ATR (Wilder 平滑)
df['atr_14'] = df['tr'].ewm(alpha=1/period, adjust=False).mean()

# 步骤3: ATR 占比 & 止损线
df['atr_pct']       = df['atr_14'] / df['close'] * 100
df['atr_stop_long'] = df['close'] - 2 * df['atr_14']

df[['high','low','close','tr','atr_14','atr_pct','atr_stop_long']].tail(10).round(3)
```

---

#### Cell 18 — Code: ATR 可视化
- 上面板：收盘价 + ATR 止损线
- 中面板：ATR 绝对值（港元）
- 下面板：ATR% 百分比

---

#### Cell 19 — Markdown: 综合分析
引导读者交叉解读四项指标：趋势(MACD) + 动量(RSI) + 位置(布林带) + 波动率(ATR)。

---

#### Cell 20 — Code: 综合面板 Dashboard
4 合 1 子图布局（2行×2列 或 4行×1列），每个子图对应一个指标，共享 X 轴日期。

---

#### Cell 21 — Code: 导出结果
```python
output_cols = [
    'close',
    'rsi_14',
    'dif', 'dea', 'macd_hist',
    'bb_ma', 'bb_upper', 'bb_lower', 'bb_pct_b', 'bb_bandwidth',
    'tr', 'atr_14', 'atr_pct', 'atr_stop_long'
]
result = df[output_cols].copy()
result.to_csv(OUTPUT_DIR / "smic_00981HK_indicators.csv", encoding='utf-8-sig')
print(f"已导出 {len(result)} 条记录至 output/smic_00981HK_indicators.csv")
result.tail(5).round(3)
```

---

#### Cell 22 — Markdown: 结论与延伸
- 指标局限性（滞后性、震荡市失效等）
- 组合使用建议
- 后续方向：回测、参数优化、多标的对比

---

## 6. 输出规范

### 6.1 目录结构
```
task2/
├── stock_data_spec.md                          # 取数规范（已有）
├── task02_indicator_lab.md                     # 本规范文件
├── notebooks/
│   └── indicator_lab_smic_00981HK.ipynb        # Notebook 主体
├── data/
│   └── 00981_HK/
│       └── daily_quotes.csv                    # 输入数据
└── output/
    ├── smic_00981HK_indicators.csv             # 指标计算结果
    └── charts/                                  # Notebook 生成的图表
        ├── rsi_chart.png
        ├── macd_chart.png
        ├── bollinger_chart.png
        ├── atr_chart.png
        └── dashboard.png
```

### 6.2 输出 CSV 字段
文件: `output/smic_00981HK_indicators.csv`

| 字段 | 说明 | 来源指标 |
|------|------|----------|
| trade_date | 交易日期（索引） | — |
| close | 收盘价 | 原始数据 |
| rsi_14 | RSI(14) | RSI |
| dif | 离差值 | MACD |
| dea | 信号线 | MACD |
| macd_hist | MACD柱状图 | MACD |
| bb_ma | 布林带中轨 | BOLL |
| bb_upper | 布林带上轨 | BOLL |
| bb_lower | 布林带下轨 | BOLL |
| bb_pct_b | %B 位置 | BOLL |
| bb_bandwidth | 带宽(%) | BOLL |
| tr | 真实波幅 | ATR |
| atr_14 | ATR(14) | ATR |
| atr_pct | ATR占比(%) | ATR |
| atr_stop_long | 多头止损线 | ATR |

### 6.3 可视化配色（沿用 stock_data_spec.md §6.1）
| 用途 | 颜色 | 色值 |
|------|------|------|
| 上涨/正向 | 红色 | `#e74c3c` |
| 下跌/负向 | 绿色 | `#27ae60` |
| 中性线/均线 | 蓝色 | `#2c3e50` |
| 超买区背景 | 浅红 | `#fcebeb` |
| 超卖区背景 | 浅绿 | `#eaf3de` |
| 网格线 | 浅灰 | `alpha=0.3, linestyle='--'` |

### 6.4 图表规格
- 尺寸: 14×5 英寸（单指标），16×12 英寸（Dashboard）
- DPI: 120
- 双面板/三面板图使用 `gridspec` 控制高度比例（如 3:2 或 3:1:1）
- X 轴: 日期格式 `YYYY-MM`，`mdates.DateFormatter`
- 每张图必须有标题、轴标签、图例

---

## 7. 计算校验规则

### 7.1 数值校验

| 指标 | 校验项 | 规则 |
|------|--------|------|
| RSI | 取值范围 | 0 ≤ RSI ≤ 100 |
| RSI | 前热身期 | 前 14 行 RSI 可能为 NaN 或不稳定，属正常 |
| MACD | DIF 限制 | DIF 应在价格数量级内（±几港元） |
| MACD | 柱状图一致性 | macd_hist ≈ (dif - dea) × 2 |
| 布林带 | 通道顺序 | bb_lower ≤ bb_ma ≤ bb_upper |
| 布林带 | %B 范围 | 大部分 0~1，极端行情可超出 |
| ATR | 正值 | tr > 0, atr_14 > 0 |
| ATR | 单位 | atr_14 单位 = 港元，与 close 同量级 |

### 7.2 前热身期处理
各指标在前 N 行（热身期）因数据不足，会产生 NaN：
- RSI: 前 14 行
- MACD: 前 26 行（受慢线EMA影响）
- 布林带: 前 20 行
- ATR: 前 15 行（需 shift(1) 获取昨收）

**处理策略**: 保留 NaN 不删除，在可视化时由 matplotlib 自动忽略。在 `.tail()` 展示和 CSV 导出时说明热身期含义。

---

## 8. 工作流程

```
Step 1  准备输入数据
        ↓  确认 data/00981_HK/daily_quotes.csv 存在且符合规范
Step 2  创建 Notebook
        ↓  按 §5 逐单元格设计创建 indicator_lab_smic_00981HK.ipynb
Step 3  运行并验证
        ↓  从头到尾执行所有单元格，确认无报错
Step 4  数值校验
        ↓  按 §7 检查各指标取值范围与逻辑一致性
Step 5  图表审查
        ↓  确认每张图标题/轴标签/图例完整，配色正确
Step 6  导出结果
        →  输出 indicators.csv + charts/*.png
```

---

## 9. 变更记录

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 1.0 | 2026-07-04 | 初始版本，定义 RSI/MACD/布林带/ATR 四指标计算规范，Notebook 22 个单元格设计 |
