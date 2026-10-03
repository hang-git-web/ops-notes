# MySQL 主从复制(Replication)笔记

> 主从复制是一种数据库复制技术:把主库(Master/Source)的数据变更,实时或准实时地复制到一个或多个从库(Slave/Replica)。
> 本质是"主库记录变更 → 从库拉取 → 从库重放",因此它是**异步复制**:默认不保证零丢失,只保证最终一致。

## 一、核心流程

```text
主库:数据变更 → 写入 binlog(二进制日志)
                        ↓
从库 I/O 线程:拉取主库 binlog → 写入本地 relay log(中继日志)
                        ↓
从库 SQL 线程:读取 relay log → 重放变更 → 与主库保持一致
```

| 角色 | 线程/文件 | 作用 |
| --- | --- | --- |
| 主库 | binlog | 记录所有数据变更 |
| 从库 | I/O 线程 | 拉取主库 binlog 并写入 relay log |
| 从库 | relay log | 本地中继日志 |
| 从库 | SQL 线程 | 重放 relay log 中的变更 |

## 二、复制模式

| 模式 | 说明 | 特点 |
| --- | --- | --- |
| SBR(基于语句) | 记录执行的 SQL 语句 | 日志小,但某些函数(如 `NOW()`、`UUID()`)可能导致不一致 |
| RBR(基于行) | 记录每一行的变更 | 最安全、最常用,日志量较大 |
| Mixed(混合) | 由 MySQL 自动在两者间选择 | 兼容性折中 |

> 生产环境推荐 `binlog_format = ROW`。

## 三、应用场景

| 场景 | 说明 |
| --- | --- |
| 读写分离 | 主库负责写,从库负责读,提升整体吞吐 |
| 数据热备 | 主库故障时可快速切换到从库 |
| 负载均衡 | 多从库分散高并发查询压力 |
| 数据分析 | 在从库执行统计查询,避免影响主库 |

注意:由于是异步复制,从库数据可能**短暂落后**于主库,读从库时要有延迟容忍度(如"写入后立即读"的场景要走主库)。

## 四、环境准备

| 角色 | 示例 IP | MySQL 版本 | 操作系统 |
| --- | --- | --- | --- |
| 主库 | 172.22.4.3 | 8.0 | CentOS 7.9 |
| 从库 | 172.22.4.4 | 8.0 | CentOS 7.9 |

要求:

- 主从 **MySQL 版本一致**(大版本必须相同,建议小版本也一致)
- `server-id` 全局唯一
- 主从网络互通,开放 3306 端口
- 两台机器时间同步(chrony/NTP),字符集与排序规则一致

## 五、主库配置

### 1. 修改配置文件

```bash
vim /etc/my.cnf
```

```ini
[mysqld]
server-id = 1                  # 主库唯一标识
log-bin = mysql-bin            # 启用二进制日志
binlog_format = ROW            # 推荐使用行格式
binlog_expire_logs_seconds = 604800   # binlog 保留 7 天
max_binlog_size = 512M
```

```bash
systemctl restart mysqld
```

> 提示:自定义日志路径(如 `/var/log/mysql/mysql-bin.log`)时,目录必须存在且属主为 mysql,否则服务启动失败;CentOS 上还会有 SELinux 标签问题,练习环境直接用默认路径最省事。

### 2. 创建同步专用账号

```sql
CREATE USER 'repl'@'172.22.4.4' IDENTIFIED BY 'Repl@2026';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'172.22.4.4';
FLUSH PRIVILEGES;

-- 查看账号
SELECT user, host, plugin FROM mysql.user WHERE user = 'repl';
```

> 安全建议:把账号限定到从库的具体 IP,不要用 `%`;如果从库是老版本(5.7)连 8.0 主库,账号需用 `IDENTIFIED WITH mysql_native_password`。

### 3. 查看主库状态

```sql
-- MySQL 8.4 之前
SHOW MASTER STATUS;

-- MySQL 8.4 及之后
SHOW BINARY LOG STATUS;
```

输出示例:

```text
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      747 |              |                  |
+------------------+----------+--------------+------------------+
```

记录 `File` 和 `Position`,用于从库配置。

### 4. 已有数据时的初始同步(重要)

如果主库已有业务数据,必须先做一次全量导出,并记录对应的 binlog 位置:

```bash
mysqldump -uroot -p --single-transaction --master-data=2 \
  --routines --triggers --events --all-databases \
  > /backup/full_$(date +%F).sql
```

再把这个文件导入从库,然后用文件中的 `CHANGE MASTER TO ... MASTER_LOG_FILE/POS` 注释里的位置配置从库。空库可以直接用 `SHOW MASTER STATUS` 的位置。

## 六、从库配置

### 1. 修改配置文件

```ini
[mysqld]
server-id = 2                  # 必须与主库不同
relay-log = relay-bin          # 中继日志
read_only = 1                  # 普通用户只读,防止误写
super_read_only = 1            # 超级用户也只读
replica_parallel_workers = 4   # 多线程复制,缓解延迟
replica_parallel_type = LOGICAL_CLOCK
replica_preserve_commit_order = 1
```

```bash
systemctl restart mysqld
```

> `read_only = 1` 只限制普通用户,root/SUPER 账号仍可写;需要彻底只读就加 `super_read_only = 1`(主从切换时记得关掉)。

### 2. 配置主从连接

**MySQL 8.0.23 及之后(推荐)**

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST        = '172.22.4.3',
  SOURCE_USER        = 'repl',
  SOURCE_PASSWORD    = 'Repl@2026',
  SOURCE_PORT        = 3306,
  SOURCE_LOG_FILE    = 'mysql-bin.000001',
  SOURCE_LOG_POS     = 747,
  GET_SOURCE_PUBLIC_KEY = 1,      -- 8.0 默认 caching_sha2_password 时需要
  SOURCE_SSL         = 0;

START REPLICA;
```

**MySQL 8.0.23 之前(旧语法)**

```sql
CHANGE MASTER TO
  MASTER_HOST        = '172.22.4.3',
  MASTER_USER        = 'repl',
  MASTER_PASSWORD    = 'Repl@2025',
  MASTER_LOG_FILE    = 'mysql-bin.000001',
  MASTER_LOG_POS     = 747;

START SLAVE;
```

**GTID 模式(推荐,免去手工算位置)**

主从都配置:

```ini
gtid_mode = ON
enforce_gtid_consistency = ON
```

从库只需:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='172.22.4.3',
  SOURCE_USER='repl',
  SOURCE_PASSWORD='Repl@2025',
  SOURCE_AUTO_POSITION = 1;

START REPLICA;
```

### 3. 查看同步状态

```sql
SHOW REPLICA STATUS\G;   -- MySQL 8.0.22 及之后
SHOW SLAVE STATUS\G;     -- MySQL 8.0.22 之前
```

重点指标:

| 指标 | 期望值 | 说明 |
| --- | --- | --- |
| `Replica_IO_Running` | Yes | I/O 线程正常拉取主库 binlog |
| `Replica_SQL_Running` | Yes | SQL 线程正常重放 |
| `Seconds_Behind_Source` | 接近 0 | 复制延迟秒数 |
| `Last_IO_Error` | 空 | I/O 线程错误 |
| `Last_SQL_Error` | 空 | SQL 线程错误 |

> 旧版本命令显示的字段名是 `Slave_IO_Running` / `Slave_SQL_Running`。

## 七、验证主从同步

在主库执行:

```sql
CREATE DATABASE repl_test;
USE repl_test;
CREATE TABLE t1 (id INT PRIMARY KEY, name VARCHAR(20));
INSERT INTO t1 VALUES (1, 'hello replication');
```

在从库查看:

```sql
USE repl_test;
SELECT * FROM t1;       -- 有数据说明同步成功
SELECT @@server_id;     -- 确认是从库的 server-id
SELECT @@read_only;     -- 应为 1
```

## 八、日常维护命令

```sql
-- 查看状态
SHOW REPLICA STATUS\G;

-- 停止/启动复制
STOP REPLICA;
START REPLICA;

-- 重置主从关系(清空复制信息)
STOP REPLICA;
RESET REPLICA ALL;

-- 旧语法(8.0 之前)
STOP SLAVE;
RESET SLAVE ALL;
```

主从切换(从库提升为主库)的大致操作:

```sql
STOP REPLICA;
RESET REPLICA ALL;
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;
```

然后把其他从库的复制源指向这台新主库。

## 九、常见故障与处理

| 问题 | 原因 | 解决 |
| --- | --- | --- |
| 连接失败(2003) | 网络不通、防火墙未放行、账号权限不足 | 检查 3306、云安全组、账号是否限定 IP |
| 认证失败 | 密码错误或认证插件不兼容 | `GET_SOURCE_PUBLIC_KEY=1`,或改用 `mysql_native_password` |
| `Last_IO_Error: 1236` | 指定的 binlog 文件不存在(已被清理) | 重新全量导出并重新配置位置 |
| 主键冲突(1062) | 从库被写入过数据或数据不一致 | 检查 `read_only`,校验数据后重建从库 |
| 同步延迟过高 | 大事务、慢查询、单线程重放、从库资源不足 | 开启多线程复制、拆分大事务、优化慢查询、提升硬件 |
| 主从数据不一致 | 误操作、跳过错误、非确定性语句 | 用 `pt-table-checksum` 校验并修复 |
| SQL 线程停止 | 表不存在、字段不匹配等 | 先看 `Last_SQL_Error`,修复后重启复制 |

跳过错(谨慎使用):

```sql
STOP REPLICA;
SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;   -- 旧名:SQL_SLAVE_SKIP_COUNTER
START REPLICA;
```

> 跳过错误只是让复制"继续跑",数据不一致已经产生。GTID 模式下该参数无效,需要按 GTID 单独处理;生产环境应先评估影响,能重建从库就不要跳。

## 十、复制延迟优化

| 方向 | 做法 |
| --- | --- |
| 复制线程 | `replica_parallel_workers` + `LOGICAL_CLOCK` 开启并行重放 |
| 事务粒度 | 避免大批量更新(单事务越大,从库重放越慢) |
| 日志格式 | 使用 ROW,减少不确定性 |
| 从库负载 | 只读查询压力大时增加从库或升级硬件 |
| 表结构 | 从库主键、索引与主库保持一致,避免全表扫描重放 |
| 监控 | 监控 `Seconds_Behind_Source` 与 IO/SQL 线程状态,提前告警 |

> 进阶方案:对一致性要求更高时,可考虑**半同步复制**或 **MySQL Group Replication**,代价是性能与复杂度上升。

## 十一、注意事项

1. 主从版本必须匹配,`server-id` 必须唯一
2. 默认是**异步复制**,主库故障时可能丢最后一段数据
3. 从库用 `read_only` + `super_read_only` 防止误写,切换主库时记得关闭
4. 复制账号要限定来源 IP,遵循最小权限(只需 `REPLICATION SLAVE`)
5. 已有数据的环境一定要先做**全量初始化**,再配置复制位置或 GTID
6. 定期校验主从数据一致性(`pt-table-checksum`),不要只看线程是否 Yes
7. 新版本命令已完成术语迁移:`MASTER/SLAVE` → `SOURCE/REPLICA`,写文档和脚本时注意版本差异