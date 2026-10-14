# K8s 补课：骨架层

> **这篇在补什么**：补 K8s 的"世界观"—— 组件分工、声明式机制、标签关联、YAML 结构、网络与端口。
> **为什么要补**：大部分 K8s 教程直接从"装集群 + 写 YAML"开始，把骨架层当成已知的跳过了。
> 结果是每个 YAML 都看得懂，但连起来不知道在干嘛，记了又忘。
> **怎么用**：先通读一遍，再打开《K8s 第二期》那篇笔记对照第九章，原来卡住的地方基本都会通。

---

## 〇、先看清全局

### 0.1 K8s 知识的三个层次

```text
第 1 层 | 容器        Docker / 镜像 / 容器生命周期        ← 学 K8s 的前置
第 2 层 | 骨架层      组件分工、声明式、标签、网络端口     ← ★ 本篇要补的
第 3 层 | 资源与用法   Pod / Deployment / Service / Ingress
```

**跳过第 2 层直接学第 3 层，就会出现"每步都懂，整体不懂"。**

### 0.2 一句话理解 K8s 在干什么

> **你告诉它"你想要什么状态"，它负责把现实变成那个状态，并且一直维持住。**

```text
你说的     :  "我要 2 个 nginx"
K8s 做的   :  找机器 → 起容器 → 盯着它 → 挂了重建 → 不够了补上
```

这就是"声明式"和"自动修复"的全部含义。

### 0.3 组件全景图

```text
                     ┌────────── 你的电脑 / 运维机 ──────────┐
                     │  kubectl (客户端,人用它下命令)        │
                     └───────────────────┬──────────────────┘
                                         │ HTTPS 6443
     ┌───────────────────────────────────▼────────────────────────────────────┐
     │                控制平面 (Control Plane,跑在 Master 上)                  │
     │                                                                        │
     │   kube-apiserver  ← 唯一入口,认证 / 鉴权 / 校验,读写 etcd                │
     │        │                                                               │
     │        ├──▶  etcd                    ← 数据库,存【集群所有状态】          │
     │        ├──▶  kube-scheduler          ← 决定 Pod 放哪台机器               │
     │        └──▶  kube-controller-manager ← 一堆控制器,不停对比期望与实际      │
     └───────────────────────────────────┬────────────────────────────────────┘
                                         │ apiserver 下发
     ┌───────────────────────────────────▼────────────────────────────────────┐
     │                     工作节点 (node01 / node02 …)                        │
     │                                                                        │
     │   kubelet      ← 节点管家:领任务、盯容器、报状态                          │
     │      │                                                                 │
     │      ├── CRI ──▶ containerd ──▶ runc ──▶ 容器进程                       │
     │      └── CNI ──▶ Calico     ──▶ 给 Pod 分配 IP、打通网络                 │
     │                                                                        │
     │   kube-proxy   ← 写 iptables / ipvs 规则,让 Service 能把流量转到 Pod      │
     └────────────────────────────────────────────────────────────────────────┘
```

---

## 一、架构与组件分工

### 1.1 控制平面（大脑，装在 Master）

| 组件 | 角色 | 干什么 | 出问题会怎样 |
| --- | --- | --- | --- |
| **kube-apiserver** | 前台接待 | 唯一入口。所有请求都经过它：认证、鉴权、准入校验，然后读写 etcd | 整个集群"失联"，kubectl 全部报错 |
| **etcd** | 档案室 | 键值数据库，**存集群的全部状态**（唯一数据源） | 数据全丢，集群等于重建 |
| **kube-scheduler** | 派单员 | 决定"这个 Pod 放到哪台节点" | Pod 一直 `Pending` |
| **kube-controller-manager** | 监理组 | 一堆控制器的集合，**不停对比"期望状态"和"实际状态"** | 副本不会自愈、故障不重建 |

> **记住一句话**：apiserver 是唯一入口，etcd 是唯一数据源。
> 其他组件都**不直接改集群状态**，而是通过 apiserver 读写 etcd。

### 1.2 工作节点（手脚，每台机器都有）

| 组件 | 角色 | 干什么 |
| --- | --- | --- |
| **kubelet** | 节点管家 | 向 apiserver 注册并心跳；领回"我这台该跑哪些 Pod"；调 CRI 起容器；把状态报回去 |
| **kube-proxy** | 网络邮差 | 在本节点写 iptables / ipvs 规则，实现 Service 的转发 |
| **容器运行时** | 真正干活的工人 | containerd（+ runc），负责拉镜像、起容器 |
| **CNI 插件** | 网络施工队 | Calico，给 Pod 分配 IP、打通跨节点通信 |

> ⚠️ **kubelet 和 kube-proxy 不是"只装在 Master 上的东西"** —— 每台节点（包括 Master）都有一个。
> 而且两者形态不同：**kubelet 是宿主机上的 systemd 服务，不是 Pod**；kube-proxy 是 Pod。

### 1.3 ★ 一次 `kubectl apply` 的完整链路（最该记住的一张图）

```text
你敲: kubectl apply -f nginx-deploy.yaml
        │
        ▼
① kubectl 把 YAML 发给 kube-apiserver
   (kubectl 只是"客户端",自己不干活)
        │
        ▼
② apiserver 校验格式、鉴权,把"我要 1 个 nginx"写进 etcd
   ← 这一步叫【期望状态】,只写数据库,容器还没影
        │
        ▼
③ Deployment 控制器发现:期望 1 个,实际 0 个
   → 创建一个 ReplicaSet 对象 → 再创建一个 Pod 对象(还是只写 etcd)
        │
        ▼
④ kube-scheduler 发现有 Pod 没分配节点
   → 挑一台机器,把结果写进 etcd           ← 这一步叫【调度】
        │
        ▼
⑤ 被选中的节点上,kubelet 发现"有 Pod 分给我了"
   ├── 调 CRI(containerd) → 拉镜像 → 起容器
   └── 调 CNI(Calico)     → 给 Pod 分配 IP
        │
        ▼
⑥ kubelet 把 Pod 的【实际状态】(Running / IP / 是否 Ready)报回 apiserver
        │
        ▼
⑦ 控制器对比:期望 1 = 实际 1 ✓  这次协调结束
   但它不会停 —— 它会一直盯着,一旦不一致就再来一遍
```

**看懂这张图，一半的疑问会自动消失**：

| 你的疑问 | 答案在这一步 |
| --- | --- |
| 为什么 Pod 挂了会自动重建？ | 步 ③ / ⑦ 那个循环一直在跑 |
| 节点的 `Ready` 状态是谁报的？ | 步 ⑥，kubelet 上报的 |
| 为什么装了 Calico 节点才 `Ready`？ | 步 ⑤ 里的 CNI 没就绪，Pod 网络不通 |
| `Pending` 是卡在哪？ | 步 ④ 调度不了 |
| `ContainerCreating` 卡很久？ | 步 ⑤ 镜像在拉 |
| `kubectl` 到底干了什么？ | 只是步 ①，发个请求就走了 |

### 1.4 四个最容易混的名字

| 名字 | 是什么 | 装在哪 | 形态 | 一句话 |
| --- | --- | --- | --- | --- |
| **kubectl** | 命令行客户端 | 一般只装 Master / 运维机 | 一次性命令 | **人**用来下命令的遥控器 |
| **kubelet** | 节点上的守护进程 | **每个节点**都要装 | 常驻（systemd） | **机器**用它管容器 |
| **kubeadm** | 集群安装工具 | 装的时候用 | 一次性命令，装完退场 | 把集群**搭起来** |
| **kube-proxy** | 网络代理 | 每个节点 | 常驻（以 Pod 形式） | 让 Service 的流量**转得出去** |

> 记忆法：`-ctl` 是 control（人用的控制器），`-let` 是"小东西、常在"（机器用的守护进程），`adm` 是 admin（管理员工具）。

---

## 二、声明式与控制器

### 2.1 命令式 vs 声明式

| | 命令式 | 声明式（K8s 用的） |
| --- | --- | --- |
| 你怎么说 | "做这步、再做那步" | "我要最终长这样" |
| 例子 | `docker run nginx`、`docker start xxx` | `kubectl apply -f deploy.yaml` |
| 出故障 | 你自己发现、自己重启 | 系统自己发现、自己修复 |
| 类比 | 跟司机报路线 | 告诉导航目的地 |

**这就是为什么在 K8s 里反复 `apply` 同一个文件是安全的** —— 它不看"你做了几次"，只看"现在是不是我想要的样子"。

### 2.2 期望状态 vs 实际状态

```text
etcd 里存的          = 【期望状态】"应该有 2 个 nginx Pod"
kubelet 报回来的      = 【实际状态】"现在只有 1 个在跑,另一个挂了"
                            ↓
                控制器一比对:不一致
                            ↓
                      动手补齐(再起一个)
                            ↓
                直到 实际 = 期望 为止,然后继续盯着
```

> 这个"对比 → 修正 → 再对比"的循环叫 **调谐循环（reconcile loop）**，是 K8s 一切"自动"的来源。

### 2.3 控制器（Controller）是什么

**控制器 = 一段永远在跑的代码，负责让"实际"追上"期望"。** 常见的有：

| 控制器 | 负责什么 |
| --- | --- |
| Deployment Controller | 保证副本数、处理滚动更新和回滚 |
| ReplicaSet Controller | 保证指定数量的 Pod 存在 |
| Node Controller | 节点失联时标记并处理 |
| Endpoints Controller | 根据 Service 的 selector 维护"后端 Pod 列表" |

它们都在 **kube-controller-manager** 这一个进程里跑。

### 2.4 Deployment 和 Pod 的关系

```text
Deployment  ──创建──▶  ReplicaSet  ──创建──▶  Pod
（管版本 / 回滚）      （管副本数）            （真正跑容器）
```

**为什么不直接用 Pod**：

| 直接用 Pod | 用 Deployment |
| --- | --- |
| Pod 挂了**不会重建** | 挂了自动补一个 |
| 想升级只能删了重建，会中断 | **滚动更新**，一个一个换，不中断 |
| 无法回滚 | 一条命令回滚到上一版 |

---

## 三、Label 与 Selector（K8s 最核心的关联机制）

### 3.1 为什么不用名字、不用 IP 来关联

因为 **Pod 是"朝生暮死"的**：

```text
Pod 重建一次 → 名字变了、IP 变了
→ 如果 Service 记的是"Pod 名字"或"Pod IP",重建后立刻失联
```

所以 K8s 改用**标签（Label）+ 选择器（Selector）**：

> **不看"你是谁"，只看"你身上贴着哪个标签"。**
> Pod 换成新的没关系，只要标签还在，Deployment 和 Service 自动就"认"它。

### 3.2 Label 和 Selector 分别是什么

| | 是什么 | 谁写 |
| --- | --- | --- |
| **Label** | 贴在资源上的键值对标签，如 `app: qianshan` | 谁被选，谁贴 |
| **Selector** | 筛选条件，如 `matchLabels: {app: qianshan}` | 谁要选人，谁写条件 |

### 3.3 谁在选谁

| 谁 | 用 selector 去选 | 选中的结果 |
| --- | --- | --- |
| **Deployment** | 选 Pod 的 labels | 这些 Pod 归我管（副本数、升级） |
| **Service** | 选 Pod 的 labels | 这些 Pod 是我的后端，流量转给它们 |

**一个 Deployment + Service 里，标签出现三处，必须一致**：

```yaml
# ① Deployment 的筛选条件
spec:
  selector:
    matchLabels:
      app: qianshan        # ← 第 1 处
  template:
    metadata:
      labels:
        app: qianshan      # ← 第 2 处:Pod 身上真正贴的标签
---
# ② Service 的筛选条件
spec:
  selector:
    app: qianshan          # ← 第 3 处
```

**只要有一处对不上，Service 就找不到任何 Pod** —— 表现是**访问超时**（不是报错，因为 Service 本身创建成功了）。

### 3.4 常用排查命令

```bash
kubectl get pods --show-labels              # 看 Pod 身上有哪些标签
kubectl get pods -l app=qianshan            # 按标签筛 Pod
kubectl get endpoints <service名>           # ★ 看 Service 到底选中了哪些 Pod
#   ENDPOINTS 显示 <none> → 就是标签对不上
```

---

## 四、YAML 四段式

### 4.1 所有 K8s 资源都是同一个骨架

```yaml
apiVersion: apps/v1       # ① 用哪一套 API 版本
kind: Deployment          # ② 我要创建什么资源
metadata:                 # ③ 元数据:名字、命名空间、标签
  name: my-nginx
  namespace: qianshan-ns
  labels:
    app: qianshan
spec:                     # ④ 期望状态:我希望它长什么样
  replicas: 2
  # ...
```

| 段 | 含义 | 记忆点 |
| --- | --- | --- |
| `apiVersion` | API 版本 | Pod / Service 是 `v1`；Deployment 是 `apps/v1` |
| `kind` | 资源类型 | 大驼峰，如 `Pod`、`Deployment`、`Service` |
| `metadata` | 元数据 | 名字、命名空间、**标签**都在这 |
| `spec` | **期望状态** | 你真正要写的东西全在这 |

> 还有一段 `status`（实际状态），**是系统写的，你不用写**。

### 4.2 不用背字段，用 `kubectl explain`

```bash
kubectl explain pod                        # 看 Pod 有哪些字段
kubectl explain pod.spec                   # 一层层往下钻
kubectl explain pod.spec.containers        # 看容器的字段说明
kubectl explain deploy.spec.strategy --recursive   # 递归展开,带注释
```

> **这是 K8s 里最实用的一个命令** —— 不用记几百个字段，需要什么查什么。

### 4.3 常用 kubectl 命令速查

```bash
# 创建 / 更新 / 删除
kubectl apply -f xxx.yaml              # 声明式(推荐,可重复执行)
kubectl delete -f xxx.yaml
kubectl create deployment nginx --image=nginx:1.24   # 命令式,快速试

# 查看
kubectl get pods -o wide               # 带 IP 和所在节点
kubectl get all                        # 看本命名空间的常见资源
kubectl get pod <name> -o yaml         # 看完整定义(含系统填的字段)
kubectl describe pod <name>            # ★ 排障第一站,看事件

# 排障
kubectl logs <pod> [-f]                # 看日志
kubectl exec -it <pod> -- bash         # 进容器
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## 五、Service 与"四个端口"

### 5.1 为什么需要 Service

```text
问题 1:Pod IP 会变  → 应用没法固定连它
问题 2:有 3 个 Pod  → 需要一个统一入口做负载均衡
```

**Service 就是给一组 Pod 提供"固定入口 + 负载均衡"的抽象。** 它自己有稳定的 ClusterIP 和名字，后端 Pod 换来换去都不影响。

### 5.2 四个端口的链路（★ 最容易混）

```text
外部用户
   │ 访问 节点IP:30080
   ▼
nodePort       = 30080      ← ① 节点对外暴露的端口(30000-32767)
   ▼
port           = 80         ← ② Service 自己的端口(集群内用 ClusterIP:80 访问)
   ▼  由 kube-proxy 写的 iptables / ipvs 规则转发
targetPort     = 80         ← ③ Pod 上的端口
   ▼
containerPort  = 80         ← ④ 容器里应用真正监听的端口
```

| 端口 | 在哪一层 | 谁用 |
| --- | --- | --- |
| `nodePort` | 节点 | 集群外部用户 |
| `port` | Service | 集群内部其他服务 |
| `targetPort` | Pod | kube-proxy 的转发目标 |
| `containerPort` | 容器 | 应用监听 |

> `port` 和 `targetPort` 可以不一样（比如 port 80 → targetPort 8080），
> 但 **`targetPort` 必须等于容器里应用真正监听的端口**，否则打死也连不上。

### 5.3 Service 的四种类型

| 类型 | 访问范围 | 场景 | 说明 |
| --- | --- | --- | --- |
| **ClusterIP** | 仅集群内部 | 微服务之间互相调用 | **默认类型**，分配一个集群内 IP |
| **NodePort** | 集群外部 | 私有云 / 实验 | 在 ClusterIP 基础上，每台节点开一个端口 |
| **LoadBalancer** | 公网 | 云平台生产 | 自动找云厂商创建一个负载均衡器 |
| **ExternalName** | — | 把外部服务引入集群 | 其实是一条 DNS 记录，没有代理 |

> **它们不是四选一的平行关系**：NodePort 是在 ClusterIP 之上加了一层，LoadBalancer 又是在 NodePort 之上加了一层。

### 5.4 ClusterIP 从哪来？kube-proxy 干了什么？

```text
ClusterIP 的来源:
  kubeadm.yml 里的 serviceSubnet: 10.96.0.0/16
        ↓  apiserver 从这个网段里给每个 Service 分配一个 VIP
  比如 nginx-svc 拿到 10.96.1.1
  这个 IP 是"虚拟"的,不存在于任何网卡上,ping 不通是正常的

kube-proxy 的角色:
  它在每台节点上写 iptables / ipvs 规则:
     "凡是访问 10.96.1.1:80 的流量 → 负载均衡到某个 Pod 的 10.10.x.x:80"
        ↓
  所以 Service 的负载均衡不是"有个代理进程在转发"
  而是【内核里的转发规则】在干活 —— 效率极高
```

> 这就是"**Service 的 IP ping 不通但能访问**"的原因：它只是规则匹配的目标，不是真实主机。

### 5.5 和 Docker 的对照（把两套脑子接通）

| 你在 Docker 里习惯的 | K8s 里对应 |
| --- | --- |
| `docker run -p 80:80` | **NodePort**（nodePort → port → targetPort → containerPort 四层） |
| 自定义网络里用**容器名**互访 | **Service 名字**（ClusterIP + 集群 DNS） |
| 自己起 nginx 容器 | **Deployment** 里的 Pod 模板 |
| 容器挂了手动 `docker start` | kubelet + 控制器**自动重建** |
| `docker ps` | `kubectl get pods` |

---

## 六、Pod 网络与 CNI

### 6.1 Pod 的 IP 从哪来

```text
kubeadm.yml 里声明:  podSubnet: 10.10.0.0/16
        ↓
CNI 插件(Calico)根据这个网段,给每台节点分一个小段
    node01-204 → 10.10.1.0/24
    node02-205 → 10.10.2.0/24
        ↓
节点上每起一个 Pod,就从这个段里分一个 IP
    10.10.1.5 / 10.10.1.6 / 10.10.2.3 ...
```

### 6.2 为什么 CIDR 必须和 podSubnet 一致

这是新手最常踩的坑，原理其实很简单：

```text
如果 kubeadm 里写 podSubnet = 10.10.0.0/16
但 Calico 的 IPPool   = 192.168.0.0/16(默认值,没改)
        ↓
各节点以为自己的 Pod 网段是 10.10.x.x
Calico 却按 192.168.x.x 给 Pod 发 IP
        ↓
跨节点路由表对不上 → 【Pod 之间就是不通】
        ↓
表现:同节点 Pod 能通,跨节点不通
```

所以：**改 Calico 的 `CALICO_IPV4POOL_CIDR` 必须和 `kubeadm.yml` 的 `podSubnet` 一致**。

> ⚠️ 还有个细节：`calico.yaml` 里那两行**默认是被 `#` 注释掉的**，要先取消注释再改，否则改了等于没改。

### 6.3 集群内 DNS

Service 名能被解析，靠的是 **CoreDNS**（`kube-system` 里那两个 coredns Pod）：

```text
同命名空间:  直接写服务名       nginx-svc
跨命名空间:  服务名.命名空间     nginx-svc.qianshan-ns
完整写法:    nginx-svc.qianshan-ns.svc.cluster.local
```

**每个 Pod 的 `/etc/resolv.conf` 都被自动指向 CoreDNS**，所以容器里直接写服务名就能连。

---

## 七、CRI：为什么从 Docker 换成 containerd

### 7.1 CRI 是什么

```text
kubelet 需要"起容器",但它不想绑死在某个运行时上
        ↓
于是定义了一层接口标准:  CRI (Container Runtime Interface)
        ↓
谁实现了这套接口,谁就能被 kubelet 调用
        ↓
containerd 实现了 ✓    CRI-O 实现了 ✓    Docker(直接) ✗
```

类比：**CRI 就像 USB 接口标准** —— 只要你的设备是 USB 的，插上就能用，不关心是哪家的。

### 7.2 "1.24 移除 dockershim"移除的到底是什么

```text
1.24 之前:
   kubelet ──CRI──▶ dockershim(内置适配层) ──Docker API──▶ dockerd ──▶ containerd ──▶ runc
                     ↑ 这层叫 shim(垫片),专门做翻译

1.24 之后:
   kubelet ──CRI──▶ containerd ──▶ runc
                     ↑ 中间少了两层
```

**关键认知：Docker 内部本来就在用 containerd。** K8s 只是**跳过 Docker，直接跟 containerd 说话**了。

所以这不是"换了个软件"，而是**去掉了中间那道翻译环节** —— 更轻、更稳定、少一个故障点。

### 7.3 命令对照：docker ↔ crictl

运行时换了，调试命令也换：

| Docker | crictl | 说明 |
| --- | --- | --- |
| `docker ps` | `crictl ps` | 默认只显示 Running，加 `-a` 看全部 |
| `docker images` | `crictl images` | |
| `docker pull` | `crictl pull` | |
| `docker logs` | `crictl logs` | |
| `docker exec` | `crictl exec` | |
| `docker inspect` | `crictl inspect` | |
| `docker rmi` | `crictl rmi` | |

> ⚠️ **crictl 只是调试工具，不能用来"部署应用"** —— 它起的容器 K8s 不认识。
> 部署必须走 `kubectl` + YAML。这一点和 Docker 时代完全不同。

---

## 八、Namespace

### 8.1 是什么

**逻辑上的资源分组**，用来把一堆资源"圈"在一起，便于管理和隔离。

```text
集群
 ├── default           ← 不指定就放这
 ├── kube-system       ← 系统组件都在这(apiserver / coredns / calico ...)
 ├── kube-public
 ├── kube-node-lease
 └── qianshan-ns       ← 你自己建的
```

### 8.2 两个必须知道的点

| 误区 | 实际情况 |
| --- | --- |
| "Namespace 能隔离网络" | ❌ 只能逻辑分组；**跨命名空间默认就是能通的**，网络隔离要靠 NetworkPolicy |
| "删了 Namespace 里面东西还在" | ❌ **删 Namespace 会把里面所有资源一起删掉** |

```bash
kubectl get ns                       # 看所有命名空间
kubectl get pods -n kube-system      # 看指定命名空间
kubectl get pods -A                  # 看所有命名空间(等价 --all-namespaces)
```

---

## 九、回到《K8s 第二期》笔记：逐句对照

补完骨架层，再回头看那些原来卡住的地方：

| 第二期笔记里的话 | 现在怎么理解 |
| --- | --- |
| 1.10 装 containerd / 1.11 装 kubeadm + kubelet | kubelet 是节点管家，kubeadm 是安装器，containerd 是被 CRI 调用的运行时 —— **三个不同层次的东西** |
| "kubelet 启动后反复重启，是正常的" | 它还没拿到 kubeadm 发的证书和配置，在等 |
| `kubectl get nodes` 显示 `NotReady` | 这个状态**是 kubelet 上报的**；缺 CNI 所以不算就绪 |
| 装完 Calico 变 `Ready` | CNI 就绪，Pod 网络通了，kubelet 上报 `NodeReady=True` |
| `etcd-master-203`、`kube-apiserver-master-203` | 控制平面组件是 **kubelet 拉起来的静态 Pod**，名字带节点名后缀 |
| `coredns` 两个 Pod | 集群 DNS，Service 名能解析靠它 |
| "CIDR 不一致 → Pod 之间就是不通" | 见 6.2，路由表对不上 |
| "Deployment 保证始终有 N 个副本" | 见 2.2 调谐循环 |
| "★ 三处 app 必须一致" | 见第三章 Label / Selector |
| "四个端口不要混" | 见 5.2 |
| "1.24 移除 dockershim" | 见 7.2，移掉的是中间那层翻译垫片 |
| `namespace: qianshan-ns` | 见第八章 |
| `kubectl explain` | 见 4.2，最实用的命令 |
| `ipvsadm -ln` 能看到规则说明是 ipvs 模式 | kube-proxy 写的转发规则 |

---

## 十、自检清单

能不看笔记答出下面这些，骨架层就算通了：

| # | 问题 |
| --- | --- |
| 1 | `kubectl`、`kubelet`、`kubeadm`、`kube-proxy` 分别是什么？装在哪？ |
| 2 | 一次 `kubectl apply` 之后，数据经过了哪些组件？在哪一步才真正起容器？ |
| 3 | "期望状态"存在哪里？谁负责让实际追上期望？ |
| 4 | 为什么 Service 用标签选 Pod，而不是用 Pod 名字？ |
| 5 | 一个 Deployment + Service 里，标签一共出现在几处？不一致会怎样？ |
| 6 | K8s 的 YAML 有哪四段？哪一段是你要写的，哪一段是系统填的？ |
| 7 | `nodePort` / `port` / `targetPort` / `containerPort` 分别在哪一层？ |
| 8 | `ClusterIP` 是从哪个网段分配的？为什么 ping 不通却能访问？ |
| 9 | Pod 的 IP 是谁分配的？为什么 Calico 的 CIDR 必须和 `podSubnet` 一致？ |
| 10 | CRI 是什么？1.24 "移除 dockershim"到底移除了什么？ |
| 11 | 为什么 Pod 挂了会自动重建？ |
| 12 | Namespace 能隔离网络吗？删掉会怎样？ |

### 推荐的动手练习

```bash
# 1. 看一个 Deployment 的完整定义(含系统填的字段)
kubectl create deployment demo --image=nginx:1.24 --replicas=2
kubectl get deploy demo -o yaml

# 2. 故意把 Service 的 selector 改错,观察 endpoints 变 <none>
kubectl expose deployment demo --port=80 --target-port=80 --type=NodePort
kubectl get endpoints demo
kubectl edit svc demo        # 把 selector 的 app 改成错的
kubectl get endpoints demo   # 变成 <none> → 这就是标签对不上的表现

# 3. 删掉一个 Pod,看它怎么被自动补回来
kubectl get pods
kubectl delete pod 某个Pod名
kubectl get pods -w          # 盯着看,新 Pod 会自己冒出来

# 4. 用 explain 查字段,而不是背
kubectl explain deploy.spec.strategy --recursive
```
