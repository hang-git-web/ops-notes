# notes3 实战环境规划（复制备份 / MySQL 主从 / Redis）

> **这份文档解决什么**：今天要一次做完 `notes3` 里三个模块的实战，先定清楚**要建几台虚拟机、每台装什么、按什么顺序做、哪里会踩坑**。
> **结论：3 台 Rocky Linux 8.10**（2 核 4G / 40G），IP 沿用笔记里的 `172.22.4.2 / .3 / .4`，这样笔记里的命令基本可以原样复制。
> **本文覆盖的笔记**：
> - 复制备份：`stage3-06`（备份与恢复）、`stage3-07`（mysqldump 详解）、`stage3-08`（XtraBackup）
> - MySQL 主从：`stage3-09`（多实例与远程登录）、`stage3-10`（主从配置）
> - Redis：`stage3-14/16`（部署与主从）、`stage3-17`（哨兵）、`stage3-18`（Cluster）、`stage3-19`（持久化）

---

## 一、先算清楚：每个模块最少要几台

| 笔记 | 内容 | 最少机器数 | 为什么 |
| --- | --- | --- | --- |
| 06 / 07 | mysqldump 备份恢复、binlog 按时间点恢复 | **1 台** | 备份到本地目录，再恢复到测试库验证 |
| 08 | XtraBackup 全量 + 增量热备 | **1 台** | 同机演练即可；异机恢复验证需要第 2 台（可选） |
| 09 | MySQL 多实例（3306 + 3307） | **1 台** | 同一台机器上跑两个独立实例 |
| 10 | MySQL 主从复制 | **2 台** | 一主一从 |
| 16 | Redis 主从复制 | **2 台** | 笔记明确写"准备两台机器，一台为主，一台为从" |
| 17 | Redis 哨兵（Sentinel） | **3 台** | 哨兵靠多数派投票，笔记里也是 3 台、`quorum 2` |
| 18 | Redis Cluster | **3 台** | 每台 2 个实例（7000/7001），3 主 3 从 |
| 19 | Redis 持久化（RDB / AOF） | **1 台** | 单机够 |

**取最大值 = 3 台**。三个模块复用同一批机器，串行做，不需要建 7 台。

---

## 二、主机规划（3 台）

```text
                       宿主机（你的 PC）
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
     172.22.4.2         172.22.4.3         172.22.4.4
       node1              node2              node3
   ┌────────────┐    ┌────────────┐    ┌────────────┐
   │ MySQL 8.0  │    │ MySQL 8.0  │    │ MySQL 8.0  │
   │ (可选/验证) │    │  主库      │    │  从库      │
   │            │    │ + 实例3307 │    │            │
   ├────────────┤    ├────────────┤    ├────────────┤
   │ Redis      │    │ Redis      │    │ Redis      │
   │ master     │    │ slave      │    │ slave      │
   │ sentinel   │    │ sentinel   │    │ sentinel   │
   │ cluster    │    │ cluster    │    │ cluster    │
   │ 7000/7001  │    │ 7000/7001  │    │ 7000/7001  │
   └────────────┘    └────────────┘    └────────────┘
```

| 主机名 | IP | 配置 | 承担的模块 |
| --- | --- | --- | --- |
| **node1** | 172.22.4.2 | 2 核 / 4G / 40G | Redis 哨兵 + Cluster（笔记 17/18 里的 `.2`）；备份模块的**异机恢复验证机** |
| **node2** | 172.22.4.3 | 2 核 / 4G / 40G | **MySQL 主库**（10）+ 多实例 3307（09）+ **Redis master**（16/17/18） |
| **node3** | 172.22.4.4 | 2 核 / 4G / 40G | **MySQL 从库**（10）+ **Redis slave**（16/17/18） |

### 关于笔记里 IP 的一处不一致

| 笔记 | 主节点 IP |
| --- | --- |
| `stage3-10` MySQL 主从 | 主库 `172.22.4.3`、从库 `172.22.4.4` |
| `stage3-16` Redis 主从 | 主节点 `172.22.4.3`、从节点 `172.22.4.4` |
| `stage3-17` Redis 哨兵 | 主节点 `172.22.4.2`，从节点 `.3` 和 `.4` |
| `stage3-18` Redis Cluster | 三台 `.2 / .3 / .4` |

**统一口径（按上面这张表）**：

- **MySQL 主库 = `.3`**、**Redis master = `.3`**（和笔记 10、16 一致）
- 做笔记 17（哨兵）时，把 `sentinel monitor mymaster` 的目标改成 **`172.22.4.3`**（或临时让 `.2` 当 Redis 主节点）
- Cluster 按笔记原样用三台 `.2 / .3 / .4`

> 记住这个规律：**`.3` 是主，`.4` 是从，`.2` 只在哨兵和 Cluster 里当"第三方投票节点"**。

---

## 三、操作系统：用 Rocky Linux 8.10

### 为什么不用 CentOS 7

| | CentOS 7.9 | Rocky Linux 8.10 |
| --- | --- | --- |
| 维护状态 | **已于 2024-06-30 EOL** | 8.x 支持到 2029 年 |
| yum 源 | 官方镜像已下架，**必须改 vault 源才能装包** | **开箱即用** |
| chrony 时间同步 | 需要手动装和启 | **装机自带并已启用** |
| 软件包版本 | 偏老 | 较新（gcc 8.5、python 3.6+） |
| 和本仓库其他笔记的一致性 | 部分笔记是 CentOS | **`stage2-05` Docker 三篇本来就是 Rocky 8.10**，统一后只维护一套系统 |

**结论：换 Rocky 8.10 是值得的**，省掉修源、时间同步现成、支持期长。代价只是把几处 `el7` 改成 `el8`。

### 从 CentOS 7 迁到 Rocky 8 要改的地方

| 笔记里写的 | Rocky 8.10 上改成 |
| --- | --- |
| `mysql84-community-release-el7-1.noarch.rpm` | `mysql84-community-release-el8-1.noarch.rpm`（见下方版本说明） |
| `percona-release-latest.noarch.rpm` | **不用改**，脚本会自动识别 el8 |
| `yum install xxx` | **可以照抄**，Rocky 8 的 `yum` 是 `dnf` 的软链接 |
| 服务名 `mysqld` | **不用改**，Rocky 8 装 MySQL 也是 `mysqld` |
| `systemctl restart redis-server`（笔记 16） | 改成 **`systemctl restart redis`**（Rocky/CentOS 的服务名是 `redis`） |

### MySQL 版本：用 8.4（与本机 8.4.11 一致）

**理由**：笔记 08 那张表很关键 —— **XtraBackup 必须和 MySQL 大版本严格匹配**：

| MySQL | XtraBackup |
| --- | --- |
| 5.7 | 2.4（`percona-xtrabackup-24`） |
| 8.0 | 8.0（`percona-xtrabackup-80`） |
| 8.4 | 8.4（`percona-xtrabackup-84`） |

用 8.4 的话，笔记 08 里的 `xtrabackup --backup / --prepare / --copy-back` 命令和参数可以**原样照抄**；主从那节需要的 `SOURCE` / `REPLICA` 新语法在 **8.0.23+** 就有，完全够用。

```bash
dnf install -y https://dev.mysql.com/get/mysql84-community-release-el8-1.noarch.rpm
dnf install -y mysql-community-server
systemctl enable --now mysqld
grep 'temporary password' /var/log/mysqld.log      # 拿初始密码
```

### Redis 版本：统一源码编译 7.2.0，不用 yum 装

**理由**：

- Rocky 8 仓库里的 Redis 只有 6.2 左右，和笔记 17/18 的配置对不上
- 主从、哨兵、Cluster 三节用**同一份配置模板**，`scp` 分发到三台，改一个 IP 就行
- 笔记 17/18/19 本来就是按 7.2.0 写的，命令可直接复用

```bash
dnf install -y gcc make tcl wget
mkdir -p /opt && cd /opt
wget http://download.redis.io/releases/redis-7.2.0.tar.gz
tar -zxvf redis-7.2.0.tar.gz
cd redis-7.2.0 && make && make install
redis-server --version            # 期望：Redis server v=7.2.0
```

> **`make` 一定要装**：笔记 17/18/19 写的 `yum install -y gcc tcl wget` **漏了 `make`**，照抄会在 `make && make install` 报 `make: command not found`。

---

## 四、端口规划（三套服务共存不冲突）

| 服务 | 端口 | 出现在 |
| --- | --- | --- |
| MySQL 主实例 | 3306 | 09 / 10 |
| MySQL 多实例 | 3307 | 09 |
| MySQL 多实例（可选第二个） | 3308 | — |
| Redis | 6379 | 16 / 17 / 19 |
| Redis 哨兵 | 26379 | 17 |
| Redis Cluster | 7000 / 7001 | 18 |

**互不冲突**，所以三套东西可以同时装在同一批机器上，只是在做某一节时注意别把别的服务停掉。

> **唯一要小心的地方**：做 Redis 哨兵和 Cluster 的故障演练时会 `shutdown` 主节点。如果这时正好在做 MySQL 主从，别搞混了是哪台机器的哪个服务。

---

## 五、每台机器装什么

### 三台通用（统一基础环境）

```bash
# 1) 基础工具
dnf install -y vim wget net-tools lsof bash-completion tar

# 2) 时间同步（主从复制要求时间一致，笔记 10 明确提到了）
systemctl enable --now chronyd
chronyc sources -v
date                              # 三台对比一下，应该基本一致

# 3) 实验环境：关闭 SELinux 和 firewalld
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld
getenforce                        # 期望 Permissive
systemctl is-active firewalld     # 期望 inactive

# 4) hosts 解析（三台都写）
cat >>/etc/hosts<<'EOF'
172.22.4.2 node1
172.22.4.3 node2
172.22.4.4 node3
EOF
ping -c1 node2 && ping -c1 node3  # 验证互通
```

### 按角色装

| 装什么 | 装在哪 | 用途 |
| --- | --- | --- |
| **MySQL 8.0** | node2、node3（node1 可选） | 09 / 10 / 06 / 07 |
| **Percona 源 + XtraBackup 8.0** | node2 | 08 |
| **Redis 7.2.0（源码编译）** | 三台都装 | 16 / 17 / 18 / 19 |

**XtraBackup 安装**（node2）：

```bash
dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
percona-release enable-only tools release
dnf install -y percona-xtrabackup-84
xtrabackup --version              # 期望 8.0.x
```

> **注意 `percona-release enable-only tools release`**：这样只启用 tools 仓库的 GA 版本，避免装到测试版。如果提示找不到包，先 `dnf makecache` 再试。

---

## 六、一天的执行顺序

原则：**破坏性最强的放前面**，因为后面随时可以重建。

| 顺序 | 时段 | 模块 | 在哪几台做 | 为什么这个顺序 |
| --- | --- | --- | --- | --- |
| 0 | 开工前 | 环境准备 | 三台全上 | 装系统、跑第五章的通用配置、**打快照** |
| 1 | 上午 | **备份**（06 → 07 → 08） | node2 为主，node1 做异机恢复 | 物理备份和 XtraBackup 恢复会**覆盖 `/var/lib/mysql`**，先做完，后面主从环境重建一遍就好 |
| 2 | 中午前 | **MySQL 多实例**（09） | node2（3307） | 不动 3306，不影响后面的主从 |
| 3 | 下午前段 | **MySQL 主从**（10） | node2 主 + node3 从 | 笔记里"已有数据时的初始化"正好复用第 1 步的 `--source-data=2` 备份知识 |
| 4 | 下午中段 | **Redis 持久化 + 主从**（19 → 16） | node2 主 + node3 从 | 先把单机持久化跑通，再加从节点 |
| 5 | 下午后段 | **Redis 哨兵**（17） | 三台全上 | 要 shutdown 主节点演练故障转移，**做完再进 Cluster** |
| 6 | 傍晚 | **Redis Cluster**（18） | 三台全上（7000/7001） | 最后做，要起 6 个实例、占资源最多 |

> **每完成一个模块就打一次快照**。三个模块都有不可逆操作，出事时回滚比修环境快得多。

---

## 七、每个模块的验收标准

### 模块一：复制备份（06 / 07 / 08）

```bash
# 06 + 07：mysqldump 备份并恢复到测试库
mysqldump -uroot -p --single-transaction --routines --triggers --events \
  --default-character-set=utf8mb4 mydb | gzip > /backup/mydb_$(date +%F).sql.gz
chmod 600 /backup/mydb_$(date +%F).sql.gz

mysql -uroot -p -e "CREATE DATABASE mydb_verify DEFAULT CHARACTER SET utf8mb4;"
gunzip < /backup/mydb_$(date +%F).sql.gz | mysql -uroot -p mydb_verify
mysql -uroot -p -e "USE mydb_verify; SHOW TABLES;"
```

| 检查项 | 通过标准 |
| --- | --- |
| 备份文件生成 | `ls -lh /backup/*.sql.gz` 有文件且**大小不为 0** |
| 备份文件权限 | `chmod 600` 后 `ls -l` 显示 `-rw-------` |
| **恢复到测试库能查到数据** | `SHOW TABLES` 有表，`SELECT COUNT(*)` 对得上 |
| binlog 位置已记录 | `--source-data=2` 导出的文件里有 `CHANGE REPLICATION SOURCE TO` 注释 |
| binlog 按时间点恢复 | `mysqlbinlog --start-datetime=... 管道符 mysql` 能补回数据 |
| XtraBackup 全量备份 | `/data/backup/full/` 下有 `xtrabackup_checkpoints`、`ibdata1` |
| XtraBackup 增量链衔接 | 上一份 `to_lsn` **等于**这一份的 `from_lsn` |
| **XtraBackup 恢复后能启动** | `systemctl start mysqld` 成功，`SHOW DATABASES` 数据完整 |

> **"备份成功 ≠ 备份可用"** —— 必须真的恢复一次验证过，这也是笔记 06 结尾强调的那条。

### 模块二：MySQL 主从（09 / 10）

```bash
# 09：多实例（注意：如果配置节名写成 [mysqld2]，可能没生效）
systemctl start mysqld2
ss -lntp | grep 3307
mysql -uroot -p -P 3307 -S /var/lib/mysql2/mysql.sock -e "select @@port;"
#   期望输出 3307（如果输出 3306，说明配置没生效，见第九章第 3 条）

# 10：主从状态
mysql -uroot -p -e "SHOW REPLICA STATUS\G" | grep -E "Running|Behind|Error"
```

| 检查项 | 通过标准 |
| --- | --- |
| 两个实例都在监听 | `ss -lntp` 同时看到 3306 和 3307 |
| 多实例端口正确 | `select @@port` 返回 3307 |
| 主库 binlog 已开 | `SHOW BINARY LOG STATUS` 有 File 和 Position |
| 复制账号已建 | `SELECT user,host FROM mysql.user WHERE user='repl';` |
| **从库 IO 线程** | `Replica_IO_Running: Yes` |
| **从库 SQL 线程** | `Replica_SQL_Running: Yes` |
| **复制延迟** | `Seconds_Behind_Source: 0`（或接近 0） |
| **两端都没有错误** | `Last_IO_Error` 和 `Last_SQL_Error` **都是空** |
| 从库只读 | `SELECT @@read_only;` 返回 1 |
| **写入能同步过去** | 主库建库建表插数据 → 从库能查到 |

### 模块三：Redis（16 / 17 / 18 / 19）

```bash
# 16：主从
redis-cli -h 172.22.4.3 -a 'Redis@2026' INFO replication | head -5
#   期望：role:master，下面有 connected_slaves:2

redis-cli -h 172.22.4.4 -a 'Redis@2026' INFO replication | head -5
#   期望：role:slave，master_link_status:up

# 17：哨兵
redis-cli -p 26379 sentinel masters
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster

# 18：Cluster
redis-cli -c -h 172.22.4.2 -p 7000 cluster info | grep cluster_state
#   期望：cluster_state:ok
redis-cli -c -h 172.22.4.2 -p 7000 cluster nodes | wc -l
#   期望：6
```

| 检查项 | 通过标准 |
| --- | --- |
| Redis 版本统一 | 三台 `redis-server --version` 都是 7.2.0 |
| 主从建立成功 | 从库 `master_link_status:up`、`role:slave` |
| **主写从读能通** | 主库 `SET mykey "Hello, Redis!"` → 从库 `GET mykey` 拿到同样的值 |
| 从库只读 | 从库 `SET` 报 `READONLY You can't write against a read only replica` |
| 哨兵能看到主 | `sentinel masters` 里 `num-slaves` 和 `flags` 正常 |
| **哨兵能故障转移** | 主节点 `shutdown` 后，`get-master-addr-by-name` 的 IP **变了**，日志里有 `+switch-master` |
| 原主恢复后变从 | 重启原来的主节点，它自动变成新主的从（哨兵会 `reconfig`） |
| Cluster 状态正常 | `cluster_state:ok`、`cluster_slots_assigned:16384` |
| Cluster 是 3 主 3 从 | `cluster nodes` 有 6 行，3 个 master 3 个 slave |
| **Cluster 能自动重定向** | `redis-cli -c` 写数据不报 MOVED，`get` 能读到 |
| Cluster 故障转移 | `shutdown` 一个主节点后，它的从节点变成 master |
| RDB 持久化 | `redis-cli bgsave` 后 `dump.rdb` 存在且变大 |
| AOF 持久化 | `appendonly.aof` 存在，`tail` 能看到写命令 |
| **重启后数据还在** | `pkill redis-server` → 重启 → `GET` 之前写的值还在 |
| 损坏文件能修 | `redis-check-aof --fix` 能跑通 |

---

## 八、开做之前的准备

### 1. 网络模式（要用 172.22.4.x，二选一）

| 方案 | 做法 | 适合 |
| --- | --- | --- |
| **方案 A（推荐）** | 每台加两块网卡：一块 **NAT**（上网装包、下 Redis 源码），一块 **仅主机 Host-Only**，仅主机网段设成 `172.22.4.0/24` | 你 PC 上还同时跑着 Zabbix 那套（`192.168.171.x`）时，互不影响 |
| 方案 B | 只用一块 NAT 网卡，在 VMware「虚拟网络编辑器」里把 **VMnet8 子网改成 `172.22.4.0/24`** | PC 上只有这一套实验环境 |

**验证**：

```bash
ip -4 addr | grep 172.22.4          # 三台各有自己的 IP
ping -c2 172.22.4.3                 # 三台互相 ping 通
ping -c2 www.baidu.com              # 能上外网（装包、下源码要用）
```

### 2. 快照策略

| 时机 | 打什么快照 |
| --- | --- |
| 系统装好 + 通用配置做完 | `snap-00-base`（干净基线，以后再建实验环境都从这开始） |
| 每个模块开始前 | `snap-01-backup` / `snap-02-replication` / `snap-03-redis` |
| 破坏性操作前（`--copy-back`、`shutdown` 主节点、删数据目录） | 临时快照，出事立刻回滚 |

### 3. 资源检查

| 项 | 建议 | 说明 |
| --- | --- | --- |
| 内存 | **每台 4G** | MySQL 8 + XtraBackup + Redis 同机跑，2G 会紧张（容易 OOM） |
| CPU | 2 核 | 编译 Redis、XtraBackup 备份时吃 CPU |
| 磁盘 | **每台 40G** | MySQL 数据 + binlog + XtraBackup 全量/增量 + Redis AOF |
| 宿主机总占用 | 3 台 × 4G ≈ 12G | 确认 PC 内存够，不够就把三台的 MySQL 内存参数调小 |

---

## 九、笔记里需要先改掉的地方

这几条**和系统版本无关**，是笔记本身的小问题，动手前先改，能省掉很多排查时间。

### 1. 笔记 09：`LimitNOFILE = 5000` 里 `=` 两边有空格

systemd 单元文件里 `Key = Value` 这种写法会报 **"Unknown lvalue"** 警告，并且**这一行会被忽略**。改成：

```ini
LimitNOFILE=5000
```

### 2. 笔记 17 / 18 / 19：装依赖少了 `make`

三处都写 `yum install -y gcc tcl wget`，但 `make` 不在默认安装里，`make && make install` 会直接失败。改成：

```bash
dnf install -y gcc make tcl wget
```

### 3. 笔记 09：多实例的配置节名可能是 `[mysqld2]`

用 `mysqld --defaults-file=/etc/my.cnf.d/mysql2.cnf` 启动时，mysqld 读的是文件里的 **`[mysqld]`** 节，**不会读 `[mysqld2]`**。如果节名写成 `[mysqld2]`，里面的 `port=3307` / `datadir` 可能**一条都没生效**。

两种改法，任选：

```ini
# 改法 A（推荐，简单）：直接叫 [mysqld]
[mysqld]
port = 3307
datadir = /var/lib/mysql2
...

# 改法 B（想保留 [mysqld2] 这个名字）：启动时加 --defaults-group-suffix
# 单元文件的 ExecStart 改成：
#   /usr/sbin/mysqld --defaults-file=/etc/my.cnf.d/mysql2.cnf --defaults-group-suffix=2
```

**启动后必须验证**（这是判断有没有生效的唯一标准）：

```bash
ss -lntp | grep 3307
mysql -uroot -p -P 3307 -S /var/lib/mysql2/mysql.sock -e "select @@port;"
# 期望输出 3307；如果输出 3306，说明配置没生效
```

### 4. 笔记 16：`systemctl restart redis-server` 服务名不对

`redis-server` 是 Debian/Ubuntu 的服务名。**CentOS / Rocky 上的服务名是 `redis`**：

```bash
systemctl restart redis        # Rocky / CentOS
```

### 5. 笔记 16：`requirepass` 和 `protected-mode no` 同时开时，`masterauth` 必须一致

如果主从都设了密码，从节点的 `masterauth` 必须和主节点 `requirepass` **完全一致**，否则 `master_link_status` 会一直是 `down`。

**建议**：三台统一用一个密码（比如 `Redis@2026`），配置模板直接 `scp` 分发，从节点只改 `replicaof` 那一行。

### 6. 笔记 10：自定义 binlog 路径会踩 SELinux

笔记本身已经提示了。**练习环境直接用默认路径最省事**（`log-bin = mysql-bin`，不写绝对路径），不要自定义到 `/var/log/mysql/` 之类的地方。

---

## 十、环境层排错速查

| 现象 | 可能原因 | 怎么查 |
| --- | --- | --- |
| 三台互相 ping 不通 | 网卡模式不对、网段不一致、防火墙 | `ip -4 addr` 看 IP，`systemctl status firewalld` |
| `dnf install` 报 404 | 源没配好（Rocky 8 一般不会） | `dnf repolist`，确认 baseos / appstream 都是 enabled |
| 主从 `Seconds_Behind_Source` 一直涨 | 大事务、单线程重放、从库资源不足 | 开 `replica_parallel_workers`，笔记 10 第十章有完整方案 |
| 主从 `Last_IO_Error: 1236` | 指定的 binlog 已被清理 | 重新全量导出 + 重新配置位置（笔记 10 第九章） |
| Redis 从库 `master_link_status:down` | 密码不一致、主节点没开 `bind 0.0.0.0`、端口不通 | `redis-cli -h <主> -a <密码> ping` 先测连通 |
| 哨兵不触发故障转移 | quorum 设太大或哨兵数量不够 | 三个哨兵都起，`quorum` 设 2 |
| Cluster `cluster_state:fail` | 槽没分配完、节点没起全 | `cluster info` 看 `cluster_slots_assigned` 是否 16384 |
| `cluster_state:ok` 但写数据报 MOVED | 没用 `-c` 参数 | 用 `redis-cli -c` 启动客户端 |
| `xtrabackup` 报版本不匹配 | MySQL 和 XtraBackup 大版本对不上 | `mysql --version` 和 `xtrabackup --version` 对比 |
| XtraBackup 恢复后 MySQL 起不来 | 权限或 SELinux 标签没修 | `chown -R mysql:mysql /var/lib/mysql` + `restorecon -Rv /var/lib/mysql` |
| 编译 Redis 报 `jemalloc` 相关错误 | 缺依赖 | `dnf install -y gcc make tcl` 后 `make distclean && make` |
| 磁盘突然满 | binlog / AOF / XtraBackup 备份堆积 | `df -h`、`du -sh /var/lib/mysql /data/backup`、`df -i` |

---

## 十一、开工前对照清单

- [ ] 三台 Rocky 8.10 建好，2 核 4G 40G
- [ ] 三台网卡配好（NAT + Host-Only 或改 VMnet8），IP 是 `.2 / .3 / .4`
- [ ] 三台互相 ping 通、都能上网
- [ ] 三台 SELinux 关掉、firewalld 关掉
- [ ] 三台 chronyd 正常、时间基本一致
- [ ] 三台 hosts 写好（node1 / node2 / node3）
- [ ] 三台都编译装好 Redis 7.2.0
- [ ] node2 / node3 装好 MySQL 8.0，node2 装好 XtraBackup 8.0
- [ ] **基线快照 `snap-00-base` 已打**
- [ ] 笔记 09 / 16 / 17 / 18 / 19 里那几处小问题已改（见第九章）