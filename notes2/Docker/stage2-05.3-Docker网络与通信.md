# Docker 网络与通讯（课件第五章）

## 5.1 Docker 默认网络

### 5.1.1 先看清 docker0 —— 虚拟交换机

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

### 5.1.2 四种网络模式

| 模式 | DRIVER | 说明（课件原话 + 补充） |
| --- | --- | --- |
| **Bridge** | `bridge` | **默认模式**（桥接，自己创建的网络也是用它）。此模式会为每一个容器分配、设置 IP 等，并将容器连接到一个 docker0 虚拟网桥，通过 docker0 网桥以及 iptables nat 表配置与宿主机通信 |
| **Host** | `host` | 容器**没有独立网络命名空间**，不会虚拟出自己的网卡、也不配置独立 IP，而是**直接使用宿主机的 IP 和端口** |
| **None** | `null` | 该模式**关闭了容器的网络功能**（一般不用） |
| **自定义（container）** | — | 加入另一个已存在容器的网络命名空间（了解即可） |

> **Host 模式要注意什么**：因为直接用宿主机网络，所以**不能做端口映射**（`-p` 会失效并可能报错），而且容器里 `ss -lntp` 看到的是**宿主机所有的端口**。适合做网络性能敏感的场景（少了 NAT 一层）。
>
> **None 模式**：给了你一个完全没有网络的容器，通常用于"我自己来配网络"的高级场景，或者做纯计算任务。

### 5.1.3 自定义网络（★★ 重点）

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

**★ 自定义网络 vs 默认 bridge 网络 —— 最重要的一个区别**：

| 对比项 | 默认 bridge | 自定义网络（bridge driver） |
| --- | --- | --- |
| 容器之间能否用**容器名**通信 | **不能**（只能靠 IP，而且 IP 会变） | **能**（内置 DNS 解析） |
| 网段 | 固定 172.17.0.0/16 | 自己指定 |
| 隔离 | 所有容器混在一起 | 按网络隔离，不同网络不互通 |

> **这一条是第三期 Compose 的基础**：Compose 里 `depends_on` + `networks` 能让 PHP 容器**用服务名直接连 MySQL**（`DB_HOST: mysql`），靠的就是自定义网络的 DNS。

**清理实验环境**：

```bash
docker rm -f tomcat01 tomcat02 tomcat04 tomcat05
docker network rm qianshannet
```

## 5.2 容器对外服务（SNAT / DNAT）

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

### 看真实的 iptables NAT 规则

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

### 完整验证实验：一个端口映射的完整链路

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
