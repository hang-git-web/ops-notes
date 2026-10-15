# Docker 第二期：常用命令、网络、数据卷与私有仓库

> **本篇定位**：第一篇装好了 Docker，这一篇是**真正开始用**：把命令练熟、把网络搞懂、把数据存住、把镜像推到自己的仓库。
> **环境**：Rocky Linux 8.10 + docker 26.1.3（4 核 / 3.8G），CentOS 7.9 同样适用。
> **承接关系**：本篇的 5.x（网络）、6.x（数据卷）是第三期 Compose 的基础 —— **Compose 里的 `networks:` 和 `volumes:` 就是这两节的自动化版本**。

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

## 一、Docker 常见命令

### 4.1 管理命令

#### docker version（看版本）

```bash
docker version
```

输出看后面

#### docker info（看运行信息）

```bash
docker info
```

输出看后面

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

输出看后面


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

**容器的三个状态**：`up`（运行中）、`exited`（已停止）、`deleted`（已删除）。

#### 4.3.1 docker run / stop / start / restart / kill

**`docker run` 的本质**：首次运行时，它会检测本地是否有镜像 —— **有的话直接 start，没有的话先 pull，再 start。**

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
实操过程从后面找


> **pause 和 stop 的区别**：`pause` 用 cgroup freezer **冻结进程**（不释放资源，恢复后继续跑）；`stop` 是**结束进程**（容器变 Exited，恢复要 start 重新启动）。面试问"怎么临时冻结一个容器"就是 `pause`。

#### 4.3.4 docker exec / attach（进容器）

**课件实际演示**：



**exec 和 attach 的区别**：

```bash
docker exec -it 容器ID bash      # 新起一个进程（新终端），不会影响容器主进程
docker attach 容器ID             # 直接接到容器主进程的终端上
                                 # 课件原话：不启动新的终端，如果按 ctrl+c，容器会自动退出
```

实操过程看后面

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

>源既可以是母机，也可以是容器。

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









## 五、镜像仓库 —— Harbor

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











——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
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






```bash
docker run -d --name web -p 8080:80 nginx
curl -sI http://127.0.0.1:8080 | head -1     # 期望 HTTP/1.1 200 OK

docker pause web
curl -sI --max-time 5 http://127.0.0.1:8080 | head -1   # 期望卡住/超时（没有响应）

docker unpause web
curl -sI http://127.0.0.1:8080 | head -1     # 期望恢复 200 OK

docker rm -f web
```








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