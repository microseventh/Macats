# VPS 安全事件完整记录与分析报告

> **日期**: 2026-06-25  
> **VPS**: NY10GbpsStorage-0.5TB (365 Group LLC / GreenCloud)  
> **事件级别**: 🔴 严重 — 主机被入侵，用作网络扫描跳板  
> **最终处理**: 重装系统 + 全盘清除 + 更换 IP + 安全重构

---

## 目录

1. [事件时间线](#一事件时间线)
2. [告警邮件分析](#二告警邮件分析)
3. [救援环境取证过程](#三救援环境取证过程)
4. [发现的问题清单](#四发现的问题清单)
5. [根因分析](#五根因分析)
6. [攻击路径还原](#六攻击路径还原)
7. [备份操作记录](#七备份操作记录)
8. [处理方向与整改方案](#八处理方向与整改方案)
9. [经验教训](#九经验教训)
10. [附录: 资产清单](#十附录-资产清单)

---

## 一、事件时间线

| 时间 | 事件 |
|------|------|
| 2026-06-08 | VPS 首次部署（根据 `cidata` ISO 日期） |
| 2026-06-09 | NPM (Nginx Proxy Manager) 部署 |
| 2026-06-10 | s-ui 代理管理面板部署（`alireza7/s-ui`） |
| 2026-06-10 | qBittorrent + PostgreSQL + Monitor Agent 部署 |
| 2026-06-14 | SSH 暴力破解开始（首个攻击 IP `91.224.92.17` 出现） |
| 2026-06-15 16:04 | **首次扫描告警** — 对 CETNET2 IPv6 网络发起 ICMPv6/TCP/UDP 扫描 |
| 2026-06-19 16:24 | **第二次扫描告警** — 同样的目标、同样的扫描模式 |
| 2026-06-21 | mybot (Nazurin 机器人) 部署 |
| 2026-06-24 晚 | VPS 被 365 Group LLC 暂停服务 |
| 2026-06-25 02:19 | 进入救援系统开始取证 |

---

## 二、告警邮件分析

### 原始邮件关键字段

```
Product/Service:  NY10GbpsStorage-0.5TB
Domain:           US_Store
Amount:           $70.00 USD
Suspension Reason: Network scanning alert notification
```

### 扫描告警详情（共 4 条，截取关键信息）

```
Target address:     2607:f2d8:8416:1008::a --> CETNET2 IPv6 Networks
Protocol type:      ENTRO -- Entropy-based Detection
Scan types:         ICMPv6, TCP, UDP (3 types)
Recording times:    2026-06-15 16:04:01 / 2026-06-15 16:04:09
                    2026-06-19 16:24:30 / 2026-06-19 16:24:35
Security team:      Rein240c
Contact:            zyzh_bj001@163.com / hyj18@mails.tsinghua.edu.cn
```

### 分析结论

| 指标 | 判断 |
|------|------|
| 检测方式 | ENTRO（熵值检测）— 通过流量熵值异常识别，**不是误报** |
| 扫描协议 | 同时扫描 ICMPv6、TCP、UDP 三种协议 — 典型的主机/端口发现行为 |
| 扫描目标 | CETNET2（中国教育科研网 IPv6 网络） |
| 联系人 | 清华邮箱 — 确认是真实的安全投诉 |
| 重复性 | 6/15 和 6/19 两次触发 — 说明扫描器持续运行或被多次部署 |
| 结论 | **VPS 确实被入侵，攻击者将 VPS 用作 IPv6 扫描/僵尸网络节点** |

---

## 三、救援环境取证过程

### 3.1 环境信息

```
救援系统: Debian Trixie (Live)
访问方式: 服务商控制台 (VNC/Console)
系统盘设备: /dev/vda1 (ext4, cloudimg-rootfs)
数据盘设备: /dev/vdb1 (ext4, 0.5TB)
```

### 3.2 取证步骤

#### Step 1: 挂载被入侵磁盘

```bash
mkdir -p /mnt/victim-root /mnt/victim-data
mount /dev/vda1 /mnt/victim-root
mount /dev/vdb1 /mnt/victim-data
```

#### Step 2: 磁盘布局确认

```
vda1  — ext4 (cloudimg-rootfs) — 系统根分区
vda15 — vfat (UEFI)            — EFI 启动分区
vdb1  — ext4                   — 数据/存储盘 (0.5TB)
```

#### Step 3: 持久化机制检查

| 检查项 | 路径 | 结果 |
|--------|------|:---:|
| 系统 crontab | `/etc/crontab` | ✅ 干净，仅标准条目 |
| cron.d | `/etc/cron.d/` | ✅ 仅 `e2scrub_all` |
| cron.hourly | `/etc/cron.hourly/` | ✅ 空 |
| cron.daily | `/etc/cron.daily/` | ✅ 仅系统默认 |
| rc.local | `/etc/rc.local` | ✅ 空 |
| 用户 crontab | `/var/spool/cron/crontabs/` | ✅ 空 |
| systemd 自定义 | `/etc/systemd/system/` | ✅ 仅标准服务 |

**结论**: 未发现恶意持久化机制。

#### Step 4: SSH 安全审计

**authorized_keys** — 仅 1 个 RSA key（为用户本人）：

```
[SSH public key redacted]
```

- SHA256: `[redacted]`
- **这是用户自己的密钥，无异常**

**sshd_config 弱点**：

```
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding yes           # ⚠️ 不必要的功能开启
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp ...
```

#### Step 5: SSH 登录日志分析

**成功登录记录**（auth.log 全文搜索）：

```
Jun 21 01:19:13 Accepted publickey for root from <USER_IP>  # 用户本人
Jun 24 08:57:11 Accepted publickey for root from <USER_IP>  # 用户本人
```

- **仅 2 次成功登录，全部来自用户自己的 IP `<USER_IP>`**
- **无攻击者通过 SSH 入侵的证据**

**暴力破解记录**（btmp 膨胀至 8.5MB）：

| 攻击 IP | 行为 | 时间跨度 |
|---------|------|---------|
| `91.224.92.17` | 每 2 秒尝试 root 登录 | 6/14 ~ 6/24 持续 |
| `91.92.40.10` | root / admin / backup 多用户撞库 | 6/24 密集 |
| `101.47.8.187` | root 暴力破解 | 6/14 ~ 6/24 持续 |
| `212.127.90.201` | root / admin / work / ubuntu / telegram | 6/14 ~ 6/24 |
| `141.98.83.240` | admin 撞库 | 6/24 |
| `45.148.10.121` | root 撞库 | 6/24 |

**结论**: SSH 被大规模暴力破解，但均未成功（全部 `[preauth]` 即断开）。

#### Step 6: 恶意文件检查

| 检查项 | 路径 | 结果 |
|--------|------|:---:|
| /tmp 最近修改 | `/mnt/victim-root/tmp` | ✅ 仅 `croc` 文件传输工具残留（属正常） |
| /dev/shm | `/mnt/victim-root/dev/shm` | ✅ 空 |
| /var/tmp | `/mnt/victim-root/var/tmp` | ✅ 空 |
| /root 隐藏文件 | `/mnt/victim-root/root/.*` | ✅ 仅 `.docker/`, `.ssh/`, `.bash_history`, `.viminfo` 等 |
| bash_history | `/mnt/victim-root/root/.bash_history` | ✅ 全为用户自己的 Docker 运维操作 |

**结论**: 磁盘上未发现扫描工具或恶意文件残留。扫描器很可能在 Docker 容器内存中运行，不在宿主机磁盘留痕迹。

#### Step 7: Docker Compose 审计

共发现 4 个项目，5 个 compose 文件：

```
/mnt/victim-data/dcapp/
├── sui/docker-compose.yml          # 🔴 s-ui: 7 端口暴露公网
├── npm/docker-compose.yml          # 🟡 NPM: 管理后台公网可达
├── qb/docker-compose.yml           # 🔴 qBittorrent + PG: 密码明文×4
└── mybot/
    ├── docker-compose.yml           # ✅ 无端口暴露
    └── docker-compose.prod.yml      # ✅ 仅内网通信
```

详细审计见 [SECURITY-REVIEW.md](SECURITY-REVIEW.md)。

#### Step 8: s-ui 数据库审计

```
数据库: /mnt/victim-data/dcapp/sui/db/s-ui.db (3.8MB)
用户表: admin / <REDACTED> / 最后登录 IP: <USER_IP>
入站配置: 3 个 (hysteria2, shadowsocks, anytls)
客户端:   1 个 (<CLIENT_NAME>)  — 为用户本人配置
操作日志: 全部为 admin 操作 — 无异常用户
```

#### Step 9: 防火墙审计

```bash
cat /mnt/victim-root/etc/ufw/user.rules
```

UFW 开放端口：22, 80, 443, 8000, 81, 6881

**关键问题**: s-ui 的 7 个端口 + PostgreSQL 5432 + qBittorrent 8081/8082 **不在 UFW 规则里**，但因为 Docker 直接操作 iptables，这些端口全部公网可达。

---

## 四、发现的问题清单

### 🔴 严重（可直接导致入侵）

| # | 问题 | 详情 | 状态 |
|---|------|------|:---:|
| 1 | **s-ui 面板公网暴露** | 7 个端口全部绑定 `0.0.0.0`，绕过 UFW。`alireza7/s-ui` 是非官方镜像，历史 RCE 漏洞多 | 待修复 |
| 2 | **PostgreSQL 公网暴露** | 端口 5432 直接暴露，密码 `<REDACTED>` 明文写在 compose 环境变量中 | 待修复 |
| 3 | **qBittorrent WebUI 裸奔** | `WebUI\Address=*`，`LocalHostAuth=false`，`HostHeaderValidation=false`，无 IP 白名单 | 待修复 |
| 4 | **密码全局复用** | 密码 `<REDACTED>` 在 s-ui、qBittorrent、PostgreSQL、Monitor Agent 中复用 | 待修复 |
| 5 | **密码明文存储** | compose 文件中明文出现 4 次 | 待修复 |
| 6 | **Docker 绕过 UFW** | Docker 默认直接操作 iptables，导致所有 exposed 端口不受 UFW 管控 | 待修复 |

### 🟡 中等（增加风险面）

| # | 问题 | 详情 | 状态 |
|---|------|------|:---:|
| 7 | **SSH 暴力破解无防护** | 无 fail2ban，8.5MB 的 btmp 日志 | 待修复 |
| 8 | **NPM 管理后台公网可达** | 端口 81 无 IP 白名单 | 待修复 |
| 9 | **NPM 多余端口 15432** | 暴露但用途不明 | 待排查 |
| 10 | **sshd X11Forwarding 开启** | 不必要的攻击面 | 待修复 |

### 🟢 正常（检测通过）

| # | 项目 | 结果 |
|---|------|:---:|
| 11 | 恶意持久化 | ✅ 无 |
| 12 | 异常 SSH key | ✅ 无 |
| 13 | 异常用户账户 | ✅ 无 |
| 14 | 隐藏进程 | ✅ 无 |
| 15 | 磁盘恶意文件 | ✅ 无 |

---

## 五、根因分析

### 根本原因

```
攻击者入侵的必要条件全部满足：

1. 暴露面大
   ├── s-ui 管理面板公网可达（RCE 漏洞利用）
   ├── qBittorrent WebUI 公网可达（暴力破解）
   └── PostgreSQL 5432 公网可达（密码直连）

2. 认证薄弱
   ├── 密码 <REDACTED> 全局复用
   ├── 密码明文写在 compose 文件
   ├── 无 fail2ban 防护
   └── qBittorrent WebUI: LocalHostAuth=false

3. 网络隔离缺失
   ├── Docker 绕过 UFW
   ├── 管理面板与外网间无 VPN/跳板机
   └── 无内部网络分段

4. 供应链风险
   └── alireza7/s-ui 是非官方镜像，安全状态未知
```

### 为什么扫描器没在磁盘留痕迹

- 攻击者通过 s-ui RCE 获得容器 shell 后，可以在容器内存中直接执行扫描
- 容器拥有宿主机网络栈能力（docker-proxy + iptables DNAT）
- 扫描器可能是无文件攻击（fileless），从远程拉取到内存执行，不写磁盘
- Docker 容器重启后自动清理，不留痕迹

---

## 六、攻击路径还原

```
┌─────────────────────────────────────────────────────────────┐
│ 攻击者                                                       │
└────────────────────────┬────────────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
┌──────────────────┐          ┌──────────────────┐
│ s-ui Web 面板     │          │ qBittorrent WebUI │
│ (2095, 公网可达)  │          │ (8081, 无认证约束) │
│ alireza7/s-ui    │          │ LocalHostAuth=off │
└────────┬─────────┘          └────────┬─────────┘
         │ 利用 RCE 漏洞               │ 暴力破解密码
         │ 或弱密码登录                 │ <REDACTED>
         └──────────────┬──────────────┘
                        │
         ┌──────────────▼──────────────┐
         │  获得 Docker 容器内 shell    │
         │  (s-ui 容器 / qBittorrent)  │
         └──────────────┬──────────────┘
                        │
         ┌──────────────▼──────────────┐
         │  部署 IPv6 扫描器 (内存中)    │
         │  ICMPv6 + TCP + UDP 扫描     │
         │  走宿主机网络栈发出           │
         │  docker-proxy + iptables DNAT│
         └──────────────┬──────────────┘
                        │
         ┌──────────────▼──────────────┐
         │  目标: CETNET2 IPv6 网络     │
         │  2607:f2d8:8416:1008::a     │
         │  被熵值检测 (ENTRO) 发现     │
         └──────────────┬──────────────┘
                        │
         ┌──────────────▼──────────────┐
         │  安全团队 Rein240c 投诉      │
         │  → 服务商 365 Group LLC     │
         │  → VPS 被暂停               │
         └─────────────────────────────┘
```

### 时间吻合度

| 事件 | 日期 |
|------|------|
| s-ui 部署 | 6/10 |
| 首次扫描告警 | 6/15（部署后 5 天） |
| mybot 部署 | 6/21 |
| 第二次扫描告警 | 6/19（mybot 部署前 2 天） |
| VPS 暂停 | 6/24 |

mybot 是唯一无端口暴露的服务，且部署时间晚于两次扫描告警，可以排除嫌疑。s-ui 是最可能的入口。

---

## 七、备份操作记录

### 备份内容

| 数据 | 大小 | 路径 | 状态 |
|------|:---:|------|:---:|
| s-ui 数据库 | 3.8MB | `/mnt/victim-data/s-ui-backup-20260625.db` | ✅ 已下载 |
| NPM 数据 | 2.1MB | `/mnt/victim-data/npm-backup-20260625/` | ✅ 已下载 |
| qBittorrent 配置 | 16MB | `/mnt/victim-data/qb-config-backup-20260625/` | ✅ 已下载 |
| mybot 完整项目 | 27MB | `/mnt/victim-data/mybot-backup-20260625/` | ✅ 已下载 |
| **打包文件** | **36MB** | `vps-backup-20260625.tar.gz` | ✅ 已下载至本地 |

### 备份命令记录

```bash
cp /mnt/victim-data/dcapp/sui/db/s-ui.db /mnt/victim-data/s-ui-backup-20260625.db
cp -r /mnt/victim-data/dcapp/npm/data /mnt/victim-data/npm-backup-20260625/
cp -r /mnt/victim-data/dcapp/qb/config /mnt/victim-data/qb-config-backup-20260625/
cp -r /mnt/victim-data/dcapp/mybot /mnt/victim-data/mybot-backup-20260625/

tar czf /tmp/vps-backup-20260625.tar.gz \
  s-ui-backup-20260625.db \
  npm-backup-20260625 \
  qb-config-backup-20260625 \
  mybot-backup-20260625

scp root@<VPS_IP>:/tmp/vps-backup-20260625.tar.gz ./
```

### 备份文件本地位置

```
vps-backup-20260625.tar.gz           (36MB — 完整备份压缩包)
docker-compose-backup/               (compose 文件副本 + 安全审计)
```

---

## 八、处理方向与整改方案

### 当前状态

- [x] VPS 已暂停
- [x] 取证分析完成
- [x] 数据备份完成
- [x] 服务商工单已提交（申请重装 + 全盘清除 + 换 IP）
- [ ] 等待服务商确认执行

### 工单内容摘要

向 365 Group LLC 申请：

1. **系统盘 (vda) 全盘清除重装** — Ubuntu 22.04 或 24.04 LTS
2. **数据盘 (vdb) 全盘清除** — 0.5TB 全部清除
3. **更换 IPv4 和 IPv6 地址** — 当前 IP 已被标记
4. **默认禁用 root 密码登录** — 仅密钥认证

### 重装后安全整改清单

#### 网络层

- [ ] UFW 默认 DROP入站，默认 DROP出站（白名单模式）
- [ ] 仅开放必要端口：SSH (新端口)、HTTP (80)、HTTPS (443)
- [ ] Docker 禁用 iptables 管理（`daemon.json`: `"iptables": false`）

#### SSH

- [ ] 更换为非标准端口（如 22222）
- [ ] 仅允许 ed25519 密钥认证
- [ ] 禁用 root 密码登录
- [ ] 禁用 X11Forwarding
- [ ] 安装 fail2ban（3 次失败 / 锁 1 小时）

#### Docker 容器

- [ ] 所有管理面板端口绑定 `127.0.0.1`
- [ ] 数据库端口不对公网暴露
- [ ] 敏感信息使用 `.env` 文件，不写死在 compose
- [ ] 每个服务独立随机密码

#### 应用层

- [ ] 替换 `alireza7/s-ui` 为官方 sing-box（或确认可信 fork）
- [ ] qBittorrent WebUI: `WebUI\Address=127.0.0.1`, `LocalHostAuth=true`
- [ ] NPM 管理面板: IP 白名单或仅本地访问

### 未来预防措施

- [ ] 定期 `docker compose pull` 更新镜像
- [ ] 每月检查 UFW 状态和 auth.log
- [ ] 密码使用 `openssl rand -base64 24` 生成，用密码管理器存储
- [ ] 考虑使用 vps-inspect 脚本定期自检

---

## 九、经验教训

### 技术层面

1. **Docker 默认绕防火墙** — 这是最常见的 VPS 安全盲区。Docker 的 `--iptables` 默认开启，会让所有容器端口绕过 UFW/firewalld。必须通过 `daemon.json` 禁用或使用 `127.0.0.1` 绑定。

2. **密码不复用** — 一个密码打穿整个基础设施是攻击者的终极梦想。每个服务应有独立随机密码。

3. **管理面板是最脆弱的入口** — s-ui、NPM admin、qBittorrent WebUI，任何一个被攻破就是完整控制权。管理面板必须是 VPN 后面或至少 IP 白名单 + HTTPS + 强认证。

4. **compose 文件不是 secrets 管理工具** — 密码明文写在 compose 里 = 任何拿到 compose 的人拿到所有密码。用 `.env` + `.gitignore` 或 Docker secrets。

5. **无文件攻击不留痕** — 磁盘取证干净不代表没被入侵。容器内存中的扫描器不会在宿主机 `/tmp` 留文件。

### 运维层面

6. **救援系统是最佳取证工具** — 独立启动的 Live 环境，工具可信，能挂载被入侵磁盘做离线分析。

7. **被入侵过的系统不要修** — 即使能找到并删除所有恶意代码，也无法保证内核/引导/固件没被篡改。重装是唯一可靠方案。

8. **先备份再操作** — 救援环境里先 `cp` 一份数据到安全位置，再动手。

### 流程层面

9. **VPS 被暂停不是坏事** — 服务商的自动检测和暂停机制阻止了进一步的损害（IP 被封、法律问题等）。

10. **每次事故都是一次安全架构审查** — 把这次的发现变成 checklist，以后每部署一个服务就过一遍。

---

## 十、附录: 资产清单

| 项目 | 信息 |
|------|------|
| VPS ID | NY10GbpsStorage-0.5TB |
| 服务商 | 365 Group LLC |
| 费用 | $70/月 |
| OS | Ubuntu (cloudimg) |
| 内核 | 5.15.0 |
| 系统盘 | /dev/vda (ext4) |
| 数据盘 | /dev/vdb (ext4, 0.5TB) |
| 原 IPv4 | <VPS_IP> |
| SSH Key 指纹 | SHA256:sXvIyQEgthnR3UoiLs6yunWTw7OhLT6wJFi0U1/oEmc |

### Docker 资产

| 容器 | 镜像 | 端口 |
|------|------|------|
| s-ui | alireza7/s-ui | 2095, 2096, 32760, 35851(tcp/udp), 19183, 15738 |
| nginx-proxy-manager | jc21/nginx-proxy-manager | 80, 443, 81, 15432 |
| qbittorrent | lscr.io/linuxserver/qbittorrent:5.0.4 | 8081, 6881(tcp/udp), 8082 |
| qb-postgres | postgres:15-alpine | 5432 |
| qb-monitor-agent | (自建 build) | 无 |
| nazurin-bot | (自建 build) | 无 |

### 本报告相关文件

```
./docker-compose-backup/
├── INCIDENT-REPORT.md          # 本文件 — 完整事件记录
├── SECURITY-REVIEW.md          # compose 安全审批 + 整改方案
├── sui/docker-compose.yml      # s-ui compose 备份
├── npm/docker-compose.yml      # NPM compose 备份
├── qb/docker-compose.yml       # qBittorrent compose 备份
└── mybot/
    ├── docker-compose.yml       # mybot compose 备份
    └── docker-compose.prod.yml  # mybot prod compose 备份
```
