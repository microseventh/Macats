# 🌐 Docker IPv6 NAT 网络配置与标准部署指南

本指南用于解决 **VPS 仅有单个公网 IPv6（或单段 /64）** 情况下，如何让 Docker 容器（如 qBittorrent、Nginx 等）完美支持 IPv6 出站与入站，同时避免与宿主机物理网卡发生路由冲突。

## 📌 核心设计原理

* **宿主机：** 使用商家分配的公网 IPv6 地址。
* **Docker 内部：** 采用 IPv6 唯一本地地址（ULA，即 `fd00::/84` 这种私有网段）。
* **网络路由：** 开启 Linux 内核转发，利用 Docker 20.10+ 自带的 `ip6tables` 功能自动实现 IPv6 的 NAT（网络地址转换）网络结构。

## 🛠️ 第一部分：宿主机基础环境配置（一次性配置）

在配置 Docker 之前，必须确保宿主机内核允许 IPv6 流量进行路由转发。

### 1. 开启 Linux 内核 IPv6 转发

修改系统内核参数文件 `/etc/sysctl.conf`，追加或修改以下内容：

```ini
# 开启所有网卡的 IPv6 转发
net.ipv6.conf.all.forwarding=1
net.ipv6.conf.default.forwarding=1
```

应用配置使其立即生效：

```bash
sudo sysctl -p
```

### 2. （可选）调整系统防火墙（如 UFW）

如果 VPS 启用了 UFW 防火墙，默认会拦截自定义网桥的转发流量。 打开 `/etc/default/ufw`，确保转发策略为允许：

```text
DEFAULT_FORWARD_POLICY="ACCEPT"
```

修改后重载防火墙：`sudo ufw reload`。

## ⚙️ 第二部分：Docker 全局 IPv6 开启

修改 Docker 的守护进程配置文件，使其默认具备 IPv6 处理能力。

### 1. 配置文件路径

编辑 `/etc/docker/daemon.json`（若不存在则新建），写入以下标准 JSON：

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/64",
  "experimental": true,
  "ip6tables": true
}
```

> **📝 参数详解：**
> 
> * `"ipv6": true`: 激活 Docker 的 IPv6 功能。
> * `"fixed-cidr-v6"`: 默认网桥（`docker0`）使用的私有 IPv6 网段。
> * `"ip6tables": true`: ​**最关键参数**​。让 Docker 自动管理 `ip6tables` 防火墙规则，实现自动 NAT 转换，免去手动写 `iptables` 的痛苦。

### 2. 重启服务

```bash
sudo systemctl restart docker
```

## 🚀 第三部分：应用级复用部署（以 Docker Compose 为例）

当全局环境配置好后，后续任何需要复用 IPv6 的容器项目，只需在 `docker-compose.yml` 中声明一个 ​**开启了 IPv6 功能的自定义网络**​。

### 📝 复用模板示例（qBittorrent）

```yaml
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - WEBUI_PORT=8081
    volumes:
      - ./config:/config
      - ./downloads:/downloads
    ports:
      - 8081:8081         # WebUI 网页管理端口
      - 6881:6881         # BT 监听端口（TCP）
      - 6881:6881/udp     # BT 监听端口（UDP）
    networks:
      - app_net_v6        # 👈 引用下方定义的支持 IPv6 的网络
    restart: unless-stopped

networks:
  app_net_v6:             # 👈 核心：自定义网络名称
    enable_ipv6: true     # 👈 核心：显式开启该网络的 IPv6 路由
    ipam:
      driver: default
      config:
        - subnet: fd00:dead:beef:10::/64  # 👈 核心：为此网络划分一个独立的私有 IPv6 段（不同项目可递增，如 :20::/64）
```

**启动命令：**

```bash
# 建议使用 down 清理旧网络后再 up，确保新网络拓扑被正确建立
docker compose down && docker compose up -d
```

## 🔍 第四部分：网络连通性验证维护

无论是新部署还是日常排查，都可以使用以下流水线式命令来验证。

### 1. 验证宿主机 IPv6

```bash
# 检查是否拥有公网 IPv6 (global)
ip -6 addr show

# 测试出站
curl -6 https://api6.ipify.org
```

### 2. 验证容器内部 IPv6（以 qb 容器为例）

```bash
# 强制容器使用 IPv6 访问权威节点，若返回宿主机公网 IPv6 地址则说明 NAT 完美成功
docker exec -it qbittorrent curl -6 https://api6.ipify.org
```

### 3. 应用层确认（如 PT/BT 场景）

* 进入 qBittorrent 的高级设置（Advanced Settings），确保“监听的可选 IP 地址”选择为 ​**“所有地址”**​（All addresses）。
* 观察 Peer 列表，出现带有方括号的地址（如 `[2a0c:b840:...]:端口`）即代表 IPv6 入站与出站双向完全打通。

这份文档已经做到了完全解耦，以后遇到新 VPS，直接按照一、二、三步无脑复制粘贴即可。

