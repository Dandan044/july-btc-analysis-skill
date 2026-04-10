# deployment.md - 部署配置指南

此技能在新环境中部署时，需要配置以下服务和定时任务。

---

## PM2 进程配置

### btc-alert - 警报器引擎

已在 `ecosystem.config.js` 中定义，启动方式：

```bash
cd <技能目录>
pm2 start ecosystem.config.js
pm2 save
```

**进程配置说明**：
| 参数 | 值 | 说明 |
|------|-----|------|
| script | `./skills/btc-alert/engine.js` | 警报器主引擎 |
| autorestart | true | 自动重启 |
| max_memory_restart | 500M | 内存超限重启 |
| TZ | Asia/Shanghai | 时区 |
| http_proxy | http://127.0.0.1:7890 | 代理（国内必需） |

---

## 定时任务配置

### 日报任务（Cron Jobs）

使用 OpenClaw cron 系统配置日报任务：

**早间日报（9:00 GMT+8）**：
```json
{
  "name": "btc-morning-report",
  "schedule": {
    "kind": "cron",
    "expr": "0 9 * * *",
    "tz": "Asia/Shanghai"
  },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "[SPAWN_DAILY_REPORT]执行比特币技术分析日报：获取BTC价格、恐惧指数、技术指标数据，进行技术分析，生成报告并发送",
    "thinking": "high",
    "timeoutSeconds": 0
  },
  "delivery": {
    "mode": "none"
  }
}
```

**晚间日报（21:00 GMT+8）**：
```json
{
  "name": "btc-evening-report",
  "schedule": {
    "kind": "cron",
    "expr": "0 21 * * *",
    "tz": "Asia/Shanghai"
  },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "[SPAWN_DAILY_REPORT]执行比特币技术分析日报：获取BTC价格、恐惧指数、技术指标数据，进行技术分析，生成报告并发送",
    "thinking": "high",
    "timeoutSeconds": 0
  },
  "delivery": {
    "mode": "none"
  }
}
```

**创建定时任务**：
```bash
# 使用 cron 工具添加任务
cron action=add job=<上述JSON配置>
```

---

## 通知渠道配置

技能的通知功能输出到日志。使用者需根据实际需求配置通知渠道：

### 飞书通知
配置 `delivery.channel` = `feishu`，设置 `delivery.to` 为目标用户 Open ID。

### QQ 通知
配置 `delivery.channel` = `qqbot`，设置 `delivery.to` 为目标用户 QQ ID。

---

## 数据源配置

### OKX API
需要配置 OKX CLI：
```bash
npm install -g @okx_ai/okx-trade-cli
```

配置文件路径：`~/.okx/config.toml`

### 代理配置
国内环境需要配置代理，修改 `ecosystem.config.js` 中的环境变量：
```javascript
env: {
  http_proxy: 'http://<代理地址>:<端口>',
  https_proxy: 'http://<代理地址>:<端口>',
  all_proxy: 'socks5://<代理地址>:<端口>'
}
```

---

## 完整部署步骤

1. **安装依赖**
   ```bash
   cd <技能目录>
   npm install
   ```

2. **配置数据源**
   - 安装 OKX CLI
   - 配置 API 凭证

3. **配置代理**（如需要）
   - 编辑 `ecosystem.config.js`

4. **启动警报器**
   ```bash
   pm2 start ecosystem.config.js
   pm2 save
   ```

5. **配置定时任务**
   - 使用 cron 工具添加日报任务

6. **配置通知渠道**
   - 根据需求设置 delivery 参数