XtraBackup
介绍: 一款专为 MySQL 数据库设计的开源热备份工具,它能够在数据库正常运行的状态下进行备份操作，
避免了备份过程中需要停止数据库服务，从而确保业务的连续性

备份策略介绍
全量备份：全量备份会复制数据库中的所有数据文件，涵盖了数据目录下的所有表空间文件、日志文件等。
增量备份：增量备份仅备份自上次全量备份或增量备份以来发生变化的数据。
差异备份：差异备份会备份自上次全量备份以来所有发生变化的数据

一.  XtraBackup安装环境
1. 安装依赖库（CentOS 示例）  
   yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
   percona-release enable-only tools release
   yum install percona-xtrabackup-24 -y

注意‌：需选择与 MySQL 版本匹配的 XtraBackup 版本（如 MySQL 8.0 使用 XtraBackup 8.0,MySQL 5.7 使用 XtraBackup 2.4）

2. 创建备份专用用户‌并授权
   CREATE USER 'backup'@'localhost' IDENTIFIED BY 'Backup@2025';  
   GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'backup'@'localhost';  
   FLUSH PRIVILEGES;  
   权限说明‌：
   RELOAD：刷新表状态
   REPLICATION CLIENT：读取二进制日志‌

二. 全量备份及恢复
1. 执行备份命令‌
   innobackupex --user=backup --password=Backup@2025 --no-timestamp /data/mysql/backup/full
   参数说明‌：
   --no-timestamp：禁止自动生成时间戳目录，自定义备份路径
2. 验证备份文件‌
   ls /data/mysql/backup/full_20250326
# 关键文件：ibdata1（系统表空间）、备份日志

3. 全量备份恢复
# 准备备份（应用日志）
innobackupex --apply-log /data/mysql/backup/full
# 停止 MySQL 服务并清空数据目录
systemctl stop mysql  
rm -rf /var/lib/mysql/*
# 复制备份文件
innobackupex --copy-back /data/mysql/backup/full
# 修复权限并启动服务
chown -R mysql:mysql /var/lib/mysql  
systemctl start mysql

三. 增量备份及恢复
1. 基于全量备份执行增量‌
   innobackupex --user=backup --password=Backup@2025 --no-timestamp   --incremental /data/mysql/backup/incr  --incremental-basedir=/data/mysql/backup/full
   参数说明‌：
   --incremental：指定增量备份目录
   --incremental-basedir：指向全量备份目录

2. 查看增量备份 LSN‌

cat /data/mysql/backup/incr/xtrabackup_checkpoints
# 输出示例：
# backup_type = incremental
# from_lsn = 12345678
# to_lsn = 12345999‌:ml-citation{ref="4,8" data="citationList"}

3. 部分备份
   备份指定数据库‌
   innobackupex --user=backup --password=Backup@2025 --databases="mydb" /data/mysql/backup/partial_mydb  
   备份指定表（正则匹配）‌
   innobackupex --user=backup --password=Backup@2025 --include='mydb.order_.*' /data/mysql/backup/partial_tables  
   说明‌：--include 支持正则表达式匹配表名‌

4. 增量备份恢复
# 准备全量备份
innobackupex --apply-log --redo-only /data/mysql/backup/full
# 合并增量备份
innobackupex --apply-log --redo-only /data/mysql/backup/full --incremental-dir=/data/mysql/backup/incr
# 最终应用日志并复制文件
innobackupex --apply-log /data/mysql/backup/full
innobackupex --copy-back /data/mysql/backup/full
--redo-only 表示仅应用已提交事务，

四. 备份注意事项
1. 版本兼容性‌：
   XtraBackup 8.0+ 仅支持 MySQL 8.0+，低版本需使用 XtraBackup 2.4‌

2. 备份验证‌：
   定期执行 innobackupex --verify 检查备份完整性

3. 存储引擎限制‌：
   MyISAM 表备份需短暂锁表，建议业务低峰期操作