# Docker 可视化 —— Portainer（课件第七章）

> **课件原话**：**流行的软件，应该是 portainer。**

## 4.1 完整部署（课件给的三步 + 我给你补上验证）

### 步骤 1：拉取镜像

```bash
docker pull portainer/portainer-ce
```

### 步骤 2：新建数据目录

```bash
mkdir -p /data/portainer/data
```

> **为什么要有数据目录**：Portainer 的账号、密码、你配的所有东西都存在 `/data` 里。**不挂出来，容器重建就得重新配一遍。**

### 步骤 3：启动容器

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

### 步骤 4：访问与初始化

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

## 4.2 常见坑

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 网页打不开 | 9000 没放行 / 容器没起 | `docker ps`、`ss -lntp`、防火墙、云安全组 |
| 提示连不上 Docker | 忘了挂 `docker.sock` | 重新 run，加上 `-v /var/run/docker.sock:/var/run/docker.sock` |
| 重启后配置全丢了 | 没挂 `/data` | 加上 `-v /data/portainer/data:/data` |
| 容器随机器重启没起来 | 没加 `--restart=always` | 加上重启策略 |
| **安全风险** | 9000 暴露到公网 | **只在内网开放，或加反向代理 + HTTPS + 认证** |
