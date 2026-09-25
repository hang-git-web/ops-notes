# Kubernetes 集群部署实战（kubeadm 三节点）

> 环境：RockyLinux 8.10 × 3（Master 1 台 + Worker 2 台）
> 版本：Kubernetes 1.23.17 + Docker 20.10.24 + Calico 3.24.5
> 目标：从零搭出一个能跑业务的三节点 K8s 集群，并理解每一步在干什么。

---

## 〇、先看地图：整个部署是一条链

**部署顺序链**（顺序错了会互相依赖失败）：

```text
节点环境配置 → 节点 Docker 引擎部署 → Master 部署 → Worker 节点部署
        → 集群网络配置(CNI) → 服务发布部署 → Dashboard 面板
```

对应到实操步骤：

| 阶段 | 做什么 | 在哪台机器 |
| --- | --- | --- |
| 一、节点规划 | 定主机名、IP、hosts 解析 | 全部 |
| 二、环境初始化 | 换源、升内核、关防火墙、关 swap、时间同步、ulimit、内核参数、ipvs 模块 | 全部 |
| 三、部署 Docker | 装 docker-ce + 配置镜像加速 | 全部 |
| 四、部署 K8s 组件 | 装 kubeadm / kubelet / kubectl | 全部 |
| 五、Master 初始化 | `kubeadm init`，生成集群证书与控制面 | Master |
| 六、Worker 加入 | `kubeadm join` | Worker |
| 七、部署 CNI | 装 Calico，Pod 才拿得到 IP、才能跨主机通信 | Master |
| 八、调整 kube-proxy | 切到 ipvs 模式（可选优化） | Master |
| 九、网络连通性测试 | Pod 之间、Pod 到外网 | Master |
| 十、业务功能测试 | 部署 nginx、暴露 NodePort | Master |
| 十一、Dashboard | Web 界面管理集群 | Master |

**一句话理解每个组件的分工**：

```text
kubectl   →  你的手
apiserver →  唯一的门（所有指令都从这进）
etcd      →  账本（只有 apiserver 能翻）
scheduler →  派活（决定 Pod 放哪台机器）
controller-manager → 监工（发现状态不对就去纠正）
kubelet   →  机器上的执行员（真正启动容器、上报状态）
kube-proxy→  网络调度员（把 Service 的流量转到 Pod）
CNI(Calico) → 铺路（让 Pod 之间能通）
```

---

## 一、知识点

### 1.1 容器化带来的六个问题

单个 Docker 用起来很爽，但容器一多，问题就来了：

| 问题 | 说明 |
| --- | --- |
| 容器故障怎么办 | 一个容器挂了，怎么让另一个立刻顶上去 |
| 并发变高怎么办 | 怎么横向扩展容器数量 |
| 跨主机怎么办 | 多容器跨主机提供服务 |
| 分布部署 | 多容器分布到不同节点 |
| 升级怎么办 | 多容器怎么平滑升级 |
| 怎么管 | 怎么高效管理这一堆容器 |

结论：用 compose 手工批量管理会越来越难，于是诞生了**容器编排**这个领域。

### 1.2 编排工具之争

| 工具 | 背景 |
| --- | --- |
| Swarm | Docker 自家的编排工具 |
| Mesos | Apache 的资源统一管控工具，需配合 Marathon |
| **Kubernetes** | Google 开源的容器编排工具 |
| Rancher | 云计算专家梁胜 2014 年在美国创建（他还做过 CloudStack），2020 年把 Rancher 卖给了 SUSE |

2017 年下半年 K8s 胜出。现在小规模场景偶尔还能见到 Swarm，Rancher 在私有云里也比较常见。

### 1.3 K8s 是什么

> Kubernetes 是一款开源的容器集群管理系统，能实现容器化应用的**自动化部署、扩缩容和维护**。俗称 k8s。

Google 基于内部 Borg 系统，于 2014 年 6 月开源；2014—2017 快速发展，2018 年进入成熟阶段，在中小企业广泛采用。

### 1.4 K8s 的五大能力

| 能力 | 具体表现 |
| --- | --- |
| **智能调度与弹性伸缩** | 按资源策略和节点状态自动选最优节点；按 CPU/内存负载自动调整副本数 |
| **高可用与自我修复** | 容器异常退出或健康检查失败时，自动重启或重建 |
| **服务治理与流量管理** | 服务发现、负载均衡、细粒度路由（支撑蓝绿部署、金丝雀发布） |
| **应用发布与版本控制** | 自动发布、一键回滚 |
| **存储与扩展性** | 自动挂载持久化存储；开放接口支持自定义资源与运维组件 |

### 1.5 架构：一个大号"主从"

```text
             Master（控制平台）                     Worker / Node（数据平台）
   ┌──────────────────────────────┐      ┌──────────────────────────────┐
   │  kube-apiserver              │◀────▶│  kubelet                     │
   │  etcd                        │      │  kube-proxy                  │
   │  kube-scheduler              │      │  容器运行时(Docker/containerd) │
   │  kube-controller-manager     │      │  Pod ← 真正干活的地方          │
   └──────────────────────────────┘      └──────────────────────────────┘
```

一个或多个 Master 组成控制平台，一个或多个 Worker 组成数据平台。Worker 会注册到集群，并**定时上报自己的信息**（操作系统、Docker 版本、CPU、内存、Pod 状态等）。

### 1.6 核心组件速查

| 组件 | 职责 | 一句话 |
| --- | --- | --- |
| **Controller Manager** | 集群控制管理器，维护集群状态；内含 ReplicaSet、Deployment、Service 等多种控制器 | **监工** |
| **Scheduler** | 按预设规则把 Pod 分配到合适的节点（预选 + 优选） | **派活的** |
| **API Server** | 中央通信枢纽，接收用户指令，提供认证、授权、API 注册与发现；集群状态数据由它写入 etcd | **唯一的门** |
| **etcd** | 支持分布式部署的键值数据库，持久化保存集群与资源对象元数据 | **账本** |
| **Kube-proxy** | 负责负载均衡与服务之间的网络通信（把 Service 流量转给 Pod） | **网络调度员** |
| **Kubelet** | 节点状态上报 + 本节点所有 Pod 的生命周期管理 | **执行员** |
| **Master** | 集群控制节点，接收并执行几乎所有管理命令（物理机或虚拟机均可） | **大脑** |
| **Worker / Node** | 除 Master 外的节点，承担实际工作负载 | **干活的手** |

> **重点**：`Pod` 是 K8s 中的**最小调度单元**，它**不等于容器**——一个 Pod 里可以包含多个容器（它们共享网络和存储卷，可以用 localhost 互相访问）。

### 1.7 版本策略（选版本时要懂）

- 2021 年 4 月起，K8s 从每年 4 个版本调整为 **3 个版本**，只维护最近的 **3 个次要版本**，每个约 14 个月支持。
- 版本号格式 `x.y.z`：x 主版本、y 次版本、z 补丁版本。
- 2025 年 2 月，Canonical 宣布从 Kubernetes 1.32 LTS 起，为 Ubuntu Pro 用户提供 LTS，共 12 年安全更新；**每两年发布一个 LTS 版本，从 1.32 开始**。

| 版本 | 首发时间 | 最后补丁 | 结束时间 | 说明 |
| --- | --- | --- | --- | --- |
| 1.34 | 2025-08-27 | | 2026-10-27 | |
| 1.33.1 | 2025-05-15 | | 2026-06-28 | |
| 1.32.0 | 2024-12-11 | | 2026-02-28 | LTS 起点 |
| 1.31.0 | 2024-08-13 | 1.31.13 | 2025-10-28 | |
| 1.30.0 | 2024-04-18 | 1.30.14 | 2025-07-15 | |
| 1.29 | | 1.29.14 | 2025-02-28 | |
| 1.28 | | 1.28.15 | 2024-10-22 | |
| 1.27 | | 1.27.16 | 2024-07-16 | |
| 1.26 | | 1.26.15 | 2024-02-28 | 最后补丁修复安全问题 |
| 1.25 | | 1.25.16 | 2023-10-28 | 最后补丁修复安全问题 |
| 1.24 | | 1.24.17 | 2023-07-28 | 最后补丁修复安全问题 |
| **1.23** | | 1.23.17 | 2023-02-28 | **本笔记用的版本，已 EOL** |
| 1.22 | | 1.22.17 | 2022-12-08 | 最后补丁修复安全问题 |

> ⚠️ **本笔记用的 1.23.17 早已停止支持**。学习路线照做没问题（资料最全、踩坑最少），
> 但**新环境请用 1.28 以上，或直接上 1.32 LTS**，否则面试时"用 EOL 版本"会被追问。

**版本偏差（组件版本可以差多少）**

| 组件 | 相对 kube-apiserver | 说明 |
| --- | --- | --- |
| kube-apiserver | 必须最新 | 极端情况下可比 kubectl 低 1 个次版本 |
| kubelet | 最多低 3 个次版本 | kubelet < 1.25 时只能低 2 个 |
| kube-proxy | 最多低 3 个次版本 | kube-proxy < 1.25 时只能低 2 个 |
| controller-manager / scheduler / cloud-controller-manager | 最多低 1 个次版本 | |
| kubectl | 可低 1 个，也可高 1 个 | 灵活度最高 |

**升级要求**

- **不能跨次要版本升级**：1.22.x → 1.23.x ✅；1.22.x → 1.24.x ❌
- **可以跨补丁版本升级**：1.22.0 → 1.22.16 ✅

---

## 二、节点规划

| 主机名 | IP | 角色 |
| --- | --- | --- |
| master-203 | 172.22.4.203 | Master |
| node01-204 | 172.22.4.204 | Worker |
| node02-205 | 172.22.4.205 | Worker |

**前置检查：确保每台节点的 MAC 地址和 product_uuid 唯一**

```bash
ip link
cat /sys/class/dmi/id/product_uuid
```

> **为什么**：K8s 用 `product_uuid` 和 MAC 来唯一标识节点。如果是克隆出来的虚拟机，这两项可能重复，会导致节点注册冲突、Pod 调度异常。虚拟机克隆后一定要检查。

**内核要求：大于 3.10**

RockyLinux 8.10 默认 4.18，本笔记升级到 5.19。

---

## 三、环境初始化（三台都要做）

### 3.1 更新 yum 源

```bash
curl http://k1.zhynet.net/change_rockylinux8_yum.sh | bash
```

> **为什么**：默认源在国外，装包慢甚至失败。

### 3.2 升级内核

```bash
# 安装必要软件
dnf install -y wget curl unzip tar vim gcc make net-tools lrzsz

uname -a                 # 查看当前内核

mkdir /opt/kernel519 && cd /opt/kernel519
# 用 lrzsz 把 kernel 相关 rpm 传进来
dnf localinstall -y kernel-ml-*

grubby --default-kernel                                          # 查询当前默认内核
grubby --set-default /boot/vmlinuz-5.19.12-1.el8.elrepo.x86_64   # 设置默认内核

reboot
```

> **为什么**：老内核（3.10）对 overlay2、cgroup、iptables 的支持不完整，容易出玄学问题。
> **注意**：不要直接用 `rpm -ivh` 装（缺依赖容易炸），用 `dnf localinstall` 会自动处理依赖；有条件直接搭个内网 yum 源。

### 3.3 设置主机名与 hosts 解析

```bash
# master 机器
hostnamectl set-hostname master-203 && bash

# node01 机器
hostnamectl set-hostname node01-204 && bash

# node02 机器
hostnamectl set-hostname node02-205 && bash

# 三台机器都执行
cat >>/etc/hosts<<EOF
172.22.4.203 master-203
172.22.4.204 node01-204
172.22.4.205 node02-205
EOF
```

> **为什么**：`kubeadm` 在初始化时会用主机名做节点注册，hosts 不通会导致节点无法加入。**这一步不做，后面 join 必然失败。**

### 3.4 关闭防火墙与 swap

```bash
systemctl stop firewalld && systemctl disable firewalld && systemctl status firewalld

free -m
swapoff -a && sysctl -w vm.swappiness=0

# 永久禁用 swap
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 确认已关闭
swapon -v
```

> **为什么必须关 swap**：kubelet 的资源计算、调度策略都假设节点没有 swap。**不关的话 `kubeadm init` 会直接报错退出**（或者需要加 `--ignore-preflight-errors=Swap`）。
> 防火墙是内网实验环境，图省事；生产要按需放行 6443、10250、8472 等端口。

### 3.5 时间同步

```bash
yum install -y chrony
systemctl enable chronyd && systemctl start chronyd
chronyc sources
timedatectl
```

> **为什么**：K8s 组件之间用证书通信，**时间偏差过大会导致证书校验失败、集群异常**。
> 检查点：`chronyc sources` 里应该看到 `^*` 标记的那一行。

### 3.6 配置 ulimit

```bash
cat >>/etc/security/limits.conf<<'EOF'
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```

> **为什么**：容器多、连接多，文件句柄不够会报 `Too many open files`。

### 3.7 内核参数调整

```bash
cat > /etc/sysctl.d/k8s.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=524288
fs.file-max=16777216
fs.nr_open=16777216
net.netfilter.nf_conntrack_max=2097152
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 72000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 65536
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.core.somaxconn = 16384
net.ipv4.tcp_timestamps = 0
EOF

# 让参数立即生效(讲义最后 reboot 了,这里补上更稳妥)
sysctl --system
```

关键几项的作用：

| 参数 | 作用 |
| --- | --- |
| `net.ipv4.ip_forward = 1` | 开启 IP 转发，**Pod 跨主机通信的前提** |
| `bridge-nf-call-iptables = 1` | 让 iptables 能处理网桥流量，**否则 Service 不通** |
| `vm.overcommit_memory = 1` | 允许内存超额分配 |
| `fs.inotify.max_user_watches` | 监控文件数上限 |
| `net.netfilter.nf_conntrack_max` | 连接跟踪表大小，太小会丢包 |

### 3.8 加载 ipvs 所需内核模块

```bash
dnf install -y ipvsadm

tee /etc/modules-load.d/br_netfilter_overlay.conf >/dev/null<<EOF
br_netfilter
overlay
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
nf_conntrack
EOF

reboot
```

> **为什么**：`kube-proxy` 支持 iptables 和 ipvs 两种模式。ipvs 性能更好，但它需要内核提前加载这些模块。
> 写在 `/etc/modules-load.d/` 下，**开机自动加载**，重启后 `lsmod | grep ip_vs` 能看到就说明成功。
> 想立即生效不想重启，可以 `modprobe br_netfilter overlay ip_vs`。

---

## 四、部署 Docker（三台都要）

```bash
yum -y install yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum clean all && yum makecache

dnf -y install docker-ce-20.10.24 docker-ce-cli-20.10.24 \
  docker-ce-rootless-extras-20.10.24 containerd.io-1.6.3
```

> **注意**：多台机器同时装容易触发镜像站频次限制导致失败，**建议一台装完再装下一台**。

### 4.1 配置镜像加速 + cgroup 驱动

```bash
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2",
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
EOF

systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```

| 配置项 | 为什么这么写 |
| --- | --- |
| `native.cgroupdriver=systemd` | **必须和 kubelet 的 cgroup 驱动一致**，否则 kubelet 起不来或 Pod 异常 |
| `log-opts.max-size: 100m` | 限制单个容器日志大小，**防止日志把磁盘撑满** |
| `storage-driver: overlay2` | 推荐的存储驱动，性能好 |
| `registry-mirrors` | 国内拉镜像加速 |

**检验**

```bash
docker info | grep -E "Cgroup Driver|Storage Driver"
systemctl is-active docker
```

---

## 五、安装 kubeadm / kubelet / kubectl（三台都要）

```bash
cat > /etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
EOF

dnf install -y kubeadm-1.23.17 kubelet-1.23.17 kubectl-1.23.17
systemctl enable kubelet && systemctl start kubelet
```

| 工具 | 作用 |
| --- | --- |
| **kubeadm** | 一键部署集群的"安装向导"，负责初始化控制面、生成证书、拼 join 命令 |
| **kubelet** | 每个节点都要跑的"执行员"，systemd 托管 |
| **kubectl** | 命令行客户端，用来操作集群 |

> `systemctl enable kubelet` 之后它是**反复重启状态**，这是正常的——因为还没有配置文件，等 `kubeadm init/join` 生成配置后才会稳定运行。

---

## 六、Master 初始化（只在 master-203 上）

```bash
kubeadm init \
  --kubernetes-version 1.23.17 \
  --image-repository registry.aliyuncs.com/google_containers \
  --apiserver-advertise-address=172.22.4.203 \
  --service-cidr=10.1.0.0/16 \
  --pod-network-cidr=10.2.0.0/16
```

> ⚠️ **原文勘误**：讲义里把注释 `#master的ip地址` 写在了续行符 `\` 后面，这样会把注释当成命令的一部分，**执行必然报错**。注释要单独一行。

参数逐个解释：

| 参数 | 含义 | 为什么必须写 |
| --- | --- | --- |
| `--kubernetes-version` | 要部署的版本 | 和装的 kubeadm 保持一致，避免版本漂移 |
| `--image-repository` | 镜像仓库 | 默认是 `k8s.gcr.io`，**国内拉不动**，换成阿里云 |
| `--apiserver-advertise-address` | API Server 对外通告的地址 | **多网卡必须显式指定**，Worker 靠这个地址连进来 |
| `--service-cidr` | Service 的虚拟 IP 网段 | 集群内部虚拟 IP 从这里分配 |
| `--pod-network-cidr` | Pod 网段 | **必须和 CNI 插件的配置一致**（后面 Calico 要设成同一个） |

### 6.1 配置 kubectl 的认证文件

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> **为什么**：kubectl 需要凭据才能连上 API Server。`kubeadm init` 生成的 `admin.conf` 就是管理员凭据，拷到 `~/.kube/config` 后 kubectl 才能免密操作。
> 这一步不做，后面所有 `kubectl` 命令都会报 `The connection to the server localhost:8080 was refused`。

**检验**

```bash
kubectl get nodes          # 此时 master 已经是 NotReady(正常,还没装网络插件)
kubectl get pods -A        # 能看到 kube-system 下的控制面组件在跑
```

---

## 七、Worker 节点加入集群（在 node01 / node02 上）

```bash
kubeadm join 172.22.4.203:6443 --token 17xih5.t7tbw21usy1gb93w \
  --discovery-token-ca-cert-hash sha256:8f20476435fd827c0bc11782d812525342acbd583b129d07629aa843459720b5
```

> **token 24 小时后会过期**。过期了不用重新 init，在 master 上执行：
> ```bash
> kubeadm token create --print-join-command
> ```
> 重新生成的 join 命令直接拿去用即可。

```bash
# 在 master 上查看集群节点情况
kubectl get nodes
```

**此刻 `NotReady` 是正常的**——因为**网络组件还没装**，节点还没准备好接活。

---

## 八、部署 Calico 网络组件

### 8.1 为什么必须要装 CNI

kubeadm 只装了集群的"骨架"，**不包含网络插件（CNI）**。没有 CNI 会有什么后果：

| 现象 | 原因 |
| --- | --- |
| 节点一直是 `NotReady` | kubelet 等不到网络就绪 |
| Pod 一直 `Pending` 或 `ContainerCreating` | 拿不到 Pod IP |
| 不同 Node 上的 Pod 互相 ping 不通 | 没有跨主机路由 |
| `ClusterIP` 的 Service 访问不通 | 没有网络转发规则 |

**CNI 就是给 Pod"铺路"的**：分配 IP、打通跨主机通信、让 Service 能工作。常见的实现有 Calico、Flannel、Cilium。这里选 Calico，因为它支持网络策略（NetworkPolicy）、BGP 路由、性能好。

### 8.2 部署 Calico 3.24.5

```bash
vim calico_3.24.5.yaml

# 找到这一段并修改 CIDR,必须和 kubeadm init 的 --pod-network-cidr 一致
# 原内容
# - name: CALICO_IPV4POOL_CIDR
#   value: "192.168.0.0/16"

# 修改为
- name: CALICO_IPV4POOL_CIDR
  value: "10.2.0.0/16"

kubectl apply -f calico_3.24.5.yaml

# 查看 calico 容器是否正常
kubectl get pods -A
```

> **CIDR 不一致会怎样**：Calico 会按它自己的网段给 Pod 分 IP，结果 Pod 的 IP 不在 kube-proxy 和路由认识的网段内，**Pod 之间就是不通**。这是新手最常见的坑之一。

**遇到的真实问题**：装完发现有两个 Pod 不是 `Running`。

**是不是网络问题？——多数不是，最常见是"镜像还在下载"。** 判断方法：

```bash
kubectl get pods -A -w                  # 持续观察状态变化
kubectl describe pod -n kube-system <pod名>   # 看 Events 里在做什么
```

| Events 里看到 | 含义 | 处理 |
| --- | --- | --- |
| `Pulling image "..."` | 正在拉镜像 | 等，或配置镜像加速 |
| `Pulled` / `Created` / `Started` | 正常流程 | 等它变 Running |
| `ImagePullBackOff` | 拉不到镜像 | 检查网络、镜像地址、加速器 |
| `FailedScheduling` | 调度不了 | 看节点资源、污点 |

**结论**：讲义里等一会儿全部变成 `Running`，就是典型的**镜像拉取慢**，不是网络故障。

---

## 九、调整网络负载组件为 ipvs（可选优化）

```bash
ipvsadm -ln      # 看看当前有没有规则

# 编辑 kube-proxy 的 ConfigMap
kubectl edit configmap -n kube-system kube-proxy
```

在打开的编辑器里找到这一行，改成 ipvs：

```yaml
# 把
mode: ""
# 改成
mode: "ipvs"
```

> ⚠️ **讲义这里没说清楚改什么**——`kubectl edit` 打开后要改的是 `mode` 字段，不是随便改。

```bash
# 删除 kube-proxy pod,触发重建让它读取新配置
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# 再看 ipvs 规则
ipvsadm -ln
```

```text
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InConn
TCP  10.1.0.1:443 rr
  -> 172.22.4.203:6443            Masq    1      1          0
TCP  10.1.0.10:53 rr
  -> 10.2.132.193:53              Masq    1      0          0
  -> 10.2.132.194:53              Masq    1      0          0
TCP  10.1.0.10:9153 rr
  -> 10.2.132.193:9153            Masq    1      0          0
  -> 10.2.132.194:9153            Masq    1      0          0
UDP  10.1.0.10:53 rr
  -> 10.2.132.193:53              Masq    1      0          0
  -> 10.2.132.194:53              Masq    1      0          0
```

**这段输出怎么读**：

| 看到的 | 含义 |
| --- | --- |
| `10.1.0.1:443 → 172.22.4.203:6443` | kubernetes 这个 Service 指向 API Server |
| `10.1.0.10:53 → 10.2.132.x:53` | **CoreDNS 有两个副本**，Service 做负载均衡；这正是"两个 Pod 的 IP" |
| `rr` | 调度算法：轮询 |
| `Masq` | 转发方式：NAT（类似 iptables 的 MASQUERADE） |

> mode 从 iptables 换成 ipvs 的意义：服务数量多时 ipvs 用哈希表查找，**性能明显优于 iptables 的线性规则匹配**。

---

## 十、CoreDNS 与 Calico 网络功能测试

### 10.1 确认 CoreDNS 正常

```bash
kubectl get svc -n kube-system kube-dns
```

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.1.0.10    <none>        53/UDP,53/TCP,9153/TCP   29m
```

| 字段 | 含义 |
| --- | --- |
| `CLUSTER-IP 10.1.0.10` | 集群内 DNS 地址，Pod 通过它解析服务名 |
| `53/UDP、53/TCP` | DNS 端口 |
| `9153/TCP` | 监控指标端口 |

> **为什么关心 DNS**：K8s 里服务之间是靠**服务名**互相访问的（如 `http://nginx`），解析全靠 CoreDNS。它不正常，服务发现就废了。

### 10.2 创建两个测试 Pod，验证跨 Pod 通信

```bash
kubectl run lgh-test-pod1 --image=alpine --command -- sleep 600000
kubectl run lgh-test-pod2 --image=alpine --command -- sleep 600000

kubectl get pod -o wide
```

```text
NAME           READY   STATUS    RESTARTS   AGE   IP           NODE
lgh-test-pod1  1/1     Running   0          29s   10.2.15.129  node01-204
lgh-test-pod2  1/1     Running   0          28s   10.2.15.130  node01-204
```

> **怎么看结果**：`IP` 是 `10.2.x.x`（落在 `--pod-network-cidr` 里）说明 **Calico 分配 IP 成功**；`NODE` 列告诉你在哪台机器上。
> 想验证跨主机，需要让两个 Pod 分到**不同节点**（可以给节点打污点或指定 nodeName），讲义这两个恰好都在 node01 上。

### 10.3 进入 Pod 测试连通性

```bash
kubectl exec -it lgh-test-pod1 -- sh
```

```text
/ # ping www.qq.com
PING www.qq.com (121.14.77.221): 56 data bytes
64 bytes from 121.14.77.221: seq=0 ttl=51 time=23.416 ms
```

```bash
# 在 Pod 里继续测：能不能 ping 通另一个 Pod
ping 10.2.15.130
# 能不能解析服务名
nslookup kubernetes.default.svc.cluster.local
```

> **原文勘误**：`kubectl exec -it pod sh` 会提示 `is DEPRECATED... Use kubectl exec [POD] -- [COMMAND]`，
> 正确写法是 **`kubectl exec -it <pod> -- sh`**（中间那个 `--` 不能省）。
> 同理 `kubectl run ... --command sleep 600000` 建议写成 `--command -- sleep 600000`，语义更清晰。

**测试的三个维度**（讲义只做了第一个）：

| 测试 | 命令 | 期望 |
| --- | --- | --- |
| Pod → 外网 | `ping www.qq.com` | 通 |
| Pod → Pod | `ping <另一个Pod的IP>` | 通 |
| Pod → Service | `nslookup kubernetes.default` | 能解析出 10.1.0.1 |

---

## 十一、集群业务功能测试：部署 nginx

```bash
# 创建 Deployment
kubectl create deployment nginx --image=nginx:1.29-alpine

# 暴露端口(暴露了才能从浏览器访问)
kubectl expose deploy nginx --port=80 --target-port=80 --type=NodePort

# 查看
kubectl get pod,svc
```

```text
NAME                          READY   STATUS    RESTARTS   AGE
pod/lgh-test-pod1             1/1     Running   0          11m
pod/lgh-test-pod2             1/1     Running   0          10m
pod/nginx-7cd5cf89b4-hr5fl    1/1     Running   0          73s

NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.1.0.1      <none>        443/TCP        43m
service/nginx        NodePort    10.1.94.47    <none>        80:30274/TCP   56s
```

**输出怎么读**：

| 字段 | 含义 |
| --- | --- |
| `pod/nginx-7cd5cf89b4-hr5fl` | Deployment 创建出来的 Pod，名字 = Deployment名 + ReplicaSet哈希 + 随机串 |
| `service/nginx` 的 `10.1.94.47` | ClusterIP，集群内部访问用 |
| `80:30274/TCP` | **左边 80 是 Service 端口，右边 30274 是每个 Node 上开放的端口** |
| `type: NodePort` | 通过 `任意节点IP:30274` 就能访问 |

```bash
# 浏览器访问
http://172.22.4.203:30274
http://172.22.4.204:30274
http://172.22.4.205:30274     # 三台节点都能访问,这就是 NodePort 的意义
```

| Service 类型 | 谁能访问 | 用途 |
| --- | --- | --- |
| ClusterIP | 只有集群内部 | 内部服务互调（默认类型） |
| **NodePort** | 集群外，通过 `节点IP:30000-32767` | 实验、简单对外暴露 |
| LoadBalancer | 集群外，通过云厂商 LB | 云上生产 |
| ExternalName | — | 把外部服务映射成集群内的名字 |

---

## 十二、Web 管理界面 Dashboard

### 12.1 部署 Dashboard

```bash
kubectl apply -f k8s-dashboard-2.5.1.yaml
kubectl get pods -n kubernetes-dashboard
```

```text
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-799d786dbf-djsns   1/1     Running   0          75s
kubernetes-dashboard-fb8648fd9-8n5c6         1/1     Running   0          75s
```

> **疑问解答：这个 yaml 文件是需要导入或者下载的吗？**
>
> **需要，必须先有文件**。`kubectl apply -f xxx.yaml` 只是"按这个文件创建资源"，文件本身得先存在。来源有三种：
>
> ```bash
> # ① 官方仓库下载(推荐)
> wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.1/aio/deploy/recommended.yaml -O k8s-dashboard-2.5.1.yaml
>
> # ② 官网页面复制内容,自己保存成文件
> #    https://github.com/kubernetes/dashboard/releases
>
> # ③ 离线环境:先在有网的机器下载好,再上传到服务器
> ```
>
> **注意版本匹配**：Dashboard 对 K8s 版本有要求（2.5.x 对应 K8s 1.20~1.22 左右）。配 1.23 一般能用，但**新环境建议用 v2.7.x**，否则可能出现接口不兼容。
> 另外官方推荐文件里创建的是 `ClusterIP` 类型的 Service，这就是下一步要改它的原因。

### 12.2 把服务改成 NodePort

```bash
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```

在打开的编辑器里把：

```yaml
type: ClusterIP
```

改成：

```yaml
type: NodePort
```

保存退出，然后查看：

```bash
kubectl get svc -n kubernetes-dashboard
```

> **疑问解答：这一步是干什么的？**
>
> `kubectl edit` 就是**在线打开这个资源的 YAML 让你改，保存后立即生效**（相当于"改配置 + apply"）。
> 这里改的是 **Service 的类型**：
>
> - 默认 `ClusterIP` 只能在**集群内部**访问 → 浏览器根本连不上
> - 改成 `NodePort` 后，K8s 会在**每台节点**上开放一个 30000-32767 的端口 → 外部才能访问
>
> 改完后 `kubectl get svc` 的 PORT(S) 列会变成类似 `443:31940/TCP`，那个 `31940` 就是你浏览器要用的端口。

访问（Dashboard 默认是 HTTPS 自签证书，浏览器会告警）：

```text
https://172.22.4.203:31940/
```

```bash
# Chrome 忽略证书告警的启动方式
chrome.exe --ignore-certificate-errors --user-data-dir=C:\temp-chrome
```

### 12.3 创建登录 Token（RBAC 授权）

```bash
# 创建一个服务账号
kubectl create serviceaccount dashboard-admin -n kube-system

# 把它绑定到集群管理员角色
kubectl create clusterrolebinding dashboard-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:dashboard-admin

# 查看 Token
kubectl describe secrets -n kube-system \
  $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

> **原文勘误**：`awk '/dashboard‑admin/...'` 里的连字符是从文档复制的特殊字符（非普通 `-`），直接粘贴会匹配不到。必须手敲普通的减号。

把输出的 `token:` 后面那一长串复制到 Dashboard 登录页即可。

**这三步在做什么**：

| 步骤 | 作用 |
| --- | --- |
| `create serviceaccount` | 创建一个"机器人账号"（不是给人用的用户，而是给程序/令牌用的身份） |
| `create clusterrolebinding` | **授权**：把这个账号绑到 `cluster-admin` 这个内置角色上，它就有了全套管理权限 |
| `describe secrets` | 取出这个账号的 Token，当作登录密码用 |

> 这套机制叫 **RBAC（基于角色的访问控制）**。K8s 1.24 以后 ServiceAccount 默认不再自动生成长期 Token，需要用 `kubectl create token dashboard-admin -n kube-system` 获取，要注意版本差异。

---

## 十三、疑问集中解答

把笔记里散落的问号统一回答：

### Q1：`k8s-dashboard-2.5.1.yaml` 是需要导入或下载的吗？
需要。`kubectl apply -f` 不联网也不生成文件，它只负责"照着这个文件创建资源"。文件要先从官方仓库下载（见 12.1）。

### Q2：`kubectl edit svc kubernetes-dashboard` 是干什么的？
在线编辑这个 Service 的 YAML，把类型从 `ClusterIP` 改成 `NodePort`，让集群外部能通过 `节点IP:端口` 访问 Dashboard。

### Q3：为什么需要部署 Calico 网络组件？
kubeadm 不装 CNI，没有它：节点 `NotReady`、Pod 拿不到 IP、跨主机 Pod 不通、Service 不通。Calico 就是给 Pod 铺路、分配 IP、打通跨主机通信的组件（见 8.1）。

### Q4：节点差两个没 `Running`，是网络问题吗？
**多数不是**，最常见是镜像还在下载。用 `kubectl describe pod -n kube-system <pod名>` 看 Events：显示 `Pulling image` 就是慢，显示 `ImagePullBackOff` 才是真有问题（见 8.2）。

### Q5：K8s 和 Docker 镜像有什么区别？
**本质是同一个东西**——K8s 用的就是标准的 OCI 镜像，`docker pull nginx` 和 K8s 里 `image: nginx` 拉的是同一份。区别在"怎么用"：

| 维度 | Docker 直接用 | 在 K8s 里用 |
| --- | --- | --- |
| 存储位置 | 宿主机的 `/var/lib/docker` | 每个 Node 自己的容器运行时里（多节点要保证每个节点都有） |
| 拉取方式 | 手动 `docker pull` | `imagePullPolicy` 控制（Always / IfNotPresent / Never） |
| 使用方式 | `docker run nginx:1.24` | YAML 里写 `image: nginx:1.24` |
| 关键差异 | — | **K8s 1.24+ 移除了 dockershim，运行时换成 containerd，集群里可以完全没有 Docker** |

> 本笔记的集群是 1.23 + docker-ce，还通过 dockershim 用 Docker 当运行时，所以 `docker images` 能看到 K8s 拉的镜像。

### Q6：Pod 和容器是一回事吗？
不是。**Pod 是最小调度单元，一个 Pod 可以包含多个容器**。同一个 Pod 里的容器共享网络命名空间（可以用 `localhost` 互访）和存储卷。最常见的组合就是"业务容器 + sidecar（日志/代理）"。

### Q7：Deployment 里能看到节点情况吗？
能。命令行 `kubectl get pods -o wide` 看 `NODE` 列；Dashboard 里点进 Deployment 能看到副本数、每个 Pod 的状态和所在节点。

### Q8：为什么 `kubectl expose` 之后才能从浏览器访问？
`kubectl create deployment` 只创建了 Pod，Pod 的 IP 是集群内部的、会变。`kubectl expose` 创建的是一个 **Service**——给这组 Pod 一个**固定的虚拟 IP 和端口**，NodePort 类型还会在每台节点上开一个端口对外。所以**不 expose，外部永远访问不到**。

---

## 十四、闭环自检清单

| 阶段 | 检查命令 | 通过标准 |
| --- | --- | --- |
| 主机名 | `hostname` | 三台各不相同 |
| hosts 解析 | `ping -c1 node01-204` | 能解析到 IP |
| swap | `swapon -v` | 无输出（已关闭） |
| 时间同步 | `chronyc sources` | 有 `^*` 标记的行 |
| 内核 | `uname -r` | 5.19.x |
| ipvs 模块 | `lsmod \| grep ip_vs` | 有多条 ip_vs 记录 |
| Docker | `docker info \| grep -E "Cgroup\|Storage"` | `systemd` + `overlay2` |
| 组件安装 | `kubeadm version` / `kubelet --version` | 均为 1.23.17 |
| Master 初始化 | `kubectl get nodes` | master 出现（NotReady 正常） |
| Worker 加入 | `kubectl get nodes` | 三台全部出现 |
| CNI 就绪 | `kubectl get pods -n kube-calico`（或 `-A`） | 全部 `Running` |
| 节点状态 | `kubectl get nodes` | 三台全部 **Ready** |
| DNS | `kubectl get svc -n kube-system kube-dns` | 有 `10.1.0.10` |
| ipvs 生效 | `ipvsadm -ln` | 能看到 443 / 53 的规则 |
| Pod 通信 | Pod 内 `ping` 另一个 Pod | 通 |
| Pod 出网 | Pod 内 `ping www.qq.com` | 通 |
| 业务暴露 | `kubectl get svc nginx` | `NodePort`、`80:30274/TCP` |
| 浏览器访问 | `http://节点IP:30274` | 三台节点都能看到 nginx 欢迎页 |
| Dashboard | `kubectl get pods -n kubernetes-dashboard` | 全部 `Running` |
| Dash 访问 | `https://172.22.4.203:31940` | 能用 Token 登录 |

---

## 十五、常见问题与排错

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `kubeadm init` 报 swap 相关错误 | 没关 swap | `swapoff -a` + 注释 `/etc/fstab` + 重启 |
| `kubeadm init` 拉镜像超时 | 用的还是 `k8s.gcr.io` | 加 `--image-repository registry.aliyuncs.com/google_containers` |
| `kubectl` 报 `connection to localhost:8080 refused` | 没配 `~/.kube/config` | 执行 6.1 的三条命令 |
| 节点一直 `NotReady` | 没装 CNI / CNI 没起来 | 装 Calico，检查 Calico Pod 日志 |
| Pod 一直 `Pending` | 调度不了：资源不足、有污点、CNI 未就绪 | `kubectl describe pod` 看 Events |
| Pod 一直 `ContainerCreating` | 镜像在拉取中 | 等；配镜像加速 |
| `ImagePullBackOff` | 拉不到镜像 | 换镜像地址 / 手动导入镜像 |
| Calico Pod 起不来 | **CIDR 和 `--pod-network-cidr` 不一致** | 改成同一个网段后重新 apply |
| Pod 之间不通、跨主机不通 | ip_forward 没开 / 网桥参数没开 / CNI 异常 | 检查 3.7 内核参数、Calico 状态 |
| Service 访问不通 | kube-proxy 异常 | `kubectl get pod -n kube-system \| grep proxy` |
| `kubeadm join` 报 token 失效 | token 24 小时过期 | `kubeadm token create --print-join-command` |
| 装 docker 时报错 | 多台同时装触发镜像站限流 | 一台一台装 |
| Dashboard 打不开 | Service 还是 ClusterIP / 证书告警 | 改成 NodePort；浏览器忽略证书 |
| Dashboard 登录没反应 | Token 不对 | 重新 `describe secrets` 取，注意别复制换行空格 |

**排错四件套**

```bash
kubectl get nodes                                   # 节点层面
kubectl get pods -A -o wide                         # Pod 层面(带节点和IP)
kubectl describe pod -n <命名空间> <pod名>            # 看 Events,90% 的问题能定位
kubectl logs -n <命名空间> <pod名>                    # 看容器里程序的日志
```

---

## 十六、复盘与易错点

1. **Swap 必须关** —— 不关 `kubeadm init` 直接失败，这是第一个拦路虎。
2. **hosts 解析必须配** —— 三台机器互相用主机名能通，否则 join 失败。
3. **CIDR 三处要一致** —— `--pod-network-cidr`、Calico 的 `CALICO_IPV4POOL_CIDR`、Pod 实际 IP 网段，任何一处不一致都会导致网络不通。
4. **Docker 的 cgroup 驱动要和 kubelet 一致** —— 这里都用 systemd。
5. **`NotReady` 不是故障** —— 装 CNI 之前，节点本来就是 NotReady。
6. **Pod 不 Running 先看 Events，别先怀疑网络** —— 十有八九是镜像拉取慢。
7. **`kubectl apply -f` 不会帮你下载文件** —— yaml 得先有。
8. **Service 不 expose，外部访问不到** —— Pod 的 IP 会变，Service 才是稳定入口。
9. **`kubectl exec` 要加 `--`** —— 不加会有弃用告警（未来版本直接报错）。
10. **1.23 已 EOL** —— 学习照做没问题，简历里写要注意，新环境用 1.28+/1.32 LTS。
11. **文档复制粘贴的连字符是坑** —— `grubby --set-default`、`awk '/dashboard-admin/'`、`sst -tlnp` 这类地方，特殊连字符会让命令莫名其妙失败，务必手敲。

---

## 附录：常用命令速查

```bash
# 集群与节点
kubectl get nodes                            # 节点列表与状态
kubectl get nodes -o wide                    # 带内网IP、版本
kubectl describe node <节点名>                # 节点详情(资源、已分配的Pod)

# Pod / Service / Deployment
kubectl get pods -A -o wide                  # 所有命名空间的Pod
kubectl get svc -A                           # 所有 Service
kubectl get deploy -A                        # 所有 Deployment
kubectl create deployment nginx --image=nginx:1.29-alpine
kubectl expose deploy nginx --port=80 --target-port=80 --type=NodePort
kubectl scale deployment nginx --replicas=3  # 扩容
kubectl delete deployment nginx              # 删除

# 排错
kubectl describe pod <pod>                   # 看事件
kubectl logs <pod>                           # 看日志
kubectl exec -it <pod> -- sh                 # 进容器

# 集群运维
kubeadm token create --print-join-command    # 重新生成 join 命令
kubectl get componentstatuses                # 控制面组件状态
ipvsadm -ln                                  # 查看 ipvs 转发规则
```