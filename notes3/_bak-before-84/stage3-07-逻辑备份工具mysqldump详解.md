# mysqldump 逻辑备份笔记

## 一、定义与特点

**定义**:把数据库的结构和数据导出为 SQL 文本文件(可读)。

| 特点 | 说明 |
| --- | --- |
| 兼容性强 | 支持跨版本恢复、跨平台迁移 |
| 适用规模 | 小型数据库(GB 级以下) |
| 速度 | 备份和恢复较慢,本质是逐行处理 SQL 语句 |
| 优点 | 文件可读、可编辑、可单表恢复、易传输 |
| 缺点 | 大库耗时长、恢复慢、占用 CPU/IO 较高 |

## 二、核心功能

- 备份整个数据库、单个表、多个表,或按条件备份部分数据
- 支持导出表结构(`CREATE` 语句)和数据(`INSERT` 语句)
- 可一并备份存储过程、函数、触发器、事件等对象

## 三、核心参数详解

| 参数 | 说明 |
| --- | --- |
| `--single-transaction` | 事务快照保证一致性,**仅对 InnoDB 有效**,不锁表 |
| `--lock-tables` | 备份期间锁定所有表;MyISAM 需用此参数保证一致性,但会阻塞写入 |
| `--skip-lock-tables` | 不锁表,适合允许写入的场景,但可能数据不一致 |
| `--routines` | 包含存储过程和函数 |
| `--events` | 包含事件(定时任务) |
| `--triggers` | 包含触发器 |
| `--no-data` | 仅备份表结构,不备份数据 |
| `--no-create-info` | 仅备份数据,不备份表结构 |
| `--databases` | 备份多个库,导出文件含 `CREATE DATABASE` |
| `--all-databases` | 备份所有库 |
| `--ignore-table=库.表` | 忽略指定表,可重复使用 |
| `--where="条件"` | 按条件备份部分数据(对该次备份的所有表生效) |
| `--default-character-set=utf8mb4` | 指定字符集,避免中文乱码 |
| `--master-data=2` | 以注释形式记录 binlog 位置,便于按时间点恢复 |
| `--set-gtid-purged=OFF` | 避免 GTID 相关信息导致导入报错 |

> 常用组合:`--single-transaction --routines --triggers --events --default-character-set=utf8mb4`

## 四、常用命令

### 1. 基本格式

```bash
mysqldump -u用户名 -p 选项 数据库名 [表名...] > 备份文件.sql
```

注意:`-p` 与密码之间**不能有空格**;省略密码时会在执行时提示输入。

### 2. 备份

```bash
# 单个数据库(推荐参数组合)
mysqldump -uroot -p --single-transaction --routines --triggers --events \
  --default-character-set=utf8mb4 mydb > /backup/mydb_full_$(date +%F).sql

# 多个数据库
mysqldump -uroot -p --databases db1 db2 > /backup/dbs_$(date +%F).sql

# 所有数据库
mysqldump -uroot -p --all-databases --single-transaction \
  --routines --triggers --events --default-character-set=utf8mb4 \
  > /backup/all_dbs_$(date +%F).sql

# 单个表
mysqldump -uroot -p mydb table1 > /backup/mydb_table1.sql

# 多个表(带条件,只备份 id < 1000 的数据)
mysqldump -uroot -p --where="id < 1000" mydb table1 table2 > /backup/mydb_tables_partial.sql

# 忽略某张表
mysqldump -uroot -p --ignore-table=mydb.logs mydb > /backup/mydb_no_logs.sql

# 仅备份表结构 / 仅备份数据
mysqldump -uroot -p --no-data mydb        > /backup/mydb_schema.sql
mysqldump -uroot -p --no-create-info mydb > /backup/mydb_data.sql

# 压缩备份,节省空间
mysqldump -uroot -p --single-transaction mydb | gzip > /backup/mydb_full_$(date +%F).sql.gz

# 记录 binlog 位置(便于后续增量恢复)
mysqldump -uroot -p --single-transaction --master-data=2 mydb > /backup/mydb_full.sql
```

### 3. 恢复

```bash
# 恢复完整数据库(库不存在时先创建)
mysql -uroot -p -e "CREATE DATABASE IF NOT EXISTS mydb DEFAULT CHARACTER SET utf8mb4;"
mysql -uroot -p --default-character-set=utf8mb4 mydb < /backup/mydb_full.sql

# 恢复压缩备份
gunzip < /backup/mydb_full.sql.gz | mysql -uroot -p mydb

# 恢复全部数据库
mysql -uroot -p < /backup/all_dbs.sql
```

**恢复单个表**

推荐做法:直接单独备份/恢复该表

```bash
mysqldump -uroot -p mydb table1 > table1.sql
mysql -uroot -p mydb < table1.sql
```

如果只有整库备份文件,从备份中截取该表:

```bash
sed -n '/^-- Table structure for table `table1`/,/^-- Table structure for table/p' mydb_full.sql > table1.sql
mysql -uroot -p mydb < table1.sql
```

> 截取法依赖备份文件的标准格式,表名和反引号要写对;条件允许时优先用单独备份的方式。

### 4. 验证备份是否可用

```bash
# 文件大小是否正常
ls -lh /backup/mydb_full_2026-09-12.sql

# 统计建表语句数量
grep -c "CREATE TABLE" /backup/mydb_full_2026-09-12.sql

# 恢复到测试库验证(最可靠)
mysql -uroot -p -e "CREATE DATABASE mydb_verify;"
mysql -uroot -p mydb_verify < /backup/mydb_full_2026-09-12.sql
mysql -uroot -p -e "USE mydb_verify; SHOW TABLES;"
```

## 五、备份恢复注意事项

### 1. 锁表取舍

- InnoDB 表:用 `--single-transaction` 实现一致性备份,不阻塞写入
- MyISAM 表:需在 `--lock-tables`(一致性)与 `--skip-lock-tables`(可用性)之间权衡
- 混合引擎的库:以 InnoDB 为准,单独确认 MyISAM 表的一致性需求

### 2. 备份文件安全

```bash
chmod 600 /backup/*.sql      # 备份含敏感数据,限制访问权限
gzip /backup/*.sql           # 压缩,顺便降低泄露风险
```

- 备份文件加密存储或加密传输
- 不要把密码明文写在命令行(会留在历史记录和进程列表里),改用 `-p` 交互输入或 `--defaults-extra-file` 配置文件

### 3. 版本与字符集兼容性

- 高版本导出的文件恢复到低版本可能不兼容,例如 MySQL 8 的 `utf8mb4_0900_ai_ci` 排序规则在旧版本无法识别
- 备份与恢复都建议加 `--default-character-set=utf8mb4`,避免中文乱码
- 跨版本迁移前,先在测试环境验证一次恢复

### 4. 恢复时的小技巧

```sql
-- 大表恢复可临时关闭外键检查,加快导入
SET FOREIGN_KEY_CHECKS = 0;
-- ... 导入 ...
SET FOREIGN_KEY_CHECKS = 1;
```

### 5. 规模建议

- 数据量在 GB 级以内:mysqldump 足够
- 数据量很大或要求快速恢复:改用物理备份工具(如 Percona XtraBackup)

## 六、完整实操示例

```bash
# 1. 备份
mkdir -p /backup
mysqldump -uroot -p --single-transaction --routines --triggers --events \
  --default-character-set=utf8mb4 mydb | gzip > /backup/mydb_$(date +%F).sql.gz

# 2. 限制备份文件权限
chmod 600 /backup/mydb_$(date +%F).sql.gz

# 3. 验证:恢复到测试库
mysql -uroot -p -e "CREATE DATABASE mydb_verify DEFAULT CHARACTER SET utf8mb4;"
gunzip < /backup/mydb_$(date +%F).sql.gz | mysql -uroot -p mydb_verify
mysql -uroot -p -e "USE mydb_verify; SHOW TABLES;"

# 4. 确认无误后清理测试库
mysql -uroot -p -e "DROP DATABASE mydb_verify;"
```