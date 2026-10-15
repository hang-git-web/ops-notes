# Docker 常用命令（课件第四章）

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
