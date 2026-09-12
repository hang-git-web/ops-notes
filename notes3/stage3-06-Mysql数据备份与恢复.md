# MySQL 数据备份与恢复笔记

## 一、备份分类

### 1. 按备份内容

| 类型 | 说明 |
| --- | --- |
| 完全备份 | 备份整个数据库(表、数据、索引、数据库对象) |
| 部分备份 | 只备份部分数据或部分表 |

### 2. 按备份方式

| 类型 | 说明 | 特点 |
| --- | --- | --- |
| 逻辑备份 | 导出为 SQL 语句(`mysqldump`) | 兼容性强、可读、体积大、适合中小库 |
| 物理备份 | 直接复制数据文件与日志文件 | 速度快、体积小,通常需停机或专业工具 |

### 3. 按备份状态

| 类型 | 数据库状态 | 特点 |
| --- | --- | --- |
| 热备份 | 正常运行,可读写 | 不影响业务,需要工具支持 |
| 温备份 | 允许有限操作,可读、限制写 | 折中方案 |
| 冷备份 | 停止服务后备份 | 简单、一致性好,但期间不可用 |

## 二、逻辑备份与恢复(mysqldump)

### 1. 备份前检查

```bash
# 查看当前连接与是否有大量写入
mysql -u root -p -e "SHOW PROCESSLIST;"

# 确保备份目录存在
mkdir -p /backup
```

### 2. 备份命令

```bash
# 备份单个数据库(推荐参数)
mysqldump -uroot -p --single-transaction --default-character-set=utf8mb4 \
  mydb > /backup/mydb_full_$(date +%F).sql

# 备份多个数据库
mysqldump -uroot -p --databases db1 db2 > /backup/dbs_$(date +%F).sql

# 备份所有数据库(含存储过程、触发器、事件)
mysqldump -uroot -p --all-databases --single-transaction \
  --routines --triggers --events --default-character-set=utf8mb4 \
  > /backup/all_dbs_$(date +%F).sql

# 压缩备份,节省空间
mysqldump -uroot -p --single-transaction --routines --triggers --events mydb \
  | gzip > /backup/mydb_full_$(date +%F).sql.gz

# 只备份表结构 / 只备份数据
mysqldump -uroot -p --no-data mydb        > /backup/mydb_schema.sql
mysqldump -uroot -p --no-create-info mydb > /backup/mydb_data.sql
```

### 3. 常用参数说明

| 参数 | 作用 |
| --- | --- |
| `--single-transaction` | 事务快照保证一致性,**仅对 InnoDB 有效** |
| `--skip-lock-tables` | 跳过锁表;混用 MyISAM 表时可能不一致,慎用 |
| `--routines --triggers --events` | 一并备份存储过程、触发器、事件 |
| `--default-character-set=utf8mb4` | 避免中文乱码 |
| `--master-data=2` | 记录 binlog 位置(注释形式),便于按时间点恢复 |
| `--where="条件"` | 只备份满足条件的数据(部分备份) |
| `--ignore-table=库.表` | 排除指定表 |
| `--set-gtid-purged=OFF` | 避免导入时因 GTID 语句报错 |

> 说明:`--single-transaction` 已经能在 InnoDB 下实现不锁表的一致性备份,额外加 `--skip-lock-tables` 反而可能让 MyISAM 表备份不一致。只有全 InnoDB 场景可放心使用该组合。

### 4. 恢复

```bash
# 恢复单个数据库(先建空库)
mysql -uroot -p -e "CREATE DATABASE IF NOT EXISTS mydb DEFAULT CHARACTER SET utf8mb4;"
mysql -uroot -p --default-character-set=utf8mb4 mydb < /backup/mydb_full_2026-09-12.sql

# 恢复压缩备份
gunzip < /backup/mydb_full_2026-09-12.sql.gz | mysql -uroot -p mydb

# 恢复所有数据库
mysql -uroot -p < /backup/all_dbs_2026-09-12.sql
```

> 使用 `--databases` 或 `--all-databases` 导出的文件里自带 `CREATE DATABASE`,恢复时不要再指定库名。

### 5. 定时备份

```bash
crontab -e
```

```cron
# 每天凌晨 2 点全量备份,保留最近 7 天
0 2 * * * mysqldump -uroot -p'密码' --all-databases --single-transaction --routines --triggers --events --default-character-set=utf8mb4 | gzip > /backup/all_$(date +\%F).sql.gz
```

```bash
# 简单清理 7 天前的备份
find /backup -name "*.sql.gz" -mtime +7 -delete
```

## 三、物理备份与恢复

适用于数据量大、要求快速恢复的场景。原理是直接复制数据目录(通常在 `/var/lib/mysql`)。

### 1. 备份

```bash
# 1. 停止 MySQL
systemctl stop mysqld

# 2. 复制数据目录(rsync 会保留权限,比 cp 更省事)
rsync -av /var/lib/mysql/ /backup/mysql_data_$(date +%F)/

# 3. 启动 MySQL
systemctl start mysqld
```

### 2. 恢复

```bash
# 1. 停止服务
systemctl stop mysqld

# 2. 保留原目录(不要直接 rm,便于回退)
mv /var/lib/mysql /var/lib/mysql.bak.$(date +%F)

# 3. 复制备份数据回来
rsync -av /backup/mysql_data_2026-09-12/ /var/lib/mysql/

# 4. 修复属主与 SELinux 标签
chown -R mysql:mysql /var/lib/mysql
restorecon -Rv /var/lib/mysql

# 5. 启动服务
systemctl start mysqld
```

> 服务名提示:CentOS/RHEL 是 `mysqld`,Ubuntu/Debian 是 `mysql`。
> 生产环境做物理热备份建议使用 **Percona XtraBackup**,可以不停机备份 InnoDB,支持增量备份与快速恢复。

## 四、增量备份与恢复(基于 binlog)

原理:二进制日志(binlog)记录所有数据变更,全量备份 + binlog 可以恢复到任意时间点(PITR)。

### 1. 启用 binlog

```bash
vim /etc/my.cnf
```

```ini
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
# 日志保留时间(8.0 用 binlog_expire_logs_seconds)
expire_logs_days = 7
```

```bash
systemctl restart mysqld
```

### 2. 备份 binlog

```bash
# 查看当前正在使用的日志文件与位置
mysql -u root -p -e "SHOW MASTER STATUS;"

# 手动生成新日志(便于归档当前日志)
mysql -u root -p -e "FLUSH BINARY LOGS;"

# 归档指定范围的日志文件
mkdir -p /backup/binlog
cp /var/lib/mysql/mysql-bin.00000{2..5} /backup/binlog/
```

### 3. 恢复流程

```bash
# 第一步:恢复最近一次全量备份
mysql -uroot -p mydb < /backup/mydb_full_2026-09-12.sql

# 第二步:按时间顺序逐个应用 binlog(按需限制时间范围或指定库)
mysqlbinlog --start-datetime="2026-09-12 02:00:00" \
            --stop-datetime="2026-09-12 12:00:00" \
            /backup/binlog/mysql-bin.000002 | mysql -uroot -p

mysqlbinlog --database=mydb /backup/binlog/mysql-bin.000003 | mysql -uroot -p
```

常用参数:

| 参数 | 作用 |
| --- | --- |
| `--start-datetime` / `--stop-datetime` | 按时间范围恢复 |
| `--start-position` / `--stop-position` | 按位置精确恢复 |
| `--database=mydb` | 只恢复指定库的变更 |
| `--skip-gtids` | GTID 环境下避免重复执行报错 |
| `--base64-output=DECODE-ROWS -v` | 把 ROW 格式日志解码成可读 SQL,用于查看 |

MySQL 8.4 起 `SHOW MASTER STATUS` 改为 `SHOW BINARY LOG STATUS`。

## 五、备份策略建议

| 项目 | 建议 |
| --- | --- |
| 全量 | 每天/每周一次,业务低峰期执行 |
| 增量 | 常开 binlog,按小时/天归档 |
| 副本 | 遵循 3-2-1:3 份副本、2 种介质、1 份异地 |
| 验证 | 定期把备份恢复到测试库,确认真的可用 |
| 权限 | 备份账号只授予 `SELECT`、`LOCK TABLES`、`SHOW VIEW`、`TRIGGER`、`RELOAD`、`PROCESS`、`REPLICATION CLIENT` |
| 安全 | 备份文件含业务数据,注意加密、权限控制和访问审计 |

## 六、常见坑

1. `rm -rf /var/lib/mysql/*` 前一定要确认备份可用,建议用 `mv` 保留原目录
2. `--single-transaction` 只对 InnoDB 有效,MyISAM 表仍需锁表保证一致性
3. 恢复时字符集不一致会导致中文乱码,建议备份与恢复都加 `--default-character-set=utf8mb4`
4. 大数据量用 mysqldump 会很慢,考虑 XtraBackup 或从库备份
5. binlog 必须按顺序恢复;恢复前先记录清楚时间点和文件范围
6. 只备份不演练等于没有备份,**定期做恢复演练**才是真的可靠