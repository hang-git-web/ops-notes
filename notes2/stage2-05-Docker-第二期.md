# Docker 第二期：常用命令、网络、数据卷与私有仓库

> **对应课件**：《Docker基础到大神进阶(第二期)》（千山）
> **本篇定位**：第一篇装好了 Docker，这一篇是**真正开始用**：把命令练熟、把网络搞懂、把数据存住、把镜像推到自己的仓库。
> **环境**：Rocky Linux 8.10 + docker 26.1.3（4 核 / 3.8G），CentOS 7.9 同样适用。
> **承接关系**：本篇的 5.x（网络）、6.x（数据卷）是第三期 Compose 的基础 —— **Compose 里的 `networks:` 和 `volumes:` 就是这两节的自动化版本**。

---

## 〇、开篇前的准备

```bash
# 1) 确认 docker 正常
docker version && docker info | head -20

# 2) 本篇会频繁用到这几个命令，先装上
dnf install -y net-tools        # CentOS 用 yum install -y net-tools
yum install -y iproute          # 有的系统要单独装
```

> **为什么要装 net-tools**：课件里用 `ip link show type bridge` 看虚拟网桥，有些老系统还需要 `ifconfig`/`netstat` 做对照。

---

## 一、Docker 常见命令（课件第四章）

### 4.1 管理命令

#### docker version（看版本）

```bash
docker version
```

**实际输出**（节选）：

```text
Client: Docker Engine - Community
 Version:           26.1.3
 API version:       1.45
 Go version:        go1.21.10
 Git commit:        b72abbb
 Built:             Thu May 16 08:34:39 2024
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          26.1.3
  API version:      1.45 (minimum version 1.24)
  Go version:       go1.21.10
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.32
  GitCommit:        8b3b7ca2e5ce38e8f31a34f35b2b68ceb8470d89
 runc:
  Version:          1.1.12
  GitCommit:        v1.1.12-0-g51d5e94
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

> **怎么读**：`Client` 是命令行，`Server` 是守护进程，**两者版本一致最省心**。`Server` 下面还挂着三个组件：**containerd**（容器运行时）、**runc**（真正创建容器的工具）、**docker-init**（PID 1 的初始化程序）。能说出这三层，面试会加分。
>
> **API version 那一行有 `minimum version 1.24`**：意思是服务端支持的最低 API 版本，客户端 API 版本必须落在服务端支持的范围内。

#### docker info（看运行信息）

```bash
docker info
```

**实际输出**（关键部分）：

```text
Client: Docker Engine - Community
 Version:    26.1.3
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.14.0
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.27.0
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 12
 Server Version: 26.1.3
 Storage Driver: overlay2
  Backing Filesystem: xfs
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Kernel Version: 4.18.0-553.el8_10.x86_64
 Operating System: Rocky Linux 8.10 (Green Obsidian)
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 3.829GiB
 Docker Root Dir: /var/lib/docker
 Registry Mirrors:
  https://docker.m.daocloud.io/
  https://docker.imgdb.de/
  ...
 Live Restore Enabled: false
```

> **`docker info` 是排障第一站**：容器数、镜像数、存储驱动、日志驱动、cgroup 驱动、内核、加速器、Docker 根目录，全在这。

### 4.2 镜像命令

| 命令 | 作用 |
| --- | --- |
| `docker search name` | 搜索镜像（其实就是搜 hub.docker.com） |
| `docker pull name:tag` | 拉取镜像 |
| `docker images` | 查看本地镜像 |
| `docker tag 源镜像:tag 目标镜像:tag` | 给镜像打标签（**推私有仓库前必做**） |
| `docker push images:tag` | 推送镜像到仓库 |
| `docker rmi images:tag` | 删除本地镜像（按名+tag） |
| `docker rmi imagesid` | 删除本地镜像（按 ID） |
| `docker rmi -f $(docker images -aq)` | **批量删除所有镜像** |
| `docker save` / `docker load` | 导出镜像 / 导入离线镜像 |

#### docker images（实际输出）

```bash
docker images
```

```text
REPOSITORY               TAG       IMAGE ID       CREATED        SIZE
portainer/portainer-ce   latest    2ad8d683056c   7 days ago     183MB
redis                    8.2.5     d415fa2386b8   8 days ago     137MB
redis                    7.2.13    e4233e880bed   8 days ago     117MB
redis                    6.2.21    29646b9b9860   8 days ago     106MB
nginx                    1.28      1f1a56031783   8 days ago     161MB
nginx                    latest    fd204fe2f750   8 days ago     161MB
alpine                   latest    a40c03cbb81c   5 weeks ago    8.44MB
mysql                    8.0.44    e645fd9ffc4f   6 weeks ago    781MB
tomcat                   8         0a2249be3d31   23 months ago  456MB
mysql                    5.7.44    5107333e08a8   2 years ago    501MB
nginx                    1.24      6c0218f16876   2 years ago    142MB
tomcat                   7         9dfd74e6bc2f   4 years ago    533MB
```

> **观察两点**：① 同一镜像可以有多个 tag（`nginx` 有 1.28 / latest / 1.24），它们**指向不同镜像**（IMAGE ID 不同）；② `alpine` 只有 8.44MB，所以经常被用来做"精简基础镜像"，`FROM alpine` 构建出来的镜像体积很小。

#### docker save / load（离线搬运镜像）

```bash
# 导出（在能上网的机器上）
docker save -o nginx.tar nginx:1.24
ls -lh nginx.tar                # 检验：生成一个 tar 文件

# 导入（在内网机器上）
docker load -i nginx.tar
docker images | grep nginx      # 检验：能看到 nginx:1.24

# 新写法（等价）
docker image save -o nginx.tar nginx:1.24
docker image load -i nginx.tar
```

> **save 和 load 的坑**：`docker save` 保存的是**镜像**（含 tag 和所有层），`docker load` 之后 tag 还在；而 `docker export` 导出的是**容器的文件系统快照**，`docker import` 回来会**丢掉 tag 和历史层**。别混。

### 4.3 容器命令

**容器的三个状态（课件原话）**：`up`（运行中）、`exited`（已停止）、`deleted`（已删除）。

#### 4.3.1 docker run / stop / start / restart / kill

**`docker run` 的本质（课件原话）**：首次运行时，它会检测本地是否有镜像 —— **有的话直接 start，没有的话先 pull，再 start。**

**参数表**：

| 参数 | 含义 |
| --- | --- |
| `--name` | 给运行的容器命名 |
| `-d` | 后台运行（detach） |
| `-it` | 交互式运行（常常配合 `bash` 进容器） |
| `-p` | **小写 p**，指定端口映射（宿主机端口:容器端口） |
| `-P` | **大写 P**，随机分配宿主机端口 |
| `-e` | 设置环境变量 |
| `-v` | 挂载数据卷（宿主机目录:容器目录） |

**一个完整的 run 例子（跑 MySQL）**：

```bash
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD="Qianshan@123" \
  -e MYSQL_DATABASE="appdb" \
  -p 3306:3306 \
  mysql:8.0.44
```

**检验（完整闭环）**：

```bash
docker ps                                  # 1) 容器在运行
docker logs mysql-db                       # 2) 看启动日志（找 "ready for connections"）
docker port mysql-db                       # 3) 看端口映射（0.0.0.0:3306 -> 3306/tcp）
ss -lntp | grep 3306                       # 4) 宿主机确实在监听 3306
mysql -uroot -p'Qianshan@123' -h 127.0.0.1 -P 3306 -e "show databases;"
                                           # 5) 真正连进去（能看到 appdb 才算成功）
```

> **第 5 步是"闭环"的关键**：光看到容器 Up 不算成功，**能连上业务端口**才算。这也是面试时"你怎么确认服务真的起来了"的标准答法。

**启停四件套**：

```bash
docker stop 容器名/容器ID        # 优雅停止（默认发 SIGTERM，10 秒超时后 SIGKILL）
docker start 容器名/容器ID       # 第二次启动就用 start，不要再用 run（会报 Conflict）
docker restart 容器名/容器ID     # = stop + start
docker kill 容器名/容器ID        # 直接 SIGKILL（强制，不优雅）
```

> **`run` 和 `start` 的区别（高频考点）**：
> - `docker run` = **创建 + 启动**一个新容器；如果 `--name` 已存在，报 `Conflict. The container name "/xxx" is already in use`。
> - `docker start` = **启动一个已经存在的、处于停止状态的容器**。
> - 一句话：**run 是"造一台新的"，start 是"把旧的发动起来"。**

#### 4.3.2 docker ps

```bash
docker ps        # 只输出正在运行的容器
docker ps -a     # 输出所有容器（包括已退出）
docker ps -qa    # 只输出所有容器的 ID（脚本里批量操作用）
```

> **`-qa` 的用途**：`docker rm -f $(docker ps -qa)` 就是"强制删除所有容器"，写脚本清理实验环境时最常用。

#### 4.3.3 docker pause / unpause

```bash
docker pause 容器名/ID        # 暂停（进程被冻结，还在内存里）
docker unpause 容器名/ID      # 恢复
```

**实操验证"暂停"是什么效果**：

```bash
docker run -d --name web -p 8080:80 nginx
curl -sI http://127.0.0.1:8080 | head -1     # 期望 HTTP/1.1 200 OK

docker pause web
curl -sI --max-time 5 http://127.0.0.1:8080 | head -1   # 期望卡住/超时（没有响应）

docker unpause web
curl -sI http://127.0.0.1:8080 | head -1     # 期望恢复 200 OK

docker rm -f web
```

> **pause 和 stop 的区别**：`pause` 用 cgroup freezer **冻结进程**（不释放资源，恢复后继续跑）；`stop` 是**结束进程**（容器变 Exited，恢复要 start 重新启动）。面试问"怎么临时冻结一个容器"就是 `pause`。

#### 4.3.4 docker exec / attach（进容器）

**课件实际演示**：

```bash
docker exec -it qianshan-web1 bash
```

```text
root@3a642827805b:/# ls
bin  boot  dev  docker-entrypoint.d  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt
opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@3a642827805b:/# ll
bash: ll: command not found
root@3a642827805b:/# exit
exit
```

> **两个观察**：① 提示符变成 `root@3a642827805b` —— **3a642827805b 是容器 ID**，说明你已经在容器里了；② `ll` 报 `command not found` —— 因为 `ll` 是宿主机上 `alias ll='ls -l'` 定义的**别名**，容器里没有这个别名，要用 `ls -l`。
>
> **这就是"容器是一个精简环境"的最直观证据**，面试举这个例子很好用。

**exec 和 attach 的区别（课件原话 + 补充）**：

```bash
docker exec -it 容器ID bash      # 新起一个进程（新终端），不会影响容器主进程
docker attach 容器ID             # 直接接到容器主进程的终端上
                                 # 课件原话：不启动新的终端，如果按 ctrl+c，容器会自动退出
```

| | `docker exec` | `docker attach` |
| --- | --- | --- |
| 是否新进程 | 是（在容器里新开一个进程） | 否（连到主进程的 stdin/stdout） |
| 退出方式 | `exit` 只退出这个进程，**容器继续运行** | 按 `Ctrl+C` 会把信号发给主进程，**容器可能直接退出** |
| 推荐度 | **推荐**，排障首选 | 少用，容易误杀容器 |

> **一句话记忆**：**「要进容器就用 `docker exec -it`，别用 attach —— attach 按 Ctrl+C 会把容器弄停。」**

#### 4.3.5 docker logs（看日志）

```bash
docker logs 容器名/容器ID
docker logs -f -t --tail 10 qianshan-web1
```

| 参数 | 含义 |
| --- | --- |
| `-f` | follow，实时跟踪（相当于 `tail -f`） |
| `-t` | 显示时间戳 |
| `--tail 10` | 只看最后 10 行 |
| `--since 10m` | 只看最近 10 分钟 |

> **注意**：`docker logs` 看的是**容器主进程输出到 stdout/stderr 的内容**。如果应用把日志写进容器内的文件（比如 `/var/log/nginx/access.log`），`docker logs` 是看不到的 —— 那种要 `docker exec` 进去看，或者把日志目录挂载出来。

#### 4.3.6 删除容器

```bash
docker rm 容器名/ID            # 删除已停止的容器
docker rm -f 容器名/ID         # 强制删除（课件原话：不管它有没有在运行）
```

> **坑**：正在运行的容器直接 `docker rm` 会报错：
> ```text
> Error response from daemon: You cannot remove a running container ...
> Stop the container before attempting removal or force remove
> ```
> 要么先 `docker stop`，要么加 `-f`。

#### 4.3.7 docker inspect（看容器详细信息）

```bash
docker inspect 容器名/ID
docker inspect -f '{{.State.Status}}' 容器名            # 只看状态
docker inspect -f '{{.NetworkSettings.IPAddress}}' 容器名  # 只看容器 IP
docker inspect -f '{{.State.Pid}}' 容器名                # 看宿主机上的 PID
docker inspect -f '{{json .Mounts}}' 容器名 | python -m json.tool   # 看挂载
```

> **`inspect -f` 是运维最常用的姿势**：输出是一大坨 JSON，用 Go 模板 `-f` 可以精确取出想要的那个字段 —— 写监控脚本、排障脚本全靠它。

#### 4.3.8 docker cp（数据互相拷贝）

```bash
docker cp 源 目标
```

> **课件原话**：源既可以是母机，也可以是容器。

```bash
# 母机 → 容器
echo "hello" > /tmp/a.txt
docker cp /tmp/a.txt web:/tmp/a.txt

# 容器 → 母机
docker cp web:/etc/nginx/nginx.conf /tmp/nginx.conf
cat /tmp/nginx.conf | head -5        # 检验：能看到内容

# 容器 → 容器（先落地再从落地拷）
docker cp web:/tmp/a.txt /tmp/b.txt
docker cp /tmp/b.txt web2:/tmp/b.txt
```

#### 4.3.9 查看数据卷

```bash
docker volume ls
docker volume inspect 卷名
```

#### 4.3.10 查看网络

```bash
docker network ls
docker network inspect 网络名
```

#### 4.3.11 提交容器为新镜像（docker commit）

```bash
docker commit 旧容器名 新镜像名
```

**完整实操**：

```bash
# 1) 起一个容器
docker run -d --name mynginx -p 8080:80 nginx

# 2) 进容器改点东西（装个工具、改个页面）
docker exec mynginx bash -c 'echo "<h1>my page</h1>" > /usr/share/nginx/html/index.html'

# 3) 提交成新镜像
docker commit mynginx mynginx:v1
docker images | grep mynginx        # 检验：能看到 mynginx:v1

# 4) 用新镜像起容器，验证改动被"固化"了
docker rm -f mynginx
docker run -d --name mynginx2 -p 8080:80 mynginx:v1
curl -s http://127.0.0.1:8080       # 期望看到 <h1>my page</h1>
```

> **★ 但生产不推荐 commit**，原因：
> ① **不可追溯** —— 没人知道镜像里到底改了什么（不像 Dockerfile 有每一步记录）；
> ② **体积会变大** —— 每次 commit 都会多叠一层，删掉的东西还在旧层里；
> ③ **不可复用** —— 换台机器没法重现。
>
> **正确做法是写 Dockerfile**（第三期 第二章）。commit 只适合临时调试、导出问题现场。

---

## 二、Docker 网络与通讯（课件第五章）

### 5.1 Docker 默认网络

#### 5.1.1 先看清 docker0 —— 虚拟交换机

```bash
docker network ls
```

**预期输出**：

```text
NETWORK ID     NAME      DRIVER    SCOPE
ae25b791ed71   bridge    bridge    local
f6ab74179aa7   host      host      local
a2e01d665969   none      null      local
```

**看那个 `docker0` 网桥**：

```bash
dnf install -y net-tools        # 若需要
ip link show type bridge
```

**预期输出**：

```text
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
group default
    link/ether 02:42:bd:a5:9c:24 brd ff:ff:ff:ff:ff:ff
```

> **课件原话**：**docker0 是一个虚拟桥接网络，即虚拟交换机。**

**veth-pair 技术（课件原话）**：**它是一对虚拟的、点对点直接链路。**

**把这两句串起来理解整个过程**：

```text
宿主机
  ├── docker0（虚拟交换机，有自己的 IP，通常是 172.17.0.1）
  │
  ├── veth 一对网线的一端  ←→  容器里的 eth0（网线的另一端）
  ├── veth 一对网线的一端  ←→  容器里的 eth0
  └── ...

所以：
· 每个容器都有一根"虚拟网线"（veth-pair）插到 docker0 这个虚拟交换机上
· 同一台宿主机上的容器，通过 docker0 二层互通
· 容器访问外网，靠 docker0 + iptables 的 SNAT（见 5.2）
```

**实操验证：看容器的网卡和 IP**

```bash
docker run -d --name tomcat01 -P tomcat:7
docker run -d --name tomcat02 -P tomcat:7

docker exec tomcat01 ip addr          # 容器里有一块 eth0，IP 形如 172.17.0.x
docker inspect -f '{{.NetworkSettings.IPAddress}}' tomcat01
docker inspect -f '{{.NetworkSettings.IPAddress}}' tomcat02

ip addr show docker0                  # 宿主机上的 docker0，IP 通常是 172.17.0.1
```

> **注意**：这两条 `docker run` 用的是 **`-P`（大写）**，也就是让 Docker **随机分配**宿主机端口。用 `docker port tomcat01` 可以看到实际映射到了哪个端口（一般是 32768 起）。

#### 5.1.2 四种网络模式

| 模式 | DRIVER | 说明（课件原话 + 补充） |
| --- | --- | --- |
| **Bridge** | `bridge` | **默认模式**（桥接，自己创建的网络也是用它）。此模式会为每一个容器分配、设置 IP 等，并将容器连接到一个 docker0 虚拟网桥，通过 docker0 网桥以及 iptables nat 表配置与宿主机通信 |
| **Host** | `host` | 容器**没有独立网络命名空间**，不会虚拟出自己的网卡、也不配置独立 IP，而是**直接使用宿主机的 IP 和端口** |
| **None** | `null` | 该模式**关闭了容器的网络功能**（一般不用） |
| **自定义（container）** | — | 加入另一个已存在容器的网络命名空间（了解即可） |

> **Host 模式要注意什么**：因为直接用宿主机网络，所以**不能做端口映射**（`-p` 会失效并可能报错），而且容器里 `ss -lntp` 看到的是**宿主机所有的端口**。适合做网络性能敏感的场景（少了 NAT 一层）。
>
> **None 模式**：给了你一个完全没有网络的容器，通常用于"我自己来配网络"的高级场景，或者做纯计算任务。

#### 5.1.3 自定义网络（★★ 重点）

> **课件原话**：现有的网络模式不一定适合所有场景，有时候我们要自定义一个网络。

```bash
docker network create --driver bridge --subnet 192.168.100.0/24 --gateway 192.168.100.1 qianshannet
```

```bash
docker run -d -P --name=tomcat04 --net qianshannet tomcat:7
docker run -d -P --name=tomcat05 --net qianshannet tomcat:7
```

**检验（这一步很关键）**：

```bash
docker network ls                                    # 能看到 qianshannet
docker network inspect qianshannet | grep -A3 Subnet # 能看到 192.168.100.0/24
docker inspect -f '{{.NetworkSettings.IPAddress}}' tomcat04   # 192.168.100.x
docker inspect -f '{{.NetworkSettings.IPAddress}}' tomcat05

# ★ 关键验证：同一个自定义网络里，容器可以互相 ping 通（默认用容器名就能通）
docker exec tomcat04 ping -c 2 tomcat05
```

> **★ 自定义网络 vs 默认 bridge 网络 —— 最重要的一个区别**：
>
> | | 默认 bridge | 自定义网络（bridge driver） |
> | --- | --- | --- |
> | 容器之间能否用**容器名**通信 | **不能**（只能靠 IP，而且 IP 会变） | **能**（内置 DNS 解析） |
> | 网段 | 固定 172.17.0.0/16 | 自己指定 |
> | 隔离 | 所有容器混在一起 | 按网络隔离，不同网络不互通 |
>
> **这一条是第三期 Compose 的基础**：Compose 里 `depends_on` + `networks` 能让 PHP 容器**用服务名直接连 MySQL**（`DB_HOST: mysql`），靠的就是自定义网络的 DNS。

**清理实验环境**：

```bash
docker rm -f tomcat01 tomcat02 tomcat04 tomcat05
docker network rm qianshannet
```

### 5.2 容器对外服务（SNAT / DNAT）

> **课件原话（两句，必须背）**：
> **① 容器可以上网的原理，是利用 iptables 的 SNAT 技术（SNAT = 源地址转换）。**
> **② 容器服务能被外部访问，是利用了 iptables 的 DNAT 技术（DNAT = 目标地址转换）。**

**把这两句展开**：

```text
【出方向 / 容器访问外网 → SNAT（源地址转换）】
  容器(172.17.0.3) → 请求发到 docker0 → 宿主机把【源地址】改成自己的 IP
  → 发到外网 → 回包回到宿主机 → 宿主机再转回容器
  为什么必须改：172.17.0.3 是私网地址，外网不知道怎么回包

【入方向 / 外部访问容器 → DNAT（目标地址转换）】
  外部用户访问 宿主机IP:8080
  → PREROUTING 链上被 DOCKER 链匹配到
  → 把【目标地址】改成 172.17.0.3:80
  → 转给容器，容器以为是直接连它
  这就是 -p 8080:80 背后真正发生的事
```

#### 看真实的 iptables NAT 规则

```bash
iptables -S -t nat
```

**实际输出**（课件里的完整规则，节选）：

```text
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P POSTROUTING ACCEPT
-P OUTPUT ACCEPT
-N DOCKER
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 192.168.100.0/24 ! -o br-cb00ad15b848 -j MASQUERADE
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A POSTROUTING -s 172.17.0.3/32 -d 172.17.0.3/32 -p tcp -m tcp --dport 8080 -j MASQUERADE
-A POSTROUTING -s 192.168.100.2/32 -d 192.168.100.2/32 -p tcp -m tcp --dport 8080 -j MASQUERADE
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A DOCKER -i br-cb00ad15b848 -j RETURN
-A DOCKER -i docker0 -j RETURN
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 32768 -j DNAT --to-destination 172.17.0.3:8080
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 32769 -j DNAT --to-destination 172.17.0.4:8080
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 32770 -j DNAT --to-destination 172.17.0.5:8080
-A DOCKER ! -i br-cb00ad15b848 -p tcp -m tcp --dport 32771 -j DNAT --to-destination 192.168.100.2:8080
-A DOCKER ! -i br-cb00ad15b848 -p tcp -m tcp --dport 32772 -j DNAT --to-destination 192.168.100.3:8080
```

**逐条读懂（这是本节的重点）**：

| 规则 | 含义 |
| --- | --- |
| `-N DOCKER` | 新建一条自定义链，叫 `DOCKER`（docker 自己的规则都放这） |
| `-A PREROUTING ... -j DOCKER` | **入方向**的包先跳进 DOCKER 链去匹配 |
| `-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE` | **SNAT**：来自 172.17.0.0/16 且不是从 docker0 出去的包，做地址伪装（这就是"容器能上网"） |
| `-A POSTROUTING -s 192.168.100.0/24 ! -o br-cb00ad15b848 -j MASQUERADE` | 自定义网络网段的 SNAT |
| `-A DOCKER ! -i docker0 -p tcp --dport 32768 -j DNAT --to-destination 172.17.0.3:8080` | **DNAT**：外部访问宿主机的 32768 端口 → 转发到容器 172.17.0.3 的 8080（这就是"外部能访问容器"） |
| `-A DOCKER -i docker0 -j RETURN` | 从 docker0 进来的包不再往下走 DOCKER 链（避免容器互访问被 NAT） |

> **两个关键词解释**：
> - **`MASQUERADE`** = SNAT 的一种特殊形式，会自动用出口网卡的 IP 做源地址（不用写死 IP，适合动态 IP 场景）
> - **`! -i docker0`** 里的 `!` 是"非"：表示"不是从 docker0 进来的"，也就是**来自外部**的流量

#### 完整验证实验：一个端口映射的完整链路

```bash
# 1) 起一个映射到固定端口的容器
docker run -d --name web -p 8080:80 nginx

# 2) 宿主机本地访问
curl -sI http://127.0.0.1:8080 | head -1        # 期望 HTTP/1.1 200 OK

# 3) 看 DNAT 规则是否生成
iptables -t nat -S DOCKER | grep 8080
# 期望看到类似：-A DOCKER ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80

# 4) 从另一台机器（或本机用真实 IP）访问
curl -sI http://<宿主机IP>:8080 | head -1       # 期望 200 OK

# 5) 如果第 4 步不通，按顺序查：
ss -lntp | grep 8080                            # 宿主机有没有在监听
firewall-cmd --list-all 2>/dev/null || iptables -L -n | head -20
# 云服务器还要查【安全组】是否放行 8080
```

> **★ 排障思路（面试高频）**：容器端口映射后外网访问不了，按这个顺序查：
> **① 容器本身是否 Up → ② `docker port` 看映射是否生成 → ③ 宿主机 `ss -lntp` 是否在听 → ④ 防火墙（firewalld/iptables）→ ⑤ 云安全组 → ⑥ 抓包 `tcpdump -i any -nn port 8080`**。

**清理**：

```bash
docker rm -f web
```

---

## 三、Docker 数据卷（课件第六章）

> **课件原话（为什么需要数据卷）**：**数据需要保存，需要持久化，需要方便编辑。**

### 6.1 数据覆盖（★ 两条规则，必须记准）

> **课件原文**：
> **① 如果是 Linux 母机的【空目录】挂载到容器里，容器中的目录数据不会复制到母机里，并且容器目录下现有的数据会被隐藏。**
> **② 如果是 Linux 母机的【有数据目录】挂载到容器里，容器中的目录数据不会复制到母机里，并且容器目录下的现有数据会被隐藏，会显示母机当前数据目录下的数据。**

**翻译成人话**：

```text
挂载 = 用母机的目录【盖住】容器里的目录。
· 母机目录是空的  → 容器里那个目录原来的内容【看不见了】，而且母机的空目录里也不会多出东西
· 母机目录有数据  → 显示母机的数据，容器里原来的内容同样【被盖住】
· 共同点：方向永远是"母机盖住容器"，容器里的东西【不会】被复制到母机
```

**实操验证（非常值得做一遍）**：

```bash
# 1) 准备一个空目录
mkdir -p /data/empty

# 2) 把空目录挂到 nginx 的网站目录上（容器里本来有 index.html）
docker run -d --name web1 -p 8081:80 -v /data/empty:/usr/share/nginx/html nginx

# 3) 看容器里那个目录 —— 原来的 index.html 被"盖"没了
docker exec web1 ls -l /usr/share/nginx/html
# 期望：空的（没有 index.html）

# 4) 访问会看到 403 Forbidden（因为 index.html 不存在）
curl -sI http://127.0.0.1:8081 | head -1

# 5) 母机目录里也不会凭空多出容器原来的文件
ls -l /data/empty
# 期望：还是空的
```

> **这个规则的实际意义**：**部署 MySQL 时，如果挂一个空目录到 `/var/lib/mysql`，MySQL 会以为"数据目录是空的"而执行初始化**（这正好是我们要的）；但如果你把空目录挂到 Nginx 的 `/usr/share/nginx/html`，网站就"打不开"了 —— 这不是坏了，是**被盖住了**。

### 6.2 三种挂载方式（★★ 面试必问）

课件原话：**一个叫具名挂载，一个是匿名挂载，一个是路径挂载。**

#### 方式一：匿名挂载（不写母机路径）

```bash
docker run -d -P --name=nginxt001 -v /etc/nginx nginx:1.28
```

> **课件批注**：`/etc/nginx` 指的是**容器的目录**，不是母机的目录。数据会挂载到 `/var/lib/docker/volumes` 目录下。

**检验（看它到底挂到哪了）**：

```bash
docker inspect -f '{{json .Mounts}}' nginxt001 | python -m json.tool
# 期望看到 Source 形如 /var/lib/docker/volumes/<一长串随机ID>/_data

docker volume ls
# 期望看到一个名字是一长串随机十六进制字符的卷（没有可读名字）
```

> **特点**：卷名是**随机字符串**，没有可读性，回头很难认出这个卷是谁的 —— 所以生产上基本不用匿名挂载。

#### 方式二：具名挂载（指定卷名，不指定母机路径）

```bash
docker run -d -P --name nginx002 -v nginx-config:/etc/nginx nginx:1.28
```

> **课件批注**：数据会挂载到 `/var/lib/docker/volumes` 目录下；**它与匿名挂载的区别是有"具体的目录名"**。

**检验**：

```bash
docker volume ls
# 期望看到 nginx-config 这个卷（名字可读！）

docker volume inspect nginx-config
# 期望看到 Mountpoint: /var/lib/docker/volumes/nginx-config/_data
```

> **怎么区分匿名和具名（一句话）**：`-v` 后面**只有容器路径**、没有冒号 → 匿名；`-v` 后面是 **`卷名:容器路径`**（卷名不以 `/` 开头）→ 具名。

#### 方式三：路径挂载 / 绑定挂载（指定母机具体路径）

```bash
docker run -d -P --name=nginx003 -v /data/nginx:/etc/nginx nginx:1.28
```

**检验**：

```bash
docker inspect -f '{{json .Mounts}}' nginx003 | python -m json.tool
# 期望 Source 就是 /data/nginx（我们自己指定的），Type 是 bind
ls -l /data/nginx        # 母机目录里能看到容器 /etc/nginx 的内容吗？答案是：看不到（被盖住的是容器那一侧）
```

> **这三种到底怎么选（★ 面试答法）**：
>
> | 方式 | 写法 | 母机路径 | 适合 | 生产推荐度 |
> | --- | --- | --- | --- | --- |
> | 匿名挂载 | `-v /容器路径` | 随机 ID，在 `/var/lib/docker/volumes/` | 临时测试 | 不推荐（名字不可读） |
> | **具名挂载** | `-v 卷名:/容器路径` | `/var/lib/docker/volumes/卷名/_data` | **数据库、需要 Docker 管理的数据** | **推荐** |
> | **路径挂载** | `-v /母机路径:/容器路径` | 你指定的任意路径 | **配置文件、网站代码、日志** | **推荐** |
>
> **一句话总结**：**「数据交给 Docker 管就用具名卷；需要自己在母机上直接编辑、看日志的，就用路径挂载。」**

#### 常用卷管理命令

```bash
docker volume ls                    # 列出所有卷
docker volume inspect 卷名          # 看卷详情（含 Mountpoint）
docker volume create 卷名           # 手动创建卷
docker volume rm 卷名               # 删除卷（有容器在用会报错）
docker volume prune                 # 清理所有没被使用的卷（★ 会删数据，生产慎用）
```

> **★ 重要提醒**：**`docker volume prune` 会删掉未被引用卷里的所有数据，且不可恢复。** 生产上执行前一定要先 `docker volume ls` 确认，或者干脆不用它。
>
> **一个高频面试题**：「容器删了，数据怎么才能不丢？」
> 答：**用 `-v` 挂载 —— 具名卷或母机路径都在容器之外，容器删了数据还在；只有放在容器可写层里的数据才会跟着容器消失。**

#### 综合实验：把三种挂载的存活情况一次验证完

```bash
# A. 不挂载（数据在可写层）
docker run -d --name t-a -p 8081:80 nginx
docker exec t-a bash -c 'echo A > /usr/share/nginx/html/a.txt'
docker rm -f t-a
docker run -d --name t-a2 -p 8081:80 nginx
docker exec t-a2 ls /usr/share/nginx/html       # a.txt 不见了 → 数据丢了

# B. 具名卷（数据在卷里）
docker rm -f t-a2
docker run -d --name t-b -p 8081:80 -v myhtml:/usr/share/nginx/html nginx
docker exec t-b bash -c 'echo B > /usr/share/nginx/html/b.txt'
docker rm -f t-b
docker run -d --name t-b2 -p 8081:80 -v myhtml:/usr/share/nginx/html nginx
docker exec t-b2 cat /usr/share/nginx/html/b.txt    # 输出 B → 数据还在

# C. 母机路径（数据在母机目录里）
docker rm -f t-b2
mkdir -p /data/myhtml && echo C > /data/myhtml/c.txt
docker run -d --name t-c -p 8081:80 -v /data/myhtml:/usr/share/nginx/html nginx
curl -s http://127.0.0.1:8081/c.txt                 # 输出 C → 母机直接改文件、容器立即可见

# 清理
docker rm -f t-c
docker volume rm myhtml
```

> **★ 实验 C 是路径挂载最大的优势**：改配置/改页面**不用进容器**，在母机上 `vim` 一下、`docker restart` 一下就行 —— 这也是"方便编辑"的意思。

---

## 四、Docker 可视化 —— Portainer（课件第七章）

> **课件原话**：**流行的软件，应该是 portainer。**

### 4.1 完整部署（课件给的三步 + 我给你补上验证）

#### 步骤 1：拉取镜像

```bash
docker pull portainer/portainer-ce
```

#### 步骤 2：新建数据目录

```bash
mkdir -p /data/portainer/data
```

> **为什么要有数据目录**：Portainer 的账号、密码、你配的所有东西都存在 `/data` 里。**不挂出来，容器重建就得重新配一遍。**

#### 步骤 3：启动容器

```bash
docker run -d -p 8000:8000 -p 9000:9000 \
  --name=portainer-ce --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /data/portainer/data:/data \
  portainer/portainer-ce
```

**两个端口分别干什么（课件原话）**：

| 端口 | 用途 |
| --- | --- |
| **8000** | 用于通信端口（Edge Agent 用） |
| **9000** | **后台管理端口**（浏览器访问这个） |

**`-v /var/run/docker.sock:/var/run/docker.sock` 是什么（★ 关键）**：

```text
/var/run/docker.sock 是 Docker 守护进程的"控制插座"。
把它挂进容器，等于把"操作 Docker 的能力"交给了 Portainer。
所以 Portainer 才能在网页上帮你创建容器、删镜像、看日志。

★ 安全提醒：能访问这个 sock = 等于宿主机 root 权限。
  所以 9000 端口千万不要直接暴露到公网，要加防火墙/反代+认证。
```

#### 步骤 4：访问与初始化

```bash
# 先确认容器起来了
docker ps | grep portainer
docker logs --tail 20 portainer-ce

# 确认端口在听
ss -lntp | grep -E '8000|9000'
```

浏览器访问：

```text
http://<宿主机IP>:9000
```

**首次访问要做的三件事**：

```text
1) 设置 admin 账号密码（密码要求 8 位以上）
2) 选择要管理的环境：选 "Docker" → 通常自动识别 local
   （因为我们已经挂载了 docker.sock，所以直接选 Local 就能连上）
3) 进入后左侧菜单：Containers / Images / Volumes / Networks / Stacks
   —— 对应我们前面学的所有命令
```

> **页面能做什么**：`Containers` 看/起/停/删容器、`Images` 拉/删镜像、`Volumes` 看数据卷、`Networks` 看网络、**`Stacks` 可以直接粘贴 compose 文件部署**（第三期内容）。

**检验（闭环）**：

```bash
# 在 Portainer 网页上创建一个测试容器，然后在命令行确认
docker ps
# 期望能看到网页上创建的那个容器 —— 说明 Portainer 真的能操作 Docker
```

### 4.2 常见坑

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 网页打不开 | 9000 没放行 / 容器没起 | `docker ps`、`ss -lntp`、防火墙、云安全组 |
| 提示连不上 Docker | 忘了挂 `docker.sock` | 重新 run，加上 `-v /var/run/docker.sock:/var/run/docker.sock` |
| 重启后配置全丢了 | 没挂 `/data` | 加上 `-v /data/portainer/data:/data` |
| 容器随机器重启没起来 | 没加 `--restart=always` | 加上重启策略 |
| **安全风险** | 9000 暴露到公网 | **只在内网开放，或加反向代理 + HTTPS + 认证** |

---

## 五、镜像仓库 —— Harbor（课件第八章）

### 5.1 为什么要私有仓库

> **课件原话**：**默认从 Docker 公司的 Docker Hub 上拉镜像。**

但生产上不能用 Docker Hub：
- 内网机器**上不了外网**
- 公司自己的业务镜像**不能公开**
- 拉取速度慢、可能被限流

**常见的私有仓库软件（课件列了三个）**：

| 软件 | 说明 |
| --- | --- |
| `docker registry` | Docker 官方的，最简单，只有最基础的功能 |
| **`harbor`** | **VMware 出的第三方，功能全（带 Web UI、权限、镜像扫描）** ← 主流 |
| `JFrog Artifactory`（杰蛙） | 第三方的，功能更强，收费 |

### 5.2 Harbor 完整部署（★★★ 完整复现）

#### 步骤 1：下载离线安装包

```bash
# 在能上网的机器上下载，再上传到服务器（比如 /opt 目录）
wget https://github.com/goharbor/harbor/releases/download/v2.14.2/harbor-offline-installer-v2.14.2.tgz
# 或者直接本地下好后上传
ls -lh /opt/harbor-offline-installer-v2.14.2.tgz
```

> **为什么用 offline 版**：offline 包里已经含所有镜像，不用现场联网拉，内网部署最稳。

#### 步骤 2：解压

```bash
cd /opt
tar xf harbor-offline-installer-v2.14.2.tgz
cd harbor
ls        # 期望看到 harbor.yml.tmpl、install.sh、harbor.v2.14.2.tar.gz 等
```

#### 步骤 3：修改配置

```bash
cp harbor.yml.tmpl harbor.yml
vim harbor.yml
```

**课件指出要改三处**：

```yaml
# ① hostname：改成你的域名或 IP
hostname: docker.qianshan.com

# ② 屏蔽 HTTPS 服务（因为是内网实验，用 http 就行）
# https:
#   port: 443
#   certificate: /your/certificate/path
#   private_key: /your/private/key/path

# ③ harbor_admin_password：设置管理员密码
harbor_admin_password: Harbor12345
```

**http 部分保持默认**：

```yaml
# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80
```

> **★ 生产环境必须用 HTTPS**：Harbor 官方明确要求"非 localhost 必须用 https"，否则 docker 客户端会拒绝推送。实验环境可以用 http + `insecure-registries` 绕过（下面第 6 步）。

#### 步骤 4：新建数据目录

```bash
mkdir -p /data/harbor
```

> 建议在 `harbor.yml` 里把 `data_volume` 改成 `/data/harbor`，这样镜像数据都落在你规划好的大盘上，不会把根分区写满。

#### 步骤 5：安装

```bash
./install.sh
```

**预期输出（关键几行）**：

```text
[Step 0]: checking if docker is installed ...
[Step 1]: checking docker-compose is installed ...
[Step 2]: loading Harbor images ...
[Step 3]: preparing environment ...
[Step 4]: preparing harbor configs ...
[Step 5]: starting Harbor ...
✔ ----Harbor has been installed and started successfully.----
```

**检验**：

```bash
docker ps | grep harbor      # 期望看到 harbor-core、harbor-portal、harbor-db、
                             # harbor-redis、nginx、registry 等一组容器
ss -lntp | grep ':80'        # 期望 nginx 在监听 80
```

浏览器访问：

```text
http://172.22.4.201/harbor/projects
（换成你自己的 hostname / IP）
默认账号：admin，密码就是 harbor.yml 里设的那个
```

#### 步骤 6：让 docker 信任这个 http 仓库（insecure-registries）

**课件原话**：在 `/etc/docker/daemon.json` 增加一行。

```bash
vi /etc/docker/daemon.json
```

**改完的文件长这样**（课件实际内容）：

```json
{
    "insecure-registries": ["172.22.4.201", "docker.qianshan.com"],
    "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.imgdb.de",
    "https://docker-0.unsee.tech",
    "https://docker.hlmirror.com",
    "https://docker.1ms.run",
    "https://func.ink",
    "https://lispy.org",
    "https://docker.xiaogenban1993.com"
    ]
}
```

```bash
systemctl restart docker
docker info | grep -A3 "Insecure Registries"
# 期望看到 172.22.4.201 和 docker.qianshan.com
```

> **`insecure-registries` 是什么**：告诉 docker "这个仓库用 http（没有证书）也是安全的，允许连"。**只应该填自己内网的仓库地址**，千万别填公网地址 —— 那等于关掉了 TLS 校验。

#### 步骤 7：让机器能解析 hostname

**课件实际踩的坑**：

```text
[root@r810-200 ~]# ping docker.qianshan.com
ping: docker.qianshan.com: Name or service not known
```

**解决（改 /etc/hosts）**：

```bash
vi /etc/hosts
```

```text
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.22.4.201 docker.qianshan.com
```

```bash
ping -c 2 docker.qianshan.com     # 检验：能 ping 通
```

> **生产做法**：内网有 DNS 就配 DNS 记录，`/etc/hosts` 只适合两三台机器的实验环境。

#### 步骤 8：登录、打标签、推送、拉取（完整闭环）

```bash
# 1) 登录（用户名 admin，密码是 harbor.yml 里设的那个）
docker login docker.qianshan.com
# Username: admin
# Password:
# 期望：Login Succeeded
```

```bash
# 2) 给本地镜像打标签（格式：仓库地址/项目名/镜像名:标签）
docker tag redis:6.2.21 docker.qianshan.com/library/redis:v6.2.21
docker images | grep qianshan      # 检验：多出一条带仓库地址的 tag
```

```bash
# 3) 推送
docker push docker.qianshan.com/library/redis:v6.2.21
```

**预期输出（关键几行）**：

```text
The push refers to repository [docker.qianshan.com/library/redis]
...
v6.2.21: digest: sha256:xxxx size: 1234
```

```bash
# 4) 拉取（先删本地，再拉回来，才算真验证）
docker rmi docker.qianshan.com/library/redis:v6.2.21
docker pull docker.qianshan.com/library/redis:v6.2.21
docker images | grep qianshan
```

> **★ 为什么必须"先删再拉"**：如果本地已经有这个镜像，`docker pull` 可能什么都不做（层都在本地），**你根本不知道仓库里到底有没有**。**先 rmi 再 pull 才是真验证** —— 这也是面试时"你怎么确认镜像推送成功"的标准答法。

**网页验证**：

```text
登录 http://172.22.4.201/harbor/projects
→ 进入 library 项目 → 仓库列表里应该能看到 redis
→ 点进去能看到 tag v6.2.21 和它的 digest、大小
```

#### 步骤 9：Harbor 的日常运维命令

```bash
cd /opt/harbor

docker compose ps          # 看 harbor 各组件状态
docker compose stop        # 停止
docker compose start       # 启动
docker compose down        # 停止并删除容器（数据在 /data/harbor，不会丢）
docker compose up -d       # 重新拉起

# 修改配置后（比如改了 harbor.yml）要重新生成配置并重启
./prepare
docker compose up -d
```

> **注意**：Harbor 2.x 之后用的是 `docker compose`（子命令形式）；老版本教程里写 `docker-compose`，两者别混。

### 5.3 Harbor 常见坑

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `docker login` 报 `http: server gave HTTP response to HTTPS client` | 没配 `insecure-registries` | 加进 `daemon.json` 并 `systemctl restart docker` |
| `docker push` 报 `denied: requested access to the resource is denied` | 没登录，或项目/权限不对 | 先 `docker login`，确认项目存在、账号有推送权限 |
| `ping: ... Name or service not known` | 主机名解析不了 | 改 `/etc/hosts` 或配 DNS |
| `docker push` 报 `no such host` | 同上 | 同上 |
| 安装脚本报 `Need to install docker-compose` | 缺 compose | 见第一期 2.3 装 compose |
| 磁盘被 Harbor 写满 | 数据默认落在根分区 | 在 `harbor.yml` 里把 `data_volume` 指到大盘 |
| 网页 502 / 容器反复重启 | 内存不足 | Harbor 至少需要 2G 可用内存，4G 更稳 |

---

## 六、本期闭环自检清单

| 检查项 | 命令 | 通过标准 |
| --- | --- | --- |
| 版本信息看得懂 | `docker version` | 能指出 Client/Server/containerd/runc |
| 运行信息看得懂 | `docker info` | 能报出存储驱动、cgroup 驱动、根目录 |
| 能拉/搜/存镜像 | `docker search`、`docker pull`、`docker save` | 都成功 |
| 镜像能离线搬运 | `docker save -o x.tar` + `docker load -i x.tar` | 在另一台机器 load 成功 |
| 镜像能批量删 | `docker rmi -f $(docker images -aq)` | 清空 |
| run 参数都懂 | `docker run -d --name x -e .. -p .. -v .. 镜像` | 能逐参数解释 |
| 容器能启停 | `stop` / `start` / `restart` / `kill` | 状态在 up / exited 间切换 |
| 能看容器状态 | `ps` / `ps -a` / `ps -qa` | 三条输出差异能说清 |
| 能暂停恢复 | `pause` / `unpause` | curl 从超时恢复成 200 |
| 能进容器 | `exec -it x bash` | 提示符变成容器 ID |
| 能看日志 | `logs -f -t --tail 10 x` | 实时滚动 |
| 能查详情 | `inspect -f '{{.State.Pid}}' x` | 取到字段 |
| 能互拷文件 | `docker cp` 双向 | 文件确实到了 |
| 能提交镜像 | `docker commit` | 新 tag 起容器能看到改动 |
| 看懂 docker0 | `ip link show type bridge` | 找到 docker0 |
| 看懂网络模式 | `docker network ls` | bridge/host/none 三种 |
| 能建自定义网络 | `docker network create --subnet ...` | 两个容器能**用容器名**互 ping |
| 看懂 NAT 规则 | `iptables -S -t nat` | 能指出 MASQUERADE（SNAT）和 DNAT 各一行 |
| 端口映射能闭环 | 从外部 `curl http://IP:8080` | 200 OK |
| 三种挂载都会 | 匿名 / 具名 / 路径 | 能说出各自挂到哪 |
| 数据卷能存活 | 删容器重建后数据还在 | 具名卷和路径挂载都验证过 |
| Portainer 能用 | 浏览器 9000 | 能看到并操作容器 |
| Harbor 能推拉 | `login` + `tag` + `push` + `rmi` + `pull` | 先删再拉成功 |

---

## 七、自测问题（闭卷）

- [ ] `docker run` 和 `docker start` 的区别？为什么第二次启动要用 start？
- [ ] `docker exec` 和 `docker attach` 的区别？为什么推荐 exec？
- [ ] `docker save` 和 `docker export` 的区别？
- [ ] `docker pause` 和 `docker stop` 的区别？
- [ ] `docker commit` 为什么生产不推荐？正确做法是什么？
- [ ] 为什么容器里 `ll` 会报 command not found？
- [ ] `docker ps`、`ps -a`、`ps -qa` 分别输出什么？
- [ ] 什么是 docker0？什么是 veth-pair？两者怎么配合？
- [ ] bridge / host / none 三种网络模式的区别？
- [ ] 自定义网络比默认 bridge 网络强在哪？（提示：容器名解析）
- [ ] 容器为什么能上网？（SNAT）
- [ ] 外部为什么能访问容器？（DNAT）
- [ ] `iptables -S -t nat` 里 `MASQUERADE` 和 `DNAT` 各对应什么？
- [ ] 端口映射后外网访问不了，按什么顺序排查？
- [ ] 母机空目录挂进容器，容器里原来的数据会怎样？
- [ ] 匿名挂载、具名挂载、路径挂载的区别？各自数据存在哪？
- [ ] 容器删了数据怎么才能不丢？
- [ ] Portainer 为什么要挂 `/var/run/docker.sock`？有什么安全风险？
- [ ] Harbor 部署的关键步骤有哪些？改 `harbor.yml` 要改哪三处？
- [ ] 为什么要配 `insecure-registries`？生产环境该怎么处理？
- [ ] 怎么验证镜像真的推到仓库了？（先 rmi 再 pull）

---

## 八、与第三期的衔接

```text
第二期（本篇）学到的              第三期会变成什么
─────────────────────────────────────────────────────────
docker run 一条条敲        →    写成 docker-compose.yaml，一条 up -d 全起来
docker network create      →    compose 里的 networks: 段
docker run -v 具名卷/路径   →    compose 里的 volumes: 段
docker run -e 环境变量      →    compose 里的 environment: 段（还能用 .env 抽出来）
手动保证启动顺序            →    compose 里的 depends_on + healthcheck
私有仓库 Harbor            →    构建好的镜像推上去，多台机器共享
```

> **下一步**：先把本期的**三种挂载实验**和**自定义网络实验**在虚拟机里跑一遍（这两个最容易"看懂了但没真懂"），再进第三期。