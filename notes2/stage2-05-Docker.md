# Docker 安装与容器运行笔记(CentOS 7)

> 环境:CentOS 7,使用阿里云 Docker 仓库安装 docker-ce。
> 提醒:公共镜像加速地址失效非常快,文末整理了有效的配置方式与常见坑。

## 一、安装 Docker

### 1. 卸载旧版本(如果有)

```bash
sudo yum remove -y docker \
  docker-client \
  docker-client-latest \
  docker-common \
  docker-latest \
  docker-latest-logrotate \
  docker-logrotate \
  docker-engine
```

### 2. 安装依赖工具

```bash
sudo yum install -y yum-utils
```

### 3. 添加阿里云的 Docker 官方仓库

```bash
sudo yum-config-manager \
  --add-repo \
  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 4. 安装 Docker

```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io
```

> 需要 compose 的话可以一起装:`docker-compose-plugin`。

### 5. 启动 Docker 并设置开机自启

```bash
systemctl start docker
systemctl enable docker
systemctl status docker
```

### 6. 配置镜像加速

创建或编辑 `/etc/docker/daemon.json`:

```bash
mkdir -p /etc/docker
vim /etc/docker/daemon.json
```

```json
{
  "registry-mirrors": [
    "https://iju9kaj2.mirror.aliyuncs.com",
    "https://docker.m.daocloud.io",
    "https://docker.nju.edu.cn"
  ]
}
```

说明:

- `https://<你的ID>.mirror.aliyuncs.com` 换成自己在阿里云控制台申请的专属加速地址(容器镜像服务 ACR → 镜像工具 → 镜像加速器),专属地址最稳定
- 加速地址**按顺序尝试**,建议把自己可用的放前面
- 不要填 `https://hub.docker.com`、`https://cr.console.aliyun.com` 这类地址,它们不是镜像加速器
- 部分公共镜像源(如 USTC、dockerproxy、163 等)已陆续停止服务,填了会拖慢甚至导致拉取失败

修改后重启 Docker 生效:

```bash
systemctl daemon-reload
systemctl restart docker
```

### 7. 验证安装

```bash
docker version
docker info
```

确认加速器已生效:

```bash
docker info | grep -A3 "Registry Mirrors"
```

## 二、运行容器

### 1. 跑一个测试容器

```bash
docker run hello-world
```

看到 `Hello from Docker!` 说明安装和网络都正常。

### 2. 查看本地镜像与容器

```bash
docker images     # 查看本地镜像
docker ps         # 查看正在运行的容器
docker ps -a      # 查看所有容器,包括已退出的
```

### 3. 运行 nginx 网站

```bash
docker run -d --name mynginx -p 8080:80 nginx
```

参数说明:

| 参数 | 含义 |
| --- | --- |
| `-d` | 后台运行(detach) |
| `--name mynginx` | 给容器起个名字,方便后续管理 |
| `-p 8080:80` | 端口映射,**宿主机端口:容器端口** |
| `nginx` | 使用的镜像名,本地没有会自动从仓库拉取 |

浏览器访问 `http://你的服务器IP:8080`,看到 nginx 欢迎页即成功。

### 4. 验证与排错

```bash
docker ps                      # 确认容器在运行
docker logs mynginx            # 查看容器日志
docker port mynginx            # 查看端口映射
```

如果浏览器打不开,检查:

```bash
firewall-cmd --permanent --add-port=8080/tcp   # 放行端口
firewall-cmd --reload
```

> 云服务器还要在控制台安全组里放行 8080。

## 三、收尾该怎么做

### 1. 分三种情况

| 场景 | 要不要停 Docker | 说明 |
| --- | --- | --- |
| 马上继续做下一个实验(比如 compose 搭 LNMP) | 不停 | 保持运行,直接接着用 |
| 今天到此为止,但要保留环境 | 停容器即可,Docker 服务可停可不停 | 已 enable,开机自动拉起 |
| 生产服务器 | 永远不要停 | 容器要持续提供服务 |

### 2. 比"停 Docker 服务"更重要的是先优雅停容器

直接 `systemctl stop docker` 等于把所有容器的进程一次性杀掉,对 nginx 无所谓,但如果里面有 MySQL 这类在写数据的服务,就可能留下不干净的状态。规范顺序是先停容器,再决定停不停服务:

```bash
# 1. 优雅停止运行中的容器
docker stop mynginx

# 2. 确认状态
docker ps            # 应该为空
docker ps -a         # 能看到 mynginx 是 Exited

# 3. 如果实验做完了,清理掉不再需要的容器和镜像
docker rm mynginx d2f051f0435e db9af3c837bb
docker rmi hello-world

# 4. 看看占了多少空间
docker system df

# 5. 确实不需要 Docker 继续跑,再停服务(可选)
systemctl stop docker
```

### 3. 几个容易误解的点

- `docker stop mynginx` 只是让容器停下来,容器和它可写层里的数据都还在,下次 `docker start mynginx` 就恢复
- `docker rm` 才是真正的删除,可写层数据一起没了
- 停了 Docker 服务不会删任何镜像和容器,只是守护进程不运行了
- 已经 `systemctl enable docker`,下次开机 Docker 会自动启动;但容器不会自动启动,除非创建时加了 `--restart=always`

### 4. 如果只是关虚拟机

那更简单:关机前先 `docker stop mynginx`(避免文件系统写入中断),然后正常关机即可。Docker 服务和容器状态都会保留,下次开机 `systemctl status docker` 看到 `active` 就说明服务已经自动起来了,再 `docker start mynginx` 就能恢复网站。
## 四、常见坑

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `docker run hello-world` 卡住或超时 | 镜像加速器不可用 | 换成自己的阿里云专属加速地址 |
| 加速器配置不生效 | 改完没重启 Docker | `systemctl restart docker` |
| 浏览器打不开 8080 | 防火墙或云安全组未放行 | 放行端口并检查安全组 |
| 修改 daemon.json 后 Docker 起不来 | JSON 格式错误(多逗号、缺引号) | `python -m json.tool` 或在线 JSON 校验 |
| `docker rm` 删不掉 | 容器还在运行 | 先 `docker stop`,或 `docker rm -f` |
| 容器一启动就退出 | 主进程执行完毕即退出 | `docker logs` 看原因,交互式用 `-it` |