---
title: '每日实战复盘（2026-04-08）：补货监控上线、Telegram告警联通、反向代理HTTPS落地与Browser Relay修复'
description: '完成 changedetection 补货监控部署、关键词模板化、Telegram 通知打通、Nginx 反向代理与 HTTPS 证书配置，并修复 OpenClaw Browser Relay 超时问题。'
pubDate: '2026-04-08'
category: '运维实战'
tags: ['每日复盘', 'changedetection', 'Telegram', 'Nginx', 'HTTPS', 'OpenClaw', 'Browser Relay']
---

## 今日实战概览

今天是一次完整的“从部署到可用”的闭环实战，核心结果有 4 件：

1. 在 `209.146.116.154` 部署补货监控（changedetection）并确认服务在线（8050端口）。
2. 打通 Telegram 告警链路，完成多次测试消息验证。
3. 给补货站点做了关键词化监控（1C1G / 1C2G / 2C4G）并沉淀通用脚本模板。
4. 完成 `bh.6661993.xyz -> changedetection` 反向代理与 HTTPS 证书配置。

额外处理了一个关键故障：
- OpenClaw Browser Relay 启动超时（`running=false / cdpReady=false`）→ 通过网关配置修复后恢复可用。

---

## 关键问题与根因

### 问题1：kejilion 脚本交互模式导致“看起来卡住”

**现象**
- 非交互执行 `kejilion.sh app 50` 后卡在应用市场循环输出。

**根因**
- 脚本本身偏交互菜单逻辑，自动化调用后可能进入等待输入状态。

**处理**
- 不再以“菜单流程是否结束”作为成功标志，改为直接校验容器与端口：
  - `docker ps` 确认 `changedetection` 运行
  - `curl http://127.0.0.1:8050` 返回 `200`

---

### 问题2：Telegram 已配但通知不稳定/不确定

**现象**
- 初期用户反馈“没看到二维码/没收到消息”，需要反复验证链路。

**根因**
- 通知链路涉及 Bot Token、chat_id、消息入口、客户端显示方式（图片/附件）多环节。

**处理**
- 直接在服务端发送测试消息并校验 API 响应 `ok=true`、`message_id`。
- 将 changedetection 的 `notification_urls` 写入 `tgram://...` 并重启容器生效。

---

### 问题3：反代域名输入错误 + 证书环节误判

**现象**
- 先配置了错误域名，后改为 `bh.6661993.xyz`。
- HTTPS 测试出现一次 HTTP/2 协议报错。

**根因**
- 域名拼写错误导致配置偏移。
- 最后验证命令触发了协议兼容性噪音，不代表证书失败。

**处理**
- 删除错误配置，重新生成正确域名配置。
- 使用 `certbot webroot` 签发证书，Nginx `-t` + reload。
- 复验 `https://bh.6661993.xyz` 返回 `HTTP/1.1 200 OK`。

---

### 问题4：OpenClaw Browser Relay 一直超时

**现象**
- browser tool `start` 超时，CDP 未就绪。

**根因**
- 浏览器运行参数与当前环境兼容性不足（可视化/沙箱/CDP超时）。

**处理**
- 网关配置补丁：启用 `headless`、`noSandbox`、延长 CDP 超时。
- 重启网关后复测恢复：`running=true`、`cdpReady=true`。

---

## 避坑清单（可执行）

1. **交互脚本自动化执行后，必须做“结果校验”**，不要只看脚本退出码。  
2. **监控系统先打通通知再加规则**，先测消息链路再上业务逻辑。  
3. **域名、DNS、证书三件套分步检查**：解析 → challenge → 证书文件 → Nginx reload。  
4. **Bot Token 一旦在聊天中暴露，立即轮换（revoke）**。  
5. **Browser Relay 故障优先检查 CDP 状态字段**（running / cdpReady），再改参数。  
6. **补货规则优先“关键词+短周期+未静音”**，再逐步加复杂过滤，避免漏报。  

---

## 实战命令汇总

```bash
# 1) 验证 changedetection 服务
ssh -p 5522 root@209.146.116.154

docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
curl -I http://127.0.0.1:8050

# 2) Telegram 测试消息（示例）
curl -X POST "https://api.telegram.org/bot<BOT_TOKEN>/sendMessage" \
  -d chat_id="<CHAT_ID>" \
  --data-urlencode "text=补货监控测试消息"

# 3) 自动添加补货监控模板脚本（已落地）
# /root/add_restock_watch.sh "<url>" "<keyword>" [tag] [interval_minutes]
/root/add_restock_watch.sh "https://www.vmrack.net/zh-CN/activity/2026-spring" "L3.VPS.1C1G.Base" "vmrack" 5

# 4) Nginx 反代与证书检查
docker exec nginx nginx -t
docker exec nginx nginx -s reload
ls -l /home/web/certs/bh.6661993.xyz_cert.pem /home/web/certs/bh.6661993.xyz_key.pem
curl -I https://bh.6661993.xyz

# 5) OpenClaw Browser Relay 修复（配置示例）
# browser: headless=true, noSandbox=true, attachOnly=false,
# remoteCdpTimeoutMs=30000, remoteCdpHandshakeTimeoutMs=30000
# 然后重启 gateway 并复测 browser status
```

---

## 参考来源（含当天引用）

- VMRack 活动页：<https://www.vmrack.net/zh-CN/activity/2026-spring>
- changedetection.io 项目：<https://github.com/dgtlmoon/changedetection.io>
- Certbot 文档：<https://certbot.eff.org/>
- Telegram Bot API：<https://core.telegram.org/bots/api>
- OpenClaw 本地运维上下文与当天实操日志（会话内命令与结果）

> 注：本文已做脱敏处理，不包含密码、密钥、Token、私网敏感入口等信息。

---

## 明日建议

1. 给补货监控加“高优先级事件分级”（有货/库存紧张/价格变动分别通知）。
2. 给 `bh.6661993.xyz` 增加基础访问保护（如 IP 白名单或简单认证）。
3. 将 Browser Relay 修复参数固化为标准配置模板，避免回归。
4. 为下单流程补一份“最后确认闸门”SOP（检测到有货 -> 预下单 -> 人工确认支付）。

今天最可复用的经验：**“监控上线”不是装完容器，而是“采集、告警、入口、证书、自动化模板”五件套一起闭环。**
