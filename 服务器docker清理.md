好的，没问题！我把这次从遇到问题到完美解决的整个**完整复盘、核心命令、技术逻辑以及长效防护**，为你梳理成一份干净、直观、方便随时阅读和复制的结构化文档。

# 服务器系统盘满问题排查与优化指南

在 Linux 服务器（尤其是系统盘较小的 VPS）的日常维护中，系统盘爆满是非常经典的问题。本次针对 GreenCloud 服务器（24GB 系统盘 + 500GB 外挂盘）的排查与处理，完整过程与核心技术规范总结如下：

## 一、 核心问题诊断

当收到服务器磁盘报警时，首要任务是**精准定位罪魁祸首**，而不是盲目重装或删除文件。

### 1. 逐级定位文件体积的命令

寻找大文件的核心逻辑是“由浅入深、顺藤摸瓜”。使用以下命令可以按大小降序排列，列出目录下的一级子目录：

* **查根目录**：寻找是哪个大方向占用了空间
  ```bash
  sudo du -h --max-depth=1 / 2>/dev/null | sort -hr | head -n 15
  ```
* **查特定目录**（例如发现 `/var` 异常）：
  ```bash
  sudo du -h --max-depth=1 /var 2>/dev/null | sort -hr
  ```

### 2. 本次排查结果

通过上述命令层层剥离，最终锁定了高危区域：

* `/var/lib/docker/containers`**/var/lib/docker/containers（占用 7.3G）**：确认为 Docker 容器运行过程中积攒的未限制的日志文件（`*-json.log`）。
* `/var/lib/docker/overlay2`**/var/lib/docker/overlay2（占用 6.2G）**：为当前运行的 11 个 Docker 镜像本体和容器虚拟读写层。属于业务正常占用，无需也不能强制清理。

## 二、 解决方案与核心命令

### 1. 紧急避险：零风险释放 7.3G 空间

对于正在运行的 Docker 容器日志，**绝对不能直接使用 rm 命令删除**`rm`。如果直接删除，由于进程还在占用，空间不仅不会释放，还会变成“僵尸文件”。

**正确做法**：使用 `truncate`（截断）或重定向清空文件内容，在不停止业务的前提下瞬间释放空间：

```bash
sudo find /var/lib/docker/containers/ -name "*-json.log" -exec sh -c 'cat /dev/null > "{}"' \;
```

*注：执行后，containers 目录瞬间降至 316K，系统盘危机解除。*`containers`

### 2. 治本之策：锁死日志上限（长效防御）

如果不加限制，日志迟早会再次塞满系统盘。必须通过 Docker 全局配置文件限制日志大小。

* **操作步骤**： 修改或创建 `/etc/docker/daemon.json` 文件：
  ```bash
  sudo nano /etc/docker/daemon.json
  ```
* **写入防御配置**（限制单个容器日志最大 10M，最多保留 3 个轮转文件，即单个容器日志永远不超 30M）：
  ```json
  {
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "10m",
      "max-file": "3"
    }
  }
  ```
* **配置生效**：重启 Docker 服务（对数据无影响）：
  ```bash
  sudo systemctl restart docker
  ```

## 三、 长远弹性方案：Docker 整体搬迁外挂盘

当未来业务继续扩张，静态镜像（`overlay2`）将 24G 系统盘再次撑满时，无需重装，直接利用 500GB 的闲置外挂盘（假设挂载在 `/data`）进行无缝搬迁：

1. **暂停服务**：`sudo systemctl stop docker`
2. **同步数据**：`sudo rsync -avz /var/lib/docker/ /data/docker/`
3. **更改路径**：在 `/etc/docker/daemon.json` 中追加 `"data-root": "/data/docker"`。
4. **恢复并清理**：重启 Docker 验证路径生效后，清空原系统盘的 `/var/lib/docker/*` 目录。

## 四、 终极备选：重装系统并升级的防踩坑规范

由于现有的 Ubuntu 20.04 LTS 已经结束标准生命周期支持，如果未来出于系统升级目的（如升级到 **Ubuntu 24.04 LTS**）主动选择重装，需严格遵守以下**保护外挂盘**的作业规范：

### 1. GreenCloud 控制面板挂载

在控制面板选择新版 Ubuntu 镜像并点击 **Insert**，修改 **Boot Order** 为 `CD/DVD` 优先，重启进入 VNC 安装界面。

### 2. 自定义分区（存储配置关键分水岭）

在安装向导的 Guided storage configuration 界面，**必须选择 Custom storage layout（自定义存储布局）**`Custom storage layout`。

* **24G 系统盘**：划分为 `/`（根目录），并**勾选 Format（格式化）**。
* **500G 外挂盘**：**绝对不能勾选 Format**！可以直接不分配挂载点（装完系统后再挂载），确保数据无伤。

### 3. 新系统环境复活

重装完成后，将 Boot Order 改回 HDD。进入新系统后：

* 在 `/etc/fstab` 中重新绑定 500G 盘的 UUID 实现开机自动挂载。
* 安装全新 Docker 后，**第一时间**修改 `daemon.json`，把 `data-root` 直接指定在外挂盘路径。
* 这样新拉取的镜像和日志全都在外挂盘上运行，24G 系统盘永远保持空盈，彻底绝后患。
