# TOOLS.md - 技能工具笔记

## OKX 交易 API

此技能已接入 OKX API，可获取市场数据和执行交易。

### CLI 安装
```bash
npm install -g @okx_ai/okx-trade-cli
okx --version  # 1.3.0
```

### 配置文件
路径：`~/.okx/config.toml`

### 代理访问（重要！）
国内网络需要通过代理访问 OKX API。使用 wrapper 脚本：

```bash
# 从技能目录执行
./scripts/okx-proxy.sh --profile live account balance

# 或指定完整路径
<技能目录>/scripts/okx-proxy.sh --profile live account balance
```

### 常用命令

```bash
# 查看余额
okx-proxy.sh --profile live account balance USDT

# 查看持仓
okx-proxy.sh --profile live account positions

# 查看BTC价格
okx-proxy.sh market ticker BTC-USDT
```

### 技能文档
完整命令参考见：
- `~/.agents/skills/okx-cex-trade/SKILL.md` - 交易命令
- `~/.agents/skills/okx-cex-market/SKILL.md` - 市场数据
- `~/.agents/skills/okx-cex-portfolio/SKILL.md` - 账户余额/持仓

---

## PM2 进程管理 - 警报器引擎

服务名称：`btc-alert`

### 查看服务状态
```bash
pm2 list                    # 查看所有进程状态
pm2 logs btc-alert          # 查看实时日志
pm2 info btc-alert          # 查看详细信息
```

### 服务管理
```bash
pm2 start ecosystem.config.js   # 启动服务
pm2 stop btc-alert              # 停止服务
pm2 restart btc-alert           # 重启服务
pm2 delete btc-alert            # 删除服务
```

### 日志位置
- `logs/btc-alert-out.log` - 标准输出
- `logs/btc-alert-error.log` - 错误日志

---

## 报告发送方式

发送报告时，使用者需自行配置通知渠道（飞书、QQ 等）。

### 飞书文档方式

使用 `feishu_doc` 工具创建文档并发送链接。

### 分段消息方式

若飞书文档失败，使用 `message` 工具分段发送。