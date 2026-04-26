# deployment.md - 部署配置指南

此技能在智能体中安装使用时，需要配置以下服务。

---

## 前置条件

已安装 OpenClaw 框架，具备以下环境：
- Node.js >= 18
- PM2 进程管理器
- Python 环境
- proxychains4（系统自带或已安装）

---

## 一、安装技能

将技能目录放置到智能体的 skills 目录下：

```bash
# 目标路径示例
<智能体工作区>/skills/july-btc-analysis/
```

---

## 二、代理配置

### 前置：代理服务准备

国内网络访问 OKX API 需要代理服务。请自行准备代理方案：

| 方案 | 说明 |
|------|------|
| 机场订阅 | 购买机场服务，获取订阅链接或配置文件 |
| 自建代理 | 自建 VPS + 代理服务（如 V2Ray、Shadowsocks） |
| 其他方案 | 企业代理、云服务商代理等 |

**确保以下几点：**
- 本地代理程序已运行（如 mihomo/Clash、V2Ray 等）
- 本地代理端口已开启（常见端口：`7890`、`1080` 等）
- 代理可访问 `https://www.okx.com`（测试：`curl --proxy http://127.0.0.1:<端口> https://www.okx.com`）

**⚠️ 注意**：代理端口需配置到以下位置：
- Shell 环境变量（`~/.bashrc`）
- PM2 环境变量（`ecosystem.config.js`）
- OKX CLI 配置（`~/.okx/config.toml`）

---

### 代理环境变量配置

此技能采用**环境变量统一配置**方式：

| 环境变量 | 说明 | 示例 |
|---------|------|------|
| `http_proxy` | HTTP 代理地址 | `http://127.0.0.1:7890` |
| `https_proxy` | HTTPS 代理地址 | `http://127.0.0.1:7890` |
| `HTTP_PROXY_PORT` | 代理端口（okx-proxy.sh 使用） | `7890` |

### 配置方式

**方式一：Shell 环境变量（临时）**
```bash
export http_proxy="http://127.0.0.1:<端口>"
export https_proxy="http://127.0.0.1:<端口>"
export HTTP_PROXY_PORT="<端口>"
```

**方式二：写入 ~/.bashrc（永久）**
```bash
echo 'export http_proxy="http://127.0.0.1:<端口>"' >> ~/.bashrc
echo 'export https_proxy="http://127.0.0.1:<端口>"' >> ~/.bashrc
echo 'export HTTP_PROXY_PORT="<端口>"' >> ~/.bashrc
source ~/.bashrc
```

**方式三：PM2 ecosystem.config.js（警报器专用）**

见下一章节。

---

## 三、PM2 进程配置

### btc-alert - 警报器引擎

警报器引擎通过 PM2 运行，负责监控市场并在触发条件时创建即时分析任务。

#### ecosystem.config.js

在技能目录下创建或编辑 `ecosystem.config.js`：

```javascript
module.exports = {
  apps: [{
    name: 'btc-alert',
    script: './skills/july-btc-analysis/skills/btc-alert/engine.js',
    cwd: '<智能体工作区>',  // ⚠️ 替换为智能体工作区绝对路径
    
    autorestart: true,
    watch: false,
    max_restarts: 10,
    restart_delay: 3000,
    max_memory_restart: '500M',
    
    // 日志
    error_file: './logs/btc-alert.log',
    out_file: './logs/btc-alert.log',
    merge_logs: true,
    time: true,
    
    // 环境（代理配置）
    env: {
      NODE_ENV: 'production',
      TZ: 'Asia/Shanghai',
      http_proxy: 'http://127.0.0.1:<代理端口>',   // ⚠️ 替换
      https_proxy: 'http://127.0.0.1:<代理端口>',
      HTTP_PROXY_PORT: '<代理端口>'
    }
  }]
};
```

**⚠️ 必填项**：
| 字段 | 说明 |
|------|------|
| `cwd` | 智能体工作区的绝对路径 |
| `script` | 警报引擎路径，根据技能安装位置调整 |
| `http_proxy` 端口 | 代理端口，与系统代理一致 |

#### 启动命令

```bash
cd <智能体工作区>
npm install
pm2 start ecosystem.config.js
pm2 save
```

**验证运行**：
```bash
pm2 list
pm2 logs btc-alert
```

---

## 四、OKX CLI 配置

### 安装 OKX CLI

```bash
npm install -g @okx_ai/okx-trade-cli
okx --version
```

### 配置 API 凭证

创建 `~/.okx/config.toml`：

```toml
# OKX API Configuration

# 代理配置（国内网络必需）
proxy = "http://127.0.0.1:<代理端口>"

# 实盘配置
[profiles.live]
api_key = "<你的API Key>"
secret_key = "<你的Secret Key>"
passphrase = "<你的Passphrase>"
demo = false

# 模拟盘配置（可选）
[profiles.demo]
api_key = "<模拟盘API Key>"
secret_key = "<模拟盘Secret Key>"
passphrase = "<模拟盘Passphrase>"
demo = true
```

**⚠️ 安全提醒**：
- 此文件包含敏感凭证，**切勿提交到 Git**

### 验证 OKX CLI

```bash
# 使用代理 wrapper（国内网络）
./skills/july-btc-analysis/scripts/okx-proxy.sh --profile live account balance

# 或直接使用（国外网络）
okx --profile live account balance
```

---

## 五、定时任务配置

### 日报任务（Cron Jobs）

使用 OpenClaw cron 系统配置日报任务触发时间。

#### 早间日报（9:00 GMT+8）

```json
{
  "name": "<智能体ID>-btc-morning",
  "schedule": {
    "kind": "cron",
    "expr": "0 9 * * *",
    "tz": "Asia/Shanghai"
  },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "agentId": "<智能体ID>",
    "message": "[SPAWN_DAILY_REPORT]执行比特币技术分析日报：获取BTC价格、恐惧指数、技术指标数据，进行技术分析，生成报告并发送",
    "thinking": "high",
    "timeoutSeconds": 0
  },
  "delivery": {
    "mode": "none"
  }
}
```

#### 晚间日报（21:00 GMT+8）

```json
{
  "name": "<智能体ID>-btc-evening",
  "schedule": {
    "kind": "cron",
    "expr": "0 21 * * *",
    "tz": "Asia/Shanghai"
  },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "agentId": "<智能体ID>",
    "message": "[SPAWN_DAILY_REPORT]执行比特币技术分析日报：获取BTC价格、恐惧指数、技术指标数据，进行技术分析，生成报告并发送",
    "thinking": "high",
    "timeoutSeconds": 0
  },
  "delivery": {
    "mode": "none"
  }
}
```

**⚠️ 必填项**：
| 字段 | 说明 |
|------|------|
| `agentId` | 安装此技能的智能体 ID |
| `name` | 任务名称，建议格式 `<智能体ID>-btc-morning/evening` |

---

## 六、通知渠道配置（可选）

如需发送报告到飞书，可配置飞书机器人凭证。

### credentials.json

在智能体工作区创建 `.openclaw/credentials.json`：

```json
{
  "feishu": {
    "appId": "<飞书应用App ID>",
    "accountId": "<账号标识>",
    "targetOpenId": "<目标用户的Open ID>",
    "targetName": "<目标用户名称>"
  }
}
```

**⚠️ 安全提醒**：
- 此文件包含敏感凭证，**切勿提交到 Git**

---

## 七、完整部署步骤

| 步骤 | 操作 | 验证命令 |
|------|------|---------|
| 1 | 安装技能 | 检查 `skills/july-btc-analysis/` 目录存在 |
| 2 | 配置代理环境变量 | `echo $http_proxy` |
| 3 | 编辑 ecosystem.config.js | 填写 cwd、script 路径、代理端口 |
| 4 | 安装依赖 | `npm install` |
| 5 | 安装 OKX CLI | `okx --version` |
| 6 | 配置 OKX API | 创建 `~/.okx/config.toml` |
| 7 | 启动警报器 | `pm2 list` 显示 btc-alert online |
| 8 | 创建定时任务 | `openclaw cron list` 显示两个任务 |
| 9 | 配置飞书通知（可选） | 创建 `credentials.json` |

### 验证检查清单

| 检查项 | 命令 | 预期结果 |
|--------|------|---------|
| PM2 运行 | `pm2 list` | `btc-alert` 状态 online |
| 警报器日志 | `pm2 logs btc-alert --lines 20` | 有心跳日志输出 |
| Cron 任务 | `openclaw cron list` | 两个日报任务已注册 |
| OKX CLI | `okx --version` | 显示版本号 |
| 代理可用 | `curl --proxy $http_proxy https://www.okx.com` | 返回正常 |

---

## 八、常见问题

### Q: PM2 启动失败 "script not found"

检查：
- `cwd` 是否为智能体工作区绝对路径
- `script` 路径是否正确指向 `engine.js`

### Q: 警报器无法获取数据（ETIMEDOUT）

检查代理配置：
- 确认代理服务运行中
- 确认 `http_proxy` 环境变量端口正确
- 测试：`curl --proxy http://127.0.0.1:<端口> https://www.okx.com`

### Q: Cron 任务不触发

检查：
- `agentId` 是否与实际智能体 ID 一致
- 智能体是否正常运行

### Q: OKX CLI 调用失败

检查：
- `~/.okx/config.toml` 是否存在
- API 凭证是否正确
- 代理是否可用（国内网络）

