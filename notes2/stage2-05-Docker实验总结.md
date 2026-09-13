# Docker 基础理论:概念、命令与持久化

> 本文回答四个问题:Docker 有什么用、镜像和容器到底有什么区别、那些命令在操作什么、输出里的字段怎么看。
> 文中所有输出均来自实际实验环境(CentOS 7 + Docker 26.1.4)。

## 一、Docker 解决什么问题

手工装环境(LNMP)的痛点:

- 换一台机器要重装一遍,版本还不一定一致("在我机器上能跑")
- 组件之间会冲突,比如 PHP 5.4 连 MySQL 8 的认证插件、字符集问题
- 装坏了很难干净卸载,容易留下残留配置

Docker 的思路:**把"应用 + 依赖 + 配置"打包成一个镜像,到哪台机器都一模一样地跑起来。**

| 对比 | 虚拟机 | 容器 |
| --- | --- | --- |
| 装了什么 | 完整操作系统 | 只有应用和依赖 |
| 体积 | 几 GB | 几十 MB 到几百 MB |
| 启动速度 | 分钟级 | 秒级 |
| 隔离程度 | 完全隔离 | 共享宿主机内核,进程级隔离 |
| 一台机器能跑多少 | 几个 | 几十个 |

后续计划:**用 docker compose 把 LNMP 重做一遍**,让 nginx、PHP、MySQL 各自跑在独立容器里,一条命令全部起停,不再手动解决版本冲突。

## 二、镜像与容器

### 1. 一句话定义

> **镜像 = 打包好的运行环境(模板),容器 = 用这个模板跑起来的实例。**

| 镜像 | 容器 |
| --- | --- |
| 类(class) | 对象(实例) |
| 安装包 | 安装后运行的进程 |
| 菜谱 | 做出来的那道菜 |
| 只读 | 可读可写(多一层可写层) |
| `docker images` 查看 | `docker ps` 查看 |

### 2. 证据一:一个镜像可以起多个容器

实际输出:

```bash
[root@bogon ~]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
nginx         latest    5f52b9aca7a7   10 days ago    170MB
hello-world   latest    e2ac70e7319a   5 months ago   10.1kB

[root@bogon ~]# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                      PORTS                                   NAMES
34c8e8c84619   nginx         "/docker-entrypoint.…"   25 minutes ago   Up 25 minutes               0.0.0.0:8080->80/tcp, :::8080->80/tcp   mynginx
d2f051f0435e   hello-world   "/hello"                 30 minutes ago   Exited (0) 30 minutes ago                                           nostalgic_jackson
db9af3c837bb   hello-world   "/hello"                 32 minutes ago   Exited (0) 32 minutes ago                                           hungry_feistel
```

读法:

- `docker images` 里只有 **2 个镜像**
- `docker ps -a` 里却有 **3 个容器**
- 其中 hello-world 这一个镜像,对应了 2 个容器(`nostalgic_jackson`、`hungry_feistel`)

结论:镜像可以反复使用,每次 `run` 都生成一个独立的新容器。

### 3. 证据二:镜像大小差异说明它"不装整个系统"

同样是在上面那份 `docker images` 输出里:

| 镜像 | 大小 | 说明 |
| --- | --- | --- |
| hello-world | 10.1kB | 里面只有一个极简的可执行文件 |
| nginx | 170MB | 包含精简系统库 + nginx 程序 + 配置 |

两者都不包含完整操作系统——容器共享宿主机内核,镜像只带应用和它需要的那些文件。

### 4. 证据三:镜像由多个层叠加而成

拉取 nginx 时的实际过程:

```text
latest: Pulling from library/nginx
6310eb16bf42: Pull complete
956faab5efb3: Pull complete
a44b5c8be616: Pull complete
02fc02c4ab8d: Pull complete
c12f394dea35: Pull complete
07db7bf2649b: Pull complete
f340c1b7c1d6: Pull complete
Digest: sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4
Status: Downloaded newer image for nginx:latest
```

每一行 `Pull complete` 就是一个层(layer),层可以复用,所以拉取基于相同底层的镜像会更快。

### 5. 可写层:容器数据和镜像的关系

```text
容器 = 只读的镜像层 + 一层可写层
```

- 你在容器里改文件,改的是**可写层**
- 容器删除时可写层一起消失,**镜像不受影响**
- 想要数据不丢,必须挂载卷(见第五节)

## 三、命令在操作什么

Docker 命令只围绕两类对象:**镜像(模板)** 和 **容器(实例)**。

| 命令 | 操作对象 | 作用 |
| --- | --- | --- |
| `docker pull` | 镜像 | 从仓库下载 |
| `docker images` | 镜像 | 查看本地镜像 |
| `docker rmi` | 镜像 | 删除镜像 |
| `docker run` | 容器 | 创建 + 启动 |
| `docker start/stop/restart` | 容器 | 启停已有容器 |
| `docker ps` / `ps -a` | 容器 | 查看运行中 / 全部容器 |
| `docker logs` | 容器 | 查看容器输出 |
| `docker exec -it` | 容器 | 进入容器内部 |
| `docker rm` | 容器 | 删除容器 |
| `docker version` / `info` | 服务 | 查看版本 / 运行信息 |

### 1. `docker run hello-world` 背后做了五件事

1. 检查本地有没有该镜像 —— 没有
2. 从仓库下载镜像(`Pulling from library/hello-world`)
3. 用镜像**创建**容器
4. **启动**容器,执行里面的 `/hello`
5. 程序打印完退出,容器状态变为 `Exited (0)`

### 2. `docker run -d --name mynginx -p 8080:80 nginx` 参数说明

| 参数 | 含义 |
| --- | --- |
| `-d` | 后台运行 |
| `--name mynginx` | 指定容器名(不写会随机生成) |
| `-p 8080:80` | 端口映射,格式是**宿主机端口:容器端口** |
| `-v 宿主目录:容器目录` | 挂载目录,用于数据持久化 |
| `--restart=always` | 容器随 Docker 自动重启 |
| `--rm` | 容器退出后自动删除,适合一次性任务 |
| `-it` | 交互式运行,常用于临时调试 |

### 3. `run` 与 `start`、`ps` 与 `ps -a` 的区别

实际输出对照:

```bash
[root@bogon ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                                   NAMES
34c8e8c84619   nginx     "/docker-entrypoint.…"   25 minutes ago   Up 25 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   mynginx
```

```bash
[root@bogon ~]# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                      PORTS                                   NAMES
34c8e8c84619   nginx         "/docker-entrypoint.…"   25 minutes ago   Up 25 minutes               0.0.0.0:8080->80/tcp, :::8080->80/tcp   mynginx
d2f051f0435e   hello-world   "/hello"                 30 minutes ago   Exited (0) 30 minutes ago                                           nostalgic_jackson
db9af3c837bb   hello-world   "/hello"                 32 minutes ago   Exited (0) 32 minutes ago                                           hungry_feistel
```

| 命令 | 区别 |
| --- | --- |
| `docker run` | 创建 + 启动,容器名已存在会报 `Conflict` |
| `docker start` | 启动已存在的容器 |
| `docker ps` | 只看运行中的容器(上面只有 1 条) |
| `docker ps -a` | 看全部容器(上面有 3 条) |

## 四、输出字段解读

### 1. `docker pull` 的过程

```text
Unable to find image 'nginx:latest' locally     ← 本地没有,开始下载
latest: Pulling from library/nginx              ← 从官方仓库拉取
6310eb16bf42: Pull complete                     ← 一个层下载完成
Digest: sha256:05b8cb...                        ← 镜像内容唯一指纹
Status: Downloaded newer image for nginx:latest ← 拉取完成
```

### 2. `docker images` 字段

实际输出:

```bash
[root@bogon ~]# docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
nginx         latest    5f52b9aca7a7   10 days ago    170MB
hello-world   latest    e2ac70e7319a   5 months ago   10.1kB
```

| 字段 | 含义 |
| --- | --- |
| REPOSITORY | 镜像名(nginx、hello-world) |
| TAG | 版本标签,这里是 latest |
| IMAGE ID | 镜像唯一编号 |
| CREATED | 构建时间(相对时间) |
| SIZE | 占用大小 |

### 3. `docker ps` 字段

```bash
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                                   NAMES
34c8e8c84619   nginx     "/docker-entrypoint.…"   25 minutes ago   Up 25 minutes   0.0.0.0:8080->80/tcp, :::8080->80/tcp   mynginx
```

| 字段 | 含义 |
| --- | --- |
| CONTAINER ID | 容器唯一编号 |
| IMAGE | 用哪个镜像启动 |
| COMMAND | 容器内的主进程 |
| CREATED | 创建时间 |
| STATUS | `Up 25 minutes` 表示已运行 25 分钟 |
| PORTS | `0.0.0.0:8080->80/tcp` 表示宿主机 8080 → 容器 80;`:::8080` 是 IPv6 |
| NAMES | 容器名 |

### 4. `docker info` 关键字段

实际输出(节选完整内容):

```bash
[root@bogon ~]# docker info
Client: Docker Engine - Community
 Version:    26.1.4
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.14.1
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.27.1
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 3
  Running: 1
  Paused: 0
  Stopped: 2
 Images: 2
 Server Version: 26.1.4
 Storage Driver: overlay2
  Backing Filesystem: xfs
  Supports d_type: true
  Using metacopy: false
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: d2d58213f83a351ca8f528a95fbd145f5654e957
 runc version: v1.1.12-0-g51d5e94
 init version: de40ad0
 Security Options:
  seccomp
   Profile: builtin
 Kernel Version: 3.10.0-1160.el7.x86_64
 Operating System: CentOS Linux 7 (Core)
 OSType: linux
 Architecture: x86_64
 CPUs: 1
 Total Memory: 972.3MiB
 Name: bogon
 ID: 5a1d1a62-0ff9-4298-b3c9-4d12c03e371a
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Registry Mirrors:
  https://docker.m.daocloud.io/
  https://dockerproxy.com/
  https://docker.mirrors.ustc.edu.cn/
  https://docker.nju.edu.cn/
  https://iju9kaj2.mirror.aliyuncs.com/
  http://hub-mirror.c.163.com/
  https://cr.console.aliyun.com/
  https://hub.docker.com/
  http://mirrors.ustc.edu.cn/
 Live Restore Enabled: false
```

| 字段 | 说明 | 本机实际值 |
| --- | --- | --- |
| Server Version | Docker 服务端版本 | 26.1.4 |
| Containers | 容器总数 / 运行 / 暂停 / 停止 | 3 个(1 运行、2 停止) |
| Images | 本地镜像数量 | 2 |
| Storage Driver | 镜像分层与可写层的管理方式 | overlay2(xfs) |
| Logging Driver | 容器日志的默认方式 | json-file |
| Cgroup Version | 资源限制机制 | v1(CentOS 7 内核较老) |
| Docker Root Dir | 镜像与容器数据的存放位置 | /var/lib/docker |
| Operators System / Kernel | 宿主机系统信息 | CentOS 7 / 3.10.0 |
| CPUs / Total Memory | 宿主机资源 | 1 核 / 972.3 MiB |
| Registry Mirrors | 生效的镜像加速地址 | 9 条(见下方说明) |

### 5. 用一份输出交叉验证另一份

```text
docker ps -a  → 3 行      ←→ docker info 的 Containers: 3
docker ps     → 1 行      ←→ docker info 的 Running: 1
docker images → 2 行      ←→ docker info 的 Images: 2
```

三组数字完全对得上,说明这几条命令看到的是同一份状态,只是呈现角度不同。

### 6. 关于镜像加速列表的一个注意点

`docker info` 里显示 9 条加速地址,其中:

- ✅ 可用或可能可用:`docker.m.daocloud.io`、`docker.nju.edu.cn`、`iju9kaj2.mirror.aliyuncs.com`(建议换成自己控制台申请的地址)
- ❌ 已停服:`dockerproxy.com`、`docker.mirrors.ustc.edu.cn`、`hub-mirror.c.163.com`
- ❌ 不是加速器地址:`cr.console.aliyun.com`(控制台地址)、`hub.docker.com`(官方仓库)、`mirrors.ustc.edu.cn`

Docker 会按顺序尝试,第一个可用就生效,但排在前面那些失效地址会浪费超时时间,建议精简成 3 条。

## 五、容器是"临时的"吗

### 1. 两个"持久"要分开看

- **容器本身**:创建后会一直保存在本地,不会自动消失
- **容器内的数据**:跟随容器,容器删除就一起丢失

证据就在上面的输出里:`mynginx` 已经 `Up 25 minutes`,两个 hello-world 容器已退出 30 分钟,依然留在 `docker ps -a` 中。

### 2. 容器的四个状态

```text
创建(create) → 运行(Running) → 停止(Exited,仍在本地) → 删除(rm,彻底消失)
```

| 操作 | 容器还在吗 | 数据还在吗 |
| --- | --- | --- |
| `docker stop` | 在 | 在 |
| 宿主机重启 | 在(默认不自动启动) | 在 |
| `docker start` | 继续使用 | 在 |
| `docker rm` | 消失 | **消失** |
| `docker container prune` | 清理所有已退出容器 | **消失** |

### 3. 哪些内容存在本地

| 内容 | 位置 | 删容器后 | 删镜像后 |
| --- | --- | --- | --- |
| 镜像 | `/var/lib/docker/overlay2` | 保留 | 删除 |
| 容器记录 | `/var/lib/docker/containers` | 删除 | - |
| 容器可写层 | `/var/lib/docker/overlay2` | **删除** | - |
| 数据卷 volume | `/var/lib/docker/volumes` | 保留 | 保留 |
| 绑定挂载 `-v /data:/xx` | 宿主机目录 | 保留 | 保留 |

### 4. 三种持久化做法

**数据持久化:挂载卷**

```bash
mkdir -p /data/nginx/html
echo "<h1>我的页面</h1>" > /data/nginx/html/index.html

docker run -d --name web -p 8080:80 \
  -v /data/nginx/html:/usr/share/nginx/html nginx
```

容器删了重建,页面内容仍然在。

**环境持久化:写 Dockerfile**

把安装和配置步骤写进 Dockerfile 构建成新镜像,以后新容器自带这些改动。
`docker commit` 也能保存,但不推荐(不可追溯、体积大)。

**开机自启:加重启策略**

```bash
docker run -d --name web --restart=always -p 8080:80 \
  -v /data/nginx/html:/usr/share/nginx/html nginx
```

`systemctl enable docker` 只保证 Docker 服务自启,**容器默认不会跟着启动**。

### 5. 一个比喻:容器是"临时工",不是"宠物"

- 临时工随时可以换新的(容器随时可删可重建)
- 但他脑子里的东西会一起消失,重要资料要放进外面的档案柜(挂载卷)
- 招聘标准写成文档留存(Dockerfile)

**容器当消耗品,数据放外面**——这是 Docker 最核心的使用观念。

### 6. 查看存储情况的命令

```bash
docker info | grep "Docker Root Dir"
du -sh /var/lib/docker
docker system df
docker inspect mynginx | grep -i upperdir
```

## 六、三个验证小实验

**实验一:一个镜像,多个容器**

```bash
docker run -d --name a -p 8081:80 nginx
docker run -d --name b -p 8082:80 nginx
docker ps
```

两个端口都能访问,证明"一镜像多容器"。

**实验二:证明容器内的改动会随容器消失**

```bash
docker exec -it mynginx bash -c 'echo hello > /usr/share/nginx/html/a.txt'
docker rm -f mynginx
docker run -d --name mynginx -p 8080:80 nginx
docker exec mynginx ls /usr/share/nginx/html      # a.txt 已消失
```

**实验三:证明挂载后数据能保留**

```bash
docker rm -f mynginx
docker run -d --name mynginx -p 8080:80 \
  -v /data/nginx/html:/usr/share/nginx/html nginx

echo hello > /data/nginx/html/b.txt
docker rm -f mynginx
docker run -d --name mynginx -p 8080:80 \
  -v /data/nginx/html:/usr/share/nginx/html nginx

ls /data/nginx/html                                # b.txt 还在
```

## 七、自测问题

- [ ] 用一句话说清镜像和容器的关系
- [ ] `docker run` 和 `docker start` 有什么区别
- [ ] 为什么 `docker ps` 看不到 hello-world 容器
- [ ] `0.0.0.0:8080->80/tcp` 这一列表示什么
- [ ] 容器停止后,容器内的数据还在吗?删除后呢?
- [ ] 想让数据不丢,应该用什么参数
- [ ] `docker info` 里哪个字段能看出镜像加速是否生效
- [ ] 容器为什么"重启宿主机后没有自动启动"