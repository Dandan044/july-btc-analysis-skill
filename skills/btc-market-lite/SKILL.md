---
name: BTC Market Lite
description: 比特币市场数据获取工具。整合 OKX CLI、CryptoCompare、alternative.me、Deribit 多数据源，获取价格、OHLCV、资金费率、OI、多空比、Taker比、恐惧贪婪指数、期权数据。适合日报和即时分析。
metadata:
  openclaw:
    requires:
      bins: ["node"]
      files: ["scripts/api.js", "scripts/get_enhanced_analysis.js", "scripts/get_instant_data.js"]
---

# BTC Market Lite 技能

比特币市场数据获取工具，整合多数据源，支持日线、4小时、1小时、15分钟等多种粒度。

## 数据源架构

### 三层 API 架构

| API | 用途 | 需代理 | 脚本 |
|-----|------|--------|------|
| **CryptoCompare** | 警报器规则（价格、K线、交易量） | ❌ | `api.js` |
| **OKX CLI + API** | 日报/即时分析（K线、OI、多空比等） | ✅ | `get_enhanced_analysis.js`, `get_instant_data.js` |
| **alternative.me** | 恐惧贪婪指数 | ❌ | `get_enhanced_analysis.js` |
| **Deribit** | 期权数据 | ✅ | `get_enhanced_analysis.js` |

## 用法

### 日报数据获取

```bash
# 获取完整日报数据（默认使用代理）
node scripts/get_enhanced_analysis.js

# JSON 输出
node scripts/get_enhanced_analysis.js --json

# 保存到 data/YYYY-MM-DD.json
node scripts/get_enhanced_analysis.js --save

# 指定代理
node scripts/get_enhanced_analysis.js --proxy http://127.0.0.1:7890
```

### 即时分析数据获取

```bash
# 获取即时数据（4h+1h+15m+ticker）
node scripts/get_instant_data.js

# JSON 输出
node scripts/get_instant_data.js --json

# 保存到 data/instant-YYYY-MM-DD-HH-MM-SS.json
node scripts/get_instant_data.js --save
```

### 警报器 API（无需代理）

```javascript
const api = require('./scripts/api');

// 获取实时价格
const ticker = await api.getTicker('BTC');

// 获取K线
const klines = await api.getKlines('BTC', '15m', 5);

// 获取24小时交易量
const volume = await api.get24hVolume('BTC');

// 获取历史价格
const history = await api.getPriceHistory('BTC', 30);

// 获取恐惧贪婪指数
const fgi = await api.getFearGreedIndex(30);

// 通用 HTTP 请求
const data = await api.fetch('https://api.example.com/data');
```

## 数据结构

### 日报数据（`get_enhanced_analysis.js`）

| 数据类型 | 来源 | 粒度 | 说明 |
|---------|------|------|------|
| 价格历史 | OKX CLI | 日线 14 根 | OHLCV + EMA(7/12/20/26) |
| 技术指标 | OKX CLI | 日线 | RSI(14) |
| 资金费率 | OKX CLI | 8 小时 | 30 条历史 |
| 持仓量(OI) | OKX API | 日线 | 28 天历史 |
| 多空人数比 | OKX API | 日线 | 28 天历史 |
| 大户持仓比 | OKX API | 日线 | 28 天历史 |
| Taker买卖比 | OKX API | 日线 | 28 天历史 |
| 4小时K线 | OKX CLI | 4h 14 根 | 含交易侧数据 |
| 恐惧贪婪指数 | alternative.me | 日线 | 当前值 + 30 日统计 |
| 期权数据 | Deribit | - | 2 个主力到期日 |
| 斐波那契 | OKX CLI | 日线/4h/周线 | 多时间框架回调位 |

### 即时数据（`get_instant_data.js`）

| 数据类型 | 来源 | 粒度 | 说明 |
|---------|------|------|------|
| Ticker | OKX CLI | 实时 | 价格、24h涨跌、交易量 |
| 4小时K线 | OKX CLI | 4h 12 根 | 含交易侧数据 |
| 1小时K线 | OKX CLI | 1h 4 根 | 含交易侧数据 |
| 15分钟K线 | OKX CLI | 15m 8 根 | 含交易侧数据 |

### 警报器 API 数据（`api.js`）

| 方法 | 数据 | 说明 |
|------|------|------|
| `getKlines(symbol, interval, limit)` | K线 | 支持 1m/5m/15m/1h/4h/1d |
| `getTicker(symbol)` | 实时价格 | 价格、1h/24h/7d涨跌 |
| `get24hVolume(symbol)` | 交易量 | 小时级数据 |
| `getPriceHistory(symbol, days)` | 日线历史 | 指定天数 |
| `getFearGreedIndex(days)` | FGI | 历史数据 |
| `fetch(url)` | 任意 HTTP | 通用请求 |

## 指标解读

### 资金费率

| 费率范围 | 信号 | 含义 |
|----------|------|------|
| > 0.05% | 🔥 多头过热 | 做多拥挤，警惕回调 |
| 0.01% ~ 0.05% | 看多 | 多头主导 |
| -0.05% ~ 0.01% | 中性 | 多空平衡 |
| -0.05% ~ -0.01% | 看空 | 空头主导 |
| < -0.05% | ❄️ 空头过热 | 做空拥挤，警惕反弹 |

### 多空人数比

| 比值 | 信号 | 含义 |
|------|------|------|
| > 2.5 | 过度看多 | 散户FOMO，警惕反转 |
| 1.5 ~ 2.5 | 看多 | 多头情绪主导 |
| 0.8 ~ 1.5 | 中性 | 多空分歧 |
| < 0.8 | 过度看空 | 散户恐慌，可能反转 |

### Taker买卖比

| 比值 | 信号 |
|------|------|
| > 1.5 | 主动买入强势 |
| 0.7 ~ 1.5 | 相对平衡 |
| < 0.7 | 主动卖出强势 |

### Put/Call比（期权）

| 比值 | 信号 |
|------|------|
| > 1.0 | 看跌情绪占优 |
| < 1.0 | 看涨情绪占优 |

## 代理配置

国内访问 OKX/Deribit 需要代理。

**默认代理**: `http://127.0.0.1:7890`

**配置方法**:
```bash
# 方式1：命令行参数
node scripts/get_enhanced_analysis.js --proxy http://127.0.0.1:7890

# 方式2：修改脚本中的 PROXY_DEFAULT 常量

# 方式3：使用 okx-proxy.sh wrapper（推荐）
# 配置 scripts/okx-proxy.sh 脚本
```

**OKX CLI 依赖**:
- 需要安装 `okx` CLI 工具
- 需要配置 `okx-proxy.sh` wrapper（用于代理）

## 注意事项

- **CryptoCompare API** 有速率限制（约 100,000 次/月），但警报器使用频率较低，无需担心
- **OKX CLI** 需要代理访问
- **Deribit** 需要代理访问
- **alternative.me** 无需代理
- 无代理时，警报器仍可使用 `api.js` 获取 CryptoCompare 数据

## 文件说明

| 文件 | 用途 |
|------|------|
| `api.js` | 警报器专用 API（CryptoCompare） |
| `get_enhanced_analysis.js` | 日报数据获取（OKX + alternative.me + Deribit） |
| `get_instant_data.js` | 即时分析数据获取（OKX） |

## 更新日志

- 2026-04-10: **v5 重构** - 数据源从 Binance 切换到 OKX，整合 OKX CLI 工具，新增期权数据、斐波那契分析
- 2026-03-27: **v4 重构** - 整合 Binance 交易数据
- 2026-03-03: **v3 重构** - 简化输出，数据源切换为 CryptoCompare
- 2026-02-27: 初始版本