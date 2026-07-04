# 股票数据取数规范 (Stock Data Spec)

> 版本: 1.0 | 更新日期: 2026-07-04
> 本文件定义了股票数据采集、存储、可视化的标准化规范，供后续取数工作复用。

---

## 1. 概述

### 1.1 目标
为多只股票的行情与基本面数据建立统一、可复用的采集流程，确保数据格式一致、字段对齐、输出规范，便于后续分析与可视化。

### 1.2 适用范围
- A股（沪市/深市/科创板）与港股日线行情数据
- 基本面指标数据（市盈率、市净率、换手率等）
- 公司基础信息
- 可视化图表与交互式页面生成

### 1.3 数据源
| 项目 | 说明 |
|------|------|
| 主数据源 | Tushare Pro API |
| 行情接口 | `daily`（日线行情）/ `hk_daily`（港股日线）|
| 基本面接口 | `daily_basic`（每日指标）/ `hk_daily_basic`（港股每日指标）|
| 公司信息接口 | `stock_basic`（A股）/ `hk_basic`（港股）|
| 复权因子 | `adj_factor`（复权因子）|
| 交易日历 | `trade_cal`（交易日历）|

---

## 2. 标的清单 (Target List)

### 2.1 当前批次标的

| 序号 | 名称 | A股代码 | 港股代码 | 交易所 | 行业 | 上市结构 | 备注 |
|------|------|---------|---------|--------|------|----------|------|
| 1 | 中芯国际 | 688981.SH | 00981.HK | 上交所科创板 | 半导体 | A+H | 科创板龙头半导体 |
| 2 | 比亚迪 | 002594.SZ | 01211.HK | 深交所主板 | 新能源汽车 | A+H | 新能源车企龙头 |
| 3 | 长江电力 | 600900.SH | — | 上交所主板 | 水电/公用事业 | 纯A股 | 全球最大水电上市公司 |

### 2.2 标的选取原则
- **跨市场覆盖**: 沪市 / 深市 / 科创板
- **跨行业覆盖**: 科技半导体 / 新能源 / 公用事业
- **跨上市结构**: A+H 双重上市 / 纯A股
- **跨市值风格**: 大盘成长 / 大盘价值

### 2.3 扩展方式
后续新增标的时，在 `targets` 配置块追加条目即可，格式如下：
```yaml
targets:
  - name: "股票中文名"
    a_code: "XXXXXX.SH/SZ"   # A股代码，无则填 null
    hk_code: "XXXXX.HK"       # 港股代码，无则填 null
    exchange: "上交所/深交所"
    industry: "行业分类"
    structure: "A股/A+H"
```

---

## 3. 数据集定义 (Dataset Definitions)

每只标的采集以下三个数据集，分别落盘为独立 CSV 文件。

### 3.1 日线行情 (daily_quotes)
**A股** — 调用 `daily` 接口
**港股** — 调用 `hk_daily` 接口

| 字段名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| ts_code | str | 股票代码 | 688981.SH / 01810.HK |
| trade_date | str | 交易日期 (YYYYMMDD) | 20260629 |
| open | float | 开盘价 | 21.70 |
| high | float | 最高价 | 22.28 |
| low | float | 最低价 | 21.40 |
| close | float | 收盘价 | 21.86 |
| pre_close | float | 昨收价 | 21.42 |
| change | float | 涨跌额 | 0.44 |
| pct_chg | float | 涨跌幅 (%) | 2.05 |
| vol | float | 成交量 (股) | 206092162.0 |
| amount | float | 成交额 (元/港元) | 4522193067.26 |

### 3.2 每日基本面指标 (daily_basic)
**A股** — 调用 `daily_basic` 接口
**港股** — 调用 `hk_daily_basic` 接口

| 字段名 | 类型 | 说明 |
|--------|------|------|
| ts_code | str | 股票代码 |
| trade_date | str | 交易日期 (YYYYMMDD) |
| close | float | 当日收盘价 |
| turnover_rate | float | 换手率 (%) |
| pe | float | 市盈率 (TTM) |
| pb | float | 市净率 |
| total_mv | float | 总市值 (元/港元) |
| circ_mv | float | 流通市值 (元/港元) |

### 3.3 公司基础信息 (stock_info)
**A股** — 调用 `stock_basic` 接口
**港股** — 调用 `hk_basic` 接口

| 字段名 | 类型 | 说明 |
|--------|------|------|
| ts_code | str | 股票代码 |
| name | str | 股票名称 |
| industry | str | 行业 |
| market | str | 市场类型 |
| list_date | str | 上市日期 (YYYYMMDD) |

---

## 4. 取数参数规范 (Parameters)

### 4.1 时间范围
```yaml
date_range:
  start_date: "20250101"    # 起始日期 (YYYYMMDD)
  end_date: "20260704"      # 截止日期 (YYYYMMDD)，默认取当天
  default_window: "1Y"      # 默认窗口：近1年
```

### 4.2 复权方式
```yaml
adjust:
  type: "none"    # none=不复权 / qfq=前复权 / hfq=后复权
  note: "行情数据默认不复权；如需复权，通过 adj_factor 单独计算"
```

### 4.3 频率
```yaml
frequency: "D"   # D=日线 / W=周线 / M=月线，默认日线
```

---

## 5. 输出文件规范 (Output Specification)

### 5.1 目录结构
```
BA工作坊/
├── stock_data_spec.md              # 本规范文件
├── data/                            # 数据目录
│   ├── 688981_SH/                   # 以A股代码命名（下划线替换点号）
│   │   ├── daily_quotes.csv         # 日线行情
│   │   ├── daily_basic.csv          # 每日指标
│   │   └── stock_info.csv           # 公司信息
│   ├── 002594_SZ/
│   │   ├── ...
│   ├── 600900_SH/
│   │   ├── ...
│   ├── 00981_HK/                    # 港股标的（如有）
│   │   └── ...
│   └── 01211_HK/
│       └── ...
├── charts/                          # 静态图表目录
│   ├── 688981_SH_chart.png
│   ├── 002594_SZ_chart.png
│   └── 600900_SH_chart.png
├── pages/                           # 交互式HTML页面目录
│   ├── 688981_SH.html
│   ├── 002594_SZ.html
│   └── 600900_SH.html
└── scripts/                         # 脚本目录
    └── fetch_stock_data.py          # 通用取数脚本
```

### 5.2 文件命名规则
| 文件类型 | 命名格式 | 示例 |
|----------|----------|------|
| 行情数据 | `{code}/daily_quotes.csv` | `688981_SH/daily_quotes.csv` |
| 基本面数据 | `{code}/daily_basic.csv` | `688981_SH/daily_basic.csv` |
| 公司信息 | `{code}/stock_info.csv` | `688981_SH/stock_info.csv` |
| 静态图表 | `charts/{code}_chart.png` | `charts/688981_SH_chart.png` |
| 交互页面 | `pages/{code}.html` | `pages/688981_SH.html` |

> **代码转换规则**: 将 ts_code 中的 `.` 替换为 `_`，如 `688981.SH` → `688981_SH`。

### 5.3 CSV 格式规范
- 编码: UTF-8 with BOM (`utf-8-sig`)，确保 Excel 正确显示中文
- 分隔符: 逗号
- 日期格式: `YYYYMMDD`（纯数字字符串，无分隔符）
- 数值: 不加千分位，保留原始精度
- 排序: 按 `trade_date` 降序（最新在前），与 Tushare 默认一致

---

## 6. 可视化规范 (Visualization Specification)

### 6.1 配色约定（中国股市惯例）
| 涨跌 | 颜色 | 色值 | 说明 |
|------|------|------|------|
| 上涨 | 红色 | `#e74c3c` | 中国股市红涨绿跌，与国际相反 |
| 下跌 | 绿色 | `#27ae60` | |
| 平盘 | 灰色 | `#95a5a6` | |

### 6.2 静态图表 (Python + Matplotlib)
- 尺寸: 14×6 英寸，DPI 150
- 字体: Microsoft YaHei / SimHei（中文兼容）
- 背景: 白色
- 必含元素:
  - 标题（股票名称+代码+时间范围）
  - X轴：日期（YYYY-MM 格式）
  - Y轴：价格（标注币种：元/港元）
  - 最高/最低点标注（散点+箭头注释）
  - 网格线（浅灰虚线，alpha=0.3）
  - 去除顶部和右侧边框

### 6.3 交互式页面 (HTML + Chart.js)
- 框架: Chart.js 4.x（CDN 引入）
- 必含模块:
  - 顶部数据卡片（最新价、涨跌幅、最高、最低）
  - 主图表区（折线图，支持缩放/悬停）
  - 涨跌色与6.1一致
- 响应式布局，最大宽度 1100px

---

## 7. 工作流程 (Workflow)

```
Step 1  确认标的清单
        ↓  在 spec §2 标的清单中定义/更新股票代码
Step 2  检查交易日历
        ↓  调用 trade_cal 确认取数区间的实际交易日
Step 3  采集公司基础信息
        ↓  调用 stock_basic / hk_basic → stock_info.csv
Step 4  采集日线行情
        ↓  调用 daily / hk_daily → daily_quotes.csv
Step 5  采集基本面指标
        ↓  调用 daily_basic / hk_daily_basic → daily_basic.csv
Step 6  数据质量校验
        ↓  详见 §8 质量校验清单
Step 7  生成静态图表
        ↓  Python + Matplotlib → charts/{code}_chart.png
Step 8  生成交互式页面
        ↓  HTML + Chart.js → pages/{code}.html
Step 9  汇总报告
        →  输出取数摘要（标的数量、记录条数、时间跨度）
```

---

## 8. 数据质量校验 (Quality Checks)

每批次取数完成后，执行以下校验：

| 检查项 | 规则 | 处理方式 |
|--------|------|----------|
| 记录数 | 交易日数量与 trade_cal 匹配 | 缺失日记录日志，标记需补取 |
| 字段完整性 | 必填字段（ts_code, trade_date, close）非空 | 缺失行标记并告警 |
| 价格合理性 | open/high/low/close > 0；high ≥ low | 异常行标记，人工复核 |
| 涨跌幅校验 | \|pct_chg\| < 20%（正常范围） | 超阈值标记（可能停复牌） |
| 日期连续性 | 无非交易日数据（周末/节假日） | 多余行删除 |
| 跨市场对齐 | A+H标的两个代码记录数应接近 | 差异过大时检查交易日历 |

---

## 9. 配置模板 (Config Template)

以下 YAML 配置块可作为取数脚本的输入参数，实现配置驱动取数：

```yaml
# ===== 取数配置 =====
api:
  source: tushare
  token: "${TUSHARE_TOKEN}"        # 从环境变量读取，不硬编码

date_range:
  start_date: "20250101"
  end_date: "20260704"

adjust: "none"
frequency: "D"

targets:
  - name: "中芯国际"
    a_code: "688981.SH"
    hk_code: "00981.HK"
    exchange: "上交所科创板"
    industry: "半导体"
    structure: "A+H"

  - name: "比亚迪"
    a_code: "002594.SZ"
    hk_code: "01211.HK"
    exchange: "深交所主板"
    industry: "新能源汽车"
    structure: "A+H"

  - name: "长江电力"
    a_code: "600900.SH"
    hk_code: null
    exchange: "上交所主板"
    industry: "水电/公用事业"
    structure: "A股"

datasets:
  - daily_quotes
  - daily_basic
  - stock_info

output:
  encoding: "utf-8-sig"
  sort: "desc"                      # trade_date 降序
  charts: true
  html_pages: true
```

---

## 10. 变更记录

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| 1.0 | 2026-07-04 | 初始版本，定义三只标的（中芯国际/比亚迪/长江电力）的取数规范 |
