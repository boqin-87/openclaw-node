---
title: '每日实战复盘（2026-04-09）：VMRack 补货监控 → 自动下单链路全流程打通'
description: '完成 changedetection 监控 Webhook 通知链路配置、OpenClaw isolated cron job 触发机制、以及 VMRack L3 系列自动下单 Agent 指令固化，实现了从补货检测到自动下单的全自动闭环。'
pubDate: '2026-04-09'
category: '运维实战'
tags: ['每日复盘', 'VMRack', 'changedetection', 'Webhook', 'OpenClaw', '自动化下单', 'Telegram']
---

## 今日实战概览

今天完成了 VMRack 补货监控 → 自动下单这条链路的**全流程打通**，核心结果有 4 件：

1. 在 OpenClaw 本机构建并运行了 Webhook Listener（端口 20080），暴露公网端点 `/webhook/vmrack-restock`。
2. 完成 changedetection → Webhook → OpenClaw → isolated cron job → AI 自动下单的全链路串联与验收测试。
3. 在 `bh.6661993.xyz`（即 209.146.116.154 上的 changedetection）配置了 L3 系列专属监控规则，CSS 过滤器精准锁定 L3 区域。
4. 固化 VMRack 自动下单 Agent 指令模板，并将 OpenClaw exec 策略调整为 `gateway` 以消除 cron 任务执行时的确认干扰。

额外处理了一个关键排障：
- VPS SSH key 认证问题反复误判，最终确认是 key 文件格式/路径问题（非 key 本身损坏），并找到了可用的通用私钥完成验证。

---

## 关键问题与根因

### 问题1：VPS SSH key 认证反复失败

**现象**
- 多次尝试 SSH 登录 `209.146.116.154:5522` 均失败，最初判断是 key 文件损坏。
- 更换多个 key 副本后问题依旧。

**根因**
- 排障方向有误：之前“key 已损坏”的判断是误判。
- 真实原因是 key 文件格式与 ssh 工具不兼容（如 Termius 导出格式），或读取到了错误路径/错误副本。
- 关键教训：**不要在未确认根因前给问题下结论**，尤其涉及 key 问题时，应先确认是否路径错、格式错，再判断是否“损坏”。

**处理**
- 梳理所有已知 key 副本并逐一验证，最终找到一把可用的 ED25519 OpenSSH 私钥成功登录。
- 确认该 key 为当前所有 VPS 的通用 key，后续优先使用此 key。

---

### 问题2：补货监控误触发（L1/L2 干扰）

**现象**
- changedetection 配置初期，页面任何位置出现“立即使用”都会触发告警，导致 L1/L2 补货也被误报。

**根因**
- 监控粒度过粗，CSS 选择器覆盖了整个活动页而非只覆盖 L3 区域。

**处理**
- 使用精准 CSS 过滤器 `div.grid.grid-cols-3.gap-x-5.gap-y-5.mt-20` 将检测范围收窄到 L3 套餐区域。
- 同时在 OpenClaw cron 任务中固化指令：**严格只买 L3 系列**，L1/L2 有货也不下单。

---

### 问题3：cron 任务触发 exec 时反复要求确认

**现象**
- 补货触发后创建的 cron 任务执行时，每次 exec 操作都弹确认提示，导致流程卡住。

**根因**
- OpenClaw `tools.exec.host` 策略为 `ask`，所有 exec 操作均需人工确认。

**处理**
- 将 `tools.exec.host` 从 `auto` 改为 `gateway`，cron 任务的 exec 不再触发确认。
- 验证：修改后 cron 任务可自动完成浏览器操作（打开页面、登录账号）而无需人工介入。

---

### 问题4：Webhook 端点公网可达性验证

**现象**
- Webhook Listener 运行后，需确认公网能否正确访问。

**根因**
- 本机 Webhook 在 NAT 环境，依赖反向代理/端口映射对外暴露。

**处理**
- 使用 `curl -X POST https://webhook.6661993.xyz/webhook/vmrack-restock` 直接从外部测试。
- 返回 HTTP 200，任务创建成功，链路打通。

---

## 避坑清单（可执行）

1. **SSH key 认证失败时，先排查路径/格式/副本，再判断是否损坏**，不要跳过验证步骤直接下结论。  
2. **页面级监控务必使用 CSS 过滤器限制范围**，避免无关区域的同类元素造成误触发。  
3. **自动化链路（监控→通知→执行）上线前必须做模拟触发验收**，不能只靠检查配置本身。  
4. **cron 任务的 exec 策略要单独处理**，不能与交互会话共用同一策略。  
5. **下单类任务必须有明确的商品边界定义**，例如“只买 L3 系列”要同时在监控层和执行层双重校验。  
6. **关键凭据（账号密码/2FA）不在 memory 中明文存储**，如需记录则使用 secrets 目录或加密存储。  
7. **私钥/Token 一旦暴露应及时轮换**，本次涉及的 key 指纹（SHA256:pIumlPLz...）建议后续审计并按需更换。  

---

## 实战命令汇总

```bash
# 1) Webhook Listener 部署与验证
# 代码路径：/home/huhu/scripts/webhook-listener.py
# 端口：20080
# 端点：/webhook/vmrack-restock
python3 /home/huhu/scripts/webhook-listener.py
curl -X POST http://127.0.0.1:20080/webhook/vmrack-restock -d '{}'

# 2) 公网 Webhook 端点验收测试
curl -X POST https://webhook.6661993.xyz/webhook/vmrack-restock

# 3) changedetection 监控状态验证（VPS端）
docker ps --format "table {{.Names}}\t{{.Status}}"
curl -s http://127.0.0.1:8050 | grep -c "changedetection"

# 4) OpenClaw cron 任务创建示例
openclaw cron add \
  --name "vmrack-restock-order" \
  --delete-after-run \
  --model gemini-2.5-pro \
  --announce-to 758712802 \
  --delay 1m \
  --prompt "打开 https://www.vmrack.net/zh-CN/activity/2026-spring，检查 L3 系列库存，登录账号后按优先级下单（余额支付）"

# 5) SSH 登录 VPS（使用通用 ED25519 key）
ssh -p 5522 -i ~/.ssh/vps_ed25519_key root@209.146.116.154

# 6) 验收 Webhook → cron → 执行完整链路（模拟触发）
# 在 changedetection 后台，手动触发一次监控通知
# 或直接 POST 到 webhook 端点观察任务创建与执行日志
openclaw cron list

# 7) OpenClaw exec 策略检查与修改
# 配置文件路径：~/.openclaw/config.yaml
# tools.exec.host: "gateway"  # 而非 "ask" 或 "auto"
openclaw gateway restart
```

---

## 参考来源

- VMRack 活动页：<https://www.vmrack.net/zh-CN/activity/2026-spring>
- changedetection.io：<https://github.com/dgtlmoon/changedetection.io>
- OpenClaw 文档：<https://opencode.ai/docs>
- Telegram Bot API：<https://core.telegram.org/bots/api>
- OpenClaw 本地运维上下文与当天实操日志（会话内命令与结果）

> 注：本文已做脱敏处理，不包含密码、私钥、Bot Token、chat_id 等敏感信息。

---

## 明日建议

1. **将通用 VPS SSH key 写入 OpenClaw 节点配置**，省去每次手动指定 `-i` 的麻烦。  
2. **为 changedetection 监控增加"触发日志"可视化**，方便回溯每次触发的上下文（页面快照、时间戳）。  
3. **验收 Telegram 推送效果**：当前链路最终由 Telegram bot 回消息给用户，需确认消息格式是否友好。  
4. **考虑增加下单前的二次确认机制**：如“检测到有货后，先推送摘要消息，用户确认后再实际下单”，避免误购。  
5. **将该 Webhook Listener 改为 systemd 服务**，确保开机自启并有标准日志输出。  

今天最可复用的经验：**自动化链路（监控→通知→执行）不是配完组件就结束，必须用真实触发源做端到端验收，才能发现中间件兼容性、权限策略、过滤精度等问题。**
