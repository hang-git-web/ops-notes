Prometheus 监控 MySQL:

准备
1. MySQL 数据库本体:
2. mysqld_exporter:负责把指标暴露出来
3. Prometheus: 负责定时拉指标、存储、可视化

部署:
1. 给 MySQL 建一个只读账号
> CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'Exporter@123';
> GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
> FLUSH PRIVILEGES;
权限解释:
`PROCESS`：允许看进程列表
`REPLICATION CLIENT`：允许查看主从状态
`SELECT`：允许读表

2. 下载并运行 mysqld_exporter
   创建目录
   mkdir -p /usr/local/mysqld_exporter
   cd /usr/local/mysqld_exporter
   下载二进制文件
   wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
   解压
   tar -zxvf mysqld_exporter-0.15.1.linux-amd64.tar.gz
   cd mysqld_exporter-0.15.1.linux-amd64
   设置 MySQL 连接环境变量
   export DATA_SOURCE_NAME='exporter:Exporter@123@(127.0.0.1:3306)/'
   编写连接配置文件
   vim .my.cnf
   [client]
   user=exporter
   password=自定义的密码
   host=127.0.0.1
   prot=3306
   启动
   ./mysqld_exporter &

验证
在浏览器打开：
http://你的_exporter_IP:9104/metrics
能看到一大堆以 `mysql_` 开头的指标


用grafana查看
1.登录 Grafana
2.添加 Prometheus 数据源
3.从 Grafana 官方面板库里搜索：`MySQL`(比如 7362模板)