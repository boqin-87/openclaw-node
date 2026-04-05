---
title: '每日实战复盘（2026-04-05）：博客发布链路修复 + ROS/L2TP 故障定位'
description: '记录今天从博客发布失败排查，到 RouterOS L2TP 连接异常定位的完整实战：根因、避坑、命令与可复用流程。'
pubDate: '2026-04-05'
category: '运维实战'
tags: ['每日复盘', 'Astro', 'Git', 'RouterOS', 'L2TP', 'IPsec', '故障排查']
---

## 今日实战概览

今天的核心工作有两条主线：

1. **博客发布链路排障**
   - 已写完并提交文章后，线上访问仍 404。
   - 复核发现不是内容问题，而是域名入口与博客服务未对齐（首页返回的是路由器登录页，不是博客站点）。
   - 按用户要求，先确保**内网可访问**，确认新文在内网博客列表和详情页均可打开。

2. **ROS 连接异常排查（L2TP/IPsec）**
   - 用户反馈“断网无法连接内网”，并提到曾手动删除旧 WG 配置。
   - 通过 ROS 日志回放定位：故障时段主要是 **L2TP/IPsec 参数不一致** 导致握手失败（并非 ROS 整体掉线）。
   - 后续恢复备份后恢复正常，说明此前存在“误删导致依赖断裂”的高风险操作。

---

## 关键问题与根因

### 问题 1：文章已推送但线上仍 404

**现象**
- Git 已 push，部署触发成功，但线上链接 404。

**根因**
- 内容层没问题，问题在入口层：域名/反代/部署目标不一致。
- 首页命中的是非博客服务（返回了网络设备页面）。

**结论**
- 发布链路要拆成三段验收：
  1) 仓库是否有目标 commit
  2) 构建是否通过
  3) 域名入口是否命中正确服务

---

### 问题 2：L2TP 一直连不上，体感像“网络全断”

**现象**
- 长时间无法连回内网，反复修改后才恢复。

**日志特征**
- 连续出现 `parsing packet failed, possible cause: wrong password`
- 伴随 `phase1 negotiation failed due to time up`
- 故障时段内有多次 `PPP secret` 与 `L2TP Server settings` 修改记录
- 后续又出现 `ISAKMP-SA established` + `l2tp ... connected`

**根因**
- 主要是 **客户端参数与服务端参数不一致**（PSK 或账号口令不匹配）。
- 同时叠加“删除旧 WG 配置”的操作风险，导致排障复杂度上升。

**结论**
- 这类故障常被误判成“ROS 断网”，但本质多是 VPN 认证链路不一致。

---

## 避坑清单（可执行）

1. **改 VPN 前先备份**
   - 先导出 `.rsc` 与二进制备份，再改配置。
2. **一次只改一项**
   - 先改 PSK，再测；再改账号密码，再测；避免同时改多项。
3. **改完立刻看日志关键字**
   - `wrong password`、`phase1 negotiation failed`、`connected`。
4. **禁用优先，删除最后**
   - 旧 WG/PPP 先 `disabled=yes` 观察，不要直接删除。
5. **固定排障顺序**
   - 先看设备在线性（ping）→ 再看 VPN 握手（ipsec/l2tp）→ 最后看路由与防火墙。
6. **发布链路做分层验收**
   - `git push` 成功 ≠ 线上可访问，必须补一层访问验证。

---

## 实战命令汇总

```bash
# 1) 检查 Git 远端是否正确（固定 SSH）
git remote -v
# 目标应为：git@github.com:boqin-87/openclaw-node.git

# 2) 提交并推送文章
git add src/content/blog/daily-practice-2026-04-05.md
git commit -m "新增2026-04-05每日实战复盘文章"
git push origin HEAD

# 3) 本地构建校验（Astro）
npm run build

# 4) 内网访问校验（示例）
curl -I http://<blog-lan-host>:4321/blog/
curl -I http://<blog-lan-host>:4321/blog/daily-practice-2026-04-05/

# 5) RouterOS 日志排查（关键时间段）
/log print without-paging
# 或按关键字过滤：
/log print where message~"ipsec|l2tp|ppp|wireguard|peer|changed"

# 6) L2TP 服务与账号检查（RouterOS CLI）
/interface l2tp-server server print
/ppp secret print
```

---

## 参考来源（当天使用）

- 本地博客仓库与内容规范：
  - `blog-kejilion-style/src/content.config.ts`
  - `blog-kejilion-style/src/content/blog/07-vps-warp-sui-practice-notes.md`
- 当天会话记录与故障上下文：
  - OpenClaw 会话日志（当日）
  - `memory/2026-04-05.md`
  - `memory/2026-04-05-exec-policy.md`
- 部署与发布相关：
  - `blog-kejilion-style/DEPLOY.md`
  - `blog-kejilion-style/.github/workflows/deploy.yml`
- RouterOS 实时日志（当日窗口）
  - 包含 L2TP/IPsec 握手失败与恢复记录

> 注：本文已按要求脱敏，不包含密钥、口令、token、私网敏感入口等信息。

---

## 明日建议

1. 做一个“**VPN 变更前一键快照**”脚本（导出配置 + 记录时间戳）。
2. 把 L2TP/WG 的“**最小可用基线**”写成 SOP，变更必须走清单。
3. 给博客发布链路加“**部署后自动访问校验**”（已建立 23:10 二次校验任务，可继续补告警）。
4. 将今天故障时间线沉淀到长期知识库，便于后续快速对照排障。

今天最有价值的经验：**把“感觉像断网”的问题，拆成可验证的链路步骤，日志会给出答案。**
