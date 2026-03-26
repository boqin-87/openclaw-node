---
title: "OpenClaw 集群节点：通过 Docker / 云服务器部署跳出沙盒控制所有集群设备"
description: "完整详解 OpenClaw 集群节点的两种部署方式：网关节点（Docker/云服务器）和终端节点（Mac/Linux/Win），实现从任意设备通过主网关统一管控所有节点。"
pubDate: "2026-03-26"
heroImage: ""
tags: ["OpenClaw", "集群", "Docker", "节点", "HomeLab"]
---

## 什么是 OpenClaw 集群节点？

OpenClaw 的集群节点机制允许你在一台设备上部署"网关龙虾"（主控端），然后在其他设备上部署"节点龙虾"（被控端），从而实现：

- 从网关设备统一管控所有节点机器
- 在任意节点上执行命令、截图、查看屏幕等
- 突破 Docker 沙盒限制，直接控制宿主机

本文以飞牛（funnas）上的 Docker 部署为例，详解**网关龙虾**和**节点龙虾**的完整配置流程。

---

## 一、部署前准备

### 1.1 必要配置（所有平台通用）

在部署节点前，无论是 Docker、云服务器还是本地物理机，都需要执行以下配置：

```bash
# 允许所有来源访问（局域网 + 外网均可）
openclaw config set gateway.bind "lan"
openclaw config set gateway.controlUi.allowedOrigins '["*"]'
openclaw config set gateway.controlUi.allowInsecureAuth true
openclaw config set gateway.controlUi.dangerouslyDisableDeviceAuth true
```

> ⚠️ 以上配置会让网关暴露在局域网/外网，请确保网络环境安全，或配合防火墙规则限制访问。

### 1.2 各平台差异

| 平台 | 特殊说明 |
|------|---------|
| **Mac / Linux** | 直接运行上述配置命令即可 |
| **Windows** | `allowedOrigins` 参数格式略有不同，见下方 |

**Windows 用户**请使用以下命令：

```powershell
openclaw config set gateway.bind "lan"
openclaw config set gateway.controlUi.allowedOrigins --json '["\\"*\\""]'
openclaw config set gateway.controlUi.allowInsecureAuth true
openclaw config set gateway.controlUi.dangerouslyDisableDeviceAuth true
```

---

## 二、网关龙虾部署（Docker / 云服务器）

网关龙虾可以部署在任何有 Docker 的设备上，包括云服务器、NAS、树莓派等。以下以飞牛 NAS 的 Docker 环境为例。

### 2.1 编译 Docker 镜像

OpenClaw 官方暂未提供预编译的集群节点专用镜像，需要从源码自行构建。

#### 获取源码

```bash
# 克隆官方仓库
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

#### 修改编译配置（跳过不支持的平台插件）

编辑 `scripts/stage-bundled-plugin-runtime-deps.mjs`，将 `stageBundledPluginRuntimeDeps` 函数改为以下内容：

```javascript
export function stageBundledPluginRuntimeDeps(params = {}) {
  const repoRoot = params.cwd ?? params.repoRoot ?? process.cwd();
  for (const pluginDir of listBundledPluginRuntimeDirs(repoRoot)) {
    const pluginId = path.basename(pluginDir);
    // 跳过飞书和 Discord 插件的依赖安装
    if (pluginId === "feishu" || pluginId === "discord") {
      console.log(`Skipping runtime deps installation for ${pluginId} plugin`);
      continue;
    }
    const packageJson = readJson(path.join(pluginDir, "package.json"));
    const nodeModulesDir = path.join(pluginDir, "node_modules");
    removePathIfExists(nodeModulesDir);
    if (!hasRuntimeDeps(packageJson) || !shouldStageRuntimeDeps(packageJson)) {
      continue;
    }
    installPluginRuntimeDeps(pluginDir, pluginId);
  }
}
```

#### 构建镜像

```bash
docker build -t openclaw:local -f Dockerfile .
```

构建完成后，可在 Docker 的本地镜像列表中看到 `openclaw:local`。

### 2.2 docker-compose 启动

创建 `docker-compose.yml`：

```yaml
services:
  openclaw:
    image: openclaw:local
    container_name: openclaw
    ports:
      - "18789:18789"
    volumes:
      - openclaw-data:/root/.openclaw
      - /vol1:/vol1  # 映射你需要的目录
    environment:
      - OPENCLAW_GATEWAY_TOKEN=你的自定义token
      - npm_config_registry=https://registry.npmmirror.com/
    restart: unless-stopped
    command: >
      bash -c "
        openclaw config set gateway.bind 'lan' &&
        openclaw config set gateway.controlUi.allowedOrigins '[\"*\"]' &&
        openclaw config set gateway.controlUi.allowInsecureAuth true &&
        openclaw config set gateway.controlUi.dangerouslyDisableDeviceAuth true &&
        openclaw gateway run --allow-unconfigured
      "
    healthcheck:
      test: ["CMD", "curl", "-s", "--fail", "http://localhost:18789/health", "||", "exit", "0"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    network_mode: bridge

volumes:
  openclaw-data:
```

```bash
docker compose up -d
```

启动后访问 `http://你的服务器IP:18789` 按 OpenClaw 初始化引导完成配置即可。

---

## 三、节点龙虾部署（被控端）

节点龙虾部署在被控设备上，以下以 Mac 电脑为例（Linux / Windows 流程相同）。

### 3.1 安装节点服务

在被控设备（节点）上执行：

```bash
openclaw node install
openclaw node restart
```

### 3.2 查询网关设备 Token

在**网关设备**（主控端）上查询 Token：

```bash
openclaw dashboard
```

记录下显示的 Gateway Token。

### 3.3 配置节点连接网关

在**节点设备**上，将网关地址和 Token 配置进去：

```bash
# 将网关 Token 设为网关设备的 Token
openclaw config set gateway.auth.token 你的网关token
```

### 3.4 启动节点服务

在节点设备上运行（将地址替换为你的网关服务器 IP）：

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 openclaw node run --host 192.168.100.61 --port 18789
```

### 3.5 网关设备确认连接

在**网关设备**上执行：

```bash
# 查看已发现的设备
openclaw devices list

# 批准最新发现的节点
openclaw devices approve --latest

# 确认握手状态
openclaw devices list
```

看到节点状态变为 `connected` 即表示握手成功。

### 3.6 节点保持运行

节点服务需要在后台持续运行，建议使用 systemd 或 launchd 管理。

---

## 四、授权访问配置（重要）

### 4.1 默认行为

每次节点执行敏感操作（截图、执行命令等）时，网关侧会弹出授权确认。

### 4.2 完全开放权限（不再弹窗）

如果需要节点完全自治、不打扰主控端，可以在**节点设备**上执行：

```bash
# 设置 exec 模式为完全允许
openclaw config set tools.exec.security "full"
openclaw config set tools.exec.ask "off"

# 重启 gateway 使配置生效
openclaw gateway restart
```

### 4.3 恢复安全模式

```bash
openclaw config set tools.exec.security "allowlist"
openclaw config set tools.exec.ask "on-miss"
openclaw gateway restart
```

---

## 五、常见问题

### Q1: 节点一直显示 offline？

- 检查节点机器和网关机器网络是否互通（ping 一下）
- 确认节点服务是否在运行（`openclaw node status`）
- 确认 Token 和 IP 地址配置正确
- 检查防火墙是否放行了 `18789` 端口

### Q2: Docker 部署的网关无法控制宿主机？

这是正常现象——Docker 容器默认在沙盒隔离环境。要控制宿主机，必须通过**节点龙虾**机制，节点部署在宿主机上而非容器内。

### Q3: 外网访问安全吗？

建议配合 Cloudflare Tunnel、WireGuard VPN 或强密码 Token 保护网关，不要直接将端口暴露在公网。

---

## 六、总结

通过本文的部署方式，你可以：

1. ✅ 在云服务器 / NAS 上部署 **网关龙虾**，作为集群主控中心
2. ✅ 在 Mac / Linux / Windows 上部署 **节点龙虾**，作为被控终端
3. ✅ 从任意一个网关设备统一向所有节点下发命令
4. ✅ 突破 Docker 沙盒限制，真正控制集群中每一台设备

集群节点机制是 OpenClaw 区别于普通 AI 助手的关键能力，让它真正成为了一个**跨设备的智能体控制中心**。
