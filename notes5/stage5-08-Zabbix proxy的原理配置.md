# stage5-08 Zabbix proxy 的原理与配置

> 前置:第 03~07 节已完成,单机 server + agent 架构跑通。
> 本节需要第三台 Ubuntu 22.04 机器作为 proxy。

## 一、知识点

### 1. 为什么需要 proxy

当被监控主机分布在多个机房、网络隔离区或数量上千台时,让所有 agent 直连 server 会有三个问题:跨机房带宽浪费、server 压力过大、隔离网络无法直连。

Proxy 的作用就是**就近采集、集中上报**:

```text
无 proxy:
  agent(机房A) ─┐
  agent(机房B) ─┼─→ server(跨网络)
  agent(机房C) ─┘

有 proxy:
  agent(机房A) → proxyA ─┐
  agent(机房B) → proxyB ─┼─→ server
  agent(机房C) → proxyC ─┘
```

### 2. Proxy 的能力

| 能力 | 说明 |
| --- | --- |
| 数据收集 | 代替 server 采集区域内的主机数据 |
| 数据预处理 | 在 proxy 侧完成部分处理,减少上报量 |
| 断点缓存 | 与 server 断连时数据落地本地库,恢复后补传 |
| 配置缓存 | 从 server 拉取监控配置并本地保存 |

### 3. 两种工作模式(注意:这是 proxy 的模式,不是 agent 的)

| 模式 | 谁主动 | 端口要求 | 适用场景 |
| --- | --- | --- | --- |
| **主动(Active)** | proxy 主动连 server 的 10051 | proxy 侧无需开放入向端口 | 推荐,跨防火墙友好 |
| 被动(Passive) | server 主动连 proxy 的 10051 | proxy 要对外开放 10051 | 网络策略允许时可用 |

### 4. 三个关键约束

1. **Proxy 的 Hostname 必须与 Web 端注册的名字完全一致**,否则一直显示 "Not seen"
2. **Proxy 版本不能高于 server 版本**,两者主版本保持一致
3. 主机要从"由 server 监控"改成"由 proxy 监控",agent 的上报地址也要相应调整

## 二、实操

### 2.1 准备第三台机器并安装 proxy

**操作(在 192.168.171.138 上)**

```bash
sudo hostnamectl set-hostname zabbix-proxy
exec bash
sudo timedatectl set-timezone Asia/Shanghai

wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu22.04_all.deb
sudo apt update

# 使用 SQLite3 作为 proxy 的本地库,轻量、免安装数据库
sudo apt install -y zabbix-proxy-sqlite3 zabbix-sql-scripts sqlite3
```

**检验**

```bash
dpkg -l | grep zabbix-proxy
ls -l /etc/zabbix/zabbix_proxy.conf
```

### 2.2 初始化 proxy 的本地数据库

**操作**

```bash
sudo mkdir -p /var/lib/zabbix
zcat /usr/share/zabbix-sql-scripts/sqlite3/proxy.sql | sudo tee /tmp/proxy.sql > /dev/null
sudo sqlite3 /var/lib/zabbix/zabbix_proxy.db < /tmp/proxy.sql
sudo chown -R zabbix:zabbix /var/lib/zabbix
```

**检验**

```bash
ls -lh /var/lib/zabbix/zabbix_proxy.db
sudo sqlite3 /var/lib/zabbix/zabbix_proxy.db ".tables" | head
```

期望:数据库文件存在且大小约几十 KB,`.tables` 能列出 `hosts`、`items` 等表。

### 2.3 配置 proxy

**操作**

```bash
sudo vi /etc/zabbix/zabbix_proxy.conf
```

关键项改成:

```ini
Server=192.168.171.136
Hostname=zabbix-proxy-01
DBName=/var/lib/zabbix/zabbix_proxy.db
ProxyMode=0
```

说明:`ProxyMode=0` 是主动模式(推荐),`1` 是被动模式。

**检验**

```bash
grep -E '^(Server|Hostname|DBName|ProxyMode)=' /etc/zabbix/zabbix_proxy.conf
```

期望输出四行,`Hostname` 为 `zabbix-proxy-01`(这个名字后面 Web 端要一模一样地填)。

### 2.4 启动 proxy 并观察日志

**操作**

```bash
sudo systemctl restart zabbix-proxy
sudo systemctl enable zabbix-proxy
```

**检验**

```bash
systemctl is-active zabbix-proxy
sudo tail -30 /var/log/zabbix/zabbix_proxy.log
```

期望:服务 active;日志里能看到 `proxy #0 started`、`proxy is running`,主动模式下还会出现与 server 通信的记录。

### 2.5 在 Web 端注册 proxy

**操作**

```text
Administration → Proxies → Create proxy

Proxy name:  zabbix-proxy-01        ← 必须与配置文件里的 Hostname 完全一致
Proxy mode:  Active
→ Add
```

**检验**

```text
Administration → Proxies
```

| 检查项 | 期望结果 |
| --- | --- |
| Name | zabbix-proxy-01 |
| Mode | Active |
| State | 绿色,Last seen 在几十秒内更新 |
| Hosts | 之后接入主机后会显示数量 |

> 如果一直是灰色的 "Not seen":先确认名字一模一样,再确认 proxy 能访问 server 的 10051(`nc -vz 192.168.171.136 10051`)。

### 2.6 把被监控主机改为由 proxy 监控

**操作 1(agent 端)**

```bash
# 允许 proxy 来拉数据(被动采集)
sudo sed -i 's/^Server=.*/Server=192.168.171.138/' /etc/zabbix/zabbix_agentd.conf

# 主动上报给 proxy
sudo sed -i 's/^ServerActive=.*/ServerActive=192.168.171.138/' /etc/zabbix/zabbix_agentd.conf

sudo systemctl restart zabbix-agent
```

**操作 2(Web 端)**

```text
Data collection → Hosts → zabbix-agent → Monitored by proxy → 选择 zabbix-proxy-01 → Update
```

**检验**

```bash
# agent 端
grep -E '^(Server|ServerActive)=' /etc/zabbix/zabbix_agentd.conf
systemctl is-active zabbix-agent
```

```text
# Web 端
1. Administration → Proxies → 该 proxy 的 Hosts 数量应为 1
2. Data collection → Hosts → zabbix-agent → 图标仍为绿色
3. Monitoring → Latest data → 数据仍在持续更新
```

### 2.7 验证 proxy 的断点缓存(选做)

**操作**

```bash
sudo systemctl stop zabbix-proxy
# 等待 3~5 分钟,期间 agent 依旧在采集
sudo systemctl start zabbix-proxy
```

**检验**

```text
Administration → Proxies → 该 proxy 会短暂变为不可用,恢复后重新变绿
Monitoring → Latest data → 恢复后历史数据出现缺口并很快补齐
```

这验证了 proxy 的本地缓存能力:server 短暂不可达时数据不会丢。

## 三、闭环自检清单

| 检查项 | 命令或位置 | 通过标准 |
| --- | --- | --- |
| proxy 包装好 | `dpkg -l \| grep zabbix-proxy` | 有输出 |
| 本地库就绪 | `sqlite3 ... ".tables"` | 能列出表 |
| 配置正确 | `grep -E '^(Server\|Hostname\|DBName)' ...` | 四项齐全,HOSTNAME 与 Web 一致 |
| proxy 服务运行 | `systemctl is-active zabbix-proxy` | active |
| 日志正常 | `tail /var/log/zabbix/zabbix_proxy.log` | 无报错,有 started |
| Web 注册成功 | Administration → Proxies | 绿色,Last seen 更新 |
| 主机已归属 proxy | Host 的 Monitored by proxy | 显示 zabbix-proxy-01 |
| 数据未中断 | Latest data | 持续有值 |
| (选做)缓存生效 | 停启 proxy | 恢复后数据补齐 |

## 四、常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| proxy 一直 Not seen | Hostname 与 Web 端名字不一致 | 两边改成完全相同的字符串 |
| proxy 启动失败 | SQLite 库文件权限或路径错 | 确认属主为 zabbix、DBName 路径正确 |
| proxy 版本报错 | proxy 版本比 server 新 | 换成与 server 相同的主版本 |
| agent 数据不再更新 | agent 仍指向旧地址,或主机没改归属 | 检查 `ServerActive`,检查 Monitored by proxy |
| 被动模式下 server 连不上 proxy | 10051 未放行 / 地址填错 | 放行端口并在 Web 端填写正确地址 |
| proxy 日志报数据库锁 | SQLite 并发限制 | 规模大时改用 `zabbix-proxy-mysql` + MySQL |