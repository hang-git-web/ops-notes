# XtraBackup 热备份与恢复笔记

> 介绍:XtraBackup 是 Percona 开源的 MySQL **物理热备**工具,在数据库正常运行的情况下完成备份,无需停机,保证业务连续性。
> 原理:通过复制 InnoDB 的数据文件 + 持续跟踪 redo log,保证备份期间数据的一致性;备份结束前只对非事务引擎(MyISAM)短暂加锁。

## 一、备份策略

| 类型 | 说明 |
| --- | --- |
| 全量备份 | 复制数据目录下的所有表空间文件、日志文件等 |
| 增量备份 | 只备份自上次全量或上次增量以来发生变化的数据 |
| 差异备份 | 备份自上次全量以来所有变化的数据 |

> XtraBackup 没有独立的“差异备份”参数:把增量的 `--incremental-basedir` 每次都指向同一个全量备份目录,得到的就是差异备份效果。

## 二、环境准备

### 1. 版本必须严格匹配

| MySQL 版本 | XtraBackup 版本 | 包名 |
| --- | --- | --- |
| 5.7 | 2.4 | `percona-xtrabackup-24` |
| 8.0 | 8.0 | `percona-xtrabackup-80` |
| 8.4 | 8.4 | `percona-xtrabackup-84` |

注意:

- MySQL 8.0 只能用 XtraBackup 8.0;5.7 只能用 2.4,错配会直接报错
- XtraBackup **不支持 MariaDB**,MariaDB 对应的是 `mariabackup`
- XtraBackup 8.0 **已移除** `innobackupex` 命令,统一改用 `xtrabackup`;2.4 仍可用 `innobackupex`

### 2. 安装(CentOS 示例)

```bash
# 添加 Percona 官方仓库
yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
percona-release enable-only tools release

# MySQL 8.0 环境
yum install -y percona-xtrabackup-80

# MySQL 5.7 环境
# yum install -y percona-xtrabackup-24

# 验证
xtrabackup --version
```

### 3. 创建备份专用用户并授权

```sql
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'Backup@2025';

GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT, BACKUP_ADMIN
  ON *.* TO 'backup'@'localhost';

FLUSH PRIVILEGES;
```

| 权限 | 作用 |
| --- | --- |
| `RELOAD` | 刷新表状态、执行 FLUSH 操作 |
| `PROCESS` | 查看服务器进程 |
| `LOCK TABLES` | MyISAM 表短暂锁表 |
| `REPLICATION CLIENT` | 读取二进制日志位置 |
| `BACKUP_ADMIN` | MySQL 8.0 备份所需(执行 `LOCK INSTANCE FOR BACKUP`) |

### 4. 准备目录

```bash
mkdir -p /data/backup
chown -R mysql:mysql /data/backup
```

## 三、全量备份与恢复

### 1. 执行备份

```bash
# MySQL 8.0(XtraBackup 8.0)
xtrabackup --backup \
  --user=backup --password='Backup@2025' \
  --target-dir=/data/backup/full

# MySQL 5.7(XtraBackup 2.4,传统写法)
# innobackupex --user=backup --password='Backup@2025' \
#   --no-timestamp /data/backup/full
```

参数说明:

| 参数 | 说明 |
| --- | --- |
| `--target-dir` / 目录参数 | 指定备份目录,**目录必须不存在或为空** |
| `--no-timestamp` | 不自动生成时间戳子目录,自定义备份路径(2.4) |
| `--defaults-file=/etc/my.cnf` | 指定配置文件,必须放在命令第一位 |

### 2. 检查备份

```bash
ls /data/backup/full

# 关键文件:
#   ibdata1                 系统表空间
#   xtrabackup_checkpoints  LSN 与备份类型
#   xtrabackup_info         备份信息
#   xtrabackup_binlog_info  binlog 位置(用于按时间点恢复)
#   xtrabackup_logfile      备份期间产生的 redo 日志

cat /data/backup/full/xtrabackup_checkpoints
cat /data/backup/full/xtrabackup_binlog_info
```

### 3. 恢复全量备份

```bash
# 1. 准备(应用 redo 日志,使数据一致)
xtrabackup --prepare --target-dir=/data/backup/full
# 2.4 写法:innobackupex --apply-log /data/backup/full

# 2. 停止 MySQL
systemctl stop mysqld

# 3. 移走原数据目录(不要直接 rm,便于回退)
mv /var/lib/mysql /var/lib/mysql.bak.$(date +%F)

# 4. 复制备份文件回数据目录(目录需为空)
xtrabackup --copy-back --target-dir=/data/backup/full --datadir=/var/lib/mysql
# 2.4 写法:innobackupex --copy-back /data/backup/full

# 5. 修复属主与 SELinux 标签
chown -R mysql:mysql /var/lib/mysql
restorecon -Rv /var/lib/mysql

# 6. 启动 MySQL 并确认
systemctl start mysqld
mysql -uroot -p -e "SHOW DATABASES;"
```

## 四、增量备份与恢复

### 1. 执行增量备份

```bash
# 基于全量备份做第一次增量
xtrabackup --backup \
  --user=backup --password='Backup@2025' \
  --target-dir=/data/backup/incr1 \
  --incremental-basedir=/data/backup/full

# 再做第二次增量,基于上一次增量(链式)
xtrabackup --backup \
  --user=backup --password='Backup@2025' \
  --target-dir=/data/backup/incr2 \
  --incremental-basedir=/data/backup/incr1

# 2.4 写法:innobackupex --incremental /data/backup/incr1 \
#            --incremental-basedir=/data/backup/full
```

| 参数 | 说明 |
| --- | --- |
| `--incremental` | 指定增量备份目录(2.4) |
| `--incremental-basedir` | 指向上一份备份目录(全量或上次增量) |

### 2. 查看 LSN(增量链是否衔接)

```bash
cat /data/backup/full/xtrabackup_checkpoints
cat /data/backup/incr1/xtrabackup_checkpoints
```

输出示例:

```text
backup_type = incremental
from_lsn = 12345678
to_lsn   = 12345999
```

判断标准:**上一份备份的 `to_lsn` 必须等于这一份增量的 `from_lsn`**,否则增量链断裂,无法合并。

### 3. 恢复增量备份

```bash
# 1. 准备全量:--apply-log-only 只应用 redo、不回滚未提交事务,便于继续合并增量
xtrabackup --prepare --apply-log-only --target-dir=/data/backup/full
# 2.4:innobackupex --apply-log --redo-only /data/backup/full

# 2. 合并每一次增量(中间的所有增量都加 apply-log-only)
xtrabackup --prepare --apply-log-only --target-dir=/data/backup/full \
  --incremental-dir=/data/backup/incr1
# 2.4:innobackupex --apply-log --redo-only /data/backup/full \
#        --incremental-dir=/data/backup/incr1

# 3. 最后一个增量合并后,再执行一次完整的 prepare(不加 apply-log-only)
xtrabackup --prepare --target-dir=/data/backup/full
# 2.4:innobackupex --apply-log /data/backup/full

# 4. 停止服务、移走旧数据目录、复制回数据(同全量的恢复流程)
systemctl stop mysqld
mv /var/lib/mysql /var/lib/mysql.bak.$(date +%F)
xtrabackup --copy-back --target-dir=/data/backup/full --datadir=/var/lib/mysql
chown -R mysql:mysql /var/lib/mysql
restorecon -Rv /var/lib/mysql
systemctl start mysqld
```

> `--apply-log-only`(2.4 为 `--redo-only`)的含义:只应用 redo 日志、**不回滚未提交事务**,这样才能继续往上合并增量;链式合并完成后必须再做一次完整 `--prepare`,完成回滚,备份才可用。

## 五、部分备份(指定库或表)

```bash
# 备份指定数据库
xtrabackup --backup --user=backup --password='Backup@2025' \
  --databases="mydb" --target-dir=/data/backup/partial_mydb

# 备份匹配的表(8.0 用 --tables 正则;2.4 用 --include)
xtrabackup --backup --user=backup --password='Backup@2025' \
  --tables='^mydb[.]order_.*' --target-dir=/data/backup/partial_tables

# 2.4 写法:innobackupex --include='mydb.order_.*' /data/backup/partial_tables
```

注意:

- `--databases` 不会自动备份 `mysql` 库,需要时显式加上
- 部分备份恢复需要先 `--prepare --export` 生成表结构,再用 `ALTER TABLE ... IMPORT TABLESPACE` 导入,步骤比全量恢复复杂
- 部分备份属于物理级操作,建议在测试环境演练通过后再用于生产

## 六、常用参数速查

| 参数 | 作用 |
| --- | --- |
| `--backup` | 执行备份 |
| `--prepare` | 应用 redo,生成可恢复的一致性备份 |
| `--apply-log-only` | 只应用 redo 不回滚,用于合并增量(2.4 为 `--redo-only`) |
| `--copy-back` | 用备份覆盖数据目录(要求目录为空) |
| `--move-back` | 移动方式恢复,不额外占用空间 |
| `--incremental-basedir` | 增量备份的基准目录 |
| `--databases` / `--tables` | 部分备份 |
| `--compress` / `--compress-threads` | 压缩备份(8.0 用 `--decompress` 解压) |
| `--parallel` | 并行复制文件,提升速度 |
| `--stream=tar` | 流式输出,可配合 ssh 传到远程 |
| `--defaults-file` | 指定配置文件,必须写在命令最前面 |

压缩 + 解压示例:

```bash
xtrabackup --backup --user=backup --password='Backup@2025' \
  --compress --compress-threads=4 --target-dir=/data/backup/full_c

xtrabackup --decompress --target-dir=/data/backup/full_c
xtrabackup --prepare   --target-dir=/data/backup/full_c
```

## 七、注意事项与常见坑

1. **版本必须匹配**:MySQL 8.0 ↔ XtraBackup 8.0,MySQL 5.7 ↔ XtraBackup 2.4,错配直接失败
2. **8.0 没有 `innobackupex`**:该命令在 XtraBackup 8.0 中已被移除,统一使用 `xtrabackup`
3. **没有 `--verify` 参数**:正确做法是用 `--prepare` 验证一致性,再把备份恢复到一个测试实例做真实校验
4. **恢复前不要 `rm -rf` 数据目录**:先用 `mv` 重命名保留原目录,确认恢复成功后再清理
5. **恢复后要修权限和 SELinux 标签**:`chown -R mysql:mysql` + `restorecon -Rv /var/lib/mysql`,否则服务起不来
6. **备份目录必须为空或不存在**:否则 XtraBackup 会报错退出
7. **MyISAM 表需要短暂锁表**:建议在业务低峰期执行
8. **记录 binlog 位置**:`xtrabackup_binlog_info` 里的位置是后续做按时间点恢复(PITR)的依据
9. **备份要验证**:定期把备份恢复到测试库,确认真的可用;只备份不演练等于没有备份
10. **服务名差异**:CentOS/RHEL 是 `mysqld`,Ubuntu/Debian 是 `mysql`