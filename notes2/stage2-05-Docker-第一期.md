# Docker 第一期：Docker 简介、安装与镜像基础

> **对应课件**：《Docker零基础到大神进阶-1.8课件》（千山）
> **本篇定位**：Docker 的"从零开始"。先搞清楚 **Docker 是什么、为什么用它**，再**完整装一遍**（在线安装 + 离线安装 + compose 安装），最后讲清 **镜像和容器到底是怎么回事**。
> **环境约定**：CentOS 7.9 或 Rocky Linux 8 都行（本篇命令在 CentOS 7.9 + docker 24.0.9、Rocky 8.10 + docker 26.1.3 上都跑过）。建议虚拟机 **2 核 4G** 起，能联网。
> **三篇的关系**：第一期（概念 + 安装 + 镜像）→ 第二期（命令 + 网络 + 数据卷 + 私有仓库）→ 第三期（Compose + Dockerfile + 监控 + 项目实战）。
> **配套已有笔记**：`stage2-05-Docker.md`（安装速查）、`stage2-05-Docker实验总结.md`（镜像/容器/持久化理论）。

---

## 〇、实战前的环境准备（每次做实验前先过一遍）

> 课件里没写这一节，但**不做会踩坑**（尤其是 SELinux 和防火墙），所以放在最前面。

### 0.1 确认系统版本、资源、网络

```bash
cat /etc/redhat-release      # CentOS 7.9 / Rocky 8.10
uname -r                     # 内核版本
nproc                        # CPU 核数
free -h                      # 内存（建议 >= 2G，跑 LNMP/Zabbix 建议 4G）
df -h /                      # 根分区剩余空间（镜像很占地方，建议 >= 20G）
ping -c 2 www.aliyun.com     # 能不能上网
```

| 项 | 建议值 | 原因 |
| --- | --- | --- |
| 内存 | 2G 起，做 Compose/Zabbix 项目建议 4G | MySQL + PHP + Nginx 三个容器同时跑 |
| CPU | 2 核 | 构建镜像时比较吃 CPU |
| 磁盘 | 20G 以上 | 镜像 + 数据卷 |
| 网络 | 能访问外网 | 拉镜像、装包 |

### 0.2 关闭 SELinux

```bash
getenforce                   # 看当前状态（Enforcing / Permissive / Disabled）
setenforce 0                 # 临时改为 permissive（重启失效）

# 永久关闭
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
grep '^SELINUX=' /etc/selinux/config     # 检验：应输出 SELINUX=disabled
```

> **为什么**：SELinux 会拦住容器访问宿主机目录，报 `Permission denied`，实验阶段先关掉最省事；生产上要用 `setsebool` / `restorecon` 精确放行。

### 0.3 关闭 firewalld（课程做法）

```bash
systemctl stop firewalld
systemctl disable firewalld     # 禁止开机启动
systemctl status firewalld      # 检验：inactive (dead)
```

> **课件原话**：**firewalld 和 docker 会同时管理操作系统的防火墙，容易互相干扰，最好把 firewalld 关掉，关掉之后把 docker 重启一下。**
>
> **但要知道生产上的正确做法**（面试会问）：
> - Docker 启动时会自动往 iptables 的 `nat` 表和 `filter` 表写规则（`DOCKER` 链、`MASQUERADE`、`DNAT`）。**firewalld 重载时会清掉这些规则**，导致已经发布的容器端口突然访问不了。
> - 生产上不是简单关闭，而是：① 用 firewalld 的 `docker` zone（新版支持）；② 或者让 docker 不碰 iptables（`"iptables": false`）再自己写规则；③ 或者由外层防火墙 / 云安全组统一管理。
> - **实验环境：关掉；生产环境：要么用 docker zone，要么统一规划。** 这句能说出来就是加分项。

### 0.4 装好常用工具

```bash
yum install -y vim wget net-tools lsof bash-completion
```

---

## 一、Docker 简介（课件第一章）

### 1.1 Docker 的起源

```text
创始人  ：Solomon Hykes（美国人）
初始项目：DotCloud —— 一个 PaaS 类型的平台
         为了管理平台，团队内部写了一套工具，这套工具就是 Docker 的雏形
关键节点：2013 年，公司决定放弃 DotCloud 平台，
         把公司改名为 Docker Inc.，把 Docker 技术作为核心产品推向市场
```

> **面试怎么用**：被问"什么是 Docker"，不要背历史，用一句话定位 ——「**Docker 是一个用容器技术做应用打包和分发的工具，核心价值是让应用在开发、测试、生产环境里表现得一模一样。**」

### 1.2 什么是 Docker

严格来讲，**Docker 就是 Docker 引擎（Docker Engine）**。现在我们说"Docker"，指的是一整套广义的产品：

| 产品 | 是什么 |
| --- | --- |
| **Docker Desktop** | 面向开发者的桌面产品，主要用于 Windows / macOS |
| **Docker Hub** | 公共的容器镜像仓库（默认的镜像来源） |
| **Docker Engine** | Docker 引擎（真正跑容器的那个东西） |

**Docker Engine 有两个版本**：

| 版本 | 英文名 | 授权 | 技术支持 |
| --- | --- | --- | --- |
| 社区版 | **docker-ce** | 开源、免费 | 约 4 个月 |
| 商业版（企业版） | **docker-ee** | 闭源、收费 | 12 个月 |

> 我们平时装的就是 **docker-ce**。

### 1.3 Docker 到 Moby 项目

```text
2017 年，Docker 公司为了更好地区分社区版和商业版，
把 Docker 开源项目改名为 Moby。

产品线从此清晰：
  Docker CE  （社区版）
  Docker EE  （企业版）
  Docker 集群编排：Docker Swarm 与 Kubernetes 竞争，最后 K8s 胜出
```

> **为什么记住这条**：面试问"K8s 和 Docker 什么关系"时，可以顺口补一句「Docker 公司的 Swarm 在和 K8s 的竞争中落败，K8s 成了容器编排的事实标准」。

### 1.4 商业变革

```text
2019 年，Docker 公司把 Docker Enterprise（企业级业务部门）打包卖掉了，
卖给了一家云服务公司。

那 Docker 公司还剩下什么？
  剩下 Docker Desktop 和 Docker Hub —— 也就是"开发者入口"和"镜像分发入口"。
```

### 1.5 Docker 公司的 logo

那条**顶着集装箱的鲸鱼**：鲸鱼 = 宿主机，集装箱 = 容器。看一眼就记住了容器"一个箱子里装一个应用"的隐喻。

### 1.6 Docker 的核心价值（★★ 三条，必须能背）

| # | 价值 | 说明 |
| --- | --- | --- |
| **1** | **环境一致性** | 开发、测试、生产环境有差异。Docker 通过容器**封装完整的运行环境**，消除了"环境依赖"问题 |
| **2** | **资源高效利用** | 相比虚拟机，容器不是一个完整的操作系统，**资源占用超低、启动速度快** |
| **3** | **部署高效** | 容器可以**快速复制、快速启停**，天然支持自动化 |

> **一句话串起来**：**「Docker 把『应用 + 依赖 + 配置』打包成一个镜像，到哪台机器都一模一样地跑起来；而且因为共享内核，它比虚拟机更轻、更快、更省资源。」**

### 1.7 Docker 的核心组件（★★ 架构图要能画）

**组件清单**：

```text
Docker 客户端       （docker 命令）
Docker 守护进程     （dockerd，注意：podman 是没有守护进程的）
Docker 镜像         （Image，模板）
Docker 容器         （Container，实例）
Docker 仓库         （Registry，存镜像的地方）
Dockerfile          （描述"怎么构建镜像"的文件）
Docker Compose      （多容器编排）
Docker 网络         （Network）
Docker 存储         （Volume）
```

**架构与数据流（把这张图记住，面试可以画）**：

```text
         用户
          │  敲 docker 命令
          ▼
    Docker 客户端 (client)
          │  REST API
          ▼
   Docker 守护进程 (dockerd)
          │
    ┌─────┴──────┬───────────┐
    ▼            ▼           ▼
  镜像          容器       网络 / 存储
 (Images)   (Containers)
    ▲
    │ 拉取 / 推送
    ▼
  Docker 仓库 (Registry / Docker Hub)
```

> **关键理解**：**我们敲的 `docker` 命令只是客户端**，真正干活的是后台的 `dockerd` 守护进程。所以看到 `Cannot connect to the Docker daemon` 时，要去检查 `systemctl status docker`。

### 1.8 容器和虚拟机（★★ 高频对比题）

**课件里的三句话定义**：

```text
虚拟机：一台"虚拟"的电脑 = 虚拟的硬件 + 完整的操作系统
镜像  ：应用的"安装包"或"标准模具"
容器  ：镜像"运行的实例"
```

**完整对比表**：

| 对比项 | 虚拟机 | 容器 |
| --- | --- | --- |
| 装了什么 | 完整操作系统（Guest OS） | **只有应用和依赖** |
| 内核 | **每个虚机一套内核** | **共享宿主机内核** |
| 体积 | 几 GB | 几十 MB 到几百 MB |
| 启动速度 | 分钟级 | **秒级** |
| 隔离程度 | **完全隔离**（硬件级） | 进程级隔离（相对弱） |
| 一台机器能跑多少 | 几个 | **几十个** |
| 代表技术 | KVM、VMware | Docker |

> **一句话（背这个）**：**「虚拟机是『虚拟出一台完整的机器』，容器是『把进程关进一个有边界的盒子』—— 容器本质是被隔离 + 被限制的普通进程，靠 namespace 隔离、cgroup 限制资源。」**

---

## 二、Docker 安装（课件第二章）

### 2.1 在线安装（★★★ 完整复现）

**总体四步（课件原话）：设置 docker 仓库 → 安装前置依赖包 → 检测可安装的版本 → 安装并核对版本。**

#### 步骤 1：安装仓库管理工具 yum-utils

```bash
yum install -y yum-utils
```

**检验**：`yum-config-manager --version` 能输出版本号。

#### 步骤 2：添加 Docker 仓库（用阿里云镜像，国内快）

```bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

**检验**：

```bash
ls /etc/yum.repos.d/ | grep docker
# 期望看到 docker-ce.repo
```

#### 步骤 3：安装前置依赖包

```bash
yum -y install device-mapper-persistent-data lvm2
```

> **为什么要装这两个**：Docker 的存储驱动（overlay2 / devicemapper）依赖它们做设备映射和卷管理。

#### 步骤 4：查看可安装的版本

```bash
yum list docker-ce --showduplicates | sort -r
```

**预期输出**（节选，版本从高到低）：

```text
docker-ce.x86_64    3:26.1.4-1.el7     docker-ce-stable
docker-ce.x86_64    3:26.1.3-1.el7     docker-ce-stable
docker-ce.x86_64    3:24.0.9-1.el7     docker-ce-stable
...
```

> **为什么先看版本**：生产上很少装"最新的"，一般会**锁一个稳定版本**，保证多台机器一致。

#### 步骤 5：安装（两种方式二选一）

**方式 A：装最新版**（实验用）

```bash
yum -y install docker-ce docker-ce-cli containerd.io
```

**方式 B：装指定版本**（生产推荐，课件里装的是 24.0.9）

```bash
yum install -y docker-ce-24.0.9 docker-ce-cli-24.0.9 containerd.io
```

> **版本号怎么写**：`yum list` 里显示的是 `3:24.0.9-1.el7`，安装时只写 **`24.0.9`** 就行 —— `3:` 是 epoch，`-1.el7` 是 release，都不用写。

三个包分别是什么：

| 包名 | 作用 |
| --- | --- |
| `docker-ce` | Docker 服务端（dockerd） |
| `docker-ce-cli` | Docker 客户端（docker 命令） |
| `containerd.io` | 容器运行时（真正管容器生命周期的） |

#### 步骤 6：启动 Docker 并设置开机自启

```bash
systemctl start docker        # 启动
systemctl enable docker       # 开机自启
systemctl status docker       # 查看状态
```

**预期输出**：

```text
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2026-01-08 20:19:17 CST; 34s ago
     Docs: https://docs.docker.com
 Main PID: 10979 (dockerd)
   CGroup: /system.slice/docker.service
           └─10979 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

> **判读要点**：`Loaded` 那一行的 **`enabled`** 表示开机自启已设置；**`Active: active (running)`** 表示正在运行。两个都满足才算装好。

#### 步骤 7（课件重点）：处理 firewalld

```bash
systemctl stop firewalld
systemctl disable firewalld
iptables -nL                    # 看一眼当前规则
systemctl restart docker        # 关掉 firewalld 之后重启 docker，让它重新写 iptables 规则
```

> 原因见本章 0.3 节。

#### 步骤 8：验证安装

```bash
docker version
docker info
```

**`docker version` 预期输出**（重点看 Client 和 Server 都有内容，版本一致）：

```text
Client: Docker Engine - Community
 Version:           24.0.9
 API version:       1.43
 Go version:        go1.20.13
 Git commit:        2936816
 Built:             Thu Feb  1 00:51:49 2024
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          24.0.9
  ...
```

> **常见报错（重要）**：
> ```text
> Cannot connect to the Docker daemon at unix:///var/run/docker.sock.
> Is the docker daemon running?
> ```
> 这句表示**客户端能跑，但守护进程没起来**。解决：`systemctl start docker`，然后 `systemctl status docker` 看具体报错。
> **顺序观察**：`docker version` 会出现 Client 有内容、Server 报错的现象 —— 这正说明"客户端"和"守护进程"是两个东西（对应 1.7 的架构图）。

**`docker info` 关键字段**：

```bash
docker info
```

```text
Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 24.0.9
 Storage Driver: overlay2
  Backing Filesystem: xfs
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Kernel Version: 3.10.0-1160.el7.x86_64
 Operating System: CentOS Linux 7 (Core)
 OSType: linux
 Architecture: x86_64
 CPUs: 1
 Total Memory: 972.3MiB
 Docker Root Dir: /var/lib/docker
```

| 字段 | 说明 |
| --- | --- |
| `Server Version` | Docker 服务端版本 |
| `Storage Driver` | 镜像分层和可写层的管理方式，一般是 `overlay2` |
| `Logging Driver` | 容器日志默认格式，一般是 `json-file` |
| `Cgroup Driver` | 资源限制驱动，CentOS 7 是 `cgroupfs` |
| `Cgroup Version` | CentOS 7 内核较老，是 v1 |
| `Docker Root Dir` | **镜像和容器数据存放位置**（`/var/lib/docker`） |
| `Registry Mirrors` | 生效的镜像加速地址 |

#### 步骤 9：跑第一个容器（验证真的能用）

```bash
docker run hello-world
```

**预期输出（关键几行）**：

```text
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

> **这一条命令干了五件事**：① 检查本地有没有镜像 → ② 没有就从仓库下载 → ③ 用镜像**创建**容器 → ④ **启动**容器执行 `/hello` → ⑤ 程序打印完退出，容器变成 `Exited (0)`。

**检验**：

```bash
docker images        # 应该能看到 hello-world
docker ps -a         # 应该能看到刚才那个容器，STATUS 是 Exited (0)
```

> **`Exited (0)` 不是报错**：`0` 是退出码，表示正常退出。容器里的主进程执行完就退出了，容器自然就停了。

#### 步骤 10：安装验收清单

```bash
docker version                              # Client / Server 都有版本号
systemctl is-enabled docker                 # enabled
systemctl is-active docker                  # active
docker info | grep -A3 "Registry Mirrors"   # 加速器生效
docker images                               # 有 hello-world
```

---

### 2.2 离线安装（内网机器 / 装不了仓库时用）

**适用场景**：服务器不能上外网，或公司有内网仓库，需要用**二进制包手动装**，并**做成系统服务**。

#### 步骤 1：下载二进制包（在能上网的机器上）

```bash
wget https://mirrors.nju.edu.cn/docker-ce/linux/static/stable/x86_64/docker-28.5.2.tgz
```

> 也可以去 `https://download.docker.com/linux/static/stable/x86_64/` 找对应版本。下载完拷到目标服务器（`scp`）。

#### 步骤 2：解压并放到 PATH

```bash
tar -xf docker-28.5.2.tgz        # 解压出一个 docker/ 目录
cp docker/* /usr/bin/            # 把二进制都放到 /usr/bin
which dockerd                    # 检验：应输出 /usr/bin/dockerd
dockerd --version                # 检验：能输出版本号
```

#### 步骤 3：做成 systemd 服务（关键的一步）

```bash
cat >/etc/systemd/system/docker.service <<'EOA'
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOA
```

> **为什么需要这个文件**：二进制安装**不会自动生成 systemd 单元**。不写它就只能手动敲 `dockerd` 启动（关掉终端就没了），也没法开机自启、没法用 `systemctl status` 看状态。
>
> **几个字段为什么这么写**：
> - `Type=notify`：dockerd 启动完会主动通知 systemd，状态才准确
> - `LimitNOFILE=infinity`：容器多时文件句柄消耗很大
> - `Delegate=yes`：允许 docker 自己管理它的 cgroup（重要，否则容器里的 cgroup 操作会受限）
> - `KillMode=process`：只杀主进程、不动容器，避免 stop docker 时把容器一起粗暴干掉

#### 步骤 4：启动并设置开机自启

```bash
systemctl daemon-reload          # 新加了 unit 文件，必须 reload
systemctl start docker
systemctl enable docker
systemctl status docker          # 期望 active (running) + enabled
```

#### 步骤 5：验证

```bash
docker version
docker run hello-world
```

> **离线安装的三个坑**：
> ① 忘了 `systemctl daemon-reload` → 报 `Unit not found`；
> ② 二进制没放进 PATH → 敲 `docker` 提示 command not found；
> ③ 只拷了 `docker` 没拷 `dockerd` / `containerd` / `runc` → 服务起不来，用 `journalctl -u docker -n 50` 看原因。

---

### 2.3 docker-compose 安装

> **关键前提（课件原话）**：**如果是二进制安装的 Docker，是不包含 compose 功能的。** 因为 compose 是独立发布的 CLI 插件。

**先确认有没有**：

```bash
docker compose version        # 新写法（子命令形式）
docker-compose version        # 老写法（独立二进制）
```

课件环境里的实际输出是**两个不同版本**（说明两套都存在）：

```text
[root@r810-200 ~]# docker compose version
Docker Compose version v2.27.0

[root@r810-200 opt]# docker-compose version
Docker Compose version v2.38.0
```

#### 方式 A：装成独立二进制（老写法 `docker-compose`）

```bash
wget https://github.com/docker/compose/releases/download/v2.38.0/docker-compose-linux-x86_64 \
  -O /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose version        # 检验：Docker Compose version v2.38.0
```

#### 方式 B：装成 CLI 插件（新写法 `docker compose`，推荐）

```bash
mkdir -p /usr/libexec/docker/cli-plugins
mv /usr/local/bin/docker-compose /usr/libexec/docker/cli-plugins/docker-compose
docker compose version        # 检验：Docker Compose version v2.x.x
```

> **为什么放这个目录**：Docker 会自动扫描 `~/.docker/cli-plugins/`、`/usr/local/lib/docker/cli-plugins/`、`/usr/libexec/docker/cli-plugins/` 这三个目录，把里面的可执行文件注册成子命令。

> **坑（重要）**：`docker compose`（带空格）和 `docker-compose`（带横线）是**两个不同的东西**，版本可能不一样，行为也略有差别。**写文档、写脚本时统一用一种**，推荐统一用 `docker compose`。两个混着用，会出现"命令能查到但 compose 文件解析报错"的怪现象。

---

### 2.4 Namespace 与 Cgroups（容器的两块基石）

#### Namespace —— 负责"看不见"（隔离）

> **课件原文**：Namespace 是由 Linux 内核提供的一种特性，它能够将一些系统资源包装到一个抽象的空间中，并使得该空间中的进程以为这些资源是系统中仅有的资源。Namespace 是**构建容器技术的基石**，它使得容器内的进程**只能看到容器内的进程和资源**，实现与宿主系统以及其他容器的进程和资源隔离。Namespace 按操作的系统资源不同有很多种类，比如 cgroup namespace、mount namespace 等等。

**六种 Namespace（面试高频）**：

| Namespace | 隔离什么 | 效果 |
| --- | --- | --- |
| **PID** | 进程号 | 容器里有自己的 PID 1，看不到宿主机进程 |
| **NET** | 网络 | 有自己的网卡、IP、端口、路由表 |
| **MNT** | 挂载点 | 有自己的文件系统视图（根目录不一样） |
| **UTS** | 主机名 / 域名 | 容器可以有自己的 hostname |
| **IPC** | 进程间通信 | 信号量、共享内存互不干扰 |
| **USER** | 用户和 UID | 容器里的 root 可以映射成宿主机的普通用户 |

**实操验证：看看一个容器的 namespace 长什么样**

```bash
# 1) 起一个长期运行的容器
docker run -d --name ns-test nginx

# 2) 看这个容器的 PID
docker inspect -f '{{.State.Pid}}' ns-test        # 假设输出 12345

# 3) 看它的 namespace 列表
ls -l /proc/12345/ns/
# 期望输出（每一项都是一个 namespace 文件）：
# lrwxrwxrwx ... ipc -> ipc:[4026532465]
# lrwxrwxrwx ... mnt -> mnt:[4026532463]
# lrwxrwxrwx ... net -> net:[4026532468]
# lrwxrwxrwx ... pid -> pid:[4026532466]
# lrwxrwxrwx ... uts -> uts:[4026532464]

# 4) 对比：宿主机自己的 namespace 编号和容器不一样
ls -l /proc/1/ns/net
```

> **看什么**：`net:[4026532468]` 里方括号中的数字就是 **namespace 的 inode 号** —— **每个 namespace 一个号，相同就是同一个隔离空间**。这样就能"看见"隔离。

#### Cgroups —— 负责"用多少"（限制）

> **课件原文**：Cgroups 是资源限制器，作用是**限制、统计和隔离进程使用的资源**。

**能限制哪些资源**：

```text
CPU      ：能用多少核、占多少比例（cpu、cpuset）
内存      ：能用多少内存（memory），超了会被 OOM 杀掉
块设备 IO ：读写速率（blkio）
进程数    ：最多能起多少进程/线程（pids）
```

**实操验证：给容器限制内存和 CPU**

```bash
# 限制：最多用 256M 内存、最多用 1 个 CPU
docker run -d --name limit-test --memory=256m --cpus=1 nginx

# 查看限制（单位：字节 / 纳核）
docker inspect -f '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}' limit-test
# 期望：268435456 1000000000

# 看容器实际用量
docker stats --no-stream limit-test
```

> **一句话区分两块基石（背这个）**：**「namespace 决定『你能看到什么』，cgroup 决定『你能用多少』。」**

**清理实验容器**：

```bash
docker rm -f ns-test limit-test
```

---

## 三、镜像与容器（课件第三章）

### 3.1 Docker 镜像与容器

> **课件原文**：
> **Docker 镜像**：是一个轻量级、独立、可执行的软件包，包含了应用程序运行所需的所有内容：**代码、运行时、系统库、环境变量和配置文件**。可以理解为一个"软件安装包"，既可以从公共仓库下载，也可以自己制作。
> **Docker 容器**：是 Docker 镜像的**一个运行实例**。它基于镜像创建，包含了应用及其依赖，并能在任何支持 Docker 的主机上独立运行。

**一句话**：

> **镜像 = 打包好的运行环境（模板）；容器 = 用这个模板跑起来的实例。**

| 镜像 | 容器 |
| --- | --- |
| 类（class） | 对象（实例） |
| 安装包 | 安装后运行的进程 |
| 菜谱 | 做出来的那道菜 |
| **只读** | 可读可写（多一层可写层） |
| `docker images` 查看 | `docker ps` 查看 |

### 3.2 镜像加载原理（分层存储）

#### 核心：UFS（联合文件系统）

```text
Docker 镜像采用【分层存储】结构，核心技术是 UFS（联合文件系统 Union File System）。
多个层叠加起来，对外看起来像一个完整的文件系统。
```

#### 镜像的两个"核心层"（课件模型）

| 层 | 位置 | 内容 |
| --- | --- | --- |
| **bootfs**（引导文件系统） | 最底层 | 包含 bootloader 和 Linux 内核；所有 Linux 发行版的 bootfs 基本一致 |
| **rootfs**（根文件系统） | 在 bootfs 之上 | 包含 `/bin`、`/etc`、`/opt` 等目录 |

> **★ 课件原话（必须记住的一句）**：
> **Docker 镜像并不包含完整的操作系统，它只提供 rootfs 和应用程序，内核则使用母机（宿主机）的。**
>
> **补充一个现代视角（面试加分）**：bootfs/rootfs 是经典的**教学模型**。现代 Docker 镜像实际上就是"一层层只读的 rootfs 叠加"，`bootfs` 这一层在 Linux 上基本就是宿主机内核本身，镜像里并不打包内核。所以更准确的说法是：**镜像 = 应用 + 依赖 + 精简的系统库，共用宿主机的内核。**

#### 分层的好处：资源共享

```text
如果多个镜像基于同一个基础镜像构建，宿主机只需要保留一份基础层。
内存里也只需要加载一次。

举个实例：
  nginx 和 mysql 都基于某个 debian/ubuntu 基础层，
  那么这个基础层在本地只存一份，两个镜像共用。
```

#### 镜像层与容器层

```text
镜像层是【只读】的。
容器运行时写的数据（装软件、改配置、写日志）落在最上面的【可写层】。

最终的镜像 = 所有层累加
容器       = 只读的镜像层 + 一层可写层
```

> **三条推论（后面会反复用到）**：
> ① 容器里改文件，改的是**可写层**，镜像不受影响；
> ② 容器删除时**可写层一起消失**，所以数据会丢；
> ③ 想让数据不丢，必须挂载卷（见 第二期 第六章）。

**实操：亲眼看一眼镜像的层**

```bash
docker pull nginx
docker history nginx
```

**预期输出**（`docker history` 会把每一层列出来）：

```text
IMAGE          CREATED       CREATED BY                                      SIZE
fd204fe2f750   10 days ago   CMD ["nginx" "-g" "daemon off;"]                0B
<missing>      10 days ago   STOPSIGNAL SIGQUIT                              0B
<missing>      10 days ago   EXPOSE map[80/tcp:{}]                           0B
<missing>      10 days ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B
<missing>      10 days ago   COPY 30-tune-worker-processes.sh /docke…        4.62kB
...
```

> **看什么**：每一行就是一层，`SIZE` 是这一层增加的大小。有些层是 `0B`（只改了元数据，比如 `CMD`、`EXPOSE`）—— 这正好说明"指令不一定产生文件，但一定会产生一层"。

### 3.3 Docker 镜像加速（★★ 必做，不做拉不动镜像）

> **为什么必须配**：默认从 Docker Hub 拉镜像，国内网络基本拉不动。必须配镜像加速器。

**完整操作（课件里的 daemon.json 是全的，直接用）**：

```bash
mkdir -p /etc/docker

tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me",
    "https://hub-mirror.c.163.com",
    "https://my1tmn29.mirror.aliyuncs.com"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
```

**每个字段什么意思**（课件逐行讲了）：

| 字段 | 作用 |
| --- | --- |
| `registry-mirrors` | **镜像加速站点**（按顺序尝试，第一个可用就生效） |
| `exec-opts` | 设置 docker 的 **cgroup 驱动**。CentOS 7 保持 `cgroupfs` 也可以；如果宿主机 systemd 版本较新，用 `systemd` 更统一（K8s 1.24+ 要求运行时和 kubelet 的 cgroup driver 一致） |
| `log-driver` | 日志驱动，默认 `json-file` |
| `log-opts.max-size` | **单个日志文件最大 100m**（不限制的话容器日志会把磁盘写满） |
| `log-opts.max-file` | **最多保留 3 个文件**（轮转） |

> **★ 为什么必须配 `log-opts`**：容器日志默认写进 `json-file` 且**不限制大小**，一个疯狂打日志的容器能把磁盘写满。这是生产必配项，也是面试常问的"容器日志怎么控制"。

**生效与检验**：

```bash
systemctl daemon-reload        # daemon.json 改了要先 reload
systemctl restart docker       # 再重启 docker
systemctl status docker        # 确认 active (running)

docker info | grep -A5 "Registry Mirrors"
```

**预期输出**：

```text
 Registry Mirrors:
  https://docker.1ms.run/
  https://docker.xuanyuan.me/
  https://hub-mirror.c.163.com/
  https://my1tmn29.mirror.aliyuncs.com/
```

> **`docker info` 里能看到这几行，就说明加速器生效了**（这是"加速器是否生效"的唯一权威判断方式，不是看文件）。

**加速地址从哪找（课件给的三类来源）**：

| 来源 | 地址 |
| --- | --- |
| **阿里云专属加速器（最稳，推荐）** | 阿里云控制台 → 容器镜像服务 ACR → 镜像工具 → 镜像加速器，形如 `https://<你的ID>.mirror.aliyuncs.com` |
| 商业镜像同步站 | 轩辕镜像 `https://xuanyuan.cloud/`、毫秒镜像 `https://1ms.run/` |
| 镜像站（可查镜像是否存在） | `https://docker.aityp.com/`，例：`https://docker.aityp.com/image/docker.io/tomcat:9.0.107-jdk21` |

> **坑（重要）**：
> ① 公共加速地址**失效非常快**（有的已停服），强烈建议换成自己阿里云控制台申请的专属地址；
> ② **不要填这些**：`https://hub.docker.com`（官方仓库，不是加速器）、`https://cr.console.aliyun.com`（控制台地址，不是加速器）；
> ③ 失效的地址排在前面会**浪费超时时间**，拖慢每一次拉取，所以列表要精简；
> ④ 改完 `daemon.json` 一定要 `systemctl daemon-reload` + `systemctl restart docker`，**只 reload 不 restart 不生效**；
> ⑤ JSON 写错（多逗号、少引号）会导致 docker **起不来**，改完先校验：`python -m json.tool /etc/docker/daemon.json`。

### 3.4 镜像搜索

#### 方式一：web 搜索（最直观）

```text
打开官方站点 https://hub.docker.com
在搜索栏输入 nginx
```

#### 方式二：命令行搜索

```bash
docker search nginx
docker search centos        # 课件演示的就是这个
```

**预期输出**（节选，注意表头字段）：

```text
NAME                       DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
centos                     DEPRECATED; The official build of CentOS.       7780      [OK]
corpusops/centos           centos corpusops baseimage                      0
dockette/centos            My Custom CentOS Dockerfiles                    1                    [OK]
eclipse/centos             CentOS based minimal stack with only git and…   1                    [OK]
centos/postgresql-10-centos7   PostgreSQL is an advanced Object-Relational… 21
centos/redis-5-centos8                                                     0
centos/httpd-24-centos8                                                    3
centos/mysql-80-centos8                                                    0
centos/nginx-112-centos7   Platform for running nginx 1.12 or building …  16
```

**`docker search` 的几列怎么读**：

| 字段 | 含义 |
| --- | --- |
| `NAME` | 镜像名。**没有斜杠的是官方镜像**（如 `centos`），`xxx/yyy` 是第三方（如 `dockette/centos`） |
| `STARS` | 收藏数，越高说明越流行 |
| `OFFICIAL` | `[OK]` 表示 Docker 官方维护 |
| `AUTOMATED` | `[OK]` 表示是自动构建的（已逐渐弃用） |

> **注意**：
> ① `docker search` 只能搜**镜像名和描述**，不能搜 tag（比如搜不到"nginx 1.24"）。要确认某个 tag 存不存在，得去 web 站点看，或者直接 `docker pull` 试；
> ② `centos` 这个镜像官方标注了 **DEPRECATED**（CentOS 8 停止维护后官方镜像也停止更新），做实验建议用 `rockylinux` / `almalinux` / `ubuntu` 代替。

### 3.5 镜像拉取

**三种拉法**：

```bash
docker pull docker.io/library/nginx:1.24.0   # 完整格式（最严格）
docker pull nginx                            # 简写：等价于 nginx:latest，从官方仓库拉
docker pull bitnami/tomcat                   # 第三方镜像（用户名/镜像名）
```

**完整格式的组成（面试可能问）**：

```text
docker.io     /  library  /  nginx  :  1.24.0
   仓库地址       命名空间      镜像名      标签

· 省略仓库地址 → 默认 docker.io
· 省略命名空间 → 默认 library（官方镜像）
· 省略标签     → 默认 latest
```

**拉取过程解读**：

```bash
docker pull nginx:1.24
```

**预期输出**：

```text
1.24: Pulling from library/nginx
6310eb16bf42: Pull complete
956faab5efb3: Pull complete
a44b5c8be616: Pull complete
02fc02c4ab8d: Pull complete
c12f394dea35: Pull complete
07db7bf2649b: Pull complete
f340c1b7c1d6: Pull complete
Digest: sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4
Status: Downloaded newer image for nginx:1.24
docker.io/library/nginx:1.24
```

| 行 | 含义 |
| --- | --- |
| `Pulling from library/nginx` | 从官方仓库拉取 |
| `xxxxxxxxxxxx: Pull complete` | **每一行是一个层（layer）下载完成** |
| `Digest: sha256:...` | 镜像内容的唯一指纹（比 tag 更可靠，tag 会被覆盖） |
| `Status: Downloaded newer image` | 拉取完成 |

> **为什么有些层已经有、有些要下载**：层可以复用。如果本地某个镜像已经含相同的层，那一层会显示 `Already exists` 而不重新下载 —— 这也是"镜像分发快"的原因。

**检验**：

```bash
docker images
```

**预期输出**：

```text
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        1.24      6c0218f16876   2 years ago    142MB
nginx        latest    fd204fe2f750   10 days ago    161MB
hello-world  latest    e2ac70e7319a   5 months ago   10.1kB
```

**字段解读**：

| 字段 | 含义 |
| --- | --- |
| `REPOSITORY` | 镜像名 |
| `TAG` | 版本标签 |
| `IMAGE ID` | 镜像唯一编号（**前 12 位**，可以当短 ID 用） |
| `CREATED` | 构建时间（相对时间） |
| `SIZE` | 占用大小（是"压缩后的逻辑大小"，多镜像共享层时实际占用更小） |

---

## 四、本期闭环自检清单

| 检查项 | 命令或位置 | 通过标准 |
| --- | --- | --- |
| 系统版本、资源、网络确认 | `cat /etc/redhat-release`、`nproc`、`free -h`、`ping` | 能报出具体数值 |
| SELinux 已处理 | `getenforce` | `Permissive` 或 `Disabled` |
| firewalld 已处理 | `systemctl status firewalld` | `inactive (dead)` |
| Docker 已安装 | `docker version` | Client 和 Server 都有版本号 |
| Docker 已开机自启 | `systemctl is-enabled docker` | `enabled` |
| Docker 正在运行 | `systemctl is-active docker` | `active` |
| 镜像加速生效 | `docker info` 看 `Registry Mirrors` | 能看到自己配的地址 |
| 日志已限制大小 | `daemon.json` 的 `log-opts` | `max-size` / `max-file` 已配 |
| 能跑起容器 | `docker run hello-world` | 输出 `Hello from Docker!` |
| 能拉取镜像 | `docker pull nginx:1.24` | `Status: Downloaded` |
| 能看镜像历史 | `docker history nginx` | 逐层列出 |
| compose 可用 | `docker compose version` | 输出版本号 |
| 能看见 namespace | `ls -l /proc/<容器PID>/ns/` | 列出 6 项 namespace |
| 能限制资源 | `docker run --memory=256m --cpus=1` | `docker stats` 看到限制值 |

---

## 五、常见坑速查

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `Cannot connect to the Docker daemon ... Is the docker daemon running?` | 守护进程没起 | `systemctl start docker`，再看 `systemctl status docker` |
| `docker run hello-world` 卡住 / 超时 | 镜像加速器不可用或没配 | 换阿里云专属加速地址，改完 `daemon-reload` + `restart` |
| 加速器配了不生效 | 改完没重启 docker | `systemctl daemon-reload && systemctl restart docker`，用 `docker info` 验证 |
| 改完 `daemon.json` Docker 起不来 | JSON 格式错误 | `python -m json.tool /etc/docker/daemon.json` 校验 |
| `permission denied while trying to connect to the Docker daemon socket` | 当前用户不在 docker 组 | `usermod -aG docker $USER` 后重新登录（或临时用 sudo） |
| 容器端口映射后外网访问不了 | firewalld 清掉了 docker 的 iptables 规则 / 云安全组没放行 | 重启 docker；或按 0.3 节处理防火墙；云上查安全组 |
| 二进制安装后 `systemctl start docker` 报 `Unit not found` | 忘了 `daemon-reload` | `systemctl daemon-reload` |
| 镜像 `centos` 提示 DEPRECATED | CentOS 8 停止维护 | 换 `rockylinux` / `almalinux` / `ubuntu` |
| 磁盘被容器日志写满 | 默认 json-file 不限制大小 | 配 `log-opts` 的 `max-size` / `max-file` |
| `docker compose` 和 `docker-compose` 行为不一致 | 两套并存且版本不同 | 统一用一种，推荐 `docker compose` |

---

## 六、自测问题（闭卷）

- [ ] 能说出 Docker 的三条核心价值
- [ ] 能画出 client → dockerd → 镜像/容器/网络存储 → 仓库 的架构图
- [ ] 能说出 docker-ce 和 docker-ee 的区别
- [ ] 能说出容器和虚拟机的 5 个区别，并用一句话解释"容器共享宿主机内核"
- [ ] 能按顺序说出在线安装的完整步骤（装仓库 → 装依赖 → 查版本 → 装 → 启动 → 处理防火墙 → 验证）
- [ ] 知道看到 `Cannot connect to the Docker daemon` 该怎么查
- [ ] 能说出离线安装为什么要自己写 systemd 单元
- [ ] 能说出 `docker compose` 和 `docker-compose` 的区别
- [ ] 能说出 namespace 的六种类型和 cgroup 能限制的四种资源
- [ ] 能解释"镜像不含内核，内核用的是宿主机的"
- [ ] 能说出镜像分层的两大好处（资源共享、拉取快）
- [ ] 能写出完整的 `daemon.json`（加速 + cgroup 驱动 + 日志限制）并说出每个字段的作用
- [ ] 能用 `docker info` 判断加速器是否生效

---

## 七、与后续两期的衔接

```text
第一期（本篇）  概念 + 安装 + 镜像基础
      │  已经能做到：装好 Docker、拉下镜像、跑起 hello-world
      ▼
第二期          常用命令 + 网络 + 数据卷 + 私有仓库
      │  要做到：熟练管理镜像和容器、看懂 docker0 和 iptables 规则、
      │          会用三种挂载方式、能部署 Portainer 和 Harbor
      ▼
第三期          Compose + Dockerfile + Prometheus 监控 + 项目实战
                 能做到：一条命令起一套 LNMP、自己构建镜像、
                        用 compose 部署 Zabbix、接上 Prometheus + Grafana
```

> **下一步**：先把本篇第二章的**在线安装**在虚拟机里完整跑一遍（包括关闭 SELinux/firewalld、配加速器、跑 hello-world），再进第二期。