# Docker 第一期：Docker 原理

## 〇、实战前的环境准备

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

### 0.3 关闭 firewalld（练习环境做法）

> **问题1：生产上要怎么调整防火墙**

### 0.4 装好常用工具

```bash
yum install -y vim wget net-tools lsof bash-completion
```

---

## 一、Docker 理论基础

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

被问"什么是 Docker"，用一句话定位 ——「**Docker 是一个用容器技术做应用打包和分发的工具，核心价值是让应用在开发、测试、生产环境里表现得一模一样。**」

### 1.6 Docker 的核心价值（★★ 三条，必须能背）

| # | 价值 | 说明 |
| --- | --- | --- |
| **1** | **环境一致性** | 开发、测试、生产环境有差异。Docker 通过容器**封装完整的运行环境**，消除了"环境依赖"问题 |
| **2** | **资源高效利用** | 相比虚拟机，容器不是一个完整的操作系统，**资源占用超低、启动速度快** |
| **3** | **部署高效** | 容器可以**快速复制、快速启停**，天然支持自动化 |

> **一句话串起来**：**「Docker 把『应用 + 依赖 + 配置』打包成一个镜像，到哪台机器都一模一样地跑起来；而且因为共享内核，它比虚拟机更轻、更快、更省资源。」**

### 1.7 Docker 的核心组件

Docker有哪些组件：

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

Docker的组件架构：

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

> **问题：容器与虚拟机的区别**

---

## 二、Docker 安装

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

## 三、镜像与容器

### 3.1 Docker 镜像与容器

> **Docker 镜像**：是一个轻量级、独立、可执行的软件包，包含了应用程序运行所需的所有内容：**代码、运行时、系统库、环境变量和配置文件**。可以理解为一个"软件安装包"，既可以从公共仓库下载，也可以自己制作。
> **Docker 容器**：是 Docker 镜像的**一个运行实例**。它基于镜像创建，包含了应用及其依赖，并能在任何支持 Docker 的主机上独立运行。

**一句话**：

> **镜像 = 打包好的运行环境（模板）；容器 = 用这个模板跑起来的实例。**

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

> **Docker 镜像并不包含完整的操作系统，它只提供 rootfs 和应用程序，内核则使用母机（宿主机）的。**

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

### 3.3 Docker 镜像加速（默认从 Docker Hub 拉镜像，国内网络基本拉不动。必须配镜像加速器）

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

> **问题：镜像来源从哪里找？**

### 3.4 镜像搜索

#### 方式一：web 搜索（最直观）

```text
打开官方站点 https://hub.docker.com
在搜索栏输入 nginx
```

#### 方式二：命令行搜索

```bash
docker search nginx
docker search centos
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

> **问题：`docker search` 的几列怎么读？**

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

```bash
getenforce                   # 看当前状态（Enforcing / Permissive / Disabled）
setenforce 0                 # 临时改为 permissive（重启失效）

# 永久关闭
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
grep '^SELINUX=' /etc/selinux/config     # 检验：应输出 SELINUX=disabled
```

> **为什么**：SELinux 会拦住容器访问宿主机目录，报 `Permission denied`，实验阶段先关掉最省事；生产上要用 `setsebool` / `restorecon` 精确放行。

```bash
systemctl stop firewalld
systemctl disable firewalld     # 禁止开机启动
systemctl status firewalld      # 检验：inactive (dead)
```

> **firewalld 和 docker 会同时管理操作系统的防火墙，容易互相干扰，最好把 firewalld 关掉，关掉之后把 docker 重启一下。**
>
> **但要知道生产上的正确做法**（面试会问）：
> - Docker 启动时会自动往 iptables 的 `nat` 表和 `filter` 表写规则（`DOCKER` 链、`MASQUERADE`、`DNAT`）。**firewalld 重载时会清掉这些规则**，导致已经发布的容器端口突然访问不了。
> - 生产上不是简单关闭，而是：① 用 firewalld 的 `docker` zone（新版支持）；② 或者让 docker 不碰 iptables（`"iptables": false`）再自己写规则；③ 或者由外层防火墙 / 云安全组统一管理。
> - **实验环境：关掉；生产环境：要么用 docker zone，要么统一规划。** 这句能说出来就是加分项。

### 1.8 容器和虚拟机

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

**加速地址从哪找（三类来源）**：

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

| 字段 | 含义 |
| --- | --- |
| `NAME` | 镜像名。**没有斜杠的是官方镜像**（如 `centos`），`xxx/yyy` 是第三方（如 `dockette/centos`） |
| `STARS` | 收藏数，越高说明越流行 |
| `OFFICIAL` | `[OK]` 表示 Docker 官方维护 |
| `AUTOMATED` | `[OK]` 表示是自动构建的（已逐渐弃用） |

> **注意**：
> ① `docker search` 只能搜**镜像名和描述**，不能搜 tag（比如搜不到"nginx 1.24"）。要确认某个 tag 存不存在，得去 web 站点看，或者直接 `docker pull` 试；
> ② `centos` 这个镜像官方标注了 **DEPRECATED**（CentOS 8 停止维护后官方镜像也停止更新），做实验建议用 `rockylinux` / `almalinux` / `ubuntu` 代替。
