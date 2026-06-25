明白，把步骤拆解开来、手动一步步执行，能更清晰地掌控每一步的配置状态，非常适合作为长期的技术备忘录。

以下是为你整理的**纯手动、分步执行**的服务器初始化与 Docker 双栈网络部署指南（已去除自动化脚本，完全由你手动掌控），最后同样附带了未经验证的 UFW 可选方案。

## 第一部分：分步手动部署指南

### 第一步：更新系统软件包

在服务器挂载好 `/data` 硬盘后，首先同步最新的软件源并升级现有包：

```bash
# 更新软件包列表
sudo apt update

# 升级所有已安装的软件包到最新版本
sudo apt upgrade -y

# 清理不再需要的孤立依赖包
sudo apt autoremove -y
```

### 第二步：手动安装 Docker 核心组件

使用官方的安装脚本来自动配置 Docker 源并完成基础安装：

```bash
# 下载官方安装脚本
curl -fsSL https://get.docker.com -o get-docker.sh

# 运行安装脚本
sudo sh get-docker.sh
```

### 第三步：创建数据盘存储目录

确保你的大容量硬盘已成功挂载在 `/data`。接着在其中手动创建 Docker 数据目录和未来的应用编排目录：

```bash
# 创建 Docker 引擎底层数据目录
sudo mkdir -p /data/docker

# 创建 Docker Compose 项目编排目录
sudo mkdir -p /data/docker-apps
```

### 第四步：配置 Docker 的存储与 IPv6

打开（或创建） Docker 的守护进程配置文件：

```bash
sudo nano /etc/docker/daemon.json
```

在文件中完整写入以下 JSON 内容（通过把掩码设为标准的 `/64`，确保私有 IPv6 网段清爽合规）：

```json
{
  "data-root": "/data/docker",
  "ipv6": true,
  "fixed-cidr-v6": "fd00:ffff::/64",
  "ip6tables": true
}
```

保存并退出（Nano 编辑器中按 `Ctrl+O` 回车保存，`Ctrl+X` 退出）。

### 第五步：重启 Docker 服务

让刚才修改的 `daemon.json` 配置立即生效：

```bash
# 重新加载系统服务配置
sudo systemctl daemon-reload

# 重启 Docker 引擎
sudo systemctl restart docker
```

### 第六步：优化系统网络内核参数

调整内核参数以开启 BBR 拥塞控制并增强网络防范能力：

```bash
sudo nano /etc/sysctl.conf
```

滑动到文件最下方，追加以下优化参数：

```text
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
```

保存并退出后，执行以下命令使内核参数立即生效：

```bash
sudo sysctl -p
```

### 第七步：配置并持久化 IPv6 NAT 出网规则

由于关闭了防火墙，我们需要手动在系统底层补上 IPv6 的 `MASQUERADE`（伪装）规则，使容器能顺利通过宿主机访问外网 IPv6 资源，并将其持久化。

1. **临时应用规则（立即可用）**：

   ```bash
   sudo ip6tables -t nat -A POSTROUTING -s fd00:ffff::/64 ! -o docker0 -j MASQUERADE
   ```
2. **安装规则持久化工具**：

   ```bash
   sudo apt install iptables-persistent netfilter-persistent -y
   ```

   *提示：在安装过程中系统会弹出蓝色背景的提示框，询问是否保存当前的 IPv4 和 IPv6 规则，全部使用方向键选择 是/Yes 并回车。***是/Yes**
3. **手动保存当前规则（防止重启失效）**：

   ```bash
   sudo netfilter-persistent save
   ```

### 第八步：最终环境自检

运行以下命令，启动一个临时的测试容器，检查它是否能正常获取内网 IPv6 地址并顺畅 Ping 通外网：

```bash
docker run --rm -it alpine ash -c "ip -6 addr show && ping6 -c 3 google.com"
```

## 第二部分：日常运维目录结构与安全规范

### 1. 标准目录结构

建议今后所有的 Docker Compose 项目都整齐地归纳在 `/data/docker-apps` 下：

```text
/data/
├── docker/               # Docker 引擎底层数据（镜像、容器层，自动管理）
└── docker-apps/          # 你的项目编排目录（手动管理）
    ├── qbittorrent/      # 举例：下载服务
    │   ├── docker-compose.yml
    │   └── config/       # 挂载出来的配置目录
    └── nginx/
        └── docker-compose.yml
```

### 2. 端口安全暴露规范

在不启用 UFW 防火墙的环境下，Docker 映射端口的写法决定了安全边界：

* **需要外网公开的服务**（如 Web 网站、下载器监听端口）：
  ```yaml
  ports:
    - "8080:80"
  ```
* **私密/纯内网调用的服务**（如数据库、无需对外的后台、或者准备交给反向代理的服务）：
  ```yaml
  ports:
    - "127.0.0.1:3306:3306"  # 核心：加上 127.0.0.1，彻底阻断外网直接探测
  ```

## 第三部分：🧱 可选补充：UFW 防火墙共存方案（🚨 未经全面验证）

如果你在未来因为特殊需求必须重新开启 UFW 防火墙，请务必严格按照以下步骤精细化配置，否则会导致 Docker 的自定义网络全面断网。

### 1. 确保全局转发安全策略

打开 `/etc/default/ufw`：

```bash
sudo nano /etc/default/ufw
```

确保其维持安全的默认拦截值：

```text
DEFAULT_FORWARD_POLICY="DROP"
```

### 2. 精准放行 Docker 的 IPv4 与 IPv6 转发流量

打开 UFW 的 IPv4 规则文件：

```bash
sudo nano /etc/ufw/before.rules
```

在末尾 `COMMIT`**之前**加入：

```text
-A ufw-before-forward -i docker0 -j ACCEPT
-A ufw-before-forward -o docker0 -j ACCEPT
-A ufw-before-forward -s 172.16.0.0/12 -j ACCEPT
-A ufw-before-forward -d 172.16.0.0/12 -j ACCEPT
```

打开 UFW 的 IPv6 规则文件：

```bash
sudo nano /etc/ufw/before6.rules
```

在末尾 `COMMIT`**之前**加入：

```text
-A ufw6-before-forward -i docker0 -j ACCEPT
-A ufw6-before-forward -o docker0 -j ACCEPT
-A ufw6-before-forward -s fd00:ffff::/64 -j ACCEPT
-A ufw6-before-forward -d fd00:ffff::/64 -j ACCEPT
```

### 3. 注入 IPv6 NAT 规则

同样在 `/etc/ufw/before6.rules` 中，滑动到文件的**最顶部**（在 `*filter` 这一行的上方），注入独立的 `nat` 表配置：

```text
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s fd00:ffff::/64 ! -o docker0 -j MASQUERADE
COMMIT
```

### 4. 允许 SSH 并重载防火墙

```bash
sudo ufw allow 22/tcp  # 如果你修改过 SSH 端口，请换成你的实际端口
sudo ufw enable
sudo ufw reload
```
