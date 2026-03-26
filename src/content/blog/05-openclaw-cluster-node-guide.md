---
title: 'OpenClaw 集群节点实战：Docker 网关 + 远程节点接入指南'
description: '基于 Docker 网关与远程节点接入的 OpenClaw 集群部署思路，整理关键配置、接入流程与安全注意事项。'
pubDate: '2026-03-26'
category: '开发'
tags: ['openclaw', 'docker', 'cluster', 'node', 'self-hosted']
---

这篇是我根据一篇公开教程整理出来的**原创实操版**，重点不是照抄命令，而是把整个 OpenClaw 集群的部署逻辑讲清楚：

- 哪台机器适合当网关
- Docker 部署时需要开什么配置
- 本地节点如何接入远程网关
- 如何在可用性和安全性之间做取舍

如果你想把 OpenClaw 做成“**一台中心网关 + 多台可控节点**”的形态，这套方法是可行的。

---

## 一、整体架构怎么理解

OpenClaw 集群可以简单分成两个角色：

### 1. 网关（Gateway）
负责：
- 提供控制入口
- 持有管理 token
- 发现并批准节点
- 统一查看节点状态

### 2. 节点（Node）
负责：
- 跑在被控设备上
- 与网关建立连接
- 接受网关的管理与执行请求

典型拓扑如下：

```text
你的浏览器 / 控制端
        ↓
   OpenClaw Gateway
   （可放在 Docker / VPS / 本地服务器）
        ↓
  多个 OpenClaw Node
  （Mac / Linux / Windows / 其他机器）
```

如果你只是想先验证功能，完全可以：
- 在一台 Docker 主机上先跑网关
- 再在自己的 Mac / Linux 电脑上跑节点

---

## 二、部署前要明确的几个前提

### 1. 网关必须能被节点访问到
节点接入网关时，需要知道：
- 网关地址
- 网关端口
- 认证 token

所以无论你把网关放在：
- 家里机器
- Docker 主机
- 云服务器

都要确保节点能连到它。

### 2. 控制界面跨域和认证限制要考虑
有些教程为了演示方便，会把控制台改成非常宽松的模式，例如：
- 允许任意来源访问
- 关闭严格认证限制

这确实能让部署变简单，但也意味着**安全边界会明显变弱**。

如果只是内网测试，可以临时放宽；如果准备长期公网使用，建议后期逐步收紧。

---

## 三、Docker 网关部署思路

如果你想把网关养在 Docker 里，思路通常是：

1. 准备 OpenClaw 镜像
2. 映射数据目录保存配置
3. 暴露控制端口
4. 在容器启动时写入基础配置
5. 启动 gateway

### 一个简化后的 compose 思路

```yaml
services:
  openclaw:
    image: openclaw:local
    container_name: openclaw
    ports:
      - "18789:18789"
    volumes:
      - openclaw-data:/root/.openclaw
    environment:
      - OPENCLAW_GATEWAY_TOKEN=your-token
    restart: unless-stopped
```

如果是源码自行构建镜像，通常流程是：

```bash
docker build -t openclaw:local -f Dockerfile .
```

然后再通过 `docker-compose` 或 `docker compose` 启动。

---

## 四、网关侧常见配置项

以下几项通常最关键：

### 1. 绑定方式
让网关监听局域网或外网：

```bash
openclaw config set gateway.bind "lan"
```

### 2. 控制台允许来源
如果你要让 Web UI 从不同来源访问，可能会用到：

```bash
openclaw config set gateway.controlUi.allowedOrigins '["*"]'
```

### 3. 测试环境下的简化认证
教程里经常会看到：

```bash
openclaw config set gateway.controlUi.allowInsecureAuth true
openclaw config set gateway.controlUi.dangerouslyDisableDeviceAuth true
```

这里要强调一下：

> 这类配置只适合测试、内网、或你明确知道自己在做什么的环境。

如果网关是公网暴露的，长期保留这些选项并不安全。

---

## 五、节点机器怎么接入网关

节点侧大致分两步：

### 1. 安装并启动节点服务

```bash
openclaw node install
openclaw node restart
```

### 2. 指定网关 token 与地址
先把节点指向网关：

```bash
openclaw config set gateway.auth.token your-token
```

然后运行节点服务，指向网关地址：

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 openclaw node run --host 10.10.10.5 --port 18789
```

其中：
- `10.10.10.5` 要换成你的网关地址
- `18789` 要换成你的实际端口（如果改过）

---

## 六、网关和节点的握手流程

这是最关键的一段。

### 网关侧：查看是否发现节点

```bash
openclaw devices list
```

### 网关侧：查看控制信息 / token

```bash
openclaw dashboard
```

### 网关侧：批准最新节点

```bash
openclaw devices approve --latest
```

批准后，再次检查：

```bash
openclaw devices list
openclaw nodes status
```

如果一切正常，你就应该能看到：
- 节点已被识别
- 节点已批准
- 节点在线

---

## 七、最容易踩坑的地方

### 1. 节点能启动，但网关看不到
常见原因：
- 网关地址填错
- 网关端口没开放
- token 错误
- Docker 端口没映射出来

### 2. 网关能看到节点，但无法批准
常见原因：
- 认证配置不一致
- 节点没有持续运行
- 中间有反向代理/防火墙导致 WebSocket 不通

### 3. Docker 部署后控制台能开，但节点连不上
这类问题通常不是 OpenClaw 本体坏了，而是：
- 容器端口映射不对
- 网关只监听了本地而非 LAN
- 外层代理没把升级连接转发对

---

## 八、关于“控制所有设备”的现实理解

有些教程标题会写得很猛，比如“跳出沙盒控制所有设备”。

实际理解应该更稳一点：

- **网关**：负责统一管理
- **节点**：负责在目标机器上提供执行能力
- **能力边界**：取决于节点本身安装在哪台机器、给了哪些权限

所以不是凭空“远程控制一切”，而是：

> 你把 OpenClaw Node 部署到哪台机器，它就成为集群里可被网关调度的一台节点。

---

## 九、安全建议

这一段我建议一定要认真看。

### 测试时可以放宽
为了快速打通流程，你可以先：
- 允许 LAN 访问
- 放宽 control UI 的来源限制
- 在内网环境下做验证

### 稳定后建议收紧
长期用时，建议逐步改回：
- 不要允许 `allowedOrigins = ["*"]`
- 不要长期保留 `dangerouslyDisableDeviceAuth`
- 不要默认关掉所有执行确认

某些教程还会建议把执行权限改成完全允许，例如：

```bash
openclaw config set tools.exec.security "full"
openclaw config set tools.exec.ask "off"
```

这确实能减少弹窗，但安全风险也会同步上升。更稳妥的做法是：
- 仅在可信内网或测试节点上使用
- 正式环境恢复为更保守模式

---

## 十、我建议的新手部署顺序

如果你第一次搭，我建议按这个顺序来：

1. **先本地单机跑通 OpenClaw**
2. **再把网关迁到 Docker / VPS**
3. **再接入一台节点设备**
4. **确认网关发现、批准、状态查看都正常**
5. **最后再考虑权限放宽、外网访问、更多节点**

不要一上来就：
- Docker
- 公网
- 多节点
- 放宽认证
- 反向代理
- 远程执行

全堆一起。那样排错会非常痛苦。

---

## 十一、适合哪些场景

这套模式很适合：

- 多台设备统一管理
- 一台主控机 + 多台被控机
- 家庭实验室 / Homelab
- 本地电脑 + VPS + Docker 的混合协同
- 希望把 AI 助手扩展到多台设备执行

---

## 十二、结语

如果你把 OpenClaw 理解成一个“**中心网关 + 多个执行节点**”的系统，就会发现这套集群部署逻辑其实并不复杂。

真正要花心思的，不是把服务跑起来，而是：
- 网络可达性
- 授权边界
- 节点权限控制
- Docker / 代理 / WebSocket 的连通性

跑通之后，体验会很像给不同机器装上可统一调度的“手脚”。

---

## 原始参考

我这篇是基于公开教程整理、重写和补充理解后的原创版，原始参考链接如下：

- 原文链接：<https://www.vumstar.com/11022/>

如果你只是想快速照着原作者的步骤操作，也建议先阅读原文，再结合你自己的环境做调整。
