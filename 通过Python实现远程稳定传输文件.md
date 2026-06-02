# 🚀 跨机器文件传输工具（Jupyter + croc + SSH）部署与优化复盘报告

## 一、 项目背景与架构设计

本项目旨在通过 Python 脚本，在 Jupyter Notebook (`.ipynb`) 环境下自动化实现本地与远程 Linux 服务器之间的大文件安全、高速传输。

* **核心工具**：`croc`（一种基于动态单次暗号、支持断点续传和 P2P 加密的传输工具）。
* **远程连接通道**：`paramiko`（Python 的 SSHv2 协议底层库）。
* **传输后障保障**：`hashlib`（SHA-256 算法对账机制）。

## 二、 踩坑历程：为什么最初的脚本会“瞬间卡死”？

### 1. 现象描述

在最初的版本中，脚本通过 `pty`（伪终端）成功启动了本地的 `croc send`，并精准抓取到了连接暗号。远程服务器也成功接头。但**传输刚刚开始一瞬间，速度瞬间掉零，整个进程毫无征兆地陷入永久死锁（假死）**。

### 2. 原因剖析（真·死锁陷阱）

这是一个经典的\*\*“操作系统的管道缓冲区阻塞（Buffer Block）”\*\*问题：

* **伪终端的局限性**：Linux 系统为伪终端（PTY）分配的缓冲区非常小（通常只有 4KB 左右）。
* **垃圾数据的轰炸**：`croc` 在文件开始传输后，会通过 `stdout` 疯狂、高频地输出带有控制字符的“动态进度条”。
* **代码逻辑缺陷**：旧代码在通过循环拿到 `real_code`（暗号）后，直接执行了 `break`。这就导致 Python 进程停止了对 `master_fd`（伪终端主端）的读取消费。
* **连锁反应**：Python 不读 ➡️ 4KB 缓冲区瞬间被进度条数据塞满 ➡️ 操作系统为了保护内存，强制将本地 `croc` 进程的 `write()` 系统调用挂起（Block） ➡️ 本地不发了，远程自然断开。

## 三、 核心优化思路（如何解决死锁？）

为了彻底解决缓冲区阻塞，并使其完美适配 Jupyter 环境，我们采取了以下重构思路：

1. **引入“异步排气阀”机制（多线程）** 不能在拿到暗号后就对本地 `croc` 不闻不问。我们在 `break` 的同时，瞬间启动一个轻量级的后台守护线程（`drain_pty`）。这个线程就像一个“数据黑洞”，唯一的目标就是死循环读取 `master_fd`，把 `croc` 吐出来的进度条垃圾数据全部抽干排空，确保本地 `croc` 永远不会因为管道打满而窒息。
2. **Jupyter 输出缓冲流优化** Jupyter 的输出框如果直接接收远程传回的 `\r`（回车符）动画，极易导致浏览器前端内存崩溃。优化中采用了 `sys.stdout.write()` 和 `flush()` 的组合，配合标准 SSH 通道的 `get_pty=True` 参数，让远程输出在 Notebook 中平滑流式展显，不卡顿。
3. **双端数据绝对一致性（哈希对账）** 传输结束后，不盲目信任 `croc` 的返回状态。利用 Python 在本地分块（流式）读取大文件计算 SHA-256，同时驱动远程执行 `sha256sum`，双端通过十六进制摘要进行严格对账，防止大文件发生比特位翻转或截断。

## 四、 完整的 Jupyter 部署代码

请将以下代码按顺序部署在 Jupyter Notebook 的三个 Cell 中：

### Cell 1: 依赖导入与配置参数

```python
import subprocess
import os
import re
import time
import pty
import paramiko
import threading
import sys
import hashlib  #如果缺少请执行安装

# ==================== [ 用户配置区域 ] ====================
SSH_HOST = "IP"       # 你的服务端（服务器）IP地址
SSH_PORT = 22
SSH_USER = "root"
KEY_PATH = "/Users/"  # 你的 SSH 私钥本地路径，如果是密码请自行修改

REMOTE_DIR = "/opt/qb/downloads"                # 远程保存目录
LOCAL_FILE = "09.mp4"                          # 本地待传文件
# ========================================================
```

### 1: 核心组件定义

首先是这个，持续读取伪终端输出，防止出现缓冲区写满导致 croc 死锁

```python
def drain_pty(fd):
    try:
        while True:
            data = os.read(fd, 1024)
            if not data:
                break
    except OSError:
        pass # 进程结束后 fd 被关闭，正常退
```

然后是对本地文件进行croc传输的准备，获取好暗号并维持本地传输。

```python
def run_local_croc_properly():
    """启动本地 croc 并持续维持其生命"""
    if not os.path.exists(LOCAL_FILE):
        raise FileNotFoundError(f"本地未找到文件: {LOCAL_FILE}")

    print("🚀 正在通过伪终端启动本地 croc send...")
    master_fd, slave_fd = pty.openpty()
    
    cmd = ["croc", "--remember", "send", LOCAL_FILE]
    process = subprocess.Popen(
        cmd, stdin=slave_fd, stdout=slave_fd, stderr=slave_fd,
        text=True, close_fds=True
    )
    os.close(slave_fd)
    
    real_code = None
    output_buffer = ""
    start_time = time.time()
    
    print("⏳ 正在实时解析 croc 吐出的真实暗号...")
    while time.time() - start_time < 15: # 增加到 15 秒防网络波动
        try:
            char = os.read(master_fd, 1).decode('utf-8', errors='ignore')
            if not char: break
            output_buffer += char
            
            if char == '\n':
                line = output_buffer.strip()
                output_buffer = ""
                # 抓取真正的暗号
                match = re.search(r'croc\s+([a-zA-Z0-9-]+)', line)
                if match and "send" not in line and "croc-stdin" not in line:
                    real_code = match.group(1)
                    print(f"💥 【核心突破】成功抓取到真实暗号: {real_code}")
                    break
        except Exception:
            break

    # 【关键修复】拿到 code 后，启动排气线程，把后续的进度条垃圾全吸走！
    if real_code:
        t = threading.Thread(target=drain_pty, args=(master_fd,))
        t.daemon = True
        t.start()
        
    return process, master_fd, real_code
```

接着是传输函数，通过ssh链接服务端，然后自动输入传输指令。
`cd {REMOTE_DIR} && CROC_SECRET={code} croc --yes --remember --overwrite receive`

```python
def ssh_execute_receive(code):
    """远程连接并安全接收进度"""
    print(f"🔗 正在通过密钥连接服务器 {SSH_HOST}...")
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    
    try:
        private_key = paramiko.RSAKey.from_private_key_file(KEY_PATH)
        ssh.connect(hostname=SSH_HOST, port=SSH_PORT, username=SSH_USER, pkey=private_key, timeout=10)
        print("✅ 【SSH 连接成功】正在服务端激活 croc receive...")

        # 优化：使用原生语法替代环境变量，开启 get_pty 让 croc 认为自己在真实终端
        remote_cmd = f"cd {REMOTE_DIR} && CROC_SECRET={code} croc --yes --remember --overwrite receive"
        stdin, stdout, stderr = ssh.exec_command(remote_cmd, get_pty=True)
        
        channel = stdout.channel
        print("📡 【传输隧道已打通】服务端开始下载...\n")
        
        # 持续解析远程进度
        while not channel.exit_status_ready() or channel.recv_ready():
            if channel.recv_ready():
                output = channel.recv(1024).decode('utf-8', errors='ignore')
                # Notebook 优化：原样输出但依靠 sys.stdout 防止换行崩溃
                sys.stdout.write(output)
                sys.stdout.flush()
                
                if "100%" in output or "complete" in output.lower():
                    print("\n\n🎉 【大功告成】服务器端接收 100% 完成！")
                    break
            time.sleep(0.1) # 降低 CPU 占用

    except Exception as e:
        print(f"\n❌ 【SSH 异常】: {e}")
    finally:
        ssh.close()
```

最后是文件哈希值校验，确保传输文件一致。

```python
def verify_file_integrity():
    """使用 SHA-256 校验本地与服务端文件的一致性"""
    print("\n🔍 开始校验文件完整性 (SHA-256)...")
    
    # 1. 分块读取计算本地哈希（防止大文件撑爆内存）
    sha256_hash = hashlib.sha256()
    try:
        with open(LOCAL_FILE, "rb") as f:
            for byte_block in iter(lambda: f.read(4096), b""):
                sha256_hash.update(byte_block)
        local_hash = sha256_hash.hexdigest()
        print(f"💻 本地文件哈希: {local_hash}")
    except Exception as e:
        print(f"❌ 读取本地文件失败: {e}")
        return False

    # 2. 通过 SSH 获取远程文件哈希
    print(f"🔗 正在连接服务器计算远程哈希...")
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        private_key = paramiko.RSAKey.from_private_key_file(KEY_PATH)
        ssh.connect(hostname=SSH_HOST, port=SSH_PORT, username=SSH_USER, pkey=private_key, timeout=10)
        
        # 提取文件名，并构造远程路径
        file_name = os.path.basename(LOCAL_FILE)
        remote_file_path = f"{REMOTE_DIR}/{file_name}"
        
        # 让服务器执行 sha256sum 命令
        remote_cmd = f"sha256sum '{remote_file_path}'"
        stdin, stdout, stderr = ssh.exec_command(remote_cmd)
        
        output = stdout.read().decode('utf-8').strip()
        err_output = stderr.read().decode('utf-8').strip()
        
        if err_output and not output:
            print(f"❌ 远程计算哈希出错: {err_output}")
            return False
            
        # sha256sum 的输出格式为: <hash>  <filename>，我们切片取第一个元素
        remote_hash = output.split()[0]
        print(f"☁️  远程文件哈希: {remote_hash}")
        
        # 3. 终极比对
        if local_hash == remote_hash:
            print("\n🌟 【校验通过】两端文件 SHA-256 完全一致，传输完美无损！")
            return True
        else:
            print("\n🚨 【校验失败】文件哈希不一致，文件可能在传输中损坏或未下载完整！")
            return False
            
    except Exception as e:
        print(f"❌ SSH 获取远程哈希异常: {e}")
        return False
    finally:
        ssh.close()
```

我们的函数已经准备好了，那么接下来就可以直接使用了。

### 最后: 调度总控入口

```python
# Cell 3: 启动主流程 (更新版)
local_proc = None
master_fd = None

try:
    # 1. 启动并获取暗号
    local_proc, master_fd, code = run_local_croc_properly()
    
    if code:
        # 2. 传递给远程端下载
        ssh_execute_receive(code)
        
        # 3. 传输完成后，立刻进行完整性校验 <--- 新增这行
        verify_file_integrity()
        
    else:
        print("❌ 【错误】未能获取到正确的随机暗号，请确认本地网络是否正常。")

except KeyboardInterrupt:
    print("\n⚠️ [提示] 用户手动中断。")
finally:
    # 4. 完美扫尾，释放资源
    print("\n🧹 正在清理本地进程...")
    if master_fd:
        try: os.close(master_fd) 
        except: pass
    if local_proc and local_proc.poll() is None:
        local_proc.terminate()
        print("🔒 本地守护进程已安全关闭。")
```

本文以本组制作的为例，上传最后结果如下：

```bash
🚀 正在通过伪终端启动本地 croc send... 
⏳ 正在实时解析 croc 吐出的真实暗号... 
💥 【核心突破】成功抓取到真实暗号: 6233-quest-soda-bonanza 
🔗 正在通过密钥连接服务器 96... 
✅ 【SSH 连接成功】正在服务端激活 croc receive... 
📡 【传输隧道已打通】服务端开始下载...  

Receiving '09.mp4' (401.6 MB)   
Receiving (<-xxx.xxx.xxx.xxx:xxxx)  
09.mp4 100% |████████████████████| (421/421 MB, 7.4 MB/s)                  

🎉 【大功告成】服务器端接收 100% 完成！ 

 🔍 开始校验文件完整性 (SHA-256)... 💻 本地文件哈希: 27d83984e3439b174b5707edf0ce1a3cc590680d3b4ce386c6721a176c3e295c 

🔗 正在连接服务器计算远程哈希... ☁️  远程文件哈希: 27d83984e3439b174b5707edf0ce1a3cc590680d3b4ce386c6721a176c3e295c  

🌟 【校验通过】两端文件 SHA-256 完全一致，传输完美无损！  
🧹 正在清理本地进程...
```

## 五、 面向未来的生产级改进建议（他人部署参考）

如果此脚本要在团队内大面积推广，或者放入定时任务（如 Jenkins、Crontab）中作为生产级部署，建议做以下改进：

1. **凭据安全解耦（强烈推荐）**
   * **现状**：私钥路径和服务器 IP 硬编码在脚本头部。
   * **改进**：使用环境变量（`os.getenv('SSH_KEY_PATH')`）或者 `.env` 配置文件来读取凭据，防止无意间将核心资产代码推送到公共 Git 仓库。
2. **多文件或目录传输支持**
   * **建议**：修改参数支持接收 `List`，或者在调用 `croc` 之前先利用 `tar` 命令在本地将目录打包压缩，传输后再在远程解压，能大幅提高多碎文件传输时的吞吐量。
3. **更健壮的进程断电清理（Context Manager）**
   * **建议**：将 `Cell 3` 的 `try...finally` 包装为 Python 的上下文管理器（`with CrocTunnel() as tunnel:`）。即便 Jupyter 核心（Kernel）发生 Crash，也能最大限度保证底层的系统的 `pty` 文件句柄被正确关闭，避免产生僵尸进程。
4. **远程环境前置依赖检查**
   * **建议**：在执行 `croc` 接收前，先在 SSH 中跑一次 `which croc && which sha256sum`，若服务器未安装这两个关键工具，直接提前熔断报错，给出人性化的安装提示。

