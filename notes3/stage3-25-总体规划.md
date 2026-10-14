# notes3 总体规划 · 环境准备 + 节点任务 + 实战路线

> **本文定位**：`notes3` 的**总入口**。把 `stage3-00-环境规划.md` 的内容整合进来，并补上最缺的一块 —— **每个节点在这些实战里到底承担什么任务**。
> **适用环境**：3 台 Rocky Linux 8.10（2 核 / 4G / 40G）+ MySQL 8.4 + Redis 7.2.0
> **配套实战**：
> | 实战 | 文件 | 整合的原笔记 |
> | --- | --- | --- |
> | 实战一 · 备份篇 | `stage3-22-实战一-备份篇.md` | 06 / 07 / 08 |
> | 实战二 · MySQL 主从篇 | `stage3-23-实战二-MySQL主从篇.md` | 09 / 10 |
> | 实战三 · Redis 篇 | `stage3-24-实战三-Redis篇.md` | 16 / 17 / 18 / 19 |
>
> **和 `00-环境规划` 的关系**：00 解决的是"要几台机、怎么配"，本文在此基础上补上"**每台机在每个实验里干什么**"，是升级版。旧的 00 可以保留对照，也可以删掉。

---

## 一、先看全局（一句话 + 一张图）

**一句话**：**3 台机器够跑完全部三个模块**，节点角色**固定不乱** —— node2 永远是"主"，node3 永远是"从"，node1 是第三台（Redis 的第三个投票节点 + 备份的异机验证机）。

```text
                    ┌─────────────────────────────────────────┐
                    │              你的 PC                     │
                    │         VMware NAT 192.168.171.0/24      │
                    └────────────────┬────────────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
  ┌──────────────┐            ┌──────────────┐            ┌──────────────┐
  │  node1       │            │  node2       │            │  node3       │
  │ .147         │            │ .148         │            │ .146         │
  │ 第三方/验证  │            │ ★ 主节点     │            │ 从节点       │
  ├──────────────┤            ├──────────────┤            ├──────────────┤
  │ MySQL        │            │ MySQL 8.4 主 │            │ MySQL 8.4 从 │
  │ (可选)       │            │ + 3307 多实例│            │              │
  │              │            │ XtraBackup   │            │              │
  ├──────────────┤            ├──────────────┤            ├──────────────┤
  │ Redis 从     │            │ Redis 主     │            │ Redis 从     │
  │ Sentinel     │            │ Sentinel     │            │ Sentinel     │
  │ Cluster      │            │ Cluster      │            │ Cluster      │
  └──────────────┘            └──────────────┘            └──────────────┘
```

**为什么 IP 不是顺序的？** `.146` 给了 node3、`.147` 给了 node1、`.148` 给了 node2。**这不是笔误**，是因为 IP 是随虚机创建顺序分下来的。**正因为不顺序，所以每次开工前必须核对"我是哪台"**（见第二章 2.2）。

---

## 二、环境准备（四步走）

### 2.1 第一步：建三台虚拟机

| 项 | 建议值 | 为什么 |
| --- | --- | --- |
| 系统 | **Rocky Linux 8.10** | MySQL 8.4 只提供 el8/el9 包（要求 glibc 2.28+，CentOS 7 装不了）；且官方源开箱可用，不用修源 |
| CPU | **2 核**（处理器数量 2 × 每处理器内核 1） | 系统里看到的就是 2 个逻辑 CPU |
| 内存 | **4096 MB** | MySQL 8 + Redis + XtraBackup 同机跑，2G 会紧张 |
| 磁盘 | **40G 起**（选"单个文件"、不勾"立即分配"） | 不预分配所以按需增长；单个文件性能更好 |
| 网络 | **NAT** | 要能上网装包、下 Redis 源码 |
| 软件选择 | **最小安装** + 勾选 **「开发工具」** | Redis 要源码编译，装的时候带上 gcc/make 省事 |

**三台一致**，不要一台 2G 一台 4G，否则后面 Cluster/哨兵排障时容易误判。

> **如果 PC 内存吃紧**：三台先给 2G，跑到 Redis Cluster 那一步（要起 6 个实例）再临时加内存。

### 2.2 第二步：网络与主机名（★ 最容易装反的一步）

**IP 与主机名对应关系（务必一一对上）**：

| 主机名 | IP | 角色定位 |
| --- | --- | --- |
| **node1** | **192.168.171.147** | 第三方 / 验证机 |
| **node2** | **192.168.171.148** | **主节点（MySQL 主库 + Redis 主）** |
| **node3** | **192.168.171.146** | 从节点 |

**在每台上设置主机名**（node1 上就写 node1，以此类推）：

```bash
hostnamectl set-hostname node1
```

**三台都加 hosts 解析**：

```bash
cat >>/etc/hosts<<'EOF'
192.168.171.147 node1
192.168.171.148 node2
192.168.171.146 node3
EOF
```

> `/etc/hosts` 的作用就是给本机一张"IP ↔ 名字"的对照表。**不加的话 `ping node2` 会报 `Name or service not known`**（环境里没有内网 DNS）。笔记里命令都用 IP，所以**不加也能跑**，但加上之后排查方便很多。

**★ 三台都必须跑这一条，然后互相核对**：

```bash
echo "我是 $(hostname) ，IP 是 $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')"
```

| 在哪台跑 | 必须输出 |
| --- | --- |
| node1 | `我是 node1 ，IP 是 192.168.171.147/24` |
| node2 | `我是 node2 ，IP 是 192.168.171.148/24` |
| node3 | `我是 node3 ，IP 是 192.168.171.146/24` |

> ⚠️ **这一步对不上，后面所有实验都会错位**（笔记里 `replicaof 192.168.171.148`、`SOURCE_HOST='192.168.171.148'` 全指向 node2）。花 30 秒核对，能省几小时。

### 2.3 第三步：三台通用初始化（装完系统立刻做）

**三台都跑这一段**：

```bash
# 1) 关 SELinux
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# 2) 关 firewalld
systemctl disable --now firewalld

# 3) 时间同步（主从复制要求时间一致）
systemctl enable --now chronyd

# 4) 装基础工具（最小安装默认没有）
dnf install -y vim wget net-tools lsof bash-completion tar

# 5) 验证
echo "SELinux : $(getenforce)"
echo "防火墙  : $(systemctl is-active firewalld)"
echo "时间同步: $(systemctl is-active chronyd)"
echo "主机名  : $(hostname)"
echo "IP      : $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')"
which vim netstat lsof ss tar
```

**通过标准**：SELinux 是 `Permissive`、防火墙 `inactive`、chronyd `active`、后面 5 个命令都返回路径。

### 2.4 第四步：装软件（按角色装，别三台都装一遍）

| 软件 | 装在哪些节点 | 用途 |
| --- | --- | --- |
| **MySQL 8.4** | **node2、node3** | 备份篇 / 多实例 / 主从 |
| **Percona XtraBackup 8.4** | **node2** | 备份篇的物理热备 |
| **Redis 7.2.0**（源码编译） | **三台都装** | 主从 / 哨兵 / Cluster |

**① MySQL 8.4（node2、node3）**：

```bash
dnf install -y https://dev.mysql.com/get/mysql84-community-release-el8-1.noarch.rpm   # 下载 MySQL 8.4 的 YUM 源
dnf install -y mysql-community-server
systemctl enable --now mysqld
grep 'temporary password' /var/log/mysqld.log
# 用初始密码登录后改密码
mysql -uroot -p'初始密码' -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'Yuanhang@2026';"
```

**② XtraBackup 8.4（node2）**：

```bash
dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm   #装 Percona 的仓库定义包
percona-release enable-only pxb-84-lts      # 启用 8.4 版本的 XtraBackup
dnf makecache   #刷新元数据缓存（改了仓库后要刷新，否则 dnf 还不知道），只是提速，不是必须项
dnf list available 'percona-xtrabackup*'    # 先看能装到哪个版本
dnf install -y percona-xtrabackup-84
xtrabackup --version
```

**③ Redis 7.2.0（三台）**：

```bash
dnf install -y gcc make tcl wget            # ★ make 千万别漏
mkdir -p /opt && cd /opt
wget http://download.redis.io/releases/redis-7.2.0.tar.gz
tar -zxvf redis-7.2.0.tar.gz && cd redis-7.2.0
make && make install
redis-server --version                      # 期望 v=7.2.0
```

**三台都装完后，统一验收**：

```bash
echo "=== $(hostname) ==="
mysql --version 2>/dev/null || echo "MySQL 未装（node1 正常）"
xtrabackup --version 2>/dev/null | head -1 || echo "XtraBackup 未装（非 node2 正常）"
redis-server --version
```

---

## 三、节点任务分工（★ 本文的核心）

### 3.1 总表：每个节点在每个实战里的角色

| | **实战一 · 备份篇** | **实战二 · MySQL 主从篇** | **实战三 · Redis 篇** |
| --- | --- | --- | --- |
| **node1**<br>192.168.171.147 | **异机恢复验证机**（可选）<br>把 node2 的备份文件拷过来，验证"在别的机器上也能恢复" | 不参与<br>（备用的第三台） | **Redis 从节点**<br>**哨兵**（投票节点）<br>**Cluster 节点**（7000/7001）<br>★ 故障转移演练时可能被选为新主 |
| **node2**<br>192.168.171.148 | **★ 主角**<br>mysqldump 备份<br>binlog PITR 演练<br>XtraBackup 热备 | **★ 主库**<br>多实例 3306 + 3307<br>提供 binlog、复制账号 | **★ Redis 主节点**<br>**哨兵**<br>**Cluster 节点** |
| **node3**<br>192.168.171.146 | 可选的"恢复到另一台"验证 | **★ 从库**<br>relay log 重放<br>`read_only` 只读 | **Redis 从节点**<br>**哨兵**<br>**Cluster 节点** |

### 3.2 node1（192.168.171.147）的任务清单

**这台机器的定位是"第三台"** —— 前两个实战它基本闲着，Redis 篇它是主力之一。

| 时机 | 任务 | 具体做什么 | 验收 |
| --- | --- | --- | --- |
| 环境准备 | 通用初始化 | 关 SELinux/防火墙、配 hosts、装 Redis 7.2.0 | `redis-server --version` 出 7.2.0 |
| **实战一（可选）** | 异机恢复验证 | ① 从 node2 拉一份 mysqldump 备份：`scp root@192.168.171.148:/data/backup/mysql/shop_full_*.sql /root/`<br>② 本地建个空库，导入它<br>③ 查数据对不对 | 能查到和 node2 一致的数据 |
| 实战二 | 不参与 | —— | —— |
| **实战三** | Redis 从节点 | `replicaof 192.168.171.148 6379` + `masterauth` | `master_link_status:up`、`role:slave` |
| **实战三** | 哨兵 | 起 `sentinel.conf`（`sentinel monitor mymaster 192.168.171.148 6379 2`） | `info sentinel` 里 `sentinels=3` |
| **实战三** | ★ 故障转移演练 | 主节点挂掉后**它有可能被选为新主**；恢复原主后它继续当从 | 新主可写、`+switch-master` 日志 |
| **实战三** | Cluster | 起 7000 / 7001 两个实例，加入 6 节点集群 | `cluster_state:ok`、`cluster nodes` 6 行 |

### 3.3 node2（192.168.171.148）的任务清单

**这台是绝对主力，三个实战都绕不开它。**

| 时机 | 任务 | 具体做什么 | 验收 |
| --- | --- | --- | --- |
| 环境准备 | 通用初始化 + 装 MySQL 8.4 + XtraBackup 8.4 + Redis 7.2.0 | 见 2.4 | `mysql --version` / `xtrabackup --version` / `redis-server --version` |
| **实战一** | ★ mysqldump 备份 | 建 `shop` 库 → 单库/多库/全库备份 → **恢复到测试库验证** | 恢复后 `SELECT COUNT(*)` 和源库一致 |
| **实战一** | ★ binlog PITR | 开 binlog → 全量备份（`--source-data=2`）→ 写数据 → **误删** → 用 binlog 恢复到误删前 | 恢复后数据是 5 行（不是 3 行） |
| **实战一** | ★ XtraBackup | 建备份账号 → 全量备份 → `--prepare` → `--copy-back` → 启动验证 → 再做增量 | 恢复后 `systemctl start mysqld` 成功、数据完整 |
| **实战一** | 自动化 | 写备份脚本挂 crontab | `crontab -l` 能看到任务 |
| **实战二** | 多实例 | 建 `/var/lib/mysql2` + `[mysqld]` 节配置 + systemd 单元 → **启动 3307** | `SELECT @@port;` 返回 **3307**（不是 3306） |
| **实战二** | ★ 主库配置 | `server-id=1`、`log-bin`、`binlog_format=ROW` | `SHOW VARIABLES LIKE 'log_bin';` 是 ON |
| **实战二** | ★ 复制账号 | 建 `repl@192.168.171.146`，只授 `REPLICATION SLAVE` | `SELECT user,host` 能看到 |
| **实战二** | ★ 全量初始化 | `mysqldump --source-data=2` 导出 → `scp` 给 node3 | 文件里有 `CHANGE REPLICATION SOURCE TO` 注释 |
| **实战二** | 故障演练 | 停掉 mysqld 模拟主库宕机 → 恢复后**变成 node3 的从库** | `role:slave`，`master_host` 指向 146 |
| **实战三** | ★ Redis 主节点 | `requirepass` + 持久化（RDB/AOF） | `role:master`、`connected_slaves:2` |
| **实战三** | ★ 故障转移演练 | `redis-cli shutdown` **主动干掉自己** | 哨兵日志出现 `+switch-master` |
| **实战三** | Cluster | 起 7000/7001，参与 `cluster create` | `cluster_state:ok` |

### 3.4 node3（192.168.171.146）的任务清单

| 时机 | 任务 | 具体做什么 | 验收 |
| --- | --- | --- | --- |
| 环境准备 | 通用初始化 + 装 MySQL 8.4 + Redis 7.2.0 | 见 2.4 | 同上 |
| **实战一（可选）** | 异机恢复验证 | 同 node1，任选一台做一次即可 | 能恢复出数据 |
| **实战二** | ★ 从库配置 | `server-id=2`、`relay-log`、`read_only=1`、`super_read_only=1`、并行复制 | `SELECT @@server_id;` 返回 2 |
| **实战二** | ★ 导入全量 | 接收 node2 的 `scp` 文件并导入 | `SELECT COUNT(*)` 和主库一致 |
| **实战二** | ★ 建立复制 | `CHANGE REPLICATION SOURCE TO ... GET_SOURCE_PUBLIC_KEY=1, SOURCE_SSL_MODE=DISABLED;` + `START REPLICA;` | 两个 `Running: Yes` |
| **实战二** | ★ 端到端验证 | 主库写 → **自己这边读** | 能查到主库刚写的数据 |
| **实战二** | 只读验证 | 自己试着写 | 报 `--read-only` 错误（这是对的） |
| **实战二** | 主从切换 | 主库挂了之后**被提升为新主** | `set GLOBAL read_only=OFF` 后能写 |
| **实战三** | Redis 从节点 | `replicaof 192.168.171.148 6379` | `master_link_status:up` |
| **实战三** | 哨兵 | 起 sentinel | `sentinels=3` |
| **实战三** | Cluster | 起 7000/7001 | `cluster_state:ok` |

### 3.5 三台的"主线任务"速记

```text
node1  = Redis 第三节点 + 备份的异机验证机      （前两篇基本闲着）
node2  = 一切的主角：MySQL 主库 + 多实例 + Redis 主 + XtraBackup
node3  = 从节点：MySQL 从库 + Redis 从

一句话：node2 干活，node3 跟着，node1 凑够三台（哨兵投票 / 集群分片都需要 3 台）
```

---

## 四、三个实战的执行路线

### 4.1 为什么按这个顺序做

```text
实战一 备份篇  →  实战二 MySQL 主从篇  →  实战三 Redis 篇

理由：
① 备份篇有【破坏性演练】（误删表、覆盖数据目录、起停 MySQL），放最前面；
   后面两篇需要用到的库，随时可以重新造。
② 主从篇的"全量初始化"正好复用备份篇学的 mysqldump --source-data=2。
③ Redis 篇要起最多实例（6 个 Cluster 实例），资源占用最大，放最后。
④ 三篇的端口不冲突（3306/3307、6379/26379、7000/7001），但因为都跑在同一批机器上，
   建议【做完一篇停一篇】，避免排查时分不清是哪个服务的问题。
```

### 4.2 实战一 · 备份篇（在 node2 上为主）

| 项 | 内容 |
| --- | --- |
| **在哪几台** | **node2**（主）；node1 或 node3 可选做异机恢复验证 |
| **对应文件** | `stage3-22-实战一-备份篇.md` |
| **关键目标** | ① mysqldump 备份并**真的恢复一次**<br>② **PITR：把误删的数据恢复到误删前**<br>③ XtraBackup 全量 + 增量，恢复后服务能起 |
| **做完的标志** | ① 恢复到测试库能查到数据<br>② 误删演练后数据从 3 行变回 5 行<br>③ `xtrabackup --copy-back` 后 `systemctl start mysqld` 成功 |
| **最值钱的一步** | 第三章的 **binlog PITR 完整演练** |

### 4.3 实战二 · MySQL 主从篇（node2 主 + node3 从）

| 项 | 内容 |
| --- | --- |
| **在哪几台** | **node2**（主库 + 多实例）、**node3**（从库） |
| **对应文件** | `stage3-23-实战二-MySQL主从篇.md` |
| **关键目标** | ① 一台机器上跑起 **3306 + 3307** 两个实例<br>② 跑通 **node2 → node3 主从复制**<br>③ 会看状态、会做**主从切换演练** |
| **做完的标志** | ① `SELECT @@port;` 在 3307 实例上返回 **3307**<br>② 从库 `Replica_IO_Running` / `Replica_SQL_Running` 都是 **Yes**<br>③ **主库写、从库能读到**，从库写被拒 |
| **最容易错的** | `[mysqld2]` 节名（必须 `[mysqld]`）、`LimitNOFILE = 5000` 的空格、复制账号密码与 `masterauth` 不一致 |

### 4.4 实战三 · Redis 篇（三台全上）

| 项 | 内容 |
| --- | --- |
| **在哪几台** | **三台全上** |
| **对应文件** | `stage3-24-实战三-Redis篇.md` |
| **关键目标** | ① 持久化（RDB+AOF）<br>② **主从复制**（node2 主，node1/node3 从）<br>③ **哨兵自动故障转移**<br>④ **Cluster 3 主 3 从** |
| **做完的标志** | ① 强杀 Redis 重启数据还在<br>② 从节点 `master_link_status:up`<br>③ 干掉主节点后**哨兵自动选出新主**（日志里有 `+switch-master`）<br>④ `cluster_state:ok` + `cluster_slots_assigned:16384` |
| **最值钱的一步** | 第三章的 **故障转移演练**（干掉主 → 看选举 → 验证新主 → 原主恢复后自动变从） |

### 4.5 三篇之间的知识串联

```text
备份篇 ──► 主从篇
  mysqldump --source-data=2 记下的 binlog 位置
        │
        └──► 主从篇 2.4 "主库已有数据时的全量初始化" 用的就是这个备份方式

主从篇 ──► Redis 篇（对照理解）
  MySQL 主从【没有】自带自动切换（要 MHA / Orchestrator / MGR）
  Redis  把这件事做进了自己的生态（Sentinel / Cluster）
        │
        └──► 这个差别是面试高频考点（见 stage3-20 / stage3-21）

三篇的共同前提
  持久化：MySQL 靠 binlog + redo log（默认就有）；
          Redis 靠 RDB + AOF（必须显式开启）——
          所以 Redis 篇第一章先确认持久化，就是这个原因。
```

---

## 五、端口总表（三套能共存，互不冲突）

| 服务 | 端口 | 哪几台 | 对应实战 |
| --- | --- | --- | --- |
| MySQL 主实例 | **3306** | node2（主）、node3（从） | 一、二 |
| MySQL 多实例 | **3307** | 仅 node2 | 二 |
| Redis | **6379** | 三台 | 三 |
| Redis 哨兵 | **26379** | 三台 | 三 |
| Redis Cluster | **7000 / 7001** | 三台 | 三 |
| Cluster 集群总线 | **17000 / 17001** | 三台（自动开的，不用配） | 三 |

**查看端口占用的通用命令**：

```bash
ss -lntp | grep -E '3306|3307|6379|26379|7000|7001'
```

> **建议**：做每个实战之前先跑一次上面这条，**确认端口是"干净"的** —— 如果上一套服务没停干净，新的一套会起不来或行为诡异。

---

## 六、建议时间表

### 6.1 两天版（推荐）

| 时间 | 做什么 | 在哪几台 |
| --- | --- | --- |
| **第 1 天 上午** | 建 3 台虚机 + 装系统；跑第二章 2.2 / 2.3 的网络与通用初始化 | 三台 |
| **第 1 天 中午** | 装 MySQL 8.4 / XtraBackup / Redis（2.4） | 按角色 |
| **第 1 天 下午** | **实战一 · 备份篇**（mysqldump → PITR → XtraBackup） | node2 为主 |
| **第 1 天 傍晚** | 打快照 `snap-01-backup-done` | 三台 |
| **第 2 天 上午** | **实战二 · MySQL 主从篇**（多实例 → 主从 → 切换演练） | node2 + node3 |
| **第 2 天 下午** | **实战三 · Redis 篇**（持久化 → 主从 → 哨兵） | 三台 |
| **第 2 天 傍晚** | Cluster（起 6 实例 → 建集群 → 故障演练） | 三台 |

### 6.2 一天极限版

| 时段 | 内容 |
| --- | --- |
| 上午前段 | 环境准备（可以前晚先建好虚机） |
| 上午后段 | 实战一 备份篇（mysqldump + PITR，**XtraBackup 增量可跳过**） |
| 下午前段 | 实战二 MySQL 主从篇（多实例可跳过，直接做主从） |
| 下午后段 | 实战三 Redis 篇（持久化 → 主从 → 哨兵） |
| 傍晚 | Cluster |

> **一天版可以跳过的**：XtraBackup 的**增量**部分（全量跑通即可）、MySQL **多实例**（如果只看主从）。
> **绝对不能跳的**：备份的**恢复验证**、主从的**状态检查**、哨兵的**故障转移演练** —— 这三步才是这三篇的核心价值。

### 6.3 每个阶段结束都打快照

| 时机 | 快照名 | 作用 |
| --- | --- | --- |
| 系统装好 + 通用初始化完成 | `snap-00-base` | 干净基线，以后重建实验环境都从这开始 |
| 每篇实战开始前 | `snap-01-backup` / `snap-02-replication` / `snap-03-redis` | 出事能回滚到"这一篇开始前" |
| 破坏性操作前（`--copy-back`、`shutdown` 主节点、`pkill -9`） | 临时快照 | 立刻回滚 |

---

## 七、开工前检查清单（三份，每台照着勾）

### 7.1 node1（192.168.171.147）—— 第三方/验证机

```bash
echo "主机名: $(hostname)                    期望: node1"
echo "IP    : $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')   期望: 192.168.171.147/24"
getenforce
systemctl is-active firewalld
systemctl is-active chronyd
ping -c1 node2
redis-server --version
```

- [ ] 主机名 = node1，IP = 192.168.171.147
- [ ] SELinux = Permissive，firewalld = inactive，chronyd = active
- [ ] `/etc/hosts` 里三台都写了，`ping node2` / `ping node3` 通
- [ ] Redis 7.2.0 已编译安装
- [ ] 基础工具齐：`which vim netstat lsof ss tar` 都有输出

### 7.2 node2（192.168.171.148）—— 主节点（任务最重）

```bash
echo "主机名: $(hostname)                    期望: node2"
echo "IP    : $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')   期望: 192.168.171.148/24"
mysql --version
systemctl is-active mysqld
xtrabackup --version
redis-server --version
df -h /
free -h
```

- [ ] 主机名 = node2，IP = 192.168.171.148
- [ ] 通用初始化都做了（SELinux / 防火墙 / 时间 / hosts / 工具）
- [ ] **MySQL 8.4 已装、能登录**：`mysql -uroot -p'Yuanhang@2026' -e "SELECT VERSION();"`
- [ ] **XtraBackup 8.4 已装**
- [ ] Redis 7.2.0 已编译
- [ ] `/data` 目录已建，根分区剩余 > 10G（XtraBackup 很占地方）
- [ ] 已打快照 `snap-00-base`

### 7.3 node3（192.168.171.146）—— 从节点

```bash
echo "主机名: $(hostname)                    期望: node3"
echo "IP    : $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')   期望: 192.168.171.146/24"
mysql --version
systemctl is-active mysqld
redis-server --version
ping -c1 192.168.171.148
nc -vz 192.168.171.148 3306
```

- [ ] 主机名 = node3，IP = 192.168.171.146
- [ ] 通用初始化都做了
- [ ] **MySQL 8.4 已装、能登录**
- [ ] Redis 7.2.0 已编译
- [ ] **能连到 node2 的 3306**（这条不通，主从一定建不起来）

### 7.4 三台之间的连通性（在任意一台上跑）

```bash
for ip in 192.168.171.147 192.168.171.148 192.168.171.146; do
    printf "%-18s " "$ip"
    ping -c1 -W1 "$ip" &>/dev/null && echo "通" || echo "不通 ← 检查网卡/网段"
done
```

---

## 八、收工与开机的规范动作

### 8.1 每天收工（虚机不关机）

```bash
df -h | grep -v tmpfs      # 看一眼磁盘别被备份撑满
```

什么都不用停，虚机保持运行即可。

### 8.2 要关虚机（★ 按顺序停服务）

**先停服务，再关机**，避免数据文件写到一半断电：

```bash
# 三台都执行
redis-cli -a 'Redis@2026' shutdown 2>/dev/null
redis-cli -p 26379 shutdown 2>/dev/null
redis-cli -a 'Redis@2026' -p 7000 shutdown 2>/dev/null
redis-cli -a 'Redis@2026' -p 7001 shutdown 2>/dev/null
systemctl stop mysqld 2>/dev/null
systemctl stop mysqld2 2>/dev/null

# 确认没有残留进程
ps -ef | grep -E 'redis|mysql' | grep -v grep

# 关机
poweroff
```

### 8.3 开机后怎么恢复（★ 顺序很重要）

```bash
# 1) MySQL（主节点先起）
systemctl start mysqld
systemctl start mysqld2 2>/dev/null        # 只有 node2 有

# 2) Redis（先起主的，再起从的）
#    node2      : redis-server /usr/local/redis/conf/redis.conf
#    node1/node3: redis-server /usr/local/redis/conf/redis.conf

# 3) 等 5 秒，确认主从链路恢复
redis-cli -a 'Redis@2026' INFO replication | head -4
# 期望：master_link_status:up

# 4) 最后起哨兵（三台）
redis-sentinel /usr/local/redis/conf/sentinel.conf
redis-cli -p 26379 info sentinel

# 5) 如果做 Cluster 那一步，还要起 6 个 cluster 实例
redis-server /usr/local/redis-cluster/7000/redis.conf
redis-server /usr/local/redis-cluster/7001/redis.conf
```

> **顺序原因**：哨兵启动时会去连主节点，Redis 还没起的话日志里会先报错（虽然会重试连上，但日志脏、看着吓人）。

---

## 九、遇到问题怎么查（通用排障顺序）

```text
第 1 步  看"我是不是那台机"
         hostname + ip -4 addr          ← 80% 的"莫名其妙"都是连错机器

第 2 步  看"服务在不在"
         systemctl status mysqld  /  ps -ef | grep redis

第 3 步  看"端口在不在听"
         ss -lntp | grep <端口>

第 4 步  看"监听地址对不对"
         0.0.0.0 或 *   → 对外可连
         127.0.0.1      → 只能本机连（远程连不上就是它）

第 5 步  看"日志"
         MySQL  : /var/log/mysqld.log、/var/log/mysql2/error.log
         Redis  : /usr/local/redis/log/redis.log、sentinel.log
         Cluster: /usr/local/redis-cluster/700x/redis.log

第 6 步  看"网络通不通"
         ping <IP>                  → 探测主机
         nc -vz <IP> <端口>          → 探测端口

第 7 步  看"时间对不对"（主从、哨兵都对时间敏感）
         date ; chronyc tracking
```

### 各篇的报错速查表

| 实战 | 报错速查在哪 |
| --- | --- |
| 备份篇 | `stage3-22-实战一-备份篇.md` 第七章 |
| MySQL 主从篇 | `stage3-23-实战二-MySQL主从篇.md` 第五章 |
| Redis 篇 | `stage3-24-实战三-Redis篇.md` 第七章 |

---

## 十、notes3 文件索引

### 10.1 实战四篇（今天要做的）

| 文件 | 内容 |
| --- | --- |
| **`stage3-25-总体规划.md`** | **本文** —— 环境准备 + 节点任务 + 执行路线 |
| `stage3-22-实战一-备份篇.md` | mysqldump + binlog PITR + XtraBackup |
| `stage3-23-实战二-MySQL主从篇.md` | 多实例 + 远程登录 + 主从复制 |
| `stage3-24-实战三-Redis篇.md` | 持久化 + 主从 + 哨兵 + Cluster |

### 10.2 知识点笔记（预习 / 复习用）

| 文件 | 什么时候看 |
| --- | --- |
| `stage3-00-环境规划.md` | 本文的前身，保留作对照 |
| `stage3-01` 简介与安装 | 装 MySQL 前 |
| `stage3-02` 安全加固 / socket与TCP | 做实战二的远程登录前 |
| `stage3-03` 二进制安装MySQL | 想了解另一种装法时（**8.4 的包是 glibc2.28 + .tar.xz**） |
| `stage3-04` DML与DDL | 忘了建表语法时 |
| `stage3-05` 用户授权与撤销 | 建复制账号 / 备份账号前 |
| `stage3-06 / 07 / 08` | 备份篇的知识点版（已适配 8.4） |
| `stage3-09 / 10` | 主从篇的知识点版 |
| `stage3-14 / 15` | Redis 基础与数据类型 |
| `stage3-16 / 17 / 18 / 19` | Redis 篇的知识点版 |
| `stage3-20` MySQL 与 Redis 主从对比 | **Redis 篇做完后看**，把两边串起来 |
| `stage3-21` Redis 深度整合理解 | 同上 |

### 10.3 空文件（后续要补的）

`stage3-11`（读写分离）、`stage3-12`（垂直分库）、`stage3-13`（水平切分）目前是空的 —— 做完主从之后可以继续往这三个方向走。

---

## 十一、开工前最后一遍（30 秒）

```text
□ 三台虚机都起来了，IP 是 192.168.171.147 / 148 / 146
□ 三台主机名分别是 node1 / node2 / node3，且和 IP 对得上
□ 三台互相 ping 通，也能上网（ping www.baidu.com）
□ SELinux 关了、firewalld 关了、chronyd 在跑
□ node2 / node3 装了 MySQL 8.4，node2 装了 XtraBackup 8.4
□ 三台都装了 Redis 7.2.0
□ 三台都打了快照 snap-00-base

全打勾 → 从「实战一 · 备份篇」开始，按第四章的顺序走。
```



复盘：
问题一：mysql8.4安装失败
所有的匹配结果均已经被参数的模块化过滤条件筛除: mysql-community-server
错误：没有任何匹配: mysql-community-server

原因：
RHEL 8 / Rocky 8 引入了"模块化仓库（modularity）"：
· 一个模块 = 一组包的集合 + 版本流（stream）
· 模块可以声明"过滤掉某些包名"，防止和模块内的包冲突
· AppStream 里的 mysql 模块 8.0 覆盖了 mysql-server 这套包，
同时把 mysql-community-server 这类第三方包名也"屏蔽"了

所以：官方源里有包，但 dnf 的模块过滤器在中间拦了一道。

★ 这个问题只存在于 RHEL 8 / Rocky 8 / AlmaLinux 8，
Rocky 9 / RHEL 9 已经移除了模块化，不会有这个报错。

解决方案
# ① 先看一眼模块状态（可选）
dnf module list mysql

# ② 关键一步：禁用这个模块
dnf module disable mysql -y

# ③ 再装
dnf install -y mysql-community-server