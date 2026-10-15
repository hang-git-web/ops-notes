# 第三期 · Prometheus 监控（课件第三章）

| 项目 | 内容 |
| --- | --- |
| **你在哪** | 第三期 / 共三期 |
| **上一期产出** | PHP + MySQL 双容器工程（本期的 compose 经验直接复用） |
| **本期产出** | 一套能出图的监控栈（4 个容器） |
| **预计耗时** | 3–4 小时 |
| **验收标准** | 4 个容器都 Up；`curl :9100/metrics` 有指标；Targets 三个 job 全 UP；Grafana 导入 1860 面板有曲线 |
| **开工前** | 先跑一遍 `00-三期总览.md` 里的「开工前检查」；`ss -lntp` 确认 8080 没被第二期的容器占着 |

---

## 一、四个组件各干什么（课件 3.1）

> **prometheus：采集和存储指标**
> **cadvisor：采集容器指标**
> **node-exporter：采集宿主机指标**
> **grafana：展示图表**

**课件的链路图**：

```text
cadvisor 和 node-exporter ：提供数据
        │
        ▼
prometheus ：存储数据
        │
        ▼
grafana ：展示数据
```

**展开成一张完整的数据流图**：

```text
【数据源】                          【存储】            【展示】
node-exporter (宿主机的 CPU/内存/磁盘) ──┐
                                       ├──► prometheus ──► grafana
cadvisor      (容器的 CPU/内存/网络)   ──┘     (拉取+存储)     (画图)
                                                    ▲
                                          prometheus 自己也算一个
                                          target（监控自己）
```

> **两个关键理解**：
> ① **Prometheus 是"拉"（pull）模型**：它按 `prometheus.yml` 里配的 targets，**定时主动去抓**这些地址的 `/metrics` 接口。所以 exporter 只需要"暴露指标"，不用管推送。
> ② **exporter 就是"翻译器"**：把宿主机的 `/proc`、docker 的统计信息翻译成 Prometheus 能读的格式。

## 二、完整部署（课件 3.2，★★★ 照着敲）

### 步骤 1：建目录

```bash
mkdir -p /opt/monitoring/{prometheus,data/prometheus,data/grafana}
```

### 步骤 2：写 Prometheus 配置

```bash
cat >/opt/monitoring/prometheus/prometheus.yml<<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
EOF
```

> **注意 targets 写的是"服务名:端口"**，不是 IP —— 因为它们在同一个 compose 网络里，靠服务名解析（这就是 1.5 学的）。**如果这里写 IP，容器重建后 IP 变了就抓不到**。

### 步骤 3：写 compose 文件

```bash
cat > /opt/monitoring/docker-compose.yml <<'EOF'
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - /opt/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /opt/monitoring/data/prometheus:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
    extra_hosts:
      - "host.docker.internal:host-gateway"

  cadvisor:
    image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: always
    ports:
      - "8080:8080"
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: always
    ports:
      - "9100:9100"
    pid: host
    volumes:
      - /:/host:ro,rslave
    command:
      - --path.rootfs=/host

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - /opt/monitoring/data/grafana:/var/lib/grafana
EOF
```

**这套配置里几个必须理解的参数**：

| 参数 | 为什么需要 |
| --- | --- |
| `--storage.tsdb.retention.time=15d` | **数据只留 15 天**。不设的话 Prometheus 会一直占磁盘，直到写满 |
| `--web.enable-lifecycle` | 打开这个才能用 `POST /-/reload` **热加载配置**（改了 prometheus.yml 不用重启） |
| **`cadvisor` 的 `privileged: true` + 挂 `/sys`、`/var/lib/docker`** | cAdvisor 要读宿主机的 cgroup 和 docker 数据才能统计容器指标，必须提权 + 挂宿主机路径 |
| **`node-exporter` 的 `pid: host`** | 要看到宿主机的所有进程，不能只在容器自己的 PID 命名空间里 |
| `--path.rootfs=/host` + 挂 `/:/host:ro` | 让 node-exporter 读**宿主机**的 `/proc`（否则读到的是容器自己的，数字就不对了） |
| `extra_hosts: host.docker.internal:host-gateway` | 让容器能用 `host.docker.internal` 访问宿主机 |

> **★ 安全提醒**：`privileged: true` 等于把宿主机的能力给了容器，**cAdvisor 和 node-exporter 只能在内网部署**，端口不要暴露到公网。

### 步骤 4：启动

```bash
cd /opt/monitoring
docker compose up -d
docker compose ps
```

**预期输出**：

```text
NAME            IMAGE                STATUS
cadvisor        .../cadvisor:latest  Up
grafana         grafana/grafana      Up
node-exporter   prom/node-exporter   Up
prometheus      prom/prometheus      Up
```

### 步骤 5：逐个验证（★ 分层验证，不要只看容器起来没）

```bash
# 1) 四个容器都在
docker compose ps

# 2) 端口都在听
ss -lntp | grep -E '9090|8080|9100|3000'

# 3) exporter 自己能出指标（这是最关键的一步）
curl -s http://127.0.0.1:9100/metrics | head -20
# 期望：能看到 node_cpu_seconds_total、node_memory_... 之类的行

curl -s http://127.0.0.1:8080/metrics | grep -m3 container_
# 期望：能看到 container_cpu_... 之类的行

# 4) Prometheus 自己活着
curl -s http://127.0.0.1:9090/-/healthy
# 期望：Prometheus Server is Healthy.
```

**步骤 6：在 Prometheus 网页上确认 target 都是 UP（★ 这一步才算联调成功）**

浏览器访问 `http://<宿主机IP>:9090`，然后：

```text
顶部菜单 Status → Targets
```

**期望看到三个 job 的 State 都是 `UP`**：

```text
prometheus     (1/1 up)     endpoint http://prometheus:9090/metrics
cadvisor       (1/1 up)     endpoint http://cadvisor:8080/metrics
node-exporter  (1/1 up)     endpoint http://node-exporter:9100/metrics
```

> **★ 这一页是 Prometheus 排障的第一站**：
> - `UP` = 能抓到最后一次指标
> - `DOWN` = 抓不到，鼠标移上去能看到报错（常见：服务名写错、端口不对、容器没起）
> - 只要这里不是全 UP，Grafana 里一定是"没数据"的

**顺便在 Prometheus 网页上手写一个查询**（Graph 页）：

```promql
# 宿主机 CPU 使用率
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 宿主机可用内存（GB）
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024

# 运行中的容器数
count(container_last_seen{name!=""})
```

### 步骤 7：Grafana 接上 Prometheus 并出图

```text
1) 浏览器访问 http://<宿主机IP>:3000
2) 默认账号密码都是 admin / admin，首次登录会要求改密码
3) 左侧菜单 → Connections（老版本叫 Configuration）→ Data sources → Add data source
4) 选 Prometheus
5) URL 填 http://prometheus:9090     ← ★ 用服务名，不要用 localhost
6) 拉到底点 Save & test，期望提示 "Data source is working"
7) 左侧菜单 → Dashboards → New → Import
8) 填 dashboard ID，常用的：
     1860  → Node Exporter Full（宿主机总览，最经典）
     193   → Docker monitoring（容器总览）
   选好数据源点 Import
9) 回到 Dashboard 就能看到 CPU / 内存 / 磁盘 / 网络 的曲线
```

> **★ Grafana 里 URL 为什么用 `prometheus:9090`**：因为 Grafana 和 Prometheus 都在同一个 compose 网络里，Grafana 容器里访问 `localhost:9090` 是访问**它自己**，当然连不上。**这是初学者最常犯的错。**

### 步骤 8：日常运维

```bash
cd /opt/monitoring

docker compose ps
docker compose logs -f --tail 50 prometheus

# 改了 prometheus.yml 之后热加载（因为开了 --web.enable-lifecycle）
curl -X POST http://127.0.0.1:9090/-/reload
# 或者
docker compose restart prometheus

# 停止
docker compose down
```

## 三、常见坑（课件 3.3）

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| Targets 里某个 job 是 DOWN | 服务名/端口写错，或容器没起 | `docker compose ps` + `docker compose logs 服务名` |
| Grafana 报 "Data source is working" 失败 | URL 用了 localhost | 改成 `http://prometheus:9090` |
| node-exporter 的数字和宿主机对不上 | 没设 `pid: host` 或没挂 `/:/host` | 按 3.2 的配置补齐 |
| cadvisor 起不来 / 看不到容器指标 | 缺 `privileged: true` 或没挂 `/sys`、`/var/lib/docker` | 补齐挂载和权限 |
| 磁盘被 Prometheus 写满 | 没设 retention | 加 `--storage.tsdb.retention.time=15d` |
| 改了 prometheus.yml 不生效 | 没 reload | `curl -X POST http://127.0.0.1:9090/-/reload` |
| Grafana 重启后要重新配数据源 | 没挂 `/var/lib/grafana` | 加卷（本配置已挂） |

---

---

## 本期自检清单（原第五章摘出）

| 检查项 | 命令 | 通过标准 |
| --- | --- | --- |
| Prometheus 四件套起来 | `docker compose ps` | 4 个都 Up |
| exporter 有指标 | `curl :9100/metrics` | 有输出 |
| Targets 全 UP | Prometheus → Status → Targets | 3 个 job 都 UP |
| Grafana 出图 | 导入 1860 面板 | 有曲线 |

## 本期自测题（闭卷）

- [ ] Prometheus / cAdvisor / node-exporter / Grafana 各干什么？
- [ ] Prometheus 的 Targets 页面是干什么的？DOWN 了怎么查？
- [ ] Grafana 配数据源为什么不能用 `localhost:9090`？
