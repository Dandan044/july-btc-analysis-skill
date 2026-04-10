---
name: july-btc-analysis
description: BTC 技术分析技能，提供定时日报、警报触发即时分析、交易周期管理。适合加密货币分析智能体使用。
metadata:
  openclaw:
    requires:
      bins: ["node", "pm2"]
      files: ["skills/btc-market-lite/scripts/api.js"]
---

# july-btc-analysis 技能

比特币技术分析技能，提供完整的分析工作流：
- ⏰ **定时日报**：每天 9:00/21:00 执行分析
- ⚡ **即时分析**：警报触发时立即分析
- 📋 **交易周期管理**：建议生命周期追踪
- 🛠️ **警报器系统**：多维度市场监控

---

## 🚀 快速入门

### 1. 首先阅读的文档

安装此技能后，按以下顺序阅读：

| 文档 | 作用 | 必读 |
|------|------|------|
| `AGENTS.md` | **任务路由** - 根据指令类型选择执行哪个任务 | ⭐⭐⭐ |
| `tasks/daily-report.md` | 日报任务规则 - 定时分析的标准流程 | ⭐⭐⭐ |
| `tasks/instant-analysis.md` | 即时分析规则 - 警报触发时的分析流程 | ⭐⭐⭐ |
| `tasks/set-alert.md` | 警报创建规则 - 如何编写警报规则文件 | ⭐⭐ |
| `tasks/alert-management.md` | 警报管理规则 - 每次分析后的警报维护 | ⭐⭐ |

**核心原则**：收到任务指令时，**先读 AGENTS.md 确定任务类型，再读对应的 tasks/*.md 执行**。

### 2. 文件结构说明

```
<技能目录>/
├── SKILL.md                    ← 本文件（入门指南）
├── AGENTS.md                   ← 任务路由（必读）
├── IDENTITY.md                 ← 智能体身份定义
├── USER.md                     ← 服务对象信息
│
├── tasks/                      ← 任务规则文件（核心）
│   ├── daily-report.md         ← 日报任务执行流程
│   ├── instant-analysis.md     ← 即时分析执行流程
│   ├── set-alert.md            ← 警报创建流程
│   ├── alert-management.md     ← 警报维护流程
│   └── alert-debug.md          ← 警报调试报告流程
│
├── skills/                     ← 子技能
│   ├── btc-alert/              ← 警报器系统
│   │   ├── SKILL.md            ← 警报器技能说明
│   │   ├── engine.js           ← 警报器引擎（PM2运行）
│   │   └── rules/              ← 警报规则文件目录
│   │   └── rules-archive/      ← 已归档规则目录
│   │
│   └── btc-market-lite/        ← 市场数据获取
│   │   ├── SKILL.md            ← 数据源说明
│   │   └── scripts/
│   │       ├── api.js          ← 数据 API 封装
│   │       ├── get_enhanced_analysis.js  ← 日报数据获取
│   │       └── get_instant_data.js       ← 即时数据获取
│
├── active/                     ← 活跃交易周期（运行时生成）
│   └── cycle-YYYYMMDD-XXX/
│       ├── trade-suggestions.json  ← 交易建议文件
│       └── reports/                ← 本周期报告
│
├── archived/                   ← 已归档周期（运行时生成）
├── data/                       ← 原始数据（运行时生成）
└── logs/                       ← 执行日志（运行时生成）
```

### 3. 任务路由规则

收到指令时，根据内容判断执行哪个任务：

| 指令特征 | 任务类型 | 规则文件 |
|---------|---------|---------|
| `[SPAWN_DAILY_REPORT]` 前缀 | 日报任务 | `tasks/daily-report.md` |
| `[SPAWN_INSTANT_ANALYSIS]` 前缀 | 即时分析 | `tasks/instant-analysis.md` |
| "设定市场警报" | 警报创建 | `tasks/set-alert.md` |
| "警报调试报告" | 警报调试 | `tasks/alert-debug.md` |
| 定时触发（无前缀） | 日报任务 | `tasks/daily-report.md` |
| 其他对话 | 正常聊天 | 参考 `AGENTS.md` |

**⚠️ SPAWN 触发机制**：

如果消息以 `[SPAWN_*]` 开头，表示外部触发的任务请求。**必须立即 spawn 新会话执行**：

```
sessions_spawn(
  agentId: "当前智能体ID",
  mode: "run",
  timeoutSeconds: 0,  // 不等待完成
  task: 移除前缀后的完整内容
)
```

### 4. 核心工作流

#### 日报流程（定时触发）

```
定时触发 → SPAWN新会话 → 检查周期 → 获取历史报告
    ↓
获取市场数据 → 分析撰写 → 保存报告
    ↓
管理交易建议 → 归档检查 → 发送飞书
    ↓
记录日志 → 警报器管理
```

#### 即时分析流程（警报触发）

```
警报触发 → SPAWN新会话 → 解析警报数据 → 获取即时数据
    ↓
获取24h历史 → 分析撰写 → 保存报告
    ↓
检查建议触发 → 归档检查 → 发送飞书
    ↓
记录日志 → 警报器管理
```

#### 警报器运行流程

```
PM2启动引擎 → 加载规则 → 启动定时器
    ↓
每N分钟检查 → check()返回true？
    ↓ 是
collect()收集数据 → trigger()创建即时分析任务
    ↓
冷却30分钟 → 继续检查
```

---

## 📦 部署要求

### 必须提前部署

1. **警报器引擎**（PM2）

```bash
pm2 start skills/btc-alert/engine.js --name btc-alert
pm2 save
```

查看状态：`pm2 logs btc-alert`

2. **代理配置**（国内访问 Binance API）

默认代理：`http://127.0.0.1:7890`

在 `skills/btc-market-lite/scripts/get_enhanced_analysis.js` 中配置：
```javascript
const PROXY_DEFAULT = 'http://127.0.0.1:7890';
```

3. **飞书配置**

在 `credentials.json` 中配置：
```json
{
  "feishu": {
    "targetOpenId": "ou_xxx"
  }
}
```

### 可选部署

- **定时任务**：通过 OpenClaw cron 配置日报触发时间
- **热加载**：警报规则无需重启引擎，自动加载新规则

---

## 📊 数据源说明

### 三层 API 架构

| API | 用途 | 需代理 | 脚本 |
|-----|------|--------|------|
| **CryptoCompare** | 警报器规则（价格、K线、交易量） | ❌ | `api.js` |
| **OKX CLI + API** | 日报/即时分析（K线、OI、多空比、Taker比） | ✅ | `get_enhanced_analysis.js`, `get_instant_data.js` |
| **alternative.me** | 恐惧贪婪指数 | ❌ | `get_enhanced_analysis.js` |
| **Deribit** | 期权数据（持仓、Max Pain、Put/Call比） | ✅ | `get_enhanced_analysis.js` |

### 各数据源详情

#### CryptoCompare（警报器专用）
- **用途**：警报器规则 `check()` 和 `collect()` 方法
- **数据**：实时价格、K线（1m/5m/15m/1h/4h/1d）、交易量
- **优点**：无需代理，国内直连
- **API 封装**：`skills/btc-market-lite/scripts/api.js`

#### OKX CLI + API（日报/即时分析）
- **用途**：日报和即时分析的主数据源
- **数据**：
  - K线（日线、4h、1h、15m）
  - 资金费率（8小时）
  - 持仓量(OI)
  - 多空人数比
  - 大户持仓比
  - Taker买卖比
  - 技术指标（EMA、RSI）
- **需要**：代理 + OKX CLI 工具 + `okx-proxy.sh` wrapper

#### alternative.me
- **用途**：恐惧贪婪指数
- **数据**：当前值 + 30日历史
- **优点**：无需代理

#### Deribit
- **用途**：期权市场数据
- **数据**：持仓量、Put/Call比、隐含波动率、Max Pain、关键价位
- **需要**：代理

### 数据获取命令

```bash
# 日报数据（日线+4h+期权+斐波那契）
node skills/btc-market-lite/scripts/get_enhanced_analysis.js --save

# 即时数据（4h+1h+15m+ticker）
node skills/btc-market-lite/scripts/get_instant_data.js --save

# 警报器数据（api.js 方法）
const api = require('./scripts/api');
api.getTicker('BTC');
api.getKlines('BTC', '15m', 5);
```

### 代理配置

国内访问 OKX/Deribit 需要代理：

```bash
# 默认代理
http://127.0.0.1:7890

# 命令行指定
node scripts/get_enhanced_analysis.js --proxy http://127.0.0.1:7890

# 或配置 okx-proxy.sh wrapper
```

---

## 📋 交易周期系统

### 核心概念

**交易周期（Cycle）** 是管理交易建议的单位：
- 一个周期 = 从上一篇报告结束 → 所有建议关闭
- 每个周期独立运行，不参考历史归档数据

### 建议状态

| 状态 | 含义 | 说明 |
|------|------|------|
| `pending_entry` | 等待入场 | 需要警报监控入场条件 |
| `open` | 持仓中 | 需要监控止盈止损 |
| `closed` | 已平仓 | 交易结束 |

### 关键规则

- ⛔ **禁止读取 `archived/`**
- 每次报告前检查 `active/cycle-*` 是否存在
- 所有建议关闭后自动归档周期

---

## 🛠️ 警报规则编写

### 规则文件位置

`skills/btc-alert/rules/YYYY-MM-DD-<类型>-<目标>.js`

### 规则结构

```javascript
module.exports = {
  name: '规则名称',
  interval: 3 * 60 * 1000,  // 检查间隔
  lastTriggered: 0,         // 冷却状态
  
  async check() {    // 检测条件，返回 boolean
  async collect() {  // 收集数据
  async trigger(data) { // 触发动作（spawn 即时分析）
  lifetime() {       // 返回 'active'/'expired'/'completed'
};
```

### 警报类型

| 类型 | 用途 | 示例 |
|------|------|------|
| 价格警报 | 支撑/阻力位监控 | 突破 75000 美元 |
| 定时器警报 | 纯时间触发 | 4小时后检查入场 |
| 延迟触发警报 | 突破后等待确认 | 突破后30分钟确认 |
| 交易量异动 | 大资金进出 | 交易量 > 均值×2 |
| 情绪极端 | 市场恐慌/贪婪 | FGI < 10 或 > 80 |

**核心原则**：单方向单一警报（向上最多1个，向下最多1个）

---

## ⚠️ 关键注意事项

1. **SPAWN 机制**：日报和即时分析必须在**新会话**执行，避免上下文干扰
2. **冷却机制**：警报触发后 30 分钟内不重复触发
3. **错误监控**：连续失败 5 次 → 通知；连续失败 10 次 → 暂停
4. **归档检查**：归档前必须核对止盈止损是否真正触发（最低价≤止盈价，最高价≥止损价）
5. **日志记录**：无论成功失败，都必须记录日志

---

## 📖 完整文档索引

| 文档 | 内容 |
|------|------|
| `AGENTS.md` | 任务路由、文件结构、周期系统 |
| `tasks/daily-report.md` | 日报任务详细流程（报告结构、建议管理） |
| `tasks/instant-analysis.md` | 即时分析详细流程 |
| `tasks/set-alert.md` | 警报创建规范、规则示例 |
| `tasks/alert-management.md` | 警报维护流程 |
| `skills/btc-alert/SKILL.md` | 警报器技能说明 |
| `skills/btc-market-lite/SKILL.md` | 数据源、API说明 |
| `skills/btc-alert/engine.js` | 警报器引擎源码 |

---

📈 七月 BTC 分析技能 v1.0