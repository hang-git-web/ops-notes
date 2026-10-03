# 实战一 · 备份篇（mysqldump + binlog 时间点恢复 + XtraBackup）

> **本篇整合了**：`stage3-06`（备份与恢复）、`stage3-07`（mysqldump 详解）、`stage3-08`（XtraBackup）
> **环境**：node2 = 192.168.171.148（MySQL 8.4 主库，2 核 4G 40G，Rocky 8.10）
> **本篇目标**：跑完你会得到三样能力 ——
> ① 用 **mysqldump** 备份并**真的恢复一次**验证可用；
> ② 用 **全量 + binlog** 把数据恢复到**误删前的任意时刻**（PITR）；
> ③ 用 **XtraBackup** 做**不停机**的物理热备 + 增量备份，并成功恢复。
> **本文所有命令都按 MySQL 8.4 写**（`SHOW BINARY LOG STATUS`、`--source-data=2` 等），可直接复制。

---

## 〇、开工前的准备

### 0.1 确认环境

在 **node2** 上：

```bash
hostname                       # 期望 node2
mysql --version                # 期望 8.4.x
systemctl is-active mysqld     # 期望 active
df -h /                        # 根分区至少留 10G 给备份
```

**三台机器都确认一遍主机名和 IP 对得上**（最容易出错的地方）：

```bash
echo "我是 $(hostname) ，IP 是 $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')"
# node1 → 192.168.171.147
# node2 → 192.168.171.148
# node3 → 192.168.171.146
```

### 0.2 建一个"用来被备份"的实验库

备份要有东西可备，先造点数据（**这一步很关键，后面误删演练也靠它**）：

```bash
mysql -uroot -p'你的root密码' <<'EOF'
CREATE DATABASE IF NOT EXISTS shop DEFAULT CHARACTER SET utf8mb4;
USE shop;
CREATE TABLE IF NOT EXISTS orders (
  id INT PRIMARY KEY AUTO_INCREMENT,
  customer VARCHAR(50) NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO orders (customer, amount) VALUES
('zhangsan', 199.00),('lisi', 299.50),('wangwu', 88.80);
SELECT COUNT(*) AS '初始行数' FROM orders;
EOF
# 期望：初始行数 = 3
```

### 0.3 建备份目录

```bash
mkdir -p /data/backup/{mysql,binlog,xtrabackup}
chmod 700 /data/backup
ls -ld /data/backup
```

> **为什么专门用 `/data`**：不要备份到 `/var/lib/mysql` 下面（那是数据目录，备份放进去会被下一次备份自己套娃，还可能撑爆根分区）。**备份目录要独立。**

---

## 一、先花 5 分钟搞清备份的分类

| 分类维度 | 类型 | 一句话 | 你用哪个 |
| --- | --- | --- | --- |
| **按内容** | 完全备份 / 部分备份 | 全部库表 / 只备某些表 | 完全备份 |
| **按方式** | **逻辑备份** | 导出成 SQL 文本（`mysqldump`） | ✅ 本篇第一个 |
| | **物理备份** | 直接复制数据文件（XtraBackup） | ✅ 本篇第二个 |
| **按状态** | **热备份** | 服务正常运行，不停机 | ✅ XtraBackup |
| | 温备份 | 可读、限制写 | 少用 |
| | 冷备份 | 停服务再备 | 只用在小库 |

**选型口诀**：

```text
库小（GB 级以内）、要跨版本/跨平台迁移 → mysqldump
库大、要求不停机、要求快速恢复         → XtraBackup
想要"恢复到误删前那一秒"               → 全量 + binlog（PITR）
```

---

## 二、mysqldump 逻辑备份

### 2.1 核心参数（背这几个就够）

| 参数 | 作用 | 备注 |
| --- | --- | --- |
| **`--single-transaction`** | 事务快照保证一致性，**不锁表** | **只对 InnoDB 有效**；InnoDB 场景必加 |
| **`--routines --triggers --events`** | 连存储过程、触发器、事件一起备 | 生产必加，否则恢复后少了对象 |
| **`--default-character-set=utf8mb4`** | 避免中文乱码 | 备份和恢复**都要加** |
| **`--source-data=2`** | 把 binlog 位置以**注释**形式写进备份文件 | **MySQL 8.4 的新名字**（旧名 `--master-data`） |
| `--databases db1 db2` | 备份多个库，文件里带 `CREATE DATABASE` | 恢复时**不用**再指定库名 |
| `--all-databases` | 备份所有库 | 同上 |
| `--no-data` / `--no-create-info` | 只要结构 / 只要数据 | 拆开备份时用 |
| `--ignore-table=库.表` | 排除指定表 | 可重复写多次 |
| `--where="条件"` | 按条件备部分数据 | 对本次备份的所有表生效 |
| `--set-gtid-purged=OFF` | 避免 GTID 信息导致导入报错 | GTID 环境用 |
| `--compress` | 传输时压缩（5.7+） | 一般用管道 gzip 更直观 |

> ⚠️ **一个常见误区**：`--single-transaction` 已经能保证 InnoDB 的一致性，**不要再加 `--skip-lock-tables`**。加了反而可能让库里的 MyISAM 表备份不一致。

### 2.2 四种常用备份命令

```bash
cd /data/backup/mysql
D=$(date +%F)

# ① 单库（最常用）
mysqldump -uroot -p'你的root密码' \
  --single-transaction --routines --triggers --events \
  --default-character-set=utf8mb4 \
  --source-data=2 \
  shop > shop_full_${D}.sql

# ② 多库
mysqldump -uroot -p'你的root密码' --databases shop mysql > dbs_${D}.sql

# ③ 全库
mysqldump -uroot -p'你的root密码' --all-databases \
  --single-transaction --routines --triggers --events \
  --default-character-set=utf8mb4 > all_${D}.sql

# ④ 单表 / 带条件
mysqldump -uroot -p'你的root密码' shop orders > shop_orders_${D}.sql
mysqldump -uroot -p'你的root密码' --where="id < 1000" shop orders > orders_part.sql

# ⑤ 只要结构 / 只要数据
mysqldump -uroot -p'你的root密码' --no-data        shop > shop_schema_${D}.sql
mysqldump -uroot -p'你的root密码' --no-create-info shop > shop_data_${D}.sql
```

**压缩备份**（省空间，推荐）：

```bash
mysqldump -uroot -p'你的root密码' --single-transaction --routines --triggers --events \
  --default-character-set=utf8mb4 shop | gzip > shop_${D}.sql.gz
chmod 600 shop_${D}.sql.gz
ls -lh
```

### 2.3 ★ 恢复验证（不做这步等于没备份）

**恢复单个库**（先建空库）：

```bash
mysql -uroot -p'你的root密码' -e "CREATE DATABASE IF NOT EXISTS shop_verify DEFAULT CHARACTER SET utf8mb4;"
mysql -uroot -p'你的root密码' --default-character-set=utf8mb4 shop_verify < shop_full_${D}.sql
mysql -uroot -p'你的root密码' -e "SELECT COUNT(*) FROM shop_verify.orders;"
# 期望：3
```

**恢复压缩备份**：

```bash
gunzip < shop_${D}.sql.gz | mysql -uroot -p'你的root密码' shop_verify
```

**恢复全库**（`--all-databases` / `--databases` 导出的文件里自带 `CREATE DATABASE`，**不要**再指定库名）：

```bash
mysql -uroot -p'你的root密码' < all_${D}.sql
```

**三层验证方法（从弱到强）**：

```bash
# ① 文件大小是否正常
ls -lh /data/backup/mysql/shop_full_${D}.sql
# ② 建表语句数量对不对
grep -c "CREATE TABLE" /data/backup/mysql/shop_full_${D}.sql      # 期望 1
# ③ ★ 真恢复到测试库，查数据（最可靠）
mysql -uroot -p'你的root密码' -e "SELECT * FROM shop_verify.orders LIMIT 3;"
```

**验证完清掉测试库**：

```bash
mysql -uroot -p'你的root密码' -e "DROP DATABASE shop_verify;"
```

### 2.4 备份文件的安全

```bash
chmod 600 /data/backup/mysql/*.sql*          # 备份含业务数据，限制权限
```

> ⚠️ **别把密码明文写在命令行** —— 会留在 `history` 和 `ps` 里。生产做法是用 `--defaults-extra-file` 配置文件：
> ```bash
> cat >/root/.mybackup.cnf<<'EOF'
> [client]
> user=backup
> password=Backup@2026
> EOF
> chmod 600 /root/.mybackup.cnf
> # 用法：
> mysqldump --defaults-extra-file=/root/.mybackup.cnf --single-transaction shop > a.sql
> ```

**验证**：

```bash
ls -l /data/backup/mysql/*.sql* | head
# 期望权限是 -rw-------
```

---

## 三、binlog 增量备份与"恢复到误删前"（★ 本篇最实用的一节）

> **原理**：binlog 记录所有数据变更。**全量备份 + 之后的 binlog = 可以恢复到任意时间点**（Point-In-Time Recovery，PITR）。

### 3.1 先确认 binlog 开着

```bash
mysql -uroot -p'你的root密码' -e "SHOW VARIABLES LIKE 'log_bin';"
# 期望 ON
mysql -uroot -p'你的root密码' -e "SHOW VARIABLES LIKE 'server_id';"
# 期望非 0
```

**如果没开**，编辑 `/etc/my.cnf`：

```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
# 日志保留时间（8.0 起用秒级参数；expire_logs_days 已被移除）
binlog_expire_logs_seconds = 604800
max_binlog_size = 512M
```

```bash
systemctl restart mysqld
```

> ⚠️ **两个坑**：
> ① 自定义 binlog 路径（如 `/var/log/mysql/mysql-bin`）时，目录必须存在且属主为 mysql，**并且 SELinux 要放行**（练习环境用默认路径最省事）；
> ② `expire_logs_days` 在 MySQL 8.0 就被移除了，写成它会导致 **mysqld 启动失败**。

**查当前 binlog 位置（8.4 用新命令）**：

```bash
mysql -uroot -p'你的root密码' -e "SHOW BINARY LOG STATUS\G"
# 记下 File 和 Position，例如 mysql-bin.000003 / 157
```

> **8.4 的写法变化**：`SHOW MASTER STATUS` 已改名成 **`SHOW BINARY LOG STATUS`**。

### 3.2 ★ 完整演练：备份 → 写数据 → 误删 → 恢复到误删前

这个演练一定要亲手做一遍，做完你就真懂 PITR 了。

#### 第 1 步：做一次全量备份（带 binlog 位置）

```bash
cd /data/backup/mysql
D=$(date +%F)
mysqldump -uroot -p'你的root密码' \
  --single-transaction --routines --triggers --events \
  --default-character-set=utf8mb4 \
  --source-data=2 \
  shop > shop_before_${D}.sql

# ★ 确认 binlog 位置被写进文件了（这是后面恢复的锚点）
grep -n "CHANGE REPLICATION SOURCE TO" shop_before_${D}.sql | head -3
# 期望看到类似：
# -- CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='mysql-bin.000003', SOURCE_LOG_POS=157;
```

> **把这两个值抄下来**（示例，你要换成自己的）：`mysql-bin.000003` 和 `157`

#### 第 2 步：备份之后再写入新数据（模拟正常业务）

```bash
mysql -uroot -p'你的root密码' <<'EOF'
USE shop;
INSERT INTO orders (customer, amount) VALUES ('zhaoliu', 666.00), ('sunqi', 777.00);
SELECT COUNT(*) AS '现在有几行' FROM orders;      -- 期望 5
EOF

# 记录一下当前时间（后面按时间点恢复要用）
date '+%F %T' | tee /tmp/insert_time.txt
```

#### 第 3 步：模拟"误删"（这就是事故现场）

```bash
mysql -uroot -p'你的root密码' -e "USE shop; DROP TABLE orders;"
mysql -uroot -p'你的root密码' -e "USE shop; SHOW TABLES;"
# 期望：表没了
```

#### 第 4 步：恢复 —— 先恢复全量，再用 binlog 补增量

```bash
# ① 先用刚才的全量备份把库恢复回来（此时只有备份那一刻的 3 行数据）
mysql -uroot -p'你的root密码' -e "DROP DATABASE IF EXISTS shop; CREATE DATABASE shop DEFAULT CHARACTER SET utf8mb4;"
mysql -uroot -p'你的root密码' --default-character-set=utf8mb4 shop < shop_before_${D}.sql
mysql -uroot -p'你的root密码' -e "SELECT COUNT(*) FROM shop.orders;"
# 期望：3    ← 全量恢复到了备份时刻

# ② 看 binlog 里都有哪些文件
ls -lh /var/lib/mysql/mysql-bin.*

# ③ 用 mysqlbinlog 把"备份之后、误删之前"的变更重放一遍
mysqlbinlog --start-position=157 \
            --stop-datetime="$(cat /tmp/insert_time.txt)" \
            /var/lib/mysql/mysql-bin.000003 | mysql -uroot -p'你的root密码'

# ④ 验证数据回来了
mysql -uroot -p'你的root密码' -e "SELECT COUNT(*) AS '恢复后行数' FROM shop.orders;"
# 期望：5    ← 误删前的两条数据被 binlog 补回来了
```

> **★ 这就是 PITR 的价值**：全量备份只能回到"备份那一刻"，**binlog 才能带你回到"事故前那一秒"**。

> **更严谨的定位方式**（`--stop-datetime` 有时不够精确）：
> ```bash
> # 先看 binlog 里 DELETE/DROP 出现在什么时间、什么位点
> mysqlbinlog --base64-output=DECODE-ROWS -v /var/lib/mysql/mysql-bin.000003 \
>   | grep -n -i -E "DROP|DELETE" | head
> # 然后用 --stop-position 精确停在误删那条之前
> mysqlbinlog --start-position=157 --stop-position=<误删前的位点> /var/lib/mysql/mysql-bin.000003 | mysql -uroot -p
> ```

### 3.3 mysqlbinlog 常用参数

| 参数 | 作用 |
| --- | --- |
| `--start-datetime` / `--stop-datetime` | 按**时间**范围恢复 |
| `--start-position` / `--stop-position` | 按**位点**精确恢复（最准） |
| `--database=shop` | 只恢复指定库的变更 |
| `--skip-gtids` | GTID 环境下避免重复执行报错 |
| `--base64-output=DECODE-ROWS -v` | 把 ROW 格式日志**解码成可读 SQL**（只用于查看，不要拿去执行） |

**查看 binlog 里到底改了什么（排错神器）**：

```bash
mysqlbinlog --base64-output=DECODE-ROWS -v \
  --start-datetime="2026-10-03 10:00:00" /var/lib/mysql/mysql-bin.000003 | less
```

> ⚠️ **binlog 必须按文件顺序恢复**，跳着加会数据错乱。恢复前先把"从哪个文件、哪个位点开始"写清楚。

---

## 四、XtraBackup 物理热备

### 4.1 版本必须严格匹配

| MySQL | XtraBackup | 包名 |
| --- | --- | --- |
| 5.7 | 2.4 | `percona-xtrabackup-24` |
| 8.0 | 8.0 | `percona-xtrabackup-80` |
| **8.4（你的环境）** | **8.4** | **`percona-xtrabackup-84`** |

> **错配会直接报错**，这是 XtraBackup 最常见的坑。另外 **XtraBackup 不支持 MariaDB**（MariaDB 要用 `mariabackup`）。

### 4.2 安装（node2）

```bash
dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm

# 启用 8.4 的仓库
percona-release enable-only pxb-84-lts

dnf makecache
dnf list available 'percona-xtrabackup*'     # ★ 先看能装到哪个版本
dnf install -y percona-xtrabackup-84
xtrabackup --version                          # 期望 8.4.x
```

> **如果 `dnf list` 里找不到 `percona-xtrabackup-84`**，说明仓库名不对。Percona 对不同版本用了不同的 repo 名，依次试：
> ```bash
> percona-release enable-only tools release && dnf makecache && dnf list available 'percona-xtrabackup*'
> percona-release enable pxb-84-lts && dnf makecache && dnf list available 'percona-xtrabackup*'
> ```

### 4.3 建专用备份账号（最小权限）

```bash
mysql -uroot -p'你的root密码' <<'EOF'
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'Backup@2026';
GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT, BACKUP_ADMIN
  ON *.* TO 'backup'@'localhost';
FLUSH PRIVILEGES;
SELECT user, host, plugin FROM mysql.user WHERE user='backup';
EOF
```

| 权限 | 为什么需要 |
| --- | --- |
| `RELOAD` | 刷表状态、执行 FLUSH |
| `PROCESS` | 看服务器进程 |
| `LOCK TABLES` | MyISAM 表短暂锁表 |
| `REPLICATION CLIENT` | 读 binlog 位置（用于 PITR） |
| `BACKUP_ADMIN` | **MySQL 8.0+ 备份必需**（执行 `LOCK INSTANCE FOR BACKUP`） |

> MySQL 8.4 默认认证插件是 `caching_sha2_password`，XtraBackup 8.4 支持它，**不用改成 mysql_native_password**。

```bash
mkdir -p /data/backup/xtrabackup && chown -R mysql:mysql /data/backup/xtrabackup
```

### 4.4 全量备份 → prepare → 恢复（完整闭环）

```bash
# ① 备份（--target-dir 指向的目录必须不存在或为空！）
xtrabackup --backup \
  --user=backup --password='Backup@2026' \
  --target-dir=/data/backup/xtrabackup/full

# ② 看备份产生了什么
ls /data/backup/xtrabackup/full
cat /data/backup/xtrabackup/full/xtrabackup_checkpoints      # backup_type / to_lsn
cat /data/backup/xtrabackup/full/xtrabackup_binlog_info      # ★ binlog 位置，PITR 要用

# ③ prepare：应用 redo 日志，把备份变成"一致状态"（这一步必须做）
xtrabackup --prepare --target-dir=/data/backup/xtrabackup/full
```

**恢复（会覆盖数据目录，务必按顺序）**：

```bash
# 1) 停 MySQL
systemctl stop mysqld

# 2) ★ 先把原数据目录改名保留（不要 rm！出错能回退）
mv /var/lib/mysql /var/lib/mysql.bak.$(date +%F_%H%M)

# 3) 把备份拷回去（目标目录必须为空）
xtrabackup --copy-back \
  --target-dir=/data/backup/xtrabackup/full \
  --datadir=/var/lib/mysql

# 4) 修属主
chown -R mysql:mysql /var/lib/mysql
# restorecon -Rv /var/lib/mysql      # SELinux 开启时才需要

# 5) 启动并验证
systemctl start mysqld
mysql -uroot -p'你的root密码' -e "SELECT COUNT(*) FROM shop.orders;"
# 期望：3（备份那一刻的数据）
```

> **★ 确认数据没问题再删 `.bak` 目录**，留几天都行。

### 4.5 增量备份与合并恢复

```bash
# 先确认 apply-log-only 这个参数在你这版 8.4 里还在
xtrabackup --help 2>&1 | grep -i apply-log

# ① 第一次增量（基于全量）
xtrabackup --backup --user=backup --password='Backup@2026' \
  --target-dir=/data/backup/xtrabackup/incr1 \
  --incremental-basedir=/data/backup/xtrabackup/full

# ② 造点新数据，再做第二次增量
mysql -uroot -p'你的root密码' -e "USE shop; INSERT INTO orders(customer,amount) VALUES('incr_test', 1.00);"

xtrabackup --backup --user=backup --password='Backup@2026' \
  --target-dir=/data/backup/xtrabackup/incr2 \
  --incremental-basedir=/data/backup/xtrabackup/incr1

# ③ ★ 检查增量链有没有断开
echo "--- full ---";  cat /data/backup/xtrabackup/full/xtrabackup_checkpoints
echo "--- incr1 ---"; cat /data/backup/xtrabackup/incr1/xtrabackup_checkpoints
echo "--- incr2 ---"; cat /data/backup/xtrabackup/incr2/xtrabackup_checkpoints
# 判断标准：上一份的 to_lsn == 这一份的 from_lsn，否则链条断裂、无法合并
```

**合并恢复**：

```bash
# ① 准备全量：--apply-log-only 表示"只应用 redo、不回滚未提交事务"，这样才能继续往上合并
xtrabackup --prepare --apply-log-only --target-dir=/data/backup/xtrabackup/full

# ② 依次合并每个增量（中间的都要带 --apply-log-only）
xtrabackup --prepare --apply-log-only --target-dir=/data/backup/xtrabackup/full \
  --incremental-dir=/data/backup/xtrabackup/incr1

xtrabackup --prepare --apply-log-only --target-dir=/data/backup/xtrabackup/full \
  --incremental-dir=/data/backup/xtrabackup/incr2

# ③ ★ 最后一个增量合并完，必须再做一次【不带 --apply-log-only】的 prepare，完成事务回滚
xtrabackup --prepare --target-dir=/data/backup/xtrabackup/full

# ④ 停服务、挪走旧数据目录、拷回、改属主、启动（同 4.4）
systemctl stop mysqld
mv /var/lib/mysql /var/lib/mysql.bak2.$(date +%F_%H%M)
xtrabackup --copy-back --target-dir=/data/backup/xtrabackup/full --datadir=/var/lib/mysql
chown -R mysql:mysql /var/lib/mysql
systemctl start mysqld

# ⑤ 验证：应该能看到 incr_test 那条数据
mysql -uroot -p'你的root密码' -e "SELECT * FROM shop.orders;"
```

> ⚠️ **如果 8.4 报 `unknown option '--apply-log-only'`**：以 `xtrabackup --help | grep -i apply-log` 的输出为准；实在没有就先把**全量恢复**（4.4）跑通，增量部分对着官方文档 `docs.percona.com/xtrabackup` 核对参数。

### 4.6 XtraBackup 参数速查

| 参数 | 作用 |
| --- | --- |
| `--backup` | 执行备份 |
| `--prepare` | 应用 redo，生成可恢复的一致备份 |
| `--apply-log-only` | 只应用 redo 不回滚，用于合并增量 |
| `--copy-back` | 用备份覆盖数据目录（要求目录为空） |
| `--move-back` | 移动方式恢复，不额外占空间 |
| `--incremental-basedir` | 增量备份的基准目录 |
| `--databases` / `--tables` | 部分备份 |
| `--parallel=N` | 并行复制文件，提速 |
| `--stream=tar` | 流式输出，可配合 ssh 传到远程 |
| `--defaults-file=` | 指定配置文件，**必须写在命令最前面** |

> **XtraBackup 8.0 起已移除 `innobackupex`**，统一用 `xtrabackup`。

---

## 五、把备份变成自动任务

### 5.1 每天全量备份脚本

```bash
cat >/shell/mysql_backup.sh<<'EOF'
#!/bin/bash
# mysqldump 每日全量备份，保留 7 天
set -uo pipefail

BACKUP_DIR=/data/backup/mysql
KEEP_DAYS=7
log() { echo "[$(date '+%F %T')] $*"; }

mkdir -p "$BACKUP_DIR"
log "开始备份..."

if mysqldump -uroot -p'你的root密码' \
     --single-transaction --routines --triggers --events \
     --default-character-set=utf8mb4 --all-databases \
     | gzip > "$BACKUP_DIR/all_$(date +%F).sql.gz"; then
    log "备份成功：$BACKUP_DIR/all_$(date +%F).sql.gz"
else
    log "【错误】备份失败，请检查！"
    exit 1
fi

# 校验文件非空
if [ -s "$BACKUP_DIR/all_$(date +%F).sql.gz" ]; then
    log "文件大小：$(du -h "$BACKUP_DIR/all_$(date +%F).sql.gz" | cut -f1)"
else
    log "【错误】备份文件为空！"; exit 1
fi

# 清理旧备份
find "$BACKUP_DIR" -name '*.sql.gz' -mtime +$KEEP_DAYS -delete
log "已清理 $KEEP_DAYS 天前的旧备份，当前共 $(ls -1 "$BACKUP_DIR"/*.sql.gz 2>/dev/null | wc -l) 份"
EOF

chmod +x /shell/mysql_backup.sh
bash -n /shell/mysql_backup.sh          # 语法检查
```

### 5.2 挂到定时任务

```bash
crontab -e
# 每天凌晨 2 点跑，日志单独一份
0 2 * * * /shell/mysql_backup.sh >> /var/log/mysql_backup.log 2>&1
```

**验证**：

```bash
bash /shell/mysql_backup.sh              # 手动先跑一次
tail -20 /var/log/mysql_backup.log
ls -lh /data/backup/mysql/
```

> **cron 的两个经典坑**（复习一下）：① 脚本里**必须用绝对路径**（cron 环境变量极简）；② **必须自己重定向输出**（`>> 日志 2>&1`），否则报错你看不到。

---

## 六、闭环自检清单

| # | 检查项 | 命令 | 通过标准 |
| --- | --- | --- | --- |
| 1 | 实验库有数据 | `SELECT COUNT(*) FROM shop.orders;` | 3 行（后面变 5 行） |
| 2 | mysqldump 单库备份成功 | `ls -lh /data/backup/mysql/shop_full_*.sql` | 文件存在且非空 |
| 3 | **恢复到测试库能查到数据** | `SELECT COUNT(*) FROM shop_verify.orders;` | 3 |
| 4 | 备份文件权限收紧 | `ls -l /data/backup/mysql/*.sql*` | `-rw-------` |
| 5 | binlog 已开启 | `SHOW VARIABLES LIKE 'log_bin';` | ON |
| 6 | 备份里记录了 binlog 位置 | `grep -n "CHANGE REPLICATION SOURCE TO" shop_before_*.sql` | 有输出 |
| 7 | **PITR 把误删的数据救回来** | 演练第 4 步 | 恢复后 5 行 |
| 8 | XtraBackup 版本匹配 | `xtrabackup --version` | 8.4.x |
| 9 | 全量备份有 checkpoints | `cat .../full/xtrabackup_checkpoints` | 有 `backup_type` / `to_lsn` |
| 10 | **XtraBackup 恢复后服务能起** | `systemctl start mysqld` | active，数据完整 |
| 11 | 增量链衔接 | 对比 `to_lsn` / `from_lsn` | 上一份 to == 这一份 from |
| 12 | 备份脚本能跑 | `bash /shell/mysql_backup.sh` | 日志显示"备份成功" |
| 13 | 定时任务已挂 | `crontab -l` | 能看到 `0 2 * * *` |

---

## 七、常见报错速查

| 报错 / 现象 | 原因 | 处理 |
| --- | --- | --- |
| `mysqldump: Got error: 1045 Access denied` | 账号密码错 / 权限不足 | 检查密码；备份账号至少要有 `SELECT, LOCK TABLES, SHOW VIEW, TRIGGER, RELOAD, PROCESS, REPLICATION CLIENT` |
| `Unknown option '--master-data'` | 8.4 已改名 | 改用 **`--source-data=2`** |
| `mysqldump: Couldn't execute 'SHOW MASTER STATUS'` | 8.4 改了命令名 | 别手工敲 `SHOW MASTER STATUS`，用 `SHOW BINARY LOG STATUS`；mysqldump 会自动适配 |
| 恢复后中文乱码 | 备份/恢复字符集不一致 | 两边都加 `--default-character-set=utf8mb4` |
| `expire_logs_days` 导致 mysqld 起不来 | 8.0 起已移除 | 改用 `binlog_expire_logs_seconds` |
| `xtrabackup: unrecognized option '--apply-log-only'` | 版本差异 | 先 `xtrabackup --help \| grep -i apply-log` 确认；没有就先用全量恢复 |
| `xtrabackup: Error: target dir ... is not empty` | 备份目录必须为空 | 换个新目录，或先清空 |
| `xtrabackup: error while loading shared libraries` | 依赖缺失 | `dnf install -y libev numactl-libs` |
| XtraBackup 恢复后 MySQL 起不来 | 属主 / SELinux 标签不对 | `chown -R mysql:mysql /var/lib/mysql`（SELinux 开着再加 `restorecon -Rv`） |
| `--copy-back` 报数据目录非空 | 没挪走旧目录 | 先 `mv /var/lib/mysql /var/lib/mysql.bak.x` |
| PITR 后数据多了/少了 | binlog 起止位点选错 | 用 `--base64-output=DECODE-ROWS -v` 先看 binlog 内容，再用 `--stop-position` 精确定位 |
| 磁盘被备份写满 | 没清理旧备份 | `find /data/backup -name '*.sql.gz' -mtime +7 -delete`，并挂监控 |

---

## 八、本篇小结（面试可用的三句话）

```text
① 备份分逻辑（mysqldump）和物理（XtraBackup）两类：
   库小、要跨版本迁移用 mysqldump；库大、要不停机用 XtraBackup。

② mysqldump 的关键参数是 --single-transaction（InnoDB 不锁表）、
   --routines --triggers --events（带上存储过程/触发器/事件）、
   --source-data=2（记录 binlog 位置，8.4 的新名字）。

③ 全量备份只能回到"备份那一刻"，配合 binlog 才能做 PITR，
   恢复到"误删前那一秒"—— 这才是备份真正的价值。

④ 最重要的一句：只备份不演练 = 没有备份。定期把备份恢复一次才算数。
```
