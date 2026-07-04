# AI Quant Lab

股票技术指标交互式实验室 - 基于纯 HTML/CSS/JS 和 Chart.js 构建，支持多股票切换和指标参数实时调节。

## 功能特性

- **多股票切换**：中芯国际 (00981.HK)、长江电力 (600900.SH)
- **四大技术指标**：
  - **RSI** (相对强弱指标) - 可调周期、超买/超卖阈值
  - **MACD** (指数平滑异同移动平均线) - 可调快线/慢线/信号线周期
  - **Bollinger Bands** (布林带) - 可调周期和标准差倍数
  - **ATR** (平均真实波幅) - 可调周期
- **交互式图表**：参数实时调节，图表自动重绘
- **价格展示**：支持收盘价折线图和 K 线柱状图切换
- **中国市场配色**：涨红跌绿

## 目录结构

```
ai_quant_lab/
├── index.html              # 主应用页面
├── README.md               # 项目说明
├── .gitignore
├── build_data.py           # 数据构建脚本 (CSV -> data.js)
├── src/
│   └── data.js             # 内嵌股票数据 (自动生成)
├── data/
│   ├── 00981_HK.csv        # 中芯国际港股日线数据
│   └── 600900_SH.csv       # 长江电力A股日线数据
├── notebooks/
│   └── indicator_lab_smic_00981HK.ipynb  # Jupyter Notebook 指标计算
├── output/
│   └── smic_00981HK_indicators.csv       # 中芯国际指标计算结果
└── docs/
    ├── design.md           # 产品设计文档
    ├── stock_data_spec.md  # 取数规范 spec
    └── task02_indicator_lab.md  # 指标实验室 spec
```

## 快速开始

直接在浏览器中打开 `index.html` 即可使用，无需安装任何依赖。

或在本地启动一个简易服务器：

```bash
python -m http.server 8000
# 然后访问 http://localhost:8000
```

## 技术栈

- **前端**：纯 HTML5 + CSS3 + JavaScript (ES6)
- **图表库**：Chart.js v4.4.1
- **数据源**：Tushare API
- **计算引擎**：浏览器端实时计算所有指标

## 指标计算说明

### RSI (Relative Strength Index)
- 使用 Wilder 平滑法
- 默认周期 14，超买线 70，超卖线 30

### MACD
- DIF = EMA(fast) - EMA(slow)
- DEA = EMA(DIF, signal)
- MACD柱 = 2 × (DIF - DEA)
- 默认参数：12/26/9

### Bollinger Bands
- 中轨 = SMA(close, period)
- 上下轨 = 中轨 ± stdDev × σ
- 默认参数：周期 20，标准差倍数 2

### ATR (Average True Range)
- TR = max(H-L, |H-PC|, |L-PC|)
- ATR 使用 Wilder 平滑法
- 默认周期 14

## 数据说明

- 数据时间范围：2025-01-02 至 2026-07-03
- 数据来源：Tushare API (`hk_daily` / `daily`)
- 所有数据仅供参考学习，不构成投资建议

## License

MIT
