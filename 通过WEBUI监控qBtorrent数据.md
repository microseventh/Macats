# qBittorrent 流量监控与离线分析系统构建指南

本指南旨在构建一套完整的 qBittorrent (qB) 做种流量监控系统。系统通过宿主机 Python 脚本轮询 qB API，将 Peer 的高频活动特征存入 PostgreSQL 数据库，并提供一键导出为轻量级 SQLite `.db` 文件的方案，方便后续在本地 Mac + Jupyter 环境下进行数据透视与吸血客户端拉黑策略分析。

## 1. 目录与路径规划

建议在 VPS 上统一一个根目录（例如 `/opt/qb`）来管理所有相关文件与挂载卷，确保结构清晰，对于qb的部署所需要的文件夹，都会放在这个目录下面：

```text
/opt/qb/
├── docker-compose.yaml       # 容器编排配置文件
├── Dockerfile           # 监控脚本的构建文件
└── monitor.py           # 核心轮询脚本
```

## 2. 核心网络设置：IPv6 ULA 与 NAT 转发

为了让 qBittorrent 能够连接到更多的 IPv6 节点，最大化做种效率，需要为 Docker 开启 IPv6 支持并配置 ULA（唯一本地地址）与 NAT 转发。

详细可以参考文章：[Docker IPv6 NAT 网络配置与标准部署指南](./Docker IPv6 NAT 网络配置与标准部署指南.md)

​前置宿主机配置（/etc/docker/daemon.json）：`/etc/docker/daemon.json`

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/48",
  "experimental": true,
  "ip6tables": true
}
```

*​修改完毕后记得执行 systemctl restart docker 重启 Docker 服务。​*`systemctl restart docker`

## 3. Docker Compose 配置 (`docker-compose.yml`)

将 qBittorrent、PostgreSQL 数据库以及我们自己写的 Python Monitor 脚本编排在一起，确保它们在同一个自定义网络下相互通信。

```yaml
version: '3.8'

services:
  # 1. 你的 qBittorrent 容器（完全保留你原有的网络和环境配置）
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London  # 保持英国时区，请修改为您的时区。
      - WEBUI_PORT=8081 #默认使用8081，不占用8080端口。
    volumes:
      - /opt/qb/config:/config
      - /opt/qb/downloads:/downloads
    ports:
      - 8081:8081         # NPM 将通过宿主机的这个端口来访问它。
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
    networks:
      - qb_net_v6         # 保持你的 IPv6 下载网络。

  # 2. 生产级中心数据库：PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: qb-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=your_db_secure_password  # ⚠️请修改为你自定义的数据库密码
      - POSTGRES_DB=qb_monitor
    ports:
      - '5432:5432'       # 暴露 5432 供你的 Mac 本地直连
    volumes:
      - /opt/postgres_data:/var/lib/postgresql/data
    networks:
      - qb_net_v6

  # 3. 内部高频数据采集 Agent
  qb-monitor-agent:
    build: .
    container_name: qb-monitor-agent
    restart: unless-stopped
    environment:
      - TZ=Europe/London  # 时区保持一致
    depends_on:
      - qbittorrent
      - postgres
    networks:
      - qb_net_v6         # 在同一个 compose 下，Agent 通过 http://qbittorrent:8081 连接，完全不受外网影响

networks:
  qb_net_v6:              # 👈 原封不动保留你的 IPv6 网络配置
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: fd00:dead:beef:10::/64 # 这里默认即可。
```

## 4. 监控端容器构建 (`monitor/Dockerfile`)

保持监控端镜像轻量级，使用 Python Slim 版本。

```dockerfile
FROM python:3.10-slim
WORKDIR /app
RUN apt-get update && apt-get install -y libpq-dev gcc && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir qbittorrent-api psycopg2-binary
COPY monitor.py .
CMD ["python", "-u", "monitor.py"]
```

## 5. 核心监控脚本 (`monitor/monitor.py`)

该脚本负责登录 qB，抓取种子列表，遍历 Peer 详情，并将核心字段清洗后入库。

```python
import time
import psycopg2
from datetime import datetime
from qbittorrentapi import Client

# ==================== 生产环境内部配置 ====================
# 走 Docker 内网 DNS 通信，端口为你的原有设置 8081，安全且不走公网流量
QB_HOST = "http://qbittorrent:8081"
QB_USER = "admin"                  # 替换为你 qbee WebUI 真实的用户名
QB_PASS = "adminadmin"             # 替换为你 qbee WebUI 真实的密码，一定要替换！

DB_CONFIG = {
    "host": "postgres",            # 对应 compose 中的服务名
    "database": "qb_monitor",
    "user": "user",
    "password": "your_db_secure_password" # ⚠️必须与 compose 中的密码完全一致
}
# =========================================================

def init_pg_db():
    """初始化 PostgreSQL 数据表"""
    while True:
        try:
            conn = psycopg2.connect(**DB_CONFIG)
            cursor = conn.cursor()
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS peer_minute_stats (
                    id SERIAL PRIMARY KEY,
                    timestamp TIMESTAMP,
                    torrent_hash VARCHAR(40),
                    torrent_name TEXT,
                    ip VARCHAR(45),
                    country VARCHAR(10),
                    client TEXT,
                    uploaded_mb REAL,
                    downloaded_mb REAL,
                    avg_upspeed_kbs REAL,
                    max_upspeed_kbs REAL,
                    peer_progress VARCHAR(10)
                );
                CREATE INDEX IF NOT EXISTS idx_torrent_hash ON peer_minute_stats(torrent_hash);
                CREATE INDEX IF NOT EXISTS idx_timestamp ON peer_minute_stats(timestamp);
            ''')
            conn.commit()
            cursor.close()
            conn.close()
            print("✅ PostgreSQL 数据表结构初始化/检查成功。")
            break
        except Exception as e:
            print(f"⌛ 等待数据库服务就绪... 错误原因: {e}")
            time.sleep(5)

def start_agent():
    qb = Client(host=QB_HOST, username=QB_USER, password=QB_PASS)
    try:
        qb.auth_log_in()
        print("✅ 成功通过内网网络登录 qBittorrent WebUI！")
    except Exception as e:
        print(f"❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: {e}")
        return

    init_pg_db()
    
    memory_buffer = {}
    last_flush_time = time.time()
    print("🚀 优化版高频采样引擎在服务器后台开始运转...")
    
    while True:
        try:
            current_time = time.time()
            active_torrents = qb.torrents_info(statusFilter="active")
            
            for torrent in active_torrents:
                t_hash = torrent["hash"]
                t_name = torrent["name"]
                
                try:
                    peers_dict = qb.sync.torrent_peers(torrent_hash=t_hash)
                    peers = peers_dict.get("peers", {})
                except Exception:
                    continue
                
                for p_ip_port, p_info in peers.items():
                    ip = p_info.get("ip", p_ip_port.split(":")[0])
                    if p_info.get("uploaded", 0) == 0 and p_info.get("upspeed", 0) == 0 and p_info.get("downloaded", 0) == 0:
                        continue
                    
                    key = (t_hash, ip)
                    if key not in memory_buffer:
                        memory_buffer[key] = {
                            "name": t_name, "country": p_info.get("country", "Unknown"), "client": p_info.get("client", "Unknown"),
                            "speeds": [], "uploaded": 0, "downloaded": 0, "progress": "0%"
                        }
                    memory_buffer[key]["speeds"].append(p_info.get("upspeed", 0) / 1024)
                    memory_buffer[key]["uploaded"] = p_info.get("uploaded", 0) / (1024 * 1024)
                    memory_buffer[key]["downloaded"] = p_info.get("downloaded", 0) / (1024 * 1024)
                    memory_buffer[key]["progress"] = f"{p_info.get('progress', 0) * 100:.1f}%"

            # 每 60 秒触发一次内存向 PostgreSQL 的大批量写入
            if current_time - last_flush_time >= 60:
                minute_str = datetime.now().strftime("%Y-%m-%d %H:%M:00")
                if memory_buffer:
                    conn = psycopg2.connect(**DB_CONFIG)
                    cursor = conn.cursor()
                    insert_data = []
                    for (t_hash, ip), info in memory_buffer.items():
                        speeds = info["speeds"]
                        avg_speed = sum(speeds) / len(speeds) if speeds else 0
                        max_speed = max(speeds) if speeds else 0
                        insert_data.append((
                            minute_str, t_hash, info["name"], ip, info["country"], info["client"],
                            info["uploaded"], info["downloaded"], round(avg_speed, 2), round(max_speed, 2), info["progress"]
                        ))
                    
                    cursor.executemany('''
                        INSERT INTO peer_minute_stats 
                        (timestamp, torrent_hash, torrent_name, ip, country, client, uploaded_mb, downloaded_mb, avg_upspeed_kbs, max_upspeed_kbs, peer_progress)
                        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                    ''', insert_data)
                    conn.commit()
                    cursor.close()
                    conn.close()
                    print(f"💾 [{datetime.now().strftime('%H:%M:%S')}] 成功合并写入 {len(insert_data)} 条活跃 Peer 记录至 PostgreSQL。")
                
                memory_buffer.clear()
                last_flush_time = current_time
                
        except Exception as e:
            print(f"⚠️ Agent 运行期间捕获异常: {e}")
            time.sleep(5)
            
        time.sleep(1)

if __name__ == "__main__":
    start_agent()
```

## 6. 启动 docker

这里比较简单，上述文件完成好部署后，直接在命令行中输入如下命令：

```bash
# 1. 停止并移除原有的旧 qb 容器（数据不会丢，因为挂载在 /opt/qb）
docker compose down

# 2. 自动构建新 Agent 镜像，并全量拉起整套微服务架构
docker compose up -d --build
```

我这里没有挂NPM代理，所以数据取回直接使用`IP：5432`链接数据可即可。

如果你有域名，请自行绑定域名。

如果全部完成，你会看到如下输出：

```bash
[+] Running 5/5
 ✔ qb-monitor-agent            Built                                                  0.0s 
 ✔ Network qb_qb_net_v6        Created                                                0.1s 
 ✔ Container qb-postgres       Started                                                0.5s 
 ✔ Container qbittorrent       Started                                                0.5s 
 ✔ Container qb-monitor-agent  Started                                                0.7s 
root@UK-GCD:/opt/qb#
```

这时候使用命令查看 `docker logs`：

```bash
docker compose logs -f qb-monitor-agent
```

如果看到这样的失败：

```bash
qb-monitor-agent  | ❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: 
qb-monitor-agent  | ❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: 
qb-monitor-agent  | ❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: 
qb-monitor-agent  | ❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: 
qb-monitor-agent  | ❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: 
qb-monitor-agent  | ❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: 因為多次驗證失敗，您的 IP 位址已經被封鎖。
qb-monitor-agent  | ❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: 因為多次驗證失敗，您的 IP 位址已經被封鎖。
qb-monitor-agent  | ❌ 无法登录 WebUI，请确认用户名和密码是否正确。错误: 因為多次驗證失敗，您的 IP 位址已經被封鎖。
root@UK-GCD:/opt/qb#
```

**说明你忘记修改 WEBUI的登陆密码！！！！**

「给我回去把自己的qB密码填上去，而不是只会复制粘贴我的代码！」

然后修改好再次跑`docker compose logs -f qb-monitor-agent`，你会看到如下的输出：

```bash
root@UK-GCD:/opt/qb# docker compose logs -f qb-monitor-agent
qb-monitor-agent  | ✅ 成功通过内网网络登录 qBittorrent WebUI！
qb-monitor-agent  | ✅ PostgreSQL 数据表结构初始化/检查成功。
qb-monitor-agent  | 🚀 优化版高频采样引擎在服务器后台开始运转...
```

OK，完成了。

## 7. 数据导出与本地直读优化方案

### 方法一：导出全数据

为了彻底规避 `pg_dump` 导出的 `.sql` 文本文件在本地环境（Mac）通过 Pandas 解析时的语法兼容性问题和繁琐转换，最佳实践是在 **服务器端** 拦截数据流，直接将其封装为 SQLite 二进制文件。

​**服务端一键生成 .db 文件的命令：**​`.db` 在 VPS 的 `/opt/qb` 目录下直接执行以下组合命令：

```bash
docker exec -t qb-postgres pg_dump -U user --data-only --inserts qb_monitor | awk '
BEGIN { print "PRAGMA synchronous = OFF; PRAGMA journal_mode = MEMORY; BEGIN TRANSACTION;" }
/^INSERT INTO/ { gsub("public\\.", ""); print }
END { print "COMMIT;" }
' | sqlite3 qb_monitor_server.db -cmd "CREATE TABLE IF NOT EXISTS peer_minute_stats (id INTEGER PRIMARY KEY, timestamp TEXT, torrent_hash TEXT, torrent_name TEXT, ip TEXT, country TEXT, client TEXT, uploaded_mb REAL, downloaded_mb REAL, avg_upspeed_kbs REAL, max_upspeed_kbs REAL, peer_progress TEXT);"
```

**原理解释：**

1. 让 PostgreSQL 容器以最基础的 `INSERT` 格式输出流数据。
2. 利用 `awk` 动态剥离 `public.` schema 前缀，并注入 SQLite 的内存写入加速指令。
3. 利用服务器原生自带的 `sqlite3` 命令接住数据流，瞬间在当前目录落盘生成 `qb_monitor_server.db` 文件。

执行完毕后，你只需将 `qb_monitor_server.db` 下载回你的 Mac，即可在 Jupyter 中使用 `create_engine("sqlite:///qb_monitor_server.db")` 极速直读并开展高级分析。

注意，如果上述命令执行失败，说明你没有在服务端安装 `sqlite3`，直接使用 `apt install sqlite3`即可。

---

### 方法二：远程调用数据库

当然，对于非得登陆服务器才能下载，我们还可以直接使用 `python` 远程调用数据库，获取相关数据即可，下面这种方法，可以直接获取到全部的 SQL 数据库并导入到本地db文件里面，用于数据分析。

```python
import pandas as pd
from sqlalchemy import create_engine, text
from datetime import datetime
import os
import shutil

# 配置
CONFIG = {
    "REMOTE_URL": "postgresql://user:your_db_secure_password@IP:5432/qb_monitor", 
    # 一定要修改这里（REMOTE_URL）的数据！！！！！！！！！
    "LOCAL_DB": "qb_monitor_local.db",
    "TABLE": "peer_minute_stats",
    "PK_COLUMN": "id"  # 请确保该表有自增主键，如 id
}

def get_last_synced_id(local_engine, table, pk):
    """获取本地数据库中已有的最大 ID"""
    if not os.path.exists(CONFIG["LOCAL_DB"]):
        return 0
    try:
        query = f"SELECT MAX({pk}) FROM {table}"
        with local_engine.connect() as conn:
            result = conn.execute(text(query)).scalar()
            return result if result is not None else 0
    except:
        return 0

def backup_db_file():
    """将当前的 sqlite 文件复制一份带时间戳的备份"""
    timestamp = datetime.now().strftime("%Y%m%d%H%M%S")
    backup_name = f"qb_monitor_local_{timestamp}.db"
    shutil.copy2(CONFIG["LOCAL_DB"], backup_name)
    return backup_name

def sync_data():
    remote_engine = create_engine(CONFIG["REMOTE_URL"])
    local_engine = create_engine(f"sqlite:///{CONFIG['LOCAL_DB']}")
    
    # 1. 获取断点
    last_id = get_last_synced_id(local_engine, CONFIG["TABLE"], CONFIG["PK_COLUMN"])
    print(f"📌 上次同步的最大 ID 为: {last_id}")

    # 2. 增量获取
    print("📥 正在从远程拉取新数据...")
    query = f"SELECT * FROM {CONFIG['TABLE']} WHERE {CONFIG['PK_COLUMN']} > {last_id} ORDER BY {CONFIG['PK_COLUMN']} ASC"
    
    try:
        new_df = pd.read_sql_query(query, con=remote_engine)
        
        if new_df.empty:
            print("✅ 本地数据已是最新。")
        else:
            print(f"📤 发现 {len(new_df)} 条新数据，正在写入本地...")
            new_df.to_sql(CONFIG["TABLE"], con=local_engine, if_exists="append", index=False)
            print(f"✅ 数据同步成功。")

        # 3. 备份 DB 文件
        backup_file = backup_db_file()
        print(f"🎉 大功告成！当前数据库已备份至: {backup_file}")
        print(f"📊 当前数据库大小: {os.path.getsize(CONFIG['LOCAL_DB']) / (1024*1024):.2f} MB")

    except Exception as e:
        print(f"❌ 同步失败: {e}")

if __name__ == "__main__":
    sync_data()
```

这种方法在逻辑上就是最彻底的“增量同步”，它能将数据获取量从“全量（即所有历史记录）”压缩到“仅新增的增量部分”，减少获取部分。

## 8. 结语

通过本指南，我们成功搭建了一套**低耦合、高内聚**的 qBittorrent 流量监控与数据留存系统。整个系统利用 Docker 将全套微服务彻底容器化，既保证了数据采集 Agent 的高频运行（1 秒采样，60 秒批量落盘，最大程度兼顾了数据实时性与 VPS 磁盘寿命），又通过 IPv6 ULA 网络确保了做种效率的最大化。

从盲目做种到“数据说话”，你现在已经掌握了服务器流量的底层细节。无论是为了排查不上传的“吸血雷”，还是为了优化不同 PT 站点的做种策略，这些本地化的 `.db` 文件都将是你开展高级数据透视的最强底气。

## 9. 后续改进方向

这套系统目前已经完美解决了“数据收集”与“离线传输”的痛点，但它依然有进阶的演进空间。以下是推荐的后续优化方向：

### 1. 闭环自动化：从“离线分析”到“实时拦截”

* ​**痛点**​：目前拉黑吸血客户端需要人工分析后手动操作，存在滞后性。
* ​**改进**​：在 `monitor.py` 中直接引入过滤规则（例如：下载量巨大但上传量极低、客户端特征异常等），一旦触发阈值，直接调用 `qbittorrent-api` 的 `qb.transfer.ban_peers()` 接口，实现​**全自动无人值守拉黑**​。

### 2. 可视化升级：引入 Grafana 仪表盘

* ​**痛点**​：每次查看流量必须通过 Jupyter 写代码，不够直观。
* ​**改进**​：由于中心数据库采用了 PostgreSQL，可以直接在 VPS 上通过 Docker 并行拉起一个 Grafana 容器，连接 PostgreSQL 后，配置几张炫酷的实时流量折线图、Peer 国家分布饼图和客户端占比图，实现一网通展。

参考修改 `docker-compose.yaml`：

```yaml
version: '3.8'

services:
  # 1. qBittorrent 容器
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - WEBUI_PORT=8081
    volumes:
      - /opt/qb/config:/config
      - /opt/qb/downloads:/downloads
    ports:
      - 8081:8081
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
    networks:
      - qb_net_v6

  # 2. PostgreSQL 数据库
  postgres:
    image: postgres:15-alpine
    container_name: qb-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=your_db_secure_password
      - POSTGRES_DB=qb_monitor
    ports:
      - '5432:5432'
    volumes:
      - /opt/postgres_data:/var/lib/postgresql/data
    networks:
      - qb_net_v6

  # 3. 监控 Agent
  qb-monitor-agent:
    build: .
    container_name: qb-monitor-agent
    restart: unless-stopped
    environment:
      - TZ=Asia/Shanghai
    depends_on:
      - qbittorrent
      - postgres
    networks:
      - qb_net_v6

  # 4. 新增：Grafana 可视化平台
  grafana:
    image: grafana/grafana:latest
    container_name: qb-grafana
    restart: unless-stopped
    ports:
      - '3000:3000' # 浏览器访问 http://IP:3000
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=xxx# ⚠️设置你的 Grafana 管理员密码
      - GF_INSTALL_PLUGINS=grafana-piechart-panel # 可选：安装饼图插件
    volumes:
      - /opt/qb/grafana_data:/var/lib/grafana # 保持面板和配置不丢失
    depends_on:
      - postgres
    networks:
      - qb_net_v6

networks:
  qb_net_v6:
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: fd00:dead:beef:10::/64
```

### 3. 数据生命周期管理（Data Aging）

* ​**痛点**​：每分钟都在写入高频数据，长期运行会导致 PostgreSQL 的存储空间和索引暴涨。
* ​**改进**​：编写一个简单的数据库定时任务（Cron Job 或 pg\_cron），例如自动清理 30 天以前的明细数据，或者将历史明细数据按天聚合成统计表，只保留核心指标。

### 4. 敏感信息配置剥离

* ​**痛点**​：目前 `monitor.py` 中还残留着部分硬编码的密码和配置。
* ​**改进**​：将 `QB_PASS` 和 `DB_CONFIG` 的密码等敏感配置彻底改成从系统环境变量（`os.environ.get()`）读取，并在 `docker-compose.yml` 中通过 `.env` 文件统一管理，避免代码不小心分享出去导致泄密。

### 5. 反向代理（可选）

* 如果你在使用 Nginx Proxy Manager (NPM)，现在你可以给数据库或事 Grafana 分配一个子域名（如 `stats.yourdomain.com`），并将域名指向宿主机的 `3000` 端口，实现随时随地通过手机查看做种状态。

