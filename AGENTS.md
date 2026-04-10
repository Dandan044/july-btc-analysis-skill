# AGENTS.md - 七月 BTC 分析技能

BTC 技术分析技能，提供专业的比特币交易分析能力。

**核心原则：一切判断必须基于数据，拒绝任何猜测或主观臆断。**

---

## 技能说明

提供以下核心能力：
- 警报器引擎 (btc-alert)
- 市场数据获取 (btc-market-lite)
- 任务定义文件
- 辅助脚本

通过读取任务规则文件来执行相应分析任务。


---

## 文件结构

```
<技能目录>/                    # 技能根目录
├── active/                      # 活跃交易周期（最多1个）
│   └── cycle-YYYYMMDD-XXX/      # 当前周期文件夹
│       ├── trade-suggestions.json  # 交易建议文件
│       └── reports/             # 本周期报告
│           ├── btc-report-YYYY-MM-DD-HHMM.md
│           └── instant-report-YYYY-MM-DD-HHMM.md
│
├── archived/                    # 已归档周期
│   └── cycle-YYYYMMDD-XXX/      # 历史周期（结构同 active）
│
├── data/                        # 原始 JSON 数据（当天覆盖）
├── logs/                        # 执行日志（追加）
└── tasks/                       # 任务规则文件
```

---

## 交易周期系统

### 核心概念

**交易周期（Cycle）** 是七月管理交易建议的核心单位。一个周期从上一篇报告结束开始，到所有交易建议关闭为止。

### 周期生命周期

```
[上一周期结束]
      │
      ▼
下一篇报告生成 → 开启新周期（创建空建议文件）
      │
      ▼
周期进行中 → 报告保存到 active/cycle-xxx/reports/
          → 可能给出交易建议 → 写入 trade-suggestions.json
          → 检查价格触发止盈/止损 → 更新建议状态
      │
      ▼
所有建议关闭 → 归档（移动 active/ → archived/）
      │
      ▼
[下一周期在下一篇报告时开启]
```

### 交易建议文件结构 (`trade-suggestions.json`)

```json
{
  "cycle_id": "cycle-20260319-001",
  "status": "active",
  "started_at": "2026-03-19T09:00:00+08:00",
  "closed_at": null,
  "closed_reason": null,
  
  "suggestions": [
    {
      "id": "sug-001",
      "created_at": "2026-03-19T09:00:00+08:00",
      "triggered_by": "report-2026-03-19-morning",
      "direction": "long",
      "entry_zone": [69500, 70000],
      "stop_loss": 68000,
      "take_profit": [72000, 74000],
      "position_size": "仓位%",
      "status": "open",
      "closed_at": null,
      "close_reason": null,
      "notes": "突破阻力位后的回踩确认"
    }
  ],
  
  "summary": {
    "total": 1,
    "open": 1,
    "closed": 0
  }
}
```

### 周期管理规则

| 场景 | 操作 |
|------|------|
| `active/` 为空 | 下一篇报告开启新周期 |
| `active/` 有周期，建议文件为空 | 观望期，报告正常保存 |
| `active/` 有周期，有建议 | 持仓期，监控止盈止损 |
| 所有建议关闭 | 归档周期（移动到 `archived/`） |

### 归档规则

归档周期时，必须在 `trade-suggestions.json` 中记录以下信息：

1. **周期级别字段**：
   - `status`: 更新为 `"archived"`
   - `closed_at`: 记录归档时间（ISO 8601格式）
   - `closed_reason`: 归档原因（如 "所有建议已关闭"、"手动归档" 等）

2. **建议级别字段**（如果尚未关闭）：
   - `status`: 更新为 `"closed"`
   - `closed_at`: 记录关闭时间
   - `close_reason`: 关闭原因（如 "周期归档"）

3. **归档时机**：
   - 所有建议状态为 `closed` 或 `partial_closed` 且无剩余持仓
   - 或用户手动要求归档

### 读取当前周期状态

在每次报告生成前，检查周期状态：

```bash
# 检查是否有活跃周期
ls -d active/cycle-* 2>/dev/null

# 如果有，读取交易建议文件
cat active/cycle-*/trade-suggestions.json
```

### 不读取历史周期

**重要**：七月在进行报告分析时，**不参考 `archived/` 下的历史周期数据**。每个周期独立运行，不受上一轮交易影响。

---

## 任务路由

当收到任务指令时，**首先判断消息类型，再读取对应的任务规则文件**：

### 消息类型判断

| 消息特征 | 任务类型 | 执行方式 |
|---------|---------|---------|
| `[SPAWN_DAILY_REPORT]` 前缀 | 日报任务 | **必须 SPAWN 新会话** |
| `[SPAWN_INSTANT_ANALYSIS]` 前缀 | 即时分析 | **必须 SPAWN 新会话** |
| "设定市场警报" 关键词 | 警报创建 | 当前会话执行 |
| "警报调试报告" 关键词 | 警报调试 | 当前会话执行 |
| 定时触发（无前缀） | 日报任务 | 当前会话执行 |
| 其他对话 | 正常聊天 | 参考历史报告 + 数据 API |

### ⚠️ SPAWN 机制（必须遵守）

**如果消息以 `[SPAWN_*]` 开头，必须立即 SPAWN 新会话执行！**

原因：日报和即时分析任务执行时间较长，在当前会话执行会：
- 阻塞其他任务
- 上下文累积干扰后续判断

执行方式：
```
sessions_spawn(
  agentId: "当前智能体ID",
  mode: "run",
  timeoutSeconds: 0,  // 不等待完成，立即返回
  task: 移除前缀后的完整内容
)
```

执行后立即返回确认消息，**不要等待子会话完成**。

### 任务规则文件

| 任务类型 | 规则文件 | 必须步骤 |
|---------|---------|---------|
| 日报任务 | `tasks/daily-report.md` | 检查周期 → 获取数据 → 分析 → 保存 → 发送 → 警报管理 |
| 即时分析 | `tasks/instant-analysis.md` | 解析警报 → 获取即时数据 → 分析 → 检查触发 → 发送 |
| 警报创建 | `tasks/set-alert.md` | 理解需求 → 编写规则 → 创建文件 → 记录日志 |
| 警报调试 | `tasks/alert-debug.md` | 解析数据 → 生成报告 → 保存 → 发送 |
| 警报管理 | `tasks/alert-management.md` | 查看规则 → 评估 → 归档过期 → 创建新规则 |

### 触发来源

- **日报任务**：定时触发（9:00/21:00 GMT+8）或 `[SPAWN_DAILY_REPORT]`
- **即时分析**：警报器触发 → `[SPAWN_INSTANT_ANALYSIS]`
- **警报创建**：用户请求或分析结论需要新警报
- **警报调试**：手动调试请求
- **警报管理**：日报/即时分析完成后自动执行

---

📈 七月