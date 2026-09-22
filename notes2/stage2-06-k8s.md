
Kubernetes核心功能
自动部署: 写好配置文件，它就能部署容器
负载均衡: 同一个服务多个副本，它帮你平均访问
自动修复: 某个 Pod 挂了，它会重建
弹性扩缩容: 想跑3个变5个，很轻松
滚动升级: 更新服务时一个一个替换，不停机
命名空间隔离: 各项目之间互不影响


K8s 和 Docker 的关系
Kubernetes 并不替代 Docker，而是站在 Docker 上面
可以理解成:Docker = 打包和运行容器的工具,Kubernetes = 管理多个容器和多个机器的系统
Docker 解决一个应用的问题；K8s 解决一个系统、多个服务、多个机器之间的协作问题


k8s常见的几个核心组件
组件					类比 				干什么的
Pod				   	一张工位				容器实际工作的地方
Node				分部/分厂			一台服务器
Deployment			招聘计划				定义“我要 3 个 nginx 工人”
Service				电话号				访问 Pod 的统一入口（负载均衡）
kube-apiserver 		前台接待				所有指令都通过它进来
etcd				大脑 				集群当前状态的数据库
kube-scheduler		安排工位				决定哪个 Pod 放在哪台机器
kubelet				执行员				听命令、启动容器、上报情况
Namespace			项目隔离区			各项目互不打扰



0、新虚拟机的配置建议
项目	建议值	原因
内存	4GB(最低 2GB)	kind 节点里要跑 etcd、api-server、kubelet
CPU	2 核	dockerd + kubelet 同时工作
磁盘	40GB 以上	Docker 镜像 + kind 节点镜像(约 800MB)+ nginx 镜像
网络	NAT	要能访问外网拉镜像
系统	CentOS 7 Minimal	和你前面的笔记保持一致

一. 部署集群
1. 安装docker
   sudo yum install docker -y
   sudo systemctl start docker
   sudo systemctl enable docker

2. 安装 kubectl
   curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   需要先下载，然后传到服务器
   https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl

chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client

3. 安装 kind
   curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
   需要先下载，然后传到服务器
   mv kind-linux-amd64  kind
   chmod +x ./kind
   sudo mv ./kind /usr/local/bin/kind
   kind version	#查看版本


二. 启动一个 K8s 集群
kind create cluster

三. 查看集群状态
kubectl get nodes

kubectl get pods -A


四. 部署nginx应用
docker 拉取镜像
docker pull nginx:1.24
将镜像上传到kind节点
kind load docker-image nginx:1.24


vim   nginx-deploy.yaml（部署 + 暴露端口）
apiVersion: apps/v1           # 使用 apps/v1 版本的 API 来管理 Deployment
kind: Deployment              # 资源类型是 Deployment，负责管理 Pod 的副本和生命周期
metadata:
name: my-nginx              # Deployment 名称
labels:
app: nginx                # 给 Deployment 打标签，方便选择和管理
spec:
replicas: 1                 # 创建 1 个 Pod 副本
selector:
matchLabels:
app: nginx              # 选择标签为 app=nginx 的 Pod，Deployment 管理这些 Pod
template:
metadata:
labels:
app: nginx            # Pod 模板的标签，必须和 selector 一致
spec:
containers:
- name: nginx           # 容器名称
image: nginx:1.24     # 使用官方1.24版本的 nginx 镜像
ports:
- containerPort: 80   # 容器开放 80 端口

---

apiVersion: v1                # 使用 v1 版本的 API 来管理 Service
kind: Service                 # 资源类型是 Service，负责服务发现和负载均衡
metadata:
name: my-nginx              # Service 名称
spec:
selector:
app: nginx                # 选择标签为 app=nginx 的 Pod，转发请求给这些 Pod
type: NodePort              # 类型为 NodePort，暴露节点端口供外部访问
ports:
- port: 80                 # Service 监听的端口，集群内部使用
  targetPort: 80             # 转发到 Pod 容器的端口
  nodePort: 30080         # 集群节点暴露的端口，外部通过节点IP:30080访问
  # 端口号范围是 30000~32767，可以自行修改

使用命令部署
kubectl apply -f nginx-deploy.yaml

查看 Pod 启动情况
kubectl get pods

查看服务端口号
kubectl get svc

访问curl http://localhost:30080




