# Redis 哨兵模式：部署与故障转移

**哨兵模式（Sentinel）**

- **哨兵（Sentinel）**：独立进程，监控 Redis 主从健康状态，自动选举新主节点（类似"保安团队"）。
- **故障转移（Failover）**：主节点故障时，哨兵自动提升从节点为新主节点。

---

## 一、部署准备

三台 Rocky 8.10 服务器：

| 主机 | IP | 角色 |
| --- | --- | --- |
| node1 | 192.168.171.147 | Redis 从节点 |
| node2 | 192.168.171.148 | **Redis 主节点** |
| node3 | 192.168.171.146 | Redis 从节点 |

**所有节点执行**，安装依赖：

```bash
dnf install -y gcc make tcl wget
```

---

## 二、部署 Redis 与主从配置

### 1. 安装 Redis（所有节点）

```bash
# 下载源码包
wget http://download.redis.io/releases/redis-7.2.0.tar.gz -P /opt
tar -zxvf /opt/redis-7.2.0.tar.gz
cd /opt/redis-7.2.0 && make && make install

# 创建配置文件目录
mkdir -p /usr/local/redis/{conf,data,log}
cp /opt/redis-7.2.0/redis.conf /usr/local/redis/conf/
```

### 2. 配置主节点（node2：192.168.171.148）

```bash
vim /usr/local/redis/conf/redis.conf
```

```ini
bind 0.0.0.0
port 6379
daemonize yes
logfile "/usr/local/redis/log/redis.log"
dir /usr/local/redis/data
requirepass your_password    # 设置密码，主从需一致！
```

```bash
# 启动主节点
redis-server /usr/local/redis/conf/redis.conf
```

### 3. 配置从节点（node1：192.168.171.147、node3：192.168.171.146）

以 node1 为例：

```bash
vim /usr/local/redis/conf/redis.conf
```

```ini
replicaof 192.168.171.148 6379    # 指向主节点 IP 和端口
masterauth your_password          # 主节点密码
```

```bash
# 启动从节点
redis-server /usr/local/redis/conf/redis.conf
```

**进行主从验证**：在主节点 set 值，在两个从节点 get。

---

## 三、配置哨兵（所有节点）

### 1. 编辑配置

```bash
vim /usr/local/redis/conf/sentinel.conf
```

```ini
port 26379
daemonize yes
logfile "/usr/local/redis/log/sentinel.log"
sentinel monitor mymaster 192.168.171.148 6379 2   # 监控主节点，2 为 quorum 值
sentinel auth-pass mymaster your_password
sentinel down-after-milliseconds mymaster 5000     # 5 秒无响应判定为故障
sentinel failover-timeout mymaster 10000           # 故障转移超时时间
```

### 2. 参数解释

```ini
# 哨兵监听的端口
port 26379

# 以守护进程方式运行
daemonize yes

# 日志文件路径
logfile "/var/log/redis/sentinel.log"

# 监控主节点
#   mymaster 是主节点的别名
#   192.168.171.148 是主节点（node2）的 IP 地址
#   6379 是主节点的端口
#   2 是判定主节点客观下线所需的最少哨兵数量（quorum）
sentinel monitor mymaster 192.168.171.148 6379 2

# 如果主节点设置了密码，需要配置认证密码
sentinel auth-pass mymaster your_password

# 主节点多少毫秒内无响应，就认为主节点主观下线
sentinel down-after-milliseconds mymaster 5000

# 故障转移超时时间
sentinel failover-timeout mymaster 10000

# 执行故障转移时，最多可以有多少个从节点同时对新的主节点进行同步
sentinel parallel-syncs mymaster 1
```

### 3. 启动哨兵（所有节点）

```bash
redis-sentinel /usr/local/redis/conf/sentinel.conf
```

```bash
redis-cli -p 26379 info sentinel    # 查看状态
```

---

## 四、模拟主节点宕机（故障转移）

```bash
# 在主节点执行
redis-cli -h 192.168.171.148 shutdown
```

**观察哨兵日志**：

```bash
tail -f /usr/local/redis/log/sentinel.log    # 查看选举新主过程
```

**验证新主节点**：

```bash
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster    # 查看新主 IP
```

---

## 五、★ 原主节点恢复后会怎么样（补充）

**结论先说**：原主节点恢复上线后，**哨兵会自动把它降级成新主的从节点**；同时它会**清空自己的数据、从新主全量同步一次**。

### 5.1 恢复过程的三个阶段

| 阶段 | 会发生什么 | 日志关键字 |
| --- | --- | --- |
| **① 刚起来** | 它按自己的 `redis.conf` 启动。因为配置文件里**没有 `replicaof`**，所以它一开始**还是 `role:master`** | — |
| **② 哨兵发现它** | 哨兵周期性发 `INFO` / `PING`，发现"原主回来了，但它不是新主的从" | `-sdown slave <ip>:6379`（不再是下线状态） |
| **③ 哨兵降级它** | 哨兵**直接发 `SLAVEOF <新主IP> <端口>` 命令**，把它变成从节点 | **`+convert-to-slave slave <ip>:6379 ...`** |

### 5.2 完整验证步骤

```bash
# 1) 在原主节点上重新启动 Redis
redis-server /usr/local/redis/conf/redis.conf

# 2) 立刻看它的角色（刚起来还是 master）
redis-cli -h 192.168.171.148 -a 'your_password' INFO replication | head -3
# 期望：role:master      ← 因为配置文件里没有 replicaof

# 3) 等 10 秒左右，让哨兵处理
sleep 10

# 4) 再看一次 —— ★ 应该已经被降级了
redis-cli -h 192.168.171.148 -a 'your_password' INFO replication | head -4
# 期望：role:slave
#       master_host:<新主的 IP>

# 5) 看哨兵日志里到底发生了什么
grep -E 'convert-to-slave|sdown|switch-master' /usr/local/redis/log/sentinel.log | tail -10
```

### 5.3 ⚠️ 哨兵改的是"运行时"，不是配置文件

```text
哨兵发的是 REPLICAOF 命令（运行时生效），它【不会】去改原主节点的 redis.conf。

所以你会看到这个现象：
   运行时    ：role:slave          ← 哨兵降级的结果
   配置文件  ：没有 replicaof      ← 文件根本没被动过

★ 后果：如果你这时候【重启】原主节点的 Redis，
        它会按配置文件重新变成 master，
        然后哨兵发现后【再降级一次】。
        这个"反复"是正常现象，不是故障。
```

**想彻底固化，就把 `replicaof` 写进配置文件**：

```ini
# 在 node2 的 /usr/local/redis/conf/redis.conf 里加上（IP 换成恢复后的实际新主）
replicaof 192.168.171.147 6379
masterauth your_password
```

### 5.4 ★ 原主上的数据会怎样（会被覆盖）

```text
场景：主节点故障那一瞬间，客户端刚写了一条数据，还没同步给从节点
      → 这条数据只存在于原主上

恢复后：
   原主变成从节点 → 执行全量同步 → 【先清空自己】→ 再从新主拉 RDB
                                          │
                                          └─ 那条"独有"的数据就丢了

★ 这是异步复制的固有代价 —— 哨兵保证的是【服务可用】，
  不是【数据零丢失】。
```

**想减少丢数据的窗口**，两个方向：

| 方向 | 做法 |
| --- | --- |
| Redis 侧 | 配 `min-replicas-to-write 1` + `min-replicas-max-lag 10` —— 写入时强制至少 1 个从节点确认 |
| 应用侧 | 关键写操作做补偿/重试，或者关键数据同时落库 |

### 5.5 sentinel.conf 会被自动改写

```bash
# 看看哨兵的配置文件被改成了什么
grep -E 'monitor|known-replica|known-sentinel|current-epoch' /usr/local/redis/conf/sentinel.conf
```

你会看到类似：

```ini
sentinel monitor mymaster <新主IP> 6379 2                  ← 监控目标已经变成新主
sentinel known-replica mymaster 192.168.171.148 6379       ← 原主现在是"已知从节点"
sentinel known-sentinel mymaster 192.168.171.147 26379 ...
sentinel current-epoch 1                                    ← 配置纪元 +1
```

> ⚠️ **所以千万别手动编辑正在运行的 `sentinel.conf`** —— 哨兵会覆盖你的修改，还可能把集群状态搞乱。

### 5.6 恢复完成后，做一遍全拓扑验证

```bash
# 1) 哨兵视角：现在的主是谁
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster

# 2) 哨兵视角：有哪些从节点
redis-cli -p 26379 sentinel slaves mymaster

# 3) 三个实例各自的角色
for ip in 192.168.171.147 192.168.171.148 192.168.171.146; do
    echo "=== $ip ==="
    redis-cli -h $ip -a 'your_password' INFO replication | head -2
done
# 期望：1 个 role:master + 2 个 role:slave

# 4) 数据还在不在：在新主写，在原主（现在的从）读
redis-cli -h <新主IP> -a 'your_password' SET after_recovery "ok"
redis-cli -h 192.168.171.148 -a 'your_password' GET after_recovery
# 期望：ok
```

### 5.7 想恢复回原来的拓扑（让 node2 重新当主）

```bash
# 用哨兵做一次"计划内"故障转移
redis-cli -p 26379 sentinel failover mymaster
```

> ⚠️ **注意**：`failover` 是从**当前从节点里挑一个**当新主，**不保证一定挑到 node2**。想精确控制就得手动调 + 重启哨兵，比较折腾。
>
> **最省事的两种做法**：
> 1. **接受现状** —— 谁当主都一样，反正是主从关系，服务不受影响；
> 2. **推倒重来** —— 按笔记的初始配置，把三台的 Redis 和哨兵全部重启一遍，回到初始拓扑。

---

## 六、Redis 命令速查

```bash
# 查看主从信息
redis-cli -h <IP> info replication

# 动态修改从节点指向（脱离主从）
redis-cli -h 192.168.171.147 SLAVEOF NO ONE

# 查看哨兵监控状态
redis-cli -p 26379 sentinel masters
```

---

## 附：故障转移全过程日志对照表（便于排查）

| 日志行 | 含义 |
| --- | --- |
| `+sdown master mymaster <IP> <端口>` | **主观下线** —— 某个哨兵认为主节点没响应了 |
| `+odown master mymaster <IP> <端口>` | **客观下线** —— 认为下线的哨兵达到了 quorum（2 个） |
| `+elected-leader master mymaster ...` | 选举出一个领头哨兵来执行故障转移 |
| `+failover-state-select-slave master ...` | 开始挑选新主 |
| `+selected-slave slave <IP>:6379 ...` | 选中了某个从节点当新主 |
| `+failover-state-send-slaveof-noone ...` | 让选中的从节点脱离主从、成为独立主节点 |
| **`+switch-master mymaster <旧主IP> <端口> <新主IP> <端口>`** | **★ 切换完成** —— 主节点换了 |
| `+convert-to-slave slave <原主IP>:6379 ...` | **★ 原主恢复后被降级为从节点** |
| `-sdown slave <IP>:6379` | 某个从节点不再是"下线"状态（恢复上线了） |