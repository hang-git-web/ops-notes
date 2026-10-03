# 实战二 · MySQL 主从篇（多实例 + 主从复制）

> **本篇整合了**：`stage3-09`（多实例与远程登录）、`stage3-10`（MySQL 主从配置）
> **环境**：
> | 主机 | IP | 角色 |
> | --- | --- | --- |
> | node2 | 192.168.171.148 | **MySQL 主库（3306）** + 多实例（3307） |
> | node3 | 192.168.171.146 | **MySQL 从库** |
> | node1 | 192.168.171.147 | 备份/验证用（本篇可选） |
> **本篇目标**：
> ① 在 **node2 一台机器上跑起两个 MySQL 实例**（3306 + 3307），并会用 socket / TCP 两种方式登录；
> ② 把 **node2 → node3 的主从复制**跑通，并亲手验证"主库写、从库读"；
> ③ 会看复制状态、会处理常见故障、会做一次主从切换演练。
> **全部按 MySQL 8.4 语法写**（`CHANGE REPLICATION SOURCE TO`、`SHOW REPLICA STATUS`），可直接复制。

---

## 〇、开工前的准备

### 0.1 三台先确认同一件事

**三台都跑**（这一步别偷懒，装反了后面全错）：

```bash
echo "我是 $(hostname) ，IP 是 $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')"
```

| 主机 | 必须输出 |
| --- | --- |
| node1 | `我是 node1 ，IP 是 192.168.171.147/24` |
| node2 | `我是 node2 ，IP 是 192.168.171.148/24` |
| node3 | `我是 node3 ，IP 是 192.168.171.146/24` |

### 0.2 三台的通用配置（装完系统就该做）

```bash
# 1) 关 SELinux
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
getenforce

# 2) 关 firewalld
systemctl disable --now firewalld
systemctl is-active firewalld

# 3) 时间同步（主从要求时间一致）
systemctl enable --now chronyd
chronyc sources -v | head -5

# 4) 主机名解析（三台都写）
cat >>/etc/hosts<<'EOF'
192.168.171.147 node1
192.168.171.148 node2
192.168.171.146 node3
EOF
ping -c1 node3

# 5) 装常用工具（最小安装默认没有）
dnf install -y vim wget net-tools lsof bash-completion tar
```

> 期望：`getenforce` 输出 `Permissive`；`firewalld` 输出 `inactive`；`ping node3` 通。

### 0.3 确认 MySQL 8.4 装好（node2、node3）

```bash
mysql --version
systemctl is-active mysqld
mysql -uroot -p'你的root密码' -e "SELECT VERSION();"
```

> **如果还没装**（node2 / node3 都要装）：
> ```bash
> dnf install -y https://dev.mysql.com/get/mysql84-community-release-el8-1.noarch.rpm
> dnf install -y mysql-community-server
> systemctl enable --now mysqld
> grep 'temporary password' /var/log/mysqld.log
> mysql -uroot -p'临时密码' -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'Yuanhang@2026';"
> ```

### 0.4 先确认两台之间网络通

在 node2 上：

```bash
ping -c1 192.168.171.146
nc -vz 192.168.171.146 3306 || telnet 192.168.171.146 3306
```

> **注意**：MySQL 默认 `bind-address` 可能是 `127.0.0.1`（只允许本机连）。主从之间走 TCP，**连不上就先做第一章 1.8 节的远程访问**。

---

## 一、MySQL 多实例（在 node2 上跑 3306 + 3307）

> **什么是多实例**：在一台机器上跑**多个互相独立的 MySQL 服务**。每个实例有自己独立的**数据目录、配置文件、端口号、日志文件、进程**。
>
> **为什么要用**：省钱（一台机器当几台用）、做实验方便（一台机器就能模拟"多台"）、隔离不同业务。

### 1.1 规划（先想清楚再动手）

| 项 | 实例 1（已有的） | 实例 2（要新建的） |
| --- | --- | --- |
| 端口 | 3306 | **3307** |
| 数据目录 | `/var/lib/mysql` | **`/var/lib/mysql2`** |
| 日志目录 | `/var/log/mysqld.log` | **`/var/log/mysql2/`** |
| socket | `/var/lib/mysql/mysql.sock` | **`/var/lib/mysql2/mysql.sock`** |
| 配置文件 | `/etc/my.cnf` | **`/etc/my.cnf.d/mysql2.cnf`** |
| systemd 服务 | `mysqld` | **`mysqld2`** |
| server-id | 1 | **2** |

> **★ 端口、socket 路径、数据目录三者都必须不同**，否则第二个实例会和第一个打架，起不来。

### 1.2 建目录并给权限

```bash
mkdir -p /var/lib/mysql2 /var/log/mysql2
chown -R mysql:mysql /var/lib/mysql2 /var/log/mysql2
ls -ld /var/lib/mysql2 /var/log/mysql2
```

期望：两行都显示属主是 `mysql mysql`。

### 1.3 写第二个实例的配置文件（★ 节名必须是 `[mysqld]`）

```bash
cat >/etc/my.cnf.d/mysql2.cnf<<'EOF'
[mysqld]
port = 3307
datadir = /var/lib/mysql2
log-error = /var/log/mysql2/error.log
socket = /var/lib/mysql2/mysql.sock
server-id = 2
EOF
cat /etc/my.cnf.d/mysql2.cnf
```

> ⚠️ **这一节是必踩的坑**：很多教程把节名写成 `[mysqld2]`，但 **`mysqld` 只读 `[mysqld]` 节**。写成 `[mysqld2]`，里面的 `port`/`datadir` **一条都不生效** —— 第二个实例还会用 3306 和 `/var/lib/mysql`，然后和实例 1 抢端口起不来。
>
> **本笔记统一写 `[mysqld]`**，配 `--defaults-file` 时才能正确读到。

### 1.4 初始化第二个实例

```bash
mysqld --defaults-file=/etc/my.cnf.d/mysql2.cnf --initialize --user=mysql

# 拿初始临时密码
grep 'temporary password' /var/log/mysql2/error.log
```

**立刻验证初始化结果**：

```bash
ls /var/lib/mysql2 | head
# 期望看到 mysql、performance_schema、sys 等，说明初始化成功
```

> 报 `Permission denied` 就回 1.2 查属主；报 `unknown variable` 就回 1.3 查配置。

### 1.5 做成 systemd 服务（★ `LimitNOFILE` 不能有空格）

```bash
cat >/etc/systemd/system/mysqld2.service<<'EOF'
[Unit]
Description=MySQL Server 2
Documentation=man:mysqld(8)
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql
ExecStart=/usr/sbin/mysqld --defaults-file=/etc/my.cnf.d/mysql2.cnf
LimitNOFILE=5000
Restart=on-failure
RestartPreventExitStatus=1
PrivateTmp=false
EOF
```

> ⚠️ **`LimitNOFILE=5000` 两边不能有空格**。写成 `LimitNOFILE = 5000`，systemd 会报 `Unknown lvalue 'LimitNOFILE '`，**这一行被静默忽略**（服务能起，但限制没生效）。

```bash
systemctl daemon-reload
systemctl start mysqld2
systemctl enable mysqld2
systemctl status mysqld2 --no-pager | head -8
```

### 1.6 ★ 启动后必须验证（这一步决定成败）

```bash
# 1) 两个实例都在监听
ss -lntp | grep -E '3306|3307'

# 2) 进程有两个
ps -ef | grep mysqld | grep -v grep

# 3) ★ 确认第二个实例真的用了自己的配置
mysql -uroot -p -P 3307 -S /var/lib/mysql2/mysql.sock -e "SELECT @@port, @@datadir, @@socket;"
# 期望：3307 | /var/lib/mysql2/ | /var/lib/mysql2/mysql.sock
# 如果 port 显示 3306 → 1.3 的 [mysqld] 节没生效，回去改
```

**登录第二个实例改密码**：

```bash
mysql -uroot -p -P 3307 -S /var/lib/mysql2/mysql.sock
# 输入 1.4 拿到的临时密码
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Yuanhang@2026';
EXIT;
```

**再验证一次**：

```bash
mysql -uroot -p'Yuanhang@2026' -P 3307 -S /var/lib/mysql2/mysql.sock -e "SELECT @@port;"
# 期望 3307
```

### 1.7 socket 与 TCP：两种登录方式（★ 面试也常问）

**一句话**：`localhost` 在本机连 MySQL 时，**默认不走网络，而是走一个本机文件（socket）**。

| 你写的命令 | 实际走的路 |
| --- | --- |
| `mysql -uroot -p` | socket（默认路径） |
| `mysql -uroot -p -S /var/lib/mysql2/mysql.sock` | socket（指定路径） |
| `mysql -uroot -p -h localhost` | **socket**（`localhost` 是暗号，不是走 TCP） |
| `mysql -uroot -p -h 127.0.0.1 -P 3307` | **TCP** |

> **★ 关键点：用了 socket 的时候，`-P` 端口参数被忽略** —— 因为根本没走网络。

**最容易踩的坑**：

```bash
# 以为这样连的是第二个实例，其实连的是第一个（默认 socket）
mysql -uroot -p -P 3307

# 要连第二个实例，二选一
mysql -uroot -p -S /var/lib/mysql2/mysql.sock      # 走 socket
mysql -uroot -p -h 127.0.0.1 -P 3307               # 走 TCP
```

**进错实例怎么发现**（登录后立刻看）：

```sql
SELECT @@port, @@socket, @@datadir;
```

或在客户端里敲 `status`：

```text
Connection: Localhost via UNIX socket      ← 走 socket
Connection: 127.0.0.1 via TCP/IP           ← 走 TCP
```

**「Access denied」但密码明明是对的？** 因为账号是「用户名 + 来源」配对的：

| 走 socket | 走 TCP |
| --- | --- |
| 服务器认为来源是 `localhost` | 服务器认为来源是 `127.0.0.1` |
| 受 `'root'@'localhost'` 管 | 受 `'root'@'127.0.0.1'` 管 |

只建了 `root@localhost`，改用 TCP 就登不进去。**需要哪种来源，就建哪种账号。**

### 1.8 配置远程访问（"五关"必须全过）

在 node2 上建一个来源受限的账号：

```bash
mysql -uroot -p'Yuanhang@2026' <<'EOF'
CREATE USER 'repl_check'@'192.168.171.%' IDENTIFIED BY 'Remote@2026';
GRANT SELECT ON *.* TO 'repl_check'@'192.168.171.%';
FLUSH PRIVILEGES;
SELECT user, host FROM mysql.user WHERE user='repl_check';
EOF
```

**放开监听地址**（如果原来是 `127.0.0.1`）：

```bash
grep -n 'bind-address' /etc/my.cnf
sed -i 's/^bind-address.*/bind-address = 0.0.0.0/' /etc/my.cnf
grep 'bind-address' /etc/my.cnf
systemctl restart mysqld
ss -lntp | grep 3306
```

期望：`ss` 显示 `0.0.0.0:3306` 或 `*:3306`（不是 `127.0.0.1:3306`）。

**★ 远程连不上的"五关"，一关一关查**：

| # | 关卡 | 怎么查 |
| --- | --- | --- |
| 1 | **账号**存在且来源匹配 | `SELECT user,host FROM mysql.user;` |
| 2 | **实例在监听对外** | `bind-address` 是不是 `127.0.0.1`；有没有 `skip-networking` |
| 3 | **端口真的在 listen** | `ss -lntp \| grep 3306` |
| 4 | **防火墙 / SELinux** | `systemctl is-active firewalld`、`getenforce` |
| 5 | **网络可达** | 在 node3 上 `ping 192.168.171.148`、`nc -vz 192.168.171.148 3306` |

**最终验证**（**在 node3 上执行**，这才是真验证）：

```bash
mysql -h 192.168.171.148 -P 3306 -u repl_check -p'Remote@2026' -e "SELECT @@hostname, NOW();"
# 能连上并返回 node2 的信息 → 远程访问通了
```

> ⚠️ **安全提醒**：`bind-address = 0.0.0.0` + `'user'@'%'` + `GRANT ALL` 是风险最高的组合。**生产要限定来源 IP、只授业务库权限、不给 `WITH GRANT OPTION`。**

---

## 二、MySQL 主从复制

### 2.1 原理速览（30 秒版）

```text
主库 node2：数据变更 → 写入 binlog
                          ↓
从库 node3：I/O 线程  → 拉取主库 binlog → 写入本地 relay log（中继日志）
                          ↓
从库 node3：SQL 线程  → 重放 relay log  → 与主库保持一致
```

| 角色 | 组件 | 作用 |
| --- | --- | --- |
| 主库 | **binlog** | 记录所有数据变更 |
| 从库 | I/O 线程 | 拉主库 binlog，写 relay log |
| 从库 | relay log | 本地中继日志 |
| 从库 | SQL 线程 | 重放 relay log |

**三种 binlog 格式**：

| 格式 | 记什么 | 特点 |
| --- | --- | --- |
| SBR（statement） | 记 SQL 语句 | 日志小，但 `NOW()`、`UUID()` 这类函数可能导致主从不一致 |
| **RBR（row）** | 记每一行的变更 | **最安全、最常用**（生产推荐） |
| Mixed | 自动选择 | 折中 |

> **★ 复制是异步的**：主库发完 binlog 就回复客户端了，不等从库确认。所以主库突然宕机时，**最后一批还没传过去的写入会丢**。要更强一致就得上**半同步复制**或 **MGR**。

### 2.2 主库配置（node2）

在 `/etc/my.cnf` 的 `[mysqld]` 段加：

```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800
max_binlog_size = 512M
```

```bash
systemctl restart mysqld
mysql -uroot -p'Yuanhang@2026' -e "SHOW VARIABLES LIKE 'server_id'; SHOW VARIABLES LIKE 'log_bin'; SHOW VARIABLES LIKE 'binlog_format';"
```

> ⚠️ **别用 `expire_logs_days`**（8.0 起已移除，写了 mysqld 起不来）。自定义 binlog 路径时目录必须存在且属主为 mysql，**SELinux 还要放行** —— 练习环境用默认路径最省事。

### 2.3 建复制专用账号（node2）

```bash
mysql -uroot -p'Yuanhang@2026' <<'EOF'
CREATE USER 'repl'@'192.168.171.146' IDENTIFIED BY 'Repl@2026';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.171.146';
FLUSH PRIVILEGES;
SELECT user, host, plugin FROM mysql.user WHERE user='repl';
EOF
```

| 要点 | 说明 |
| --- | --- |
| **限定到从库具体 IP** | 不要用 `'repl'@'%'` |
| **只授 `REPLICATION SLAVE`** | 最小权限，不给 `ALL` |
| **认证插件** | MySQL 8.4 默认 `caching_sha2_password`，主从都是 8.4 时**不用改**；只有从库是老版本（5.7）才要处理插件兼容（8.4 的 `mysql_native_password` 默认不加载，需先在 my.cnf 加 `mysql_native_password=ON`） |

**查看主库当前 binlog 位置**（**8.4 用新命令**）：

```bash
mysql -uroot -p'Yuanhang@2026' -e "SHOW BINARY LOG STATUS\G"
```

输出示例：

```text
File: mysql-bin.000001
Position: 747
```

**记下这两个值**，从库配置要用。`SHOW MASTER STATUS` 是旧写法（8.4 已移除），**统一用 `SHOW BINARY LOG STATUS`**。

### 2.4 【主库已有数据时】先做全量初始化（★ 别跳过）

主库已有业务数据时，**必须先全量导一次到从库**，否则从库只有"配置复制之后"的变更，前面的数据是缺的。

```bash
mkdir -p /data/backup
mysqldump -uroot -p'Yuanhang@2026' \
  --single-transaction --routines --triggers --events \
  --default-character-set=utf8mb4 \
  --source-data=2 \
  --all-databases > /data/backup/full_$(date +%F).sql

grep -n "CHANGE REPLICATION SOURCE TO" /data/backup/full_$(date +%F).sql | head -2
scp /data/backup/full_$(date +%F).sql root@192.168.171.146:/root/
```

> 全新的空库可以跳过这一步，直接用 2.3 查到的位置。

### 2.5 从库配置（node3）

在 `/etc/my.cnf` 的 `[mysqld]` 段加：

```ini
[mysqld]
server-id = 2
relay-log = relay-bin
read_only = 1
super_read_only = 1
replica_parallel_workers = 4
replica_parallel_type = LOGICAL_CLOCK
replica_preserve_commit_order = 1
```

```bash
systemctl restart mysqld
```

> **`read_only = 1` 只限制普通用户**，root/SUPER 照样能写。要彻底只读就加 **`super_read_only = 1`**（主从切换时记得关掉）。
>
> `replica_parallel_type = LOGICAL_CLOCK` 在 8.4 已是默认值，写上无妨（用于并行重放，缓解延迟）。

**导入 2.4 的全量数据**（如果做了那一步）：

```bash
mysql -uroot -p'Yuanhang@2026' < /root/full_$(date +%F).sql
mysql -uroot -p'Yuanhang@2026' -e "SELECT COUNT(*) FROM shop.orders;"
```

### 2.6 建立复制关系（★ 8.4 的新语法）

**在 node3 上执行**：

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST        = '192.168.171.148',
  SOURCE_USER        = 'repl',
  SOURCE_PASSWORD    = 'Repl@2026',
  SOURCE_PORT        = 3306,
  SOURCE_LOG_FILE    = 'mysql-bin.000001',
  SOURCE_LOG_POS     = 747,
  GET_SOURCE_PUBLIC_KEY = 1,
  SOURCE_SSL_MODE    = DISABLED;

START REPLICA;
```

### 2.7 ★ 验证复制状态（重点看这五项）

在 **node3** 上：

```bash
mysql -uroot -p'Yuanhang@2026' -e "SHOW REPLICA STATUS\G" | grep -E "Replica_IO_Running|Replica_SQL_Running|Seconds_Behind_Source|Last_IO_Error|Last_SQL_Error"
```

| 指标 | 期望值 | 说明 |
| --- | --- | --- |
| **`Replica_IO_Running`** | **Yes** | I/O 线程正常拉主库 binlog |
| **`Replica_SQL_Running`** | **Yes** | SQL 线程正常重放 |
| **`Seconds_Behind_Source`** | **0** 或接近 0 | 复制延迟秒数 |
| **`Last_IO_Error`** | **空** | I/O 线程错误 |
| **`Last_SQL_Error`** | **空** | SQL 线程错误 |

看完整状态：

```bash
mysql -uroot -p'Yuanhang@2026' -e "SHOW REPLICA STATUS\G" | less
```

> **8.4 用 `SHOW REPLICA STATUS`**；旧版字段名是 `Slave_IO_Running` / `Slave_SQL_Running`。
>
> **★ 判断"复制是否健康"的正确标准**：两个 `Yes` **并且** 两个 Error 都是空。只看两个 Yes 是不够的 —— 有可能刚恢复过一次错误。

### 2.8 ★ 端到端验证（主库写、从库读）

**在 node2（主库）上写入**：

```bash
mysql -uroot -p'Yuanhang@2026' <<'EOF'
CREATE DATABASE IF NOT EXISTS repl_test DEFAULT CHARACTER SET utf8mb4;
USE repl_test;
CREATE TABLE IF NOT EXISTS t1 (id INT PRIMARY KEY, name VARCHAR(20));
INSERT INTO t1 VALUES (1, 'hello replication');
SELECT * FROM t1;
EOF
```

**在 node3（从库）上读**：

```bash
mysql -uroot -p'Yuanhang@2026' <<'EOF'
USE repl_test;
SELECT * FROM t1;
SELECT @@server_id;
SELECT @@read_only, @@super_read_only;
EOF
```

| 检查 | 期望 |
| --- | --- |
| `SELECT * FROM t1;` | **能看到 id=1、name=hello replication** ← 这就是同步成功 |
| `SELECT @@server_id;` | `2`（确认是 node3 自己） |
| `SELECT @@read_only;` | `1`（从库只读） |

**再从库试着写一下（应该被拒绝）**：

```bash
mysql -uroot -p'Yuanhang@2026' -e "USE repl_test; INSERT INTO t1 VALUES(2,'should fail');"
# 期望报错：ERROR 1290 (HY000): The MySQL server is running with the --read-only option
```

> **★ 到这一步，主从复制就算真跑通了**：主库写 → 从库能读到 → 从库不能写。

---

## 三、日常维护与故障演练

### 3.1 看复制延迟

```bash
# 简单看延迟秒数
mysql -uroot -p'Yuanhang@2026' -e "SHOW REPLICA STATUS\G" | grep Seconds_Behind_Source

# 更准确：用 pt-heartbeat（Percona Toolkit）
# 有兴趣可以装：dnf install -y percona-toolkit
# 然后：pt-heartbeat --host=192.168.171.148 --user=root --password=xxx --create-table --update --daemonize
```

> **`Seconds_Behind_Source` 有时不准**（大事务、长时间不写入都会失真），生产上更信 `pt-heartbeat`。

**延迟高了怎么办**：

| 方向 | 做法 |
| --- | --- |
| 复制线程 | `replica_parallel_workers` + `LOGICAL_CLOCK` 开启并行重放 |
| 事务粒度 | 避免大批量更新（单事务越大，从库重放越慢） |
| 日志格式 | 用 ROW，减少不确定性 |
| 从库负载 | 只读查询压力大就加从库或升级硬件 |
| 监控 | 监控 `Seconds_Behind_Source` 和 IO/SQL 线程状态，提前告警 |

### 3.2 常用维护命令

```sql
-- 看状态
SHOW REPLICA STATUS\G;

-- 停止 / 启动复制
STOP REPLICA;
START REPLICA;

-- 重置主从关系（清空复制信息）
STOP REPLICA;
RESET REPLICA ALL;
```

> ⚠️ **旧语法 `STOP SLAVE` / `RESET SLAVE ALL` 在 8.4 已移除**，用 `REPLICA`。

### 3.3 ★ 故障演练：模拟主库宕机 + 主从切换

**第 1 步：模拟主库故障**

```bash
# 在 node2 上停掉 MySQL
systemctl stop mysqld
```

**第 2 步：在从库 node3 上确认状态**

```bash
mysql -uroot -p'Yuanhang@2026' -e "SHOW REPLICA STATUS\G" | grep -E "Replica_IO_Running|Last_IO_Error" | head -4
# 期望：Replica_IO_Running: Connecting 或 No，并且 Last_IO_Error 提示连不上主库
```

**第 3 步：把从库提升为主库（切换）**

```sql
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;
```

```bash
# 验证它现在可写了
mysql -uroot -p'Yuanhang@2026' -e "SELECT @@read_only, @@super_read_only;"
# 期望两个都是 0

mysql -uroot -p'Yuanhang@2026' -e "USE repl_test; INSERT INTO t1 VALUES(3,'written on new master'); SELECT * FROM t1;"
# 期望能写进去
```

> **★ 注意**：这一步只是"手动提升"。**真正的自动故障切换需要 MHA / Orchestrator / MGR** —— MySQL 主从本身**不自带**自动切换（这点和 Redis 的 Sentinel 不一样）。

**第 4 步：恢复原主库，并让它变成新主的从库**

```bash
# node2 重新启动
systemctl start mysqld
```

在 node2 上把它配置成 node3 的从库（步骤和 2.4-2.6 一样，方向反过来）：

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '192.168.171.146',
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'Repl@2026',
  SOURCE_AUTO_POSITION = 1;
START REPLICA;
```

> **这个演练做完记得恢复原状**（把 node2 重新当主、node3 当从），否则后面的实验方向会反。

### 3.4 跳过错误（⚠️ 谨慎使用）

```sql
STOP REPLICA;
SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;   -- 旧名：SQL_SLAVE_SKIP_COUNTER
START REPLICA;
```

> ⚠️ **跳过错误只是让复制"继续跑"，数据不一致已经产生了。** GTID 模式下这个参数无效，要按 GTID 单独处理。**生产上能重建从库就不要跳。**

### 3.5 常见故障对照

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 连接失败（2003） | 网络不通 / 防火墙没放行 / 账号来源不匹配 | 查 3306、`bind-address`、账号是否限定 IP |
| 认证失败 | 密码错 / 认证插件不兼容 | **8.4 首选 `GET_SOURCE_PUBLIC_KEY=1`**；老客户端才考虑启用 `mysql_native_password` |
| `Last_IO_Error: 1236` | 指定的 binlog 文件不存在（已被清理） | 重新全量导出 + 重新配置位置 |
| 主键冲突（1062） | 从库被写过数据 / 数据不一致 | 检查 `read_only`，校验数据后重建从库 |
| 同步延迟过高 | 大事务、慢查询、单线程重放、从库资源不足 | 开并行复制、拆大事务、优化慢查询 |
| 主从数据不一致 | 误操作、跳错、非确定性语句 | 用 `pt-table-checksum` 校验并修复 |
| SQL 线程停止 | 表不存在、字段不匹配 | 先看 `Last_SQL_Error`，修复后 `START REPLICA` |
| 从库能写 | 忘了 `read_only` | `SET GLOBAL read_only=1; SET GLOBAL super_read_only=1;` |

---

## 四、闭环自检清单

### 4.1 多实例部分（node2）

| # | 检查项 | 命令 | 通过标准 |
| --- | --- | --- | --- |
| 1 | 两个实例都在监听 | `ss -lntp \| grep -E '3306\|3307'` | 两个端口都有 |
| 2 | 两个 mysqld 进程 | `ps -ef \| grep mysqld \| grep -v grep` | 两行 |
| 3 | **第二个实例的配置真的生效** | `SELECT @@port, @@datadir, @@socket;` | `3307 / /var/lib/mysql2/ / ...` |
| 4 | 两个实例密码都能登 | `mysql -uroot -p -P 3307 -S /var/lib/mysql2/mysql.sock` | 能进 |
| 5 | **没进错实例** | 登录后 `SELECT @@port;` | 和你要连的一致 |
| 6 | 服务开机自启 | `systemctl is-enabled mysqld mysqld2` | 都是 enabled |
| 7 | 远程能连 | 在 node3 上 `mysql -h 192.168.171.148 -u repl_check -p` | 能连 |

### 4.2 主从部分

| # | 检查项 | 命令（在哪台） | 通过标准 |
| --- | --- | --- | --- |
| 8 | 主库 binlog 已开 | node2：`SHOW VARIABLES LIKE 'log_bin';` | ON |
| 9 | 主库 server-id | node2：`SELECT @@server_id;` | 1 |
| 10 | 从库 server-id 不同 | node3：`SELECT @@server_id;` | 2 |
| 11 | 复制账号存在 | node2：`SELECT user,host FROM mysql.user WHERE user='repl';` | `repl \| 192.168.171.146` |
| 12 | **IO 线程** | node3：`SHOW REPLICA STATUS\G` | `Replica_IO_Running: Yes` |
| 13 | **SQL 线程** | node3：同上 | `Replica_SQL_Running: Yes` |
| 14 | **没有错误** | node3：同上 | `Last_IO_Error` / `Last_SQL_Error` 都为空 |
| 15 | **延迟正常** | node3：同上 | `Seconds_Behind_Source: 0` |
| 16 | 从库只读 | node3：`SELECT @@read_only;` | 1 |
| 17 | **端到端同步** | node2 写 → node3 读 | node3 能查到主库刚写的数据 |
| 18 | 从库写被拒 | node3 尝试 INSERT | 报 `--read-only` 错误 |
| 19 | 会做切换演练 | 3.3 节 | 能从库提升为主并写入 |

---

## 五、常见报错速查

| 报错 | 原因 | 处理 |
| --- | --- | --- |
| `Job for mysqld2.service failed` | 配置节名写成 `[mysqld2]`、端口冲突、目录权限不对 | 看 `/var/log/mysql2/error.log`；确认 1.3 节名是 `[mysqld]` |
| `Unknown lvalue 'LimitNOFILE '` | unit 文件里 `=` 两边有空格 | 改成 `LimitNOFILE=5000` |
| `mysqld: Can't create/write to file` | 数据目录属主不对 | `chown -R mysql:mysql /var/lib/mysql2` |
| 第二个实例起不来且端口是 3306 | `[mysqld2]` 节没被读 | 改成 `[mysqld]`，重启 |
| `ERROR 2002: Can't connect ... socket` | socket 路径写错 / 实例没起 | 核对 `-S` 路径，`systemctl status mysqld2` |
| `ERROR 1045: Access denied for user 'root'@'127.0.0.1'` | 走 TCP 但只建了 `root@localhost` | 建 `root@127.0.0.1`，或者用 socket 连 |
| `SHOW MASTER STATUS` 报语法/弃用 | 8.4 改名 | 用 `SHOW BINARY LOG STATUS` |
| `CHANGE MASTER TO` 报错 | 8.4 已移除旧语法 | 改用 `CHANGE REPLICATION SOURCE TO` |
| `ERROR 2061: Authentication plugin 'caching_sha2_password' reported error` | 少写 `GET_SOURCE_PUBLIC_KEY=1` | 在 `CHANGE REPLICATION SOURCE TO` 里加上它 |
| `Last_IO_Error: error connecting to master ... 2003` | 网络/端口/监听地址 | node3 上 `nc -vz 192.168.171.148 3306`；检查 `bind-address` |
| `Last_IO_Error: 1236` | binlog 文件已被清理 | 重新全量导出 + 重新配置位置（或用 GTID 模式） |
| `Last_SQL_Error: 1062 Duplicate entry` | 从库被写过数据 | 检查 `read_only`，用 `pt-table-checksum` 校验，必要时重建从库 |
| 复制延迟一直涨 | 大事务 / 单线程重放 | 开 `replica_parallel_workers`，拆大事务 |
| `expire_logs_days` 导致 mysqld 起不来 | 8.0 起已移除 | 改用 `binlog_expire_logs_seconds` |

---

## 六、本篇小结（面试可用的四句话）

```text
① 多实例的关键是"三独立"：独立端口、独立数据目录、独立 socket；
   配置节名必须是 [mysqld]（写 [mysqld2] 等于没配）。

② 登录方式分 socket 和 TCP：写 localhost 走 socket（-P 端口被忽略），
   写 IP 走 TCP；账号是"用户名+来源"配对的，只建 root@localhost 用 TCP 连不上。

③ 主从复制的本质是"主库写 binlog → 从库 IO 线程拉 → SQL 线程重放"，
   异步复制，主库宕机可能丢最后一段数据。

④ MySQL 8.4 的命令改了一大批名字：
   SHOW BINARY LOG STATUS / CHANGE REPLICATION SOURCE TO / START REPLICA /
   SHOW REPLICA STATUS，看到 MASTER/SLAVE 就说明是老教程。
```

> **`SOURCE_LOG_FILE` / `SOURCE_LOG_POS` 填 2.3（或 2.4 备份文件里）记下的值。**

| 参数 | 说明 |
| --- | --- |
| `GET_SOURCE_PUBLIC_KEY = 1` | **8.4 必需** —— 默认认证插件是 `caching_sha2_password`，不走 SSL 时要用它取公钥做加密握手 |
| `SOURCE_SSL_MODE = DISABLED` | 内网练习环境关掉 SSL（8.4 的新参数，旧的 `SOURCE_SSL=0` 已弃用） |

> ⚠️ **旧语法（8.4 已移除，别抄）**：`CHANGE MASTER TO ... MASTER_HOST=...` / `START SLAVE;`。看到旧教程要自动翻译成 `SOURCE_` / `REPLICA`。

**GTID 模式（可选，免去手工算位置）** —— 主从两边都加：

```ini
gtid_mode = ON
enforce_gtid_consistency = ON
```

从库只需：

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='192.168.171.148',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='Repl@2026',
  SOURCE_AUTO_POSITION = 1;
START REPLICA;
```
