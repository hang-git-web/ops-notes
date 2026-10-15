# 镜像仓库 —— Harbor（课件第八章）

## 5.1 为什么要私有仓库

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

## 5.2 Harbor 完整部署（★★★ 完整复现）

### 步骤 1：下载离线安装包

```bash
# 在能上网的机器上下载，再上传到服务器（比如 /opt 目录）
wget https://github.com/goharbor/harbor/releases/download/v2.14.2/harbor-offline-installer-v2.14.2.tgz
# 或者直接本地下好后上传
ls -lh /opt/harbor-offline-installer-v2.14.2.tgz
```

> **为什么用 offline 版**：offline 包里已经含所有镜像，不用现场联网拉，内网部署最稳。

### 步骤 2：解压

```bash
cd /opt
tar xf harbor-offline-installer-v2.14.2.tgz
cd harbor
ls        # 期望看到 harbor.yml.tmpl、install.sh、harbor.v2.14.2.tar.gz 等
```

### 步骤 3：修改配置

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

### 步骤 4：新建数据目录

```bash
mkdir -p /data/harbor
```

> 建议在 `harbor.yml` 里把 `data_volume` 改成 `/data/harbor`，这样镜像数据都落在你规划好的大盘上，不会把根分区写满。

### 步骤 5：安装

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

### 步骤 6：让 docker 信任这个 http 仓库（insecure-registries）

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

### 步骤 7：让机器能解析 hostname

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

### 步骤 8：登录、打标签、推送、拉取（完整闭环）

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

### 步骤 9：Harbor 的日常运维命令

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

## 5.3 Harbor 常见坑

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `docker login` 报 `http: server gave HTTP response to HTTPS client` | 没配 `insecure-registries` | 加进 `daemon.json` 并 `systemctl restart docker` |
| `docker push` 报 `denied: requested access to the resource is denied` | 没登录，或项目/权限不对 | 先 `docker login`，确认项目存在、账号有推送权限 |
| `ping: ... Name or service not known` | 主机名解析不了 | 改 `/etc/hosts` 或配 DNS |
| `docker push` 报 `no such host` | 同上 | 同上 |
| 安装脚本报 `Need to install docker-compose` | 缺 compose | 见第一期 2.3 装 compose |
| 磁盘被 Harbor 写满 | 数据默认落在根分区 | 在 `harbor.yml` 里把 `data_volume` 指到大盘 |
| 网页 502 / 容器反复重启 | 内存不足 | Harbor 至少需要 2G 可用内存，4G 更稳 |
