---
title: PanSou 网盘搜索部署手册
---

# PanSou 网盘搜索部署手册



**PanSou 本地部署 · 完整操作手册**

> 适用系统：Windows 10/11 + WSL 2 + Docker Desktop
> 部署对象：`ghcr.io/fish2018/pansou-web`（前后端集成版，带网页界面）
> 默认访问地址：`http://127.0.0.1:8080`
> 最后整理日期：2026-09-26

---

### 📖 使用须知

本手册按「从零到能用到能删」的完整生命周期编写。你可以从头读一遍建立认知，之后日常只需查阅第八节「常用命令速查卡」和第六节「排错速查表」。

动手前需要明确两点：

1. **PanSou 是一个网盘资源聚合搜索工具。** 它搜到的内容可能涉及版权问题，本手册只讲技术部署，搜到什么、怎么用，请自行把握分寸。
2. **程序跑在你自己的电脑上。** 它不是网页服务，不需要联网也能打开界面；但要真正搜出结果，必须联网（因为它要实时去外部频道和插件抓取链接）。

---

### 第一章 认知准备：先搞懂你在装什么

#### 1.1 PanSou 是什么

一个用 **Go 语言**编写的网盘链接聚合搜索后端，官方提供两种形态：

| 镜像 | 说明 | 默认端口 | 推荐度 |
|---|---|---|---|
| `pansou-web` | 前后端集成版，带网页界面，开箱即用 | 容器内 80（Nginx） | ⭐⭐⭐ 新手选这个 |
| `pansou` | 纯后端 API 版，只返回 JSON，无界面 | 8888 | 给开发者用 |

本手册全程使用 `pansou-web`。

#### 1.2 三个必须分清的名词

| 名词 | 比喻 | 准确含义 |
|---|---|---|
| **Docker** | 集装箱码头管理系统 | 容器化平台，负责下载、运行、管理容器 |
| **镜像（Image）** | 一张光盘 / 安装包 | 静止的模板，打包了程序+环境+配置 |
| **容器（Container）** | 用光盘装好的一台正在运行的机器 | 镜像的运行实例，是"活着的" |

关系：**镜像 →（run）→ 容器**。同一个镜像可以启动多个容器。

#### 1.3 为什么是 WSL 而不是 PowerShell

- **WSL** 不是一个终端，它是一台住在 Windows 里的完整 Linux 系统（WSL 2 = 轻量虚拟机 + 真实 Linux 内核）。
- **PowerShell** 是 Windows 的 Shell，脚下是 Windows NT 内核，只能跑 `.exe`。
- Docker 引擎全世界只有一份，住在专门的 `docker-desktop` 发行版里；而 `docker` 命令行客户端有两份（Linux 版在 WSL 里，`.exe` 在 Windows 里），**连的是同一个引擎**。

所以：**以后所有部署操作，统一在 WSL 里做。** 网上 99% 的教程都是 Linux 命令，原样照抄即可。

---

### 第二章 环境准备（一次性，装完永久有效）

#### 2.1 检查虚拟化是否开启

`Ctrl + Shift + Esc` → 任务管理器 →「性能」→「CPU」→ 看右下角：

- `虚拟化：已启用` → 通过 ✅
- `虚拟化：已禁用` → 重启进 BIOS，找 `Virtualization Technology` / `SVM Mode`，改为 Enabled

#### 2.2 安装 WSL 2

以**管理员身份**打开 PowerShell，执行：

```powershell
wsl --install
```

完成后**必须重启电脑**。重启后验证：

```powershell
wsl -l -v
```

看到 `Ubuntu` 且 VERSION 为 `2` 即成功。

#### 2.3 安装 Docker Desktop

1. 官网下载 `docker.com/products/docker-desktop`，选 Windows 版
2. 安装时勾选：✅ **Use WSL 2 instead of Hyper-V**；❌ 不要勾 "Allow Windows Containers"
3. 装完按提示重启或注销
4. 从开始菜单打开 Docker Desktop，接受许可，等右下角鲸鱼图标 🐳 稳定（首次初始化 1~2 分钟）

#### 2.4 验证安装

在 WSL 里执行：

```bash
docker --version
docker compose version
docker run hello-world
```

前两条输出版本号，第三条看到 `Hello from Docker! This message shows that your installation appears to be working correctly.` 即为成功。

#### 2.5 配置镜像加速源（国内网络必做，否则拉取 ghcr.io 会超时）

方法 A：图形界面
右键鲸鱼图标 → Settings → Docker Engine → 在 JSON 中加入：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live"
  ],
  ...其他原有配置不动...
}
```
点 Apply & restart。

方法 B：直接改文件（找不到图标时用这个）
按 `Win + R`，输入 `%USERPROFILE%\.docker\daemon.json`，若不存在则新建，内容为：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live"
  ]
}
```
保存后在任务管理器结束 Docker Desktop 进程，再重新打开。

> ⚠️ 国内加速源经常失效，拉不动时随时更换可用源。

#### 2.6 确认 Docker Desktop 开机自启

任务管理器 →「启动应用」→ 找 **Docker Desktop** → 确保状态为「已启用」。
（也可在 Docker Settings → General → 勾选 Start Docker Desktop when you log in）

---

### 第三章 部署 PanSou（核心步骤）

#### 3.1 启动命令

在 WSL 中执行：

```bash
docker run -d --name pansou --restart unless-stopped -p 8080:80 ghcr.io/fish2018/pansou-web:latest
```

> 💡 建议第一次就带上 `--restart unless-stopped`，省去后续补设的步骤。

首次运行会下载几百 MB，耐心等待进度条走完。成功后返回一串容器 ID（长串字母数字）。

#### 3.2 命令逐词拆解

| 片段 | 作用 |
|---|---|
| `docker` | 调用 Docker 程序 |
| `run` | 新建并启动容器（= pull + create + start 三合一） |
| `-d` | detached，后台运行，不霸占终端 |
| `--name pansou` | 给容器起名，方便后续管理 |
| `--restart unless-stopped` | 开机/Docker 启动时自动跟随启动；若你手动 stop 过则保持停止 |
| `-p 8080:80` | 端口映射：**电脑的 8080 → 容器内的 80** |
| `ghcr.io/.../pansou-web:latest` | 镜像地址（仓库域名/作者/镜像名:标签） |

**重点理解 `-p 8080:80`**：容器有独立的门牌号系统。80 是容器内部 Nginx 的端口，外界看不见；8080 是你电脑上开的窗户。因此访问地址是 `http://127.0.0.1:8080`，不是 `:80`。这也解释了为什么它和你之前 DTK 的 8000 端口互不冲突——各走各的门。

#### 3.3 三步验证

**① 看容器状态**

```bash
docker ps
```

找到 `pansou` 一行：`STATUS` 应为 `Up X minutes`；`PORTS` 应显示 `0.0.0.0:8080->80/tcp`（箭头方向不能反）。

**② 看日志**

```bash
docker logs pansou
```

末尾出现 `server started` / `listening on :8888` / nginx 启动成功字样即为正常。

**③ 浏览器访问**

打开 `http://127.0.0.1:8080`，看到带搜索框的 PanSou 页面即部署成功。

可选的程序员式验证：

```bash
curl http://127.0.0.1:8080/api/health
```

返回 JSON 健康检查信息。

---

### 第四章 日常管理（每天会用到的部分）

#### 4.1 常用操作

```bash
docker ps                  # 查看运行中的容器
docker ps -a               # 查看全部（含已停止的）
docker start pansou        # 启动
docker stop pansou         # 停止（保留容器和数据）
docker restart pansou      # 重启
docker rm pansou           # 删除容器（必须先 stop）
docker logs pansou         # 查看日志
docker logs -f pansou      # 实时滚动日志（Ctrl+C 退出）
docker images              # 查看本地有哪些镜像
docker rmi 镜像名          # 删除镜像，释放磁盘空间
```

#### 4.2 开机自启（如果创建时忘了加 --restart）

```bash
docker update --restart unless-stopped pansou
```

验证是否写入：

```bash
docker inspect pansou --format '{{.HostConfig.RestartPolicy.Name}}'
```

输出 `unless-stopped` 即成功。

> 三种策略对比：`no`（默认，停了就停了）／`always`（永远自动起，哪怕你手动 stop 过）／`unless-stopped`（尊重你的手动操作，最推荐）。

#### 4.3 重要时间差

Docker Desktop 从启动到引擎就绪通常需 **30~60 秒**。重启电脑后不要立刻开浏览器，先等一分钟或执行 `docker ps` 确认 STATUS 为 `Up` 再访问。一开机就点遇到"连接被拒绝"属于正常现象，并非故障。

#### 4.4 想彻底零点击：做成开机任务

方法 A：桌面快捷脚本
新建文本文件，粘贴以下内容，保存为 `启动PanSou.bat`（后缀必须是 .bat）：

```bat
@echo off
echo 正在启动 PanSou，请稍候...
wsl -d Ubuntu -e docker start pansou
timeout /t 5 >nul
start http://127.0.0.1:8080
```

双击即可，窗口闪一下自动关闭属正常现象。注意 `wsl -d Ubuntu -e` 执行完即刻退出，无需提前打开 WSL 窗口。

方法 B：注册为开机任务
任务计划程序 → 创建基本任务 → 名称填 `启动PanSou` → 触发器选「当我登录时」→ 操作选「启动程序」→ 程序填 `wsl`，参数填 `-d Ubuntu -e docker start pansou` → 完成。建议再双击该任务勾选「使用最高权限运行」。

#### 4.5 两套文件系统互访（实用技巧）

```bash
# WSL 里访问 Windows 桌面
cd /mnt/c/Users/你的用户名/Desktop && ls
```

Windows 文件资源管理器地址栏输入 `\\wsl$\Ubuntu\home\你的用户名` 可查看 Linux 内文件。

---

### 第五章 排错速查表

| 现象 | 原因 | 解决办法 |
|---|---|---|
| `docker` 不是内部或外部命令（PowerShell 里） | Windows PATH 没写进去 / 老窗口未继承新 PATH | 关窗重开；或手动加路径：`[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Program Files\Docker\Docker\resources\bin", [EnvironmentVariableTarget]::User)` |
| `Conflict. The container name "/pansou" is already in use` | 同名容器残留 | `docker rm pansou` 后重跑 |
| `Ports are not available ... bind: address already in use` | 8080 被占 | 换端口重来：`-p 8081:80`；查占用：`netstat -ano \| findstr 8080` |
| 卡在 Pulling / timeout / connection refused | ghcr.io 连不上 | 确认加速源生效并重启 Docker；换源；或走离线加载 Plan B |
| 容器 Up 但网页打不开 / 502 | 服务还没准备好 | 等 30 秒再刷新；否则 `docker logs pansou` 看报错 |
| 搜不出任何结果 | 没联网 / TG 频道与插件未配置 / 需要代理 | 检查网络；进入「搜索配置」页添加频道；配置 SOCKS5/HTTP 代理 |
| `error during connect ... Is the docker daemon running?` | Docker 引擎没启动 | 手动打开 Docker Desktop 并等待初始化 |
| Docker Desktop 一直卡在 Starting | WSL 异常 / 虚拟化未开 / 内存不足 | `wsl --update`；检查 BIOS 虚拟化；或用 `.wslconfig` 限制内存 |
| 托盘找不到鲸鱼图标 | 藏在折叠区 / 新版默认不显示 | 点任务栏 `^` 小箭头；或直接开 Docker Desktop 主窗口点右上角 ⚙️ 齿轮进设置 |

**Plan B：离线加载镜像（绕过网络问题）**
若有 `pansou.tar` 离线包：

```bash
docker load -i pansou.tar
docker run -d --name pansou --restart unless-stopped -p 8080:80 ghcr.io/fish2018/pansou-web:latest
```

---

### 第六章 卸载与清理（未来一定会用到）

#### 6.1 只想暂时停用

```bash
docker stop pansou        # 停掉，配置和数据保留
docker start pansou       # 随时恢复
```

#### 6.2 彻底删除 PanSou

按顺序执行，不可颠倒：

```bash
# 1. 停止容器
docker stop pansou

# 2. 删除容器
docker rm pansou

# 3. 删除镜像（释放几个 GB 空间）
docker rmi ghcr.io/fish2018/pansou-web:latest

# 4. （可选）清理 dangling 无用残留
docker system prune
```

> ⚠️ 关键提醒：`pansou-web` 集成版**没有挂载数据卷（volume）**，因此删掉容器等于**配置全部重置**（搜索源、插件设置等都会丢失）。若想长期保留配置，重建时应加 `-v` 挂载目录，例如 `-v pansou-data:/app/data`（具体路径需参照该项目最新版文档确认）。

#### 6.3 连带卸载整个 Docker 环境

1. Windows 设置 → 应用 → 卸载 Docker Desktop
2. 删除残留目录：`%USERPROFILE%\.docker\`
3. 若要连 WSL 一起清：PowerShell 管理员执行 `wsl --unregister docker-desktop` 和 `wsl --unregister Ubuntu`（⚠️ Ubuntu 里你自己装的东西会一并消失）
4. 若要彻底关掉 WSL：`wsl --shutdown`

---

### 第七章 进阶方向（想深入时再看）

1. **配搜索源让它真能搜到东西**：进入页面「搜索配置」，添加 Telegram 频道、启用/禁用插件、配置代理。能力来自外部频道和插件，不在程序本身。
2. **改用纯后端版**：`ghcr.io/fish2018/pansou`，需手动配 `-e PANSOU_PORT=8888 -p 8888:8888`，自己接前端或调 API。
3. **走源码编译路线**：装 Go 1.18+ → `git clone https://github.com/fish2018/pansou.git` → `go build` → `./pansou`。直观感受没有 Docker 黑盒子时要干多少活。
4. **升级版本**：`docker pull ghcr.io/fish2018/pansou-web:latest` → `docker stop pansou` → `docker rm pansou` → 重新 `docker run`（注意 restart 策略要重写）。
5. **让局域网其他设备访问**：把 `-p 8080:80` 改为 `-p 0.0.0.0:8080:80`，并用本机局域网 IP 访问（注意安全风险）。

---

### 第八章 常用命令速查卡（打印贴桌上）

```bash
# —— 启动类 ——
docker start pansou                  # 唤醒
docker ps                            # 看活着没
docker logs -f pansou                # 实时看日志

# —— 部署类 ——
docker run -d --name pansou --restart unless-stopped -p 8080:80 ghcr.io/fish2018/pansou-web:latest
docker update --restart unless-stopped pansou     # 补设自启

# —— 排错类 ——
docker ps -a                         # 含已停止的
docker logs pansou                   # 看报错
docker inspect pansou --format '{{.HostConfig.RestartPolicy.Name}}'   # 查自启策略

# —— 清理类 ——
docker stop pansou && docker rm pansou
docker rmi ghcr.io/fish2018/pansou-web:latest
docker system prune                  # 清残留
```

---

### 第九章 核心认知总结（这条主线值得记住）

同一份开源代码，可以被任何人部署在任何地方：

- 你部署在自己电脑 → `127.0.0.1:8080`，私有、可控、仅本机可见、关机就停
- 别人部署在公网服务器 → `so.252035.xyz`，全网可访问、7×24 在线、但配置和数据在他手里

Docker 的价值在于把复杂的环境配置打包成黑盒子，用户只需管一个端口。全程无需安装 Go、Node.js 或配置环境变量。

而 WSL 的存在，是因为 Docker 引擎依赖 Linux 内核特性（cgroups/namespace），Windows 上必须借一层真实的 Linux 内核来承载它。

---
