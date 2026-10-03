# K8S 快速进阶（第二期）—— k8s v1.28 + containerd 部署与资源对象

> 内容来源：《K8S快速进阶(第二期)3.12》讲义
> **本期两条主线**：**① 用 containerd 部署 k8s 1.28 集群**；**② 学 k8s 的资源对象**（Pod / Deployment / Service / Ingress）
> 配套：**与第一期的详细对比在第五章**。

---

## 〇、先看全局

### 0.1 本期做什么

```text
第一部分：把集群搭起来（1.1 ~ 1.15）
   主机规划 → 环境初始化 → 部署 containerd → 安装 kubeadm → 初始化集群
   → 装 Calico → 业务测试 → 命令补全

第二部分：学会用资源对象（二、三章）
   资源概述 → Pod → Deployment（nginx / redis）→ Service → Ingress
```

### 0.2 用到的版本

| 组件 | 版本 | 说明 |
| --- | --- | --- |
| **Kubernetes** | **1.28.12** | 比第一期的 1.23 新了 5 个小版本 |
| **容器运行时** | **containerd 1.6.32** | ★ **不再用 Docker**（K8s 1.24 移除了 dockershim）|
| **Calico** | **3.28.5** | CNI 网络插件 |
| 操作系统 | rockylinux 8.10 | 与第一期一致 |
| 内核 | 5.19.12 | 需要升级（默认内核版本太低）|

---

## 一、k8s v1.28 + containerd 部署

### 1.1 主机规划

| 角色 | 主机名 | IP 地址 | 操作系统 | 硬件配置 | 组件 |
| --- | --- | --- | --- | --- | --- |
| **Master** | master-203 | 172.22.4.203 | rockylinux8.10 | 2 核 / 4G / 50G | apiserver 1.28.12、controller-manager 1.28.12、scheduler 1.28.12、kubelet 1.28.12 |
| **Worker** | node01-204 | 172.22.4.204 | rockylinux8.10 | 2 核 / 4G / 50G | kube-proxy、kubelet 1.28.12 |
| **Worker** | node02-205 | 172.22.4.205 | rockylinux8.10 | 2 核 / 4G / 50G | kube-proxy、kubelet 1.28.12 |

### 1.2 环境配置

> 讲义此节为空标题，内容并入 1.3。

### 1.3 host 配置

**① 升级内核**（RockyLinux 8.10 默认内核版本偏低）

```bash
mkdir /opt/kernel519 && cd /opt/kernel519

# 把 kernel-ml-* 的 rpm 包用 lrzsz 传进来
dnf localinstall -y kernel-ml-*

grubby --default-kernel                                          # 查看当前默认内核
grubby --set-default /boot/vmlinuz-5.19.12-1.el8.elrepo.x86_64   # 设置成新内核
reboot
```

**② 设置主机名**（三台分别执行）

```bash
# master-203 机器
hostnamectl set-hostname master-203 && bash

# node01-204 机器
hostnamectl set-hostname node01-204 && bash

# node02-205 机器
hostnamectl set-hostname node02-205 && bash
```

**③ 三台机器都配置 hosts 解析**

```bash
cat >>/etc/hosts<<EOF
172.22.4.203 master-203
172.22.4.204 node01-204
172.22.4.205 node02-205
EOF
```

> **为什么必须做**：kubeadm 用**主机名**做节点注册，hosts 不通会导致节点无法加入集群。

### 1.4 关闭防火墙与 SELinux

```bash
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i '/^SELINUX=/ c SELINUX=disabled' /etc/selinux/config
```

### 1.5 时间同步

```bash
dnf install -y wget tree lrzsz psmisc net-tools vim chrony
systemctl restart chronyd
chronyc sources          # 确认有 ^* 的同步源
```

### 1.6 禁用 swap

```bash
swapoff -a && sed -i 's/.*swap.*/#&/' /etc/fstab && free -m
```

> **为什么必须关**：kubelet 的资源计算和调度策略都假设节点没有 swap，**不关 `kubeadm init` 会直接报错退出**。

### 1.7 配置 ulimit

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

> ⚠️ **原文勘误**：讲义里写的是 `* seft memlock unlimited`，`seft` 是 `soft` 的笔误，照抄会报错。

### 1.8 调整内核参数

```bash
# ① 加载内核模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
lsmod | grep br_netfilter

# ② 调整内核参数
cat >> /etc/sysctl.d/k8s.conf << EOF
vm.swappiness=0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# ③ 应用设置
sysctl --system
```

| 参数 | 作用 |
| --- | --- |
| `overlay` 模块 | containerd 的 overlayfs 存储驱动需要它 |
| `br_netfilter` 模块 | 让 iptables 能处理网桥流量（**否则 Service 不通**）|
| `bridge-nf-call-iptables = 1` | 网桥流量交给 iptables 处理 |
| `ip_forward = 1` | 开启 IP 转发（**Pod 跨主机通信的前提**）|

### 1.9 配置 ipvs

```bash
dnf install ipset ipvsadm -y
mkdir -p /etc/sysconfig/modules

cat <<EOF >/etc/sysconfig/modules/ipvs.modules
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

# 加执行权限并运行，验证模块是否加载成功
chmod +x /etc/sysconfig/modules/ipvs.modules && \
/bin/bash /etc/sysconfig/modules/ipvs.modules && \
lsmod | grep -e ip_vs -e nf_conntrack

reboot
```

> **为什么**：kube-proxy 支持 iptables 和 **ipvs** 两种模式。**ipvs 性能更好**，但需要内核提前加载这些模块。本期是**直接在部署阶段就启用 ipvs**。

### 1.10 部署 containerd（★ 本期最大的变化）

> **为什么用 containerd 而不是 Docker**：**K8s 1.24 起移除了 dockershim**，不再原生支持 Docker 作为运行时。containerd 更轻量、是 CRI 标准实现，也是现在的默认选择。

```bash
# ① 安装依赖并添加 docker-ce 源（containerd 的包也在这个源里）
dnf install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# ② 安装 containerd
dnf install -y containerd.io-1.6.32

# ③ 查看可用版本
yum list containerd.io.x86_64 --showduplicates | sort -r

# ④ 备份原配置，生成默认配置
cp /etc/containerd/config.toml /etc/containerd/config.toml.lgh.bak
containerd config default | tee /etc/containerd/config.toml
```

**⑤ 关键修改一：cgroup 驱动改成 systemd**

```bash
sed -i "s#SystemdCgroup\ \=\ false#SystemdCgroup\ \=\ true#g" /etc/containerd/config.toml
```

> **为什么必须改**：kubelet 用 systemd 管理 cgroup，containerd 也要用同一个驱动，**两边不一致会导致 kubelet 起不来或 Pod 异常**。

**⑥ 关键修改二：把 sandbox 镜像仓库换成国内地址**

```bash
sed -i "s#registry.k8s.io#registry.aliyuncs.com/google_containers#g" /etc/containerd/config.toml
```

> **为什么**：k8s 的 pause 镜像默认从 `registry.k8s.io` 拉，**国内基本拉不动**。

**⑦ 关键修改三：配置镜像加速器**

```bash
# 开启 certs.d 配置目录
sed -i '147s|config_path = ""|config_path = "/etc/containerd/certs.d"|' /etc/containerd/config.toml

mkdir -p /etc/containerd/certs.d/docker.io

cat >/etc/containerd/certs.d/docker.io/hosts.toml<<'EOF'
server = "https://docker.io"

[host."https://docker.rainbond.cc"]
  capabilities = ["pull", "resolve"]

[host."https://docker.1ms.run"]
  capabilities = ["pull", "resolve"]

[host."https://hub.1panel.dev"]
  capabilities = ["pull", "resolve"]
EOF
```

> **注意**：`config_path` 的**行号（147）在不同版本可能不同**，改之前用 `grep -n 'config_path' /etc/containerd/config.toml` 确认。
> 讲义里还混入了一段 `[plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]`，**那是 config.toml 的语法，不能写进 hosts.toml**，要分开放。

### 1.11 安装 kubeadm 1.28.12

**① 下载 crictl 工具**（containerd 的命令行客户端，作用类似 `docker` 命令）

```bash
cd /opt
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.28.0/crictl-v1.28.0-linux-amd64.tar.gz
tar -zxvf crictl-v1.28.0-linux-amd64.tar.gz -C /usr/local/bin
```

**② 配置 crictl 连接 containerd**

```bash
cat >/etc/crictl.yaml<<'EOF'
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
EOF
```

**③ 启动 containerd 并测试**

```bash
systemctl daemon-reload && systemctl start containerd && systemctl enable containerd

crictl pull nginx:1.18      # 从互联网拉取镜像
crictl images               # 查看本地镜像
```

**④ 配置 k8s 的 yum 源并安装组件**

```bash
cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/repodata/repomd.xml.key
EOF

dnf install -y kubeadm-1.28.12 kubelet-1.28.12 kubectl-1.28.12
```

**⑤ 配置 kubelet**

```bash
cat > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"
EOF

systemctl start kubelet
```

> **注意**：`KUBE_PROXY_MODE="ipvs"` 在 1.28 里**已不生效**（模式由 kubeadm 配置文件的 `KubeProxyConfiguration` 决定）。

### 1.12 k8s 集群初始化

**① 生成并修改 kubeadm 配置（只在 master 节点操作）**

```bash
kubeadm config print init-defaults > kubeadm.yml
```

修改后的 `kubeadm.yml`（★ 标注的是必须改的地方）：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.22.4.203        # ★ 改成 Master 的 IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock   # ★ 指向 containerd
  imagePullPolicy: IfNotPresent
  name: master-203                       # ★ 改成自己的主机名
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers   # ★ 换成国内仓库
kind: ClusterConfiguration
kubernetesVersion: 1.28.12
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/16            # ★ Service 网段
  podSubnet: 10.10.0.0/16                # ★ Pod 网段(要和 Calico 一致)
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs                               # ★ 直接启用 ipvs 模式
```

**② 拉取镜像（三个节点都操作，二选一）**

```bash
# 方式一：用离线 tar 包导入（内网/网络差时用这个）
tar xf k8s-1.28.12-master-images.tar.gz
bash crictl_load_k8s.1.28.12.images.sh

# 方式二：在线拉取
kubeadm config images pull --config kubeadm.yml
```

**③ 初始化集群（在 master 上操作）**

```bash
kubeadm init --config=kubeadm.yml --upload-certs --v=6
```

**④ 配置 kubectl 的环境变量**

```bash
cat > /etc/profile.d/k8s_env.sh <<EOF
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF

source /etc/profile
echo $KUBECONFIG
```

> **为什么**：不设 `KUBECONFIG`，kubectl 找不到凭据，会报 `connection refused`。

**⑤ 检查集群状态**

```bash
kubectl get nodes
# NAME         STATUS     ROLES           AGE    VERSION
# master-203   NotReady   control-plane   115s   v1.28.12
```

> `NotReady` 是正常的 —— 网络插件（Calico）还没装。

**⑥ 在 2 个 Worker 节点上执行加入命令**

```bash
kubeadm join 172.22.4.203:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:4968cad9700ba76eb1a0264dd62becee7a0d0525e28dfe8d6463e8a5532999e2
```

```bash
# 回到 master 查看
kubectl get nodes
# NAME         STATUS     ROLES           AGE     VERSION
# master-203   NotReady   control-plane   3m25s   v1.28.12
# node01-204   NotReady   <none>          24s     v1.28.12
# node02-205   NotReady   <none>          5s      v1.28.12
```

### 1.13 安装 Calico 组件

```bash
# ① 下载 Calico 清单
wget --no-check-certificate -O calico_3.28.5.yaml \
  https://raw.githubusercontent.com/projectcalico/calico/v3.28.5/manifests/calico.yaml

# ② 修改 CIDR（约 4962 行）—— 必须和 kubeadm.yml 里的 podSubnet 一致
# - name: CALICO_IPV4POOL_CIDR
#   value: 10.10.0.0/16
# ★ 注意后面的 value 要对齐（YAML 对缩进敏感）

# ③ 应用
kubectl apply -f calico_3.28.5.yaml

# ④ 验证
kubectl get pod -A
```

```text
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-6476886d5-4qrl9    1/1     Running   0          14s
kube-system   calico-node-cz6vm                           0/1     Running   0          14s
kube-system   calico-node-gn67n                           0/1     Running   0          14s
kube-system   calico-node-n5kbj                           0/1     Running   0          14s
kube-system   coredns-66f779496c-s79cm                    1/1     Running   0          8m4s
kube-system   coredns-66f779496c-xct66                    1/1     Running   0          8m4s
kube-system   etcd-master-203                             1/1     Running   0          8m18s
kube-system   kube-apiserver-master-203                   1/1     Running   0          8m18s
kube-system   kube-controller-manager-master-203          1/1     Running   0          8m18s
kube-system   kube-proxy-22lw7                            1/1     Running   0          5m1s
kube-system   kube-proxy-4fvd2                            1/1     Running   0          5m20s
kube-system   kube-proxy-7b6v9                            1/1     Running   0          8m4s
kube-system   kube-scheduler-master-203                   1/1     Running   0          8m18s
```

> **`calico-node` 显示 `0/1` 是正常的**（还在初始化），等一会儿会变成 `1/1`。
> **CIDR 不一致是最常见的坑**：Calico 按自己的网段给 Pod 分 IP，如果和 `podSubnet` 不一样，**Pod 之间就是不通**。

```bash
# ⑤ 等 Calico 就绪，节点会变成 Ready
kubectl get nodes
# NAME         STATUS   ROLES           AGE     VERSION
# master-203   Ready    control-plane   8m58s   v1.28.12
# node01-204   Ready    <none>          5m57s   v1.28.12
# node02-205   Ready    <none>          5m38s   v1.28.12
```

### 1.14 业务功能测试

```bash
# 创建一个 2 副本的 nginx Deployment
kubectl create deployment nginx --image=nginx:1.28 --replicas=2

# 查看 Pod 及内部 IP
kubectl get pods -o wide
```

### 1.15 命令补全（很实用）

```bash
dnf install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)

# 写进 .bashrc 永久生效
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

**查看集群里有哪些资源类型**：

```bash
kubectl api-resources
```

| 列 | 含义 |
| --- | --- |
| `NAME` | 资源类型的复数名称（如 `pods`、`services`）|
| `SHORTNAMES` | 简称（`po` = pods、`svc` = services、`cm` = configmaps）|
| `APIVERSION` | 属于哪个 API 版本 |
| `NAMESPACED` | **是否受命名空间限制** |
| `KIND` | 资源的类型名（如 `Pod`、`Service`）|

---

## 二、Pod 与 Deployment 资源

### 2.1 资源概述

> **Kubernetes 里，一切都是资源对象。**

| 资源 | 作用 |
| --- | --- |
| **Pod** | 最小运行单元 |
| **Deployment** | 无状态应用管理 |
| **StatefulSet** | 有状态应用 |
| **DaemonSet** | 节点守护进程 |
| **Service** | 服务发现 |
| **Ingress** | 外部访问入口 |
| **ConfigMap** | 配置管理 |
| **Secret** | 敏感信息 |
| **PVC / PV** | 存储 |
| **Namespace** | 资源隔离 |

### 2.2 Pod 资源

#### 2.2.1 什么是 Pod

> **Pod 是 Kubernetes 的最小调度单位**，一个 Pod 可以包含多个容器，并且包含一个**基础容器（pause）**，这些容器**共享 pause 的网络资源（网络、存储、IP）**。

**关键理解**：
- 同一个 Pod 里的容器共享网络命名空间 → 可以用 `localhost` 互相访问；
- Pod 是"调度的最小单位"，**不能只调度 Pod 里的某一个容器**。

#### 2.2.2 Pod 案例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test124
spec:
  containers:
  - name: nginx124
    image: nginx:1.24
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f nginx-pod.yaml
# pod/nginx-test124 created
```

**YAML 字段逐个解释**：

```yaml
apiVersion: v1          # api 版本（不同资源用不同版本）
kind: Pod               # 创建资源的类型
metadata:               # 元数据
  name: nginx-test124   # Pod 的名称
spec:                   # Pod 的规格（状态描述）
  containers:           # 定义容器
  - name: nginx124      # 容器名称
    image: nginx:1.24   # 镜像名
    ports:
    - containerPort: 80 # 容器监听的端口
```

> **查资源属性的神器**：`kubectl explain pod.spec.containers` —— **不用记字段，需要什么查什么**。

### 2.3 Deployment

#### 2.3.1 Deployment 是什么

> **它主要管理无状态应用**。
> **功能：副本管理、滚动更新、回滚。**

**为什么要用 Deployment 而不是直接用 Pod**：直接创建的 Pod **挂了不会自动重建**；Deployment 保证"始终有 N 个副本在跑"，还能滚动升级和回滚。

#### 2.3.2 nginx 的 Deployment 部署

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: qianshan-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  namespace: qianshan-ns
  labels:
    app: qianshan
spec:
  replicas: 1                      # 副本数，保证高可用
  selector:
    matchLabels:
      app: qianshan                # ★ 必须匹配 template 的 labels
  template:
    metadata:
      labels:
        app: qianshan
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
        resources:                 # 可选，建议生产环境配置资源限制
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: qianshan-ns
spec:
  selector:
    app: qianshan                  # ★ 选择器必须匹配 Pod 的 labels
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

**`resources` 两个字段的区别**：

| 字段 | 含义 | 不设会怎样 |
| --- | --- | --- |
| `requests` | **调度依据**（保证至少这么多）| 调度器不知道要预留多少资源 |
| `limits` | **硬上限** | 应用可能把节点资源吃光（CPU 被限流、内存超了被 OOMKilled）|

#### 2.3.3 redis 的 Deployment 部署

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-redis
  namespace: qianshan-ns
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:8.0
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: qianshan-ns
  labels:
    app: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
    nodePort: 32333
    protocol: TCP
  type: NodePort
```

---

## 三、k8s 的 Service 服务

### 3.1 Service 定义

**为什么要有 Service？**

```text
① Pod 是短暂的，IP 地址是变化的
   → 每次 Pod 重建，IP 都变，应用没法固定连它

② 解决多个 Pod 的负载均衡问题
   → 前面挂多个 Pod，需要一个统一入口把请求分下去
```

```text
┌─────────────────────────────────────────┐
│           客户端 (Client)                │
│            curl 10.96.1.1:80            │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│           Service（稳定入口）             │
│           ClusterIP: 10.96.1.1          │
│           Port: 80 → TargetPort: 80     │
└───────────────────┬─────────────────────┘
                    │ 负载均衡
        ┌───────────┼───────────┐
        ▼           ▼           ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   Pod A     │ │   Pod B     │ │   Pod C     │
│ 10.244.1.5  │ │ 10.244.1.6  │ │ 10.244.2.3  │
│   nginx     │ │   nginx     │ │   nginx     │
└─────────────┘ └─────────────┘ └─────────────┘
```

**外部访问的完整路径**：

```text
外部用户 → NodeIP:NodePort
        → iptables/ipvs 规则（kube-proxy 维护）
        → PodIP:targetPort
        → containerPort
```

> **链路上的四个"端口"不要混**：`nodePort`（节点对外）→ `port`（Service 端口）→ `targetPort`（Pod 端口）→ `containerPort`（容器实际监听）

### 3.2 Service 类型

| 类型 | 访问范围 | 适用场景 | 特点 |
| --- | --- | --- | --- |
| **ClusterIP** | **集群内部** | 微服务间通信 | **默认类型**，提供内部服务发现 |
| **NodePort** | 集群外部 | 私有云环境 | 通过 **节点 IP + 端口** 暴露服务 |
| **LoadBalancer** | 公有网络 | 云平台生产环境 | 自动创建云负载均衡器 |
| **ExternalName** | 内外互通 | 集成外部服务 | 把外部服务引入集群内部（比如数据库）|

### 3.3 比较完整的 Service 案例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-service-deployment
  labels:
    app: nginx003            # Deployment 自己的标签，随便定义
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-service2    # ★ 必须与 template.labels 中 pod 标签一致
  template:
    metadata:
      labels:
        app: nginx-service2  # Pod 的实际标签
    spec:
      containers:
      - name: nginx
        image: nginx:1.28
        ports:
        - containerPort: 80  # 容器实际监听的端口
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service003
spec:
  selector:
    app: nginx-service2      # ★ 必须与 Pod 标签一致
  ports:
  - name: http
    port: 80                 # Service 端口
    targetPort: 80           # Pod 端口（对应容器的 containerPort）
  type: ClusterIP            # 默认就是 ClusterIP，可省略
```

> ⚠️ **三处 `app:` 必须一致**：Deployment 的 `selector.matchLabels`、Pod 模板的 `labels`、Service 的 `selector`。**任何一处不一致，Service 就找不到 Pod**（表现是访问超时）。

### 3.4 Ingress 资源

> **Ingress 是集群的外部访问入口**（七层），比 NodePort 更适合生产：可以按域名/路径把流量分发到不同的 Service，还支持 HTTPS 卸载。

| | NodePort | Ingress |
| --- | --- | --- |
| 层次 | 四层（TCP）| **七层（HTTP/HTTPS）** |
| 入口数量 | 每个服务占一个端口 | **所有服务共用一个入口**（按域名/路径区分）|
| 需要额外组件 | 不需要 | **需要装 Ingress Controller** |
| 适合 | 实验、少量服务 | **生产环境** |

---

## 四、闭环自检清单

| 阶段 | 检查项 | 命令 | 通过标准 |
| --- | --- | --- | --- |
| 环境 | 主机名与 hosts | `hostname` / `ping -c1 node01-204` | 三台互通 |
| 环境 | swap 已关 | `swapon -v` | 无输出 |
| 环境 | 内核模块 | `lsmod \| grep -e ip_vs -e br_netfilter` | 都有记录 |
| containerd | 服务运行 | `systemctl is-active containerd` | active |
| containerd | cgroup 驱动 | `grep SystemdCgroup /etc/containerd/config.toml` | `true` |
| containerd | 能拉镜像 | `crictl pull nginx:1.18` | 成功 |
| 组件 | 版本正确 | `kubeadm version` | 1.28.12 |
| 集群 | 节点就绪 | `kubectl get nodes` | 三台全 **Ready** |
| 集群 | 系统 Pod | `kubectl get pod -A` | 全部 Running |
| 集群 | kube-proxy 模式 | `ipvsadm -ln` | 能看到规则（说明是 ipvs）|
| 业务 | Deployment 副本 | `kubectl get deploy -A` | READY 等于期望副本数 |
| 业务 | Service 可访问 | `curl NodeIP:NodePort` | 返回 nginx 页面 |
| 工具 | 命令补全 | 输入 `kubectl get po` + Tab | 能自动补全 |

---

## 五、★ 与第一期的对比（第二期 vs 第一期）

> 第一期笔记：`stage2-07-k8s-old.md`（k8s 1.23.17 + Docker）
> 第二期笔记：本篇（k8s 1.28.12 + containerd）

### 5.1 一张表看全部差异

| 对比项 | 第一期 | **第二期** | 为什么要改 |
| --- | --- | --- | --- |
| **K8s 版本** | **1.23.17** | **1.28.12** | 1.23 已 EOL（2023-02），1.28 更接近生产 |
| **容器运行时** | **Docker 20.10.24** | **containerd 1.6.32** | ★ **K8s 1.24 移除了 dockershim**，不再原生支持 Docker |
| **运行时管理工具** | `docker` 命令 | **`crictl`** | 运行时换了，管理命令也换 |
| **运行时配置文件** | `/etc/docker/daemon.json` | **`/etc/containerd/config.toml`** | — |
| **镜像加速方式** | daemon.json 的 `registry-mirrors` | **`/etc/containerd/certs.d/docker.io/hosts.toml`** | containerd 用 certs.d 机制 |
| **cgroup 驱动配置** | daemon.json 的 `exec-opts` | config.toml 的 **`SystemdCgroup = true`** | 同理，配置位置变了 |
| **kube-proxy 模式** | 默认 iptables，**后期手工切 ipvs** | **初始化时就声明 `mode: ipvs`** | 更规范：一次配好 |
| **初始化方式** | `kubeadm init` **命令行参数** | **`kubeadm init --config=kubeadm.yml`** | 配置文件可复用、可版本管理 |
| **Pod 网段** | `10.2.0.0/16` | **`10.10.0.0/16`** | 规划不同 |
| **Service 网段** | `10.1.0.0/16` | **`10.96.0.0/16`** | 同上 |
| **Calico 版本** | **3.24.5** | **3.28.5** | 跟随 K8s 版本升级 |
| **k8s yum 源** | `mirrors.aliyun.com/kubernetes/yum/...` | **`mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/`** | ★ 旧源已停止同步 |
| **kubelet 配置** | 无特殊配置 | `/etc/sysconfig/kubelet` 指定 cgroup 驱动 | — |
| **命令补全** | 未提及 | **1.15 节专门讲** | 实用技能 |

### 5.2 内容重心的变化（更重要）

| | 第一期 | 第二期 |
| --- | --- | --- |
| **主题** | **"怎么把集群搭起来"** | **"搭起来 + 怎么用"** |
| 部署部分 | 占绝大部分篇幅 | 压缩成 1.1~1.15，**更精炼** |
| 资源对象 | 只涉及 Deployment / Service（部署 nginx 时用）| **系统讲解了 Pod / Deployment / Service，还提到 Ingress** |
| 新增内容 | — | **资源概述、`kubectl api-resources`、`kubectl explain`、命令补全、Namespace、资源限制** |
| Dashboard | **有完整章节**（安装 + Token 登录）| **没有**（改讲 Ingress）|
| 排错内容 | 有完整章节 | **没有专门的排错章节**（需要自己补）|

### 5.3 「同样一件事，两期分别怎么做」

**① 部署容器运行时**

```bash
# 第一期：装 Docker
dnf -y install docker-ce-20.10.24 docker-ce-cli-20.10.24 containerd.io-1.6.3
systemctl start docker
docker pull nginx:1.24          # 用 docker 命令

# 第二期：装 containerd（不装 docker-ce）
dnf install -y containerd.io-1.6.32
containerd config default | tee /etc/containerd/config.toml
sed -i "s#SystemdCgroup = false#SystemdCgroup = true#g" /etc/containerd/config.toml
systemctl start containerd
crictl pull nginx:1.18          # ★ 改用 crictl 命令
```

**② 初始化集群**

```bash
# 第一期：命令行参数
kubeadm init \
  --kubernetes-version 1.23.17 \
  --image-repository registry.aliyuncs.com/google_containers \
  --apiserver-advertise-address=172.22.4.203 \
  --service-cidr=10.1.0.0/16 \
  --pod-network-cidr=10.2.0.0/16

# 第二期：配置文件
kubeadm config print init-defaults > kubeadm.yml
vim kubeadm.yml                 # 改 IP / 网段 / criSocket / imageRepository / mode: ipvs
kubeadm init --config=kubeadm.yml --upload-certs --v=6
```

**③ 切换 ipvs 模式**

```bash
# 第一期：集群跑起来后手工改 ConfigMap
kubectl edit configmap -n kube-system kube-proxy    # mode: "" → "ipvs"
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# 第二期：初始化前就在 kubeadm.yml 里声明
# apiVersion: kubeproxy.config.k8s.io/v1alpha1
# kind: KubeProxyConfiguration
# mode: ipvs
```

### 5.4 一句话总结这个变化

> **第一期解决"能不能跑起来"，第二期解决"能不能用好"。**
>
> 技术上最大的变化是 **容器运行时从 Docker 换成 containerd** —— 这不是"换个软件"，而是因为 **K8s 1.24 移除了 dockershim**，Docker 不再是受支持的运行时。
> 换句话说：**以前是"K8s 调用 Docker 起容器"，现在是"K8s 直接调用 containerd 起容器"，中间少了一层，更轻量。**

---

## 六、常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `crictl` 报连接失败 | 端点写错 / containerd 没起 | 检查 `/etc/crictl.yaml` 和 `systemctl status containerd` |
| `crictl pull` 拉不动 | 镜像加速没配好 | 检查 `certs.d/docker.io/hosts.toml` 和 config.toml 的 `config_path` |
| 节点一直 `NotReady` | 没装 Calico / Calico 没起来 | `kubectl get pod -n kube-system` |
| Pod 一直 `Pending` | 调度不了 | `kubectl describe pod`，看是不是 requests 太大 |
| Pod `ContainerCreating` 很久 | 镜像在拉取 | `crictl images` 看有没有拉下来 |
| Pod 之间不通 | **CIDR 和 podSubnet 不一致** | 改成同一网段，重新 apply Calico |
| Service 访问超时 | **三处 `app` 标签不一致** | 核对 selector 和 labels |
| `kubeadm init` 报 swap 错误 | 没关 swap | `swapoff -a` + 注释 fstab + 重启 |
| `kubectl` 报 `connection refused` | 没设 `KUBECONFIG` | 配 `/etc/profile.d/k8s_env.sh` |
| `kubeadm join` 报 token 失效 | token 24 小时过期 | `kubeadm token create --print-join-command` |

---

## 附录：两期的定位对照

| | 第一期（stage2-07）| 第二期（本篇）|
| --- | --- | --- |
| **一句话定位** | k8s 1.23 集群部署**全流程**（含排错、Dashboard）| k8s 1.28 部署 + **资源对象使用** |
| **适合谁看** | 第一次搭集群、需要排错参考 | 已搭过一次、要升级版本 + 学怎么用 |
| **两者的关系** | 打基础（原理、排错方法通用）| 进阶（新版本 + 更多资源对象）|

> **建议结合看**：用第一期理解"为什么这么做"和"出错了怎么查"，用第二期掌握"新版本怎么装"和"资源对象怎么用"。