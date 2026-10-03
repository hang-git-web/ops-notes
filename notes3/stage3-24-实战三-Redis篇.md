# 实战三 · Redis 篇（主从复制 + 哨兵 + Cluster）

> **本篇整合了**：`stage3-19`（持久化）、`stage3-16`（主从复制）、`stage3-17`（哨兵）、`stage3-18`（Cluster）
> **环境**：三台 Rocky 8.10，Redis **7.2.0**（源码编译）
>
> | 主机 | IP | 角色 |
> | --- | --- | --- |
> | **node2** | 192.168.171.148 | **Redis 主节点**（6379） |
> | **node1** | 192.168.171.147 | 从节点 + 哨兵 + Cluster 节点 |
> | **node3** | 192.168.171.146 | 从节点 + 哨兵 + Cluster 节点 |
>
> **本篇目标**：
> ① 确认 **RDB / AOF 持久化**（防止宕机丢数据）；
> ② 搭起 **node2 → node1/node3 的 Redis 主从复制**；
> ③ 用 **哨兵（Sentinel）** 实现 **自动故障转移**，并亲手演练一次"主节点挂了自动选新主"；
> ④ 用 **Cluster** 做 **3 主 3 从的分片集群**，理解哈希槽和自动重定向。
> **端口规划**（三套可以共存，互不冲突）：
> ```text
> Redis 主从   6379
> 哨兵         26379
> Cluster      7000 / 7001
> ```

---

## 〇、开工前准备

### 0.1 三台环境确认

**三台都跑**：

```bash
echo "我是 $(hostname) ，IP 是 $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')"
# node1 → 192.168.171.147
# node2 → 192.168.171.148
# node3 → 192.168.171.146

# 关 SELinux 和 firewalld（练习环境）
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld

# 三台互相 ping
ping -c1 node1 && ping -c1 node2 && ping -c1 node3
```

### 0.2 三台都装 Redis 7.2.0（源码编译）

> **为什么用源码编译**：Rocky 8 仓库里的 Redis 只有 6.x，而且**服务名/配置路径和笔记对不上**。源码编译三台版本统一，主从/哨兵/Cluster 用同一份配置模板最省心。

```bash
# ① 装编译依赖（★ make 千万别漏，否则 make 那步报 "command not found"）
dnf install -y gcc make tcl wget

# ② 下载并编译
mkdir -p /opt && cd /opt
wget http://download.redis.io/releases/redis-7.2.0.tar.gz
tar -zxvf redis-7.2.0.tar.gz
cd redis-7.2.0
make
make install

# ③ 验证
redis-server --version
redis-cli --version
# 期望：Redis server v=7.2.0 / redis-cli 7.2.0
```

> **如果 `make` 报 `jemalloc` 相关错误**：`dnf install -y gcc make tcl` 先补齐，然后 `make distclean && make` 重来。
> **如果 `make install` 后 `redis-server` 找不到**：确认 `/usr/local/bin` 在 PATH 里，或者用绝对路径 `/usr/local/bin/redis-server`。

### 0.3 目录规划（三台统一）

```bash
mkdir -p /usr/local/redis/{conf,data,log}
mkdir -p /usr/local/redis-cluster/{7000,7001}
```

### 0.4 准备一个统一的密码

**三台统一用同一个密码**，否则 `masterauth` 和 `requirepass` 不一致会导致 `master_link_status:down`：

```text
Redis 密码：Redis@2026
```

---

## 一、前置：Redis 持久化（先确认，别等丢了数据才想起来）

### 1.1 RDB 和 AOF 的区别

| | **RDB** | **AOF** |
| --- | --- | --- |
| 原理 | 定时生成内存**快照**（`.rdb`） | 记录**所有写命令**，重启时重放 |
| 文件 | 小 | 大 |
| 恢复速度 | 快 | 慢 |
| 丢数据风险 | **可能丢最后一次快照后的数据** | 少（`everysec` 最多丢 1 秒） |
| 生产建议 | 两个都开：AOF 保数据、RDB 保快速恢复和备份 | |

> **类比**：RDB 是游戏存档，AOF 是操作录像。

### 1.2 开 AOF 并验证（在 node2 上先做单机验证）

编辑 `/usr/local/redis/conf/redis.conf`，确认这几行：

```ini
daemonize yes
bind 0.0.0.0
port 6379
logfile "/usr/local/redis/log/redis.log"
dir /usr/local/redis/data
requirepass Redis@2026

# RDB
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb

# AOF
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
# 混合持久化（Redis 4.0+，推荐）
aof-use-rdb-preamble yes
```

```bash
redis-server /usr/local/redis/conf/redis.conf
redis-cli -a 'Redis@2026' ping                 # 期望 PONG

# 写数据
redis-cli -a 'Redis@2026' set test_rdb "Hello, RDB!"
redis-cli -a 'Redis@2026' set test_aof "Hello, AOF!"

# 手动触发 RDB 快照
redis-cli -a 'Redis@2026' bgsave
ls -lh /usr/local/redis/data/dump.rdb

# 看 AOF
tail -5 /usr/local/redis/data/appendonly.aof
```

**★ 模拟宕机，验证数据不丢**：

```bash
pkill -9 redis-server            # 强杀（模拟断电）
redis-server /usr/local/redis/conf/redis.conf
redis-cli -a 'Redis@2026' get test_rdb
redis-cli -a 'Redis@2026' get test_aof
# 两个都应该能取到值
```

**持久化文件损坏了怎么修**：

```bash
redis-check-rdb /usr/local/redis/data/dump.rdb
redis-check-aof --fix /usr/local/redis/data/appendonly.aof
```

> **验证完先停掉 node2 的 Redis**，接下来要用统一配置重新起：
> ```bash
> redis-cli -a 'Redis@2026' shutdown nosave 2>/dev/null
> ```

---

## 二、Redis 主从复制（node2 主，node1/node3 从）

### 2.1 原理速览

```text
主节点 node2 ──写命令──▶ 从节点 node1
            ──写命令──▶ 从节点 node3

① 异步、单向：数据只从主流向从，写只能在主节点做，从节点默认只读
② 首次建立（或断线太久）→ 全量同步：主节点 fork 出子进程生成 RDB 推给从节点
③ 之后持续 → 增量同步：主节点把每个写命令实时发给从节点
④ 主从复制本身【不含自动故障切换】—— 那要靠哨兵或 Cluster
```

| 概念 | 含义 |
| --- | --- |
| `run_id` | 每个实例启动时的 40 位随机串，标识它"这一次生命期" |
| `offset` | 复制偏移量，主从双方各记一份，**一致就说明数据一致** |
| `backlog` | 主节点的复制积压缓冲区（`repl-backlog-size`，默认 1MB），支持断线后只补差量 |

### 2.2 配置主节点（node2）

```bash
vim /usr/local/redis/conf/redis.conf
```

```ini
bind 0.0.0.0
protected-mode no
port 6379
daemonize yes
logfile "/usr/local/redis/log/redis.log"
dir /usr/local/redis/data
requirepass Redis@2026

# 持久化
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
aof-use-rdb-preamble yes
```

```bash
redis-server /usr/local/redis/conf/redis.conf
redis-cli -a 'Redis@2026' ping
redis-cli -a 'Redis@2026' INFO replication | head -5
# 期望：role:master
```

### 2.3 配置从节点（node1 和 node3）

**两份配置基本一样，只有 `replicaof` 指向同一个主节点**：

```bash
vim /usr/local/redis/conf/redis.conf
```

```ini
bind 0.0.0.0
protected-mode no
port 6379
daemonize yes
logfile "/usr/local/redis/log/redis.log"
dir /usr/local/redis/data

# ★ 从节点要写这两行
replicaof 192.168.171.148 6379
masterauth Redis@2026

# 从节点自己也有密码（客户端连它时要验），和主节点保持一致最省事
requirepass Redis@2026

# 持久化（从节点也建议开，重启能快速恢复）
appendonly yes
appendfsync everysec
```

> **★ 两个密码别搞混**：
> - **`masterauth`** = 从节点去连**主节点**时用的密码（要和主节点的 `requirepass` 一致）
> - **`requirepass`** = 别人连**从节点**时要验的密码
> - 生产上这两个可以不同，但练习环境**统一用 `Redis@2026` 最省心**。

```bash
# node1 和 node3 都执行
redis-server /usr/local/redis/conf/redis.conf
```

### 2.4 ★ 验证主从

**从节点上看角色和链路**：

```bash
# node1 和 node3 都跑
redis-cli -a 'Redis@2026' INFO replication | head -8
```

期望看到：

```text
role:slave
master_host:192.168.171.148
master_port:6379
master_link_status:up          ← ★ 必须是 up
master_last_io_seconds_ago:0   ← 越小越实时
slave_read_only:1
```

**主节点上看从节点数量**：

```bash
redis-cli -a 'Redis@2026' INFO replication | head -8
# 期望：role:master
#       connected_slaves:2
#       slave0:ip=192.168.171.147,...
#       slave1:ip=192.168.171.146,...
```

### 2.5 ★ 端到端验证（主写从读）

```bash
# 在主节点 node2 上写
redis-cli -h 192.168.171.148 -a 'Redis@2026' SET mykey "Hello, Redis!"
# 期望 OK

# 在两个从节点上读
redis-cli -h 192.168.171.147 -a 'Redis@2026' GET mykey     # 期望 "Hello, Redis!"
redis-cli -h 192.168.171.146 -a 'Redis@2026' GET mykey     # 期望 "Hello, Redis!"

# 从节点试着写（应该被拒绝）
redis-cli -h 192.168.171.147 -a 'Redis@2026' SET x 1
# 期望报错：(error) READONLY You can't write against a read only replica.
```

> **★ 到这里三步都对了，Redis 主从就算跑通**：角色对、`master_link_status:up`、主写从能读、从不能写。

### 2.6 手工主从切换（先练手工，再上哨兵）

```bash
# 在一个从节点（比如 node1）上，让它脱离主从、变成独立主节点
redis-cli -h 192.168.171.147 -a 'Redis@2026' REPLICAOF NO ONE
redis-cli -h 192.168.171.147 -a 'Redis@2026' INFO replication | head -3
# 期望：role:master

# 让另一个从节点改指向它
redis-cli -h 192.168.171.146 -a 'Redis@2026' REPLICAOF 192.168.171.147 6379

# 恢复原状（切回来）
redis-cli -h 192.168.171.147 -a 'Redis@2026' REPLICAOF 192.168.171.148 6379
redis-cli -h 192.168.171.146 -a 'Redis@2026' REPLICAOF 192.168.171.148 6379
```

> **旧命令 `SLAVEOF`** 在 Redis 5 之后叫 `REPLICAOF`（两个都能用，但新名字更规范）。
> **手工切换的问题**：靠人手敲，主节点半夜挂了没人管 —— 所以才需要**哨兵来自动做这件事**。

---

## 三、哨兵模式（三台都部署哨兵）

### 3.1 哨兵是干什么的

```text
哨兵（Sentinel）= 一个独立的进程，专门盯着 Redis 主从，干三件事：
  ① 监控：定期 ping 主节点和从节点
  ② 判断：主节点超过 down-after-milliseconds 没响应 → 判定"主观下线"
          多个哨兵都认为它下线 → 判定"客观下线"
  ③ 故障转移：从从节点里选一个当新主，并通知其它从节点改指向它

★ 为什么要 3 个哨兵？因为要靠【多数派投票】决定"主节点是不是真挂了"，
  奇数个节点才能避免平票。所以哨兵最少 3 个。
```

**拓扑**：

```text
        node1(.147)        node2(.148)        node3(.146)
        ┌─────────┐        ┌─────────┐        ┌─────────┐
        │ redis   │        │ redis   │        │ redis   │
        │ :6379   │◀──写───│ :6379   │───写──▶│ :6379   │
        │ (从)    │        │ (主)    │        │ (从)    │
        ├─────────┤        ├─────────┤        ├─────────┤
        │ sentinel│        │ sentinel│        │ sentinel│
        │ :26379  │        │ :26379  │        │ :26379  │
        └─────────┘        └─────────┘        └─────────┘
             三台哨兵互相通信，共同决定主节点是否下线
```

### 3.2 写哨兵配置（三台都一样）

```bash
vim /usr/local/redis/conf/sentinel.conf
```

```ini
port 26379
daemonize yes
logfile "/usr/local/redis/log/sentinel.log"
dir /usr/local/redis/data

# ★ 监控名为 mymaster 的主节点（IP 用 node2）
sentinel monitor mymaster 192.168.171.148 6379 2

# ★ 主节点有密码，哨兵必须知道，否则连不上
sentinel auth-pass mymaster Redis@2026

# 主节点多久没响应就判定主观下线（毫秒）
sentinel down-after-milliseconds mymaster 5000

# 故障转移超时
sentinel failover-timeout mymaster 10000

# 故障转移时，最多多少个从节点同时向新主同步
sentinel parallel-syncs mymaster 1
```

**参数逐个解释**：

| 参数 | 含义 |
| --- | --- |
| `sentinel monitor mymaster <IP> <端口> <quorum>` | 监控主节点。`mymaster` 是别名（自定义）；**`2` 是 quorum** —— 至少 2 个哨兵认为主节点下线，才判定客观下线 |
| `sentinel auth-pass mymaster <密码>` | 主节点有密码时**必须配**，否则哨兵连不上、无法监控 |
| `down-after-milliseconds` | 主观下线判定时间（5 秒） |
| `failover-timeout` | 故障转移超时（10 秒） |
| `parallel-syncs` | 转移时同时向新主同步的从节点数量，`1` 最稳（不会同时全量同步打爆新主） |

> **★ quorum 的选择**：3 个哨兵时 **quorum = 2**（多数派）。写成 1 的话，任意一个哨兵觉得主节点挂了就会发起选举，容易误判。

### 3.3 启动哨兵（三台都执行）

```bash
# 先确认 redis 主从是正常的
redis-cli -h 192.168.171.148 -a 'Redis@2026' INFO replication | head -3

# 三台都起哨兵
redis-sentinel /usr/local/redis/conf/sentinel.conf

# 验证进程和端口
ss -lntp | grep 26379
ps -ef | grep sentinel | grep -v grep
```

### 3.4 验证哨兵

```bash
# 三台都跑，看哨兵有没有认出主节点和两个从节点
redis-cli -p 26379 info sentinel
```

期望输出：

```text
sentinel_masters:1
master0:name=mymaster,status=ok,address=192.168.171.148:6379,slaves=2,sentinels=3
```

> **★ 重点看三个数字**：`status=ok`、**`slaves=2`**、**`sentinels=3`**。三个都对，说明哨兵集群认全了。

**其它常用查询**：

```bash
# 看所有被监控的主节点
redis-cli -p 26379 sentinel masters

# 只看主节点地址
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
# 期望：192.168.171.148 / 6379

# 看某个主节点的所有从节点
redis-cli -p 26379 sentinel slaves mymaster

# 看所有哨兵
redis-cli -p 26379 sentinel sentinels mymaster
```

### 3.5 ★ 故障转移演练（本篇最值钱的一步）

**第 1 步：记下当前主节点**

```bash
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
# 期望：192.168.171.148（node2）
```

**第 2 步：在另一台机器上开一个窗口盯日志**

```bash
# 比如在 node1 上
tail -f /usr/local/redis/log/sentinel.log
```

**第 3 步：干掉主节点**

```bash
# 在 node2 上执行
redis-cli -h 192.168.171.148 -a 'Redis@2026' shutdown
```

**第 4 步：看哨兵日志发生了什么（★ 关键观察）**

```text
+sdown master mymaster 192.168.171.148 6379      ← 主观下线
+odown master mymaster 192.168.171.148 6379      ← 客观下线（够 quorum 了）
+elected-leader master mymaster ...              ← 选出领头哨兵
+failover-state-select-slave master ...          ← 开始挑新主
+selected-slave slave 192.168.171.147:6379 ...   ← 选中了 node1
+failover-state-send-slaveof-noone ...           ← 让它脱离主从
+switch-master mymaster 192.168.171.148 6379 192.168.171.147 6379   ← ★ 切换完成
```

**第 5 步：验证新主节点**

```bash
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
# ★ 期望：IP 变了！变成 192.168.171.147（或 146）

# 新主节点现在能写
redis-cli -h 192.168.171.147 -a 'Redis@2026' INFO replication | head -3
# 期望：role:master
redis-cli -h 192.168.171.147 -a 'Redis@2026' SET after_failover "yes"

# 另一个从节点已经自动指向新主（哨兵帮它改了）
redis-cli -h 192.168.171.146 -a 'Redis@2026' INFO replication | head -4
# 期望：role:slave，master_host 是新主的 IP

# 数据还在吗
redis-cli -h 192.168.171.147 -a 'Redis@2026' GET mykey
# 期望：Hello, Redis!
```

**第 6 步：把原主节点恢复，看它自动变成从节点**

```bash
# 在 node2 上重新启动 Redis（哨兵会在配置里记下新主，它启动后会被告知"你是从了"）
redis-server /usr/local/redis/conf/redis.conf
sleep 5
redis-cli -h 192.168.171.148 -a 'Redis@2026' INFO replication | head -4
# ★ 期望：role:slave，master_host 指向新的主节点
```

> **★ 这一步最能体现哨兵的价值**：原主节点恢复后，**不用人工干预**，哨兵会自动把它配置成新主的从节点。

> ⚠️ **一个重要细节**：哨兵会**改写 `sentinel.conf` 文件**（把新的主节点地址写进去，并加一行 `sentinel known-replica`）。所以**别手动去编辑正在运行的 sentinel.conf**，会让哨兵混乱。

**第 7 步：恢复原状（做完演练记得回到初始状态）**

```bash
# 把 node2 重新变成主节点：先用哨兵做一次强制故障转移
redis-cli -p 26379 sentinel failover mymaster
sleep 8
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
# 反复执行 failover 直到主节点回到 192.168.171.148，或者直接重启三个哨兵和三个 redis
```

> 嫌麻烦就**三台全部重启一遍**：先起 redis（按主从配置），再起 sentinel，回到初始拓扑。

---

## 四、Redis Cluster（三台 × 2 实例 = 3 主 3 从）

### 4.1 Cluster 是什么（三个概念）

```text
① 分布式：数据分散存储在多个节点，能横向扩容
② 哈希槽（slot）：一共 16384 个槽，每个主节点负责其中一部分。
   存 key 时按 CRC16(key) mod 16384 算出槽号，就知道该去哪台机器
③ Gossip 协议：节点之间用 PING/PONG 交换状态，实现自发现和故障检测
```

**和主从 + 哨兵的区别**：

| | 主从 + 哨兵 | Cluster |
| --- | --- | --- |
| 能存多少 | 受单机内存限制 | **能横向扩容（分片）** |
| 故障切换 | 哨兵负责 | **Cluster 自带** |
| 客户端 | 普通客户端 | **必须支持 cluster 协议**（`redis-cli` 要加 `-c`） |
| 适合 | 数据量不大、要高可用 | 数据量大、要高并发 |

### 4.2 集群规划

```text
         node1(.147)         node2(.148)         node3(.146)
        ┌───────────┐       ┌───────────┐       ┌───────────┐
        │ 7000      │       │ 7000      │       │ 7000      │
        │ 7001      │       │ 7001      │       │ 7001      │
        └───────────┘       └───────────┘       └───────────┘

  6 个实例 → cluster create 会分成 3 主 3 从（--cluster-replicas 1）
  16384 个槽会平均分给 3 个主节点
```

> ⚠️ **做 Cluster 前先把前面的 Redis 和哨兵停掉**（端口不冲突，但配置容易混）：
> ```bash
> redis-cli -a 'Redis@2026' shutdown nosave 2>/dev/null
> redis-cli -p 26379 shutdown 2>/dev/null
> pkill -f redis-sentinel
> ps -ef | grep -E 'redis' | grep -v grep      # 确认干净了
> ```

### 4.3 写 6 份配置（三台都要做）

**以 7000 为例**：

```bash
cp /opt/redis-7.2.0/redis.conf /usr/local/redis-cluster/7000/redis.conf
vim /usr/local/redis-cluster/7000/redis.conf
```

```ini
port 7000
bind 0.0.0.0
protected-mode no
daemonize yes
pidfile /var/run/redis_7000.pid
logfile "/usr/local/redis-cluster/7000/redis.log"
dir /usr/local/redis-cluster/7000/

# ★ Cluster 必须开的四行
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 15000
cluster-require-full-coverage no

# 密码（集群里也要一致）
requirepass Redis@2026
masterauth Redis@2026

# ★ 如果一台机器有多块网卡（NAT + 仅主机），需要显式告诉集群"对外用哪个 IP"
# cluster-announce-ip 192.168.171.147     # 换成本机 IP

# 持久化
appendonly yes
appendfsync everysec
```

**7001 的配置**：把上面所有 `7000` 替换成 `7001` 即可：

```bash
sed 's/7000/7001/g' /usr/local/redis-cluster/7000/redis.conf > /usr/local/redis-cluster/7001/redis.conf
grep -E '^(port|pidfile|cluster-config-file)' /usr/local/redis-cluster/7001/redis.conf
# 期望：port 7001 / pidfile /var/run/redis_7001.pid / nodes-7001.conf
```

> **用 `sed` 生成第二份配置**比手敲靠谱 —— 手敲最容易漏改 `cluster-config-file`，那样两个实例会抢同一个节点配置文件，集群起不来。

### 4.4 启动 6 个实例（三台都执行）

```bash
redis-server /usr/local/redis-cluster/7000/redis.conf
redis-server /usr/local/redis-cluster/7001/redis.conf

# 验证
ss -lntp | grep -E '7000|7001'
ps -ef | grep redis-server | grep -v grep
```

期望每台看到 2 个端口、2 个进程。

> **注意端口是 17000/17001 也在监听**：Cluster 模式下每个实例会额外开一个"集群总线端口"（业务端口 +10000），这是正常的。

### 4.5 创建集群（★ 只需要在一台上执行一次）

```bash
redis-cli -a 'Redis@2026' --cluster create \
  192.168.171.147:7000 192.168.171.147:7001 \
  192.168.171.148:7000 192.168.171.148:7001 \
  192.168.171.146:7000 192.168.171.146:7001 \
  --cluster-replicas 1
```

会有一次交互确认：

```text
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
...
Can I set the above configuration? (type 'yes' to accept): yes      ← 输入 yes
```

期望结尾：

```text
[OK] All 16384 slots covered.
```

> **★ `[OK] All 16384 slots covered` 是集群创建成功的标志** —— 16384 个槽全部有主，一个都不能少。

### 4.6 ★ 验证集群

```bash
# ① 集群整体状态
redis-cli -a 'Redis@2026' -h 192.168.171.147 -p 7000 cluster info | head -8
```

期望：

```text
cluster_state:ok                    ← 必须是 ok
cluster_slots_assigned:16384        ← 必须是 16384
cluster_slots_ok:16384
cluster_known_nodes:6               ← 6 个节点
cluster_size:3                      ← 3 个主节点
```

```bash
# ② 看每个节点的角色和槽分配
redis-cli -a 'Redis@2026' -h 192.168.171.147 -p 7000 cluster nodes
```

期望看到 6 行：**3 个 `master`、3 个 `slave`**，每个 master 后面跟着一段槽范围（如 `0-5460`）。

```bash
# ③ ★ 写入测试（注意 -c 参数！）
redis-cli -a 'Redis@2026' -c -h 192.168.171.147 -p 7000 set user:1 "Alice"
redis-cli -a 'Redis@2026' -c -h 192.168.171.147 -p 7000 get user:1
# 期望：OK / "Alice"
```

> **★ `-c` 参数是关键**：它表示"自动跟随重定向"。**不加 `-c` 会报 `(error) MOVED 866 ...`** —— 因为 `user:1` 算出来的槽不在你连的这个节点上，集群会告诉你"去另一个节点"。加了 `-c` 客户端会自动跳过去。

> **不加 `-c` 看到 MOVED 是正常的**，不是错误 —— 这正是集群分片在工作的证据。

### 4.7 ★ 故障转移演练

```bash
# ① 先看清谁是主、谁是它的从
redis-cli -a 'Redis@2026' -h 192.168.171.147 -p 7000 cluster nodes | grep master

# ② 干掉一个主节点（比如 192.168.171.147:7000）
redis-cli -a 'Redis@2026' -h 192.168.171.147 -p 7000 shutdown

# ③ 等 15 秒左右（cluster-node-timeout），看它的从节点有没有顶上来
sleep 20
redis-cli -a 'Redis@2026' -h 192.168.171.148 -p 7000 cluster nodes | grep master
# ★ 期望：原来 147:7000 负责的那段槽，现在由它的从节点接管（那一行的角色变成 master）

# ④ 集群还是可用的吗
redis-cli -a 'Redis@2026' -h 192.168.171.148 -p 7000 cluster info | grep cluster_state
# 期望：cluster_state:ok        ← 挂了一个主，集群依然可用（这就是高可用）

# ⑤ 数据还能读写吗
redis-cli -a 'Redis@2026' -c -h 192.168.171.148 -p 7000 get user:1
# 期望："Alice"
```

**把节点恢复**：

```bash
# 在 node1 上重新起被关掉的实例
redis-server /usr/local/redis-cluster/7000/redis.conf
sleep 10
redis-cli -a 'Redis@2026' -h 192.168.171.147 -p 7000 cluster info | grep cluster_state
# 它会作为从节点重新加入集群
```

### 4.8 集群常用运维命令

```bash
# 看槽分配
redis-cli -a 'Redis@2026' -h 192.168.171.147 -p 7000 cluster slots

# 看某个 key 存在哪个节点
redis-cli -a 'Redis@2026' -h 192.168.171.147 -p 7000 cluster keyslot user:1

# 动态添加新节点（扩容）
redis-cli -a 'Redis@2026' --cluster add-node <新节点IP:PORT> <现有节点IP:PORT>

# 重新分片（迁移槽）
redis-cli -a 'Redis@2026' --cluster reshard <节点IP:PORT>

# 修复集群（节点重启后槽分配乱了）
redis-cli -a 'Redis@2026' --cluster fix <问题节点IP:PORT>

# 集群体检
redis-cli -a 'Redis@2026' --cluster check 192.168.171.147:7000
```

> ⚠️ **Cluster 有个限制**：**跨槽的多 key 操作（如 `MGET k1 k2`）会报 `CROSSSLOT` 错误**，除非用 `{tag}` 把 key 强制映射到同一槽，例如 `user:{1}:name` 和 `user:{1}:age`。这是使用 Cluster 时最需要提前设计的地方。

---

## 五、三套环境的启停与端口对照

### 5.1 端口总表

| 服务 | 端口 | 在哪几台 | 配置/命令 |
| --- | --- | --- | --- |
| Redis 主从 | 6379 | 三台 | `redis-server /usr/local/redis/conf/redis.conf` |
| 哨兵 | 26379 | 三台 | `redis-sentinel /usr/local/redis/conf/sentinel.conf` |
| Cluster | 7000 / 7001 | 三台 | `redis-server /usr/local/redis-cluster/700x/redis.conf` |
| Cluster 总线 | 17000 / 17001 | 三台 | 自动开启，不用配 |

**三套可以同时跑**（端口不冲突），但**做每个模块前建议把上一套停掉**，避免混淆。

### 5.2 优雅停止（★ 别用 kill -9，可能丢数据）

```bash
# 停 Redis 主从（三台）
redis-cli -a 'Redis@2026' shutdown

# 停哨兵（三台）
redis-cli -p 26379 shutdown

# 停 Cluster（三台各 2 个实例）
redis-cli -a 'Redis@2026' -p 7000 shutdown
redis-cli -a 'Redis@2026' -p 7001 shutdown
```

**确认停干净了**：

```bash
ps -ef | grep redis | grep -v grep
ss -lntp | grep -E '6379|26379|7000|7001'
# 都应该没有输出
```

### 5.3 重启后的恢复顺序（VM 重启后要按这个顺序起）

```bash
# ① 先起 Redis（主节点先起，从节点后起）
#    node2（主）: redis-server /usr/local/redis/conf/redis.conf
#    node1/node3（从）: redis-server /usr/local/redis/conf/redis.conf

# ② 再起哨兵（三台）
redis-sentinel /usr/local/redis/conf/sentinel.conf

# ③ 验证主从和哨兵
redis-cli -a 'Redis@2026' INFO replication | head -3
redis-cli -p 26379 info sentinel | head -3
```

> **顺序很重要**：哨兵启动时会去连主节点，如果 Redis 还没起，哨兵日志里会先报错，虽然它会重试连上，但日志会脏。

---

## 六、闭环自检清单

### 6.1 持久化

| # | 检查项 | 命令 | 通过标准 |
| --- | --- | --- | --- |
| 1 | AOF 已开 | `redis-cli -a 'Redis@2026' config get appendonly` | `yes` |
| 2 | RDB 有快照 | `ls -lh /usr/local/redis/data/dump.rdb` | 文件存在 |
| 3 | AOF 有内容 | `tail -3 /usr/local/redis/data/appendonly.aof` | 能看到写命令 |
| 4 | **强杀重启数据还在** | `pkill -9 redis-server` 后重启 | `GET` 能取到值 |
| 5 | 损坏文件能修 | `redis-check-aof --fix ...` | 跑通 |

### 6.2 主从复制

| # | 检查项 | 命令 | 通过标准 |
| --- | --- | --- | --- |
| 6 | 主节点角色 | node2：`INFO replication` | `role:master` |
| 7 | 从节点角色 | node1/node3：`INFO replication` | `role:slave` |
| 8 | **链路状态** | node1/node3：同上 | **`master_link_status:up`** |
| 9 | 主节点看到 2 个从 | node2：同上 | `connected_slaves:2` |
| 10 | **主写从读** | 主 `SET` → 从 `GET` | 值一致 |
| 11 | 从节点只读 | 从节点 `SET` | 报 `READONLY` |
| 12 | 密码配对正确 | 三台能互相 `redis-cli -h <IP> -a ... ping` | 都是 PONG |

### 6.3 哨兵

| # | 检查项 | 命令 | 通过标准 |
| --- | --- | --- | --- |
| 13 | 三个哨兵都起了 | 三台：`ss -lntp \| grep 26379` | 都有 |
| 14 | 哨兵认全了 | `redis-cli -p 26379 info sentinel` | `status=ok`、`slaves=2`、`sentinels=3` |
| 15 | **能查主节点** | `sentinel get-master-addr-by-name mymaster` | 返回 node2 的 IP |
| 16 | **故障转移能自动完成** | 干掉主 → 看日志 | 出现 `+switch-master` |
| 17 | **新主可写** | 切换后 `SET` | OK |
| 18 | 其它从自动跟随新主 | `INFO replication` | `master_host` 是新主 IP |
| 19 | 原主恢复后自动变从 | 重启原主 → `INFO replication` | `role:slave` |

### 6.4 Cluster

| # | 检查项 | 命令 | 通过标准 |
| --- | --- | --- | --- |
| 20 | 6 个实例都起了 | 三台：`ss -lntp \| grep -E '7000\|7001'` | 每台 2 个 |
| 21 | **集群状态** | `cluster info` | **`cluster_state:ok`** |
| 22 | **槽全覆盖** | 同上 | `cluster_slots_assigned:16384` |
| 23 | 6 个节点 | `cluster nodes` | 6 行 |
| 24 | **3 主 3 从** | `cluster nodes \| grep master` | 3 行 master |
| 25 | **能写入** | `redis-cli -c ... set` | OK |
| 26 | 自动重定向生效 | 不加 `-c` 时 | 报 `MOVED`（这是正常的） |
| 27 | **主节点挂了集群仍可用** | `shutdown` 一个主 | `cluster_state:ok`，数据能读 |
| 28 | 从节点顶上来 | `cluster nodes` | 原主的从变成 master |

---

## 七、常见报错速查

### 7.1 编译 / 启动

| 报错 | 原因 | 处理 |
| --- | --- | --- |
| `make: command not found` | 依赖没装全 | `dnf install -y gcc make tcl wget` |
| `jemalloc` 相关编译错误 | 缺依赖或残留 | `make distclean && make` |
| `Could not create server TCP listening socket *:6379: bind: Address already in use` | 端口被占 | `ss -lntp \| grep 6379`，先停掉旧的 |
| `redis-server` 找不到 | `/usr/local/bin` 不在 PATH | 用绝对路径，或加 PATH |

### 7.2 主从

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| **`master_link_status:down`** | ① `masterauth` 密码和主节点 `requirepass` 不一致 ② 主节点没开 `bind 0.0.0.0` ③ 端口不通 | 三台统一密码；主节点 `bind 0.0.0.0`；`redis-cli -h <主IP> -a <密码> ping` 先测连通 |
| `NOAUTH Authentication required` | 忘了 `-a` 或密码错 | 命令加 `-a 'Redis@2026'` |
| `READONLY You can't write against a read only replica` | 在从节点写 | 这是**正常保护**，写到主节点去 |
| 从节点 `role:master` | 配置没生效 / 重启了但没读配置 | 确认 `replicaof` 那行在，重启 `redis-server /完整路径/redis.conf` |

### 7.3 哨兵

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 哨兵日志报 `NOAUTH` / 连不上主节点 | 没配 `sentinel auth-pass` | 加上 `sentinel auth-pass mymaster Redis@2026` |
| `slaves=0` | 哨兵连上了主，但主节点没把从节点信息给它 | 确认主从本身正常（`connected_slaves` 是 2）；等 10 秒再看 |
| 不触发故障转移 | **quorum 设太大，或哨兵数量不够** | 三个哨兵都起；quorum 设 **2** |
| 触发了两遍 / 反复切换 | 原主恢复得太快，或者网络抖动 | 调大 `down-after-milliseconds`；检查网络 |
| `sentinel.conf` 被改得乱七八糟 | 哨兵会自动改写这个文件 | 正常现象，**别手动编辑运行中的 sentinel.conf** |

### 7.4 Cluster

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| `cluster_state:fail` | 槽没分配完 / 节点没起全 | `cluster info` 看 `cluster_slots_assigned` 是否 16384；`cluster nodes` 看是不是 6 个 |
| **`(error) MOVED ...`** | 没用 `-c` 参数 | **用 `redis-cli -c`**（这不是故障） |
| `(error) CLUSTERDOWN The cluster is down` | 挂掉的主节点超过半数，或槽丢失 | 把节点恢复起来 |
| `CROSSSLOT Keys in request don't hash to the same slot` | 跨槽多 key 操作 | 用 `{tag}` 把 key 强制同槽，如 `user:{1}:name` |
| 创建集群时 `[ERR] Node ... is not empty` | 节点上已经有数据或旧集群信息 | 清掉数据目录和 `nodes-7000.conf` 重来 |
| 节点重启后槽分配乱了 | 节点配置文件丢了 | `redis-cli --cluster fix <节点>` |
| 集群创建卡在 `Waiting for the cluster to join` | 集群总线端口（业务端口+10000）不通 | 确认 17000/17001 在监听、防火墙放行 |

---

## 八、本篇小结（面试可用的四句话）

```text
① Redis 主从是异步单向复制：主写从读，主从复制本身【不含自动故障切换】。

② 哨兵解决"自动切换"：3 个哨兵 + quorum 2，靠多数派投票判定主节点下线，
   自动选出新主并让其它从节点跟随；关键是别忘 sentinel auth-pass。

③ Cluster 解决"分片 + 高可用"：16384 个哈希槽分给多个主节点，
   每个主带一个从；客户端必须支持 cluster（redis-cli 要加 -c）。

④ 三者共同的前提是持久化（RDB + AOF）—— 没有持久化，
   主从和哨兵只能保证"服务不停"，保证不了"数据不丢"。
```

---

## 九、本篇做完后建议

```text
1. 把三套环境的启停脚本化（或写成 systemd 服务），别每次都手敲 redis-server 路径
2. 给哨兵/Cluster 加监控：redis_exporter + Prometheus + Grafana
3. 回头看 stage3-20（MySQL 与 Redis 主从对比）和 stage3-21（Redis 深度整合理解），
   这两篇是把你今天做的实操"拔高到体系"的，做完实验再看会很有共鸣
4. MySQL 那篇（实战二）做完，可以对照理解：
   MySQL 主从【没有】自带自动切换（要 MHA/Orchestrator/MGR），
   Redis 把这件事做进了自己的生态（Sentinel/Cluster）——
   这个差别是面试的高频考点
```
