Redis哨兵模式
哨兵模式（Sentinel）‌：
哨兵（Sentinel）‌：独立进程，监控Redis主从健康状态，自动选举新主节点（类似“保安团队”）。
故障转移（Failover）‌：主节点故障时，哨兵自动提升从节点为新主节点。‌

部署哨兵模式
一. 三台centos7服务器 ip为 172.22.4.2、172.22.4.3、172.22.4.4
所有节点执行:
安装依赖  
yum install -y gcc tcl wget

二. 部署redis与主从配置
1. 安装Redis（所有节点）
# 下载源码包
wget http://download.redis.io/releases/redis-7.2.0.tar.gz -P /opt  
tar -zxvf /opt/redis-7.2.0.tar.gz  
cd /opt/redis-7.2.0 && make && make install
# 创建配置文件目录
mkdir -p /usr/local/redis/{conf,data,log}  
cp /opt/redis-7.2.0/redis.conf /usr/local/redis/conf/

2. 配置主节点（以172.22.4.2为例）
   vim /usr/local/redis/conf/redis.conf

bind 0.0.0.0  
port 6379  
daemonize yes  
logfile "/usr/local/redis/log/redis.log"  
dir /usr/local/redis/data  
requirepass your_password  # 设置密码，主从需一致！
# 启动主节点
redis-server /usr/local/redis/conf/redis.conf

步骤3：配置从节点（172.22.4.3和172.22.4.4）

# 修改配置文件（以172.22.4.3为例）
vim /usr/local/redis/conf/redis.conf

replicaof 172.22.4.2 6379  # 指向主节点IP和端口    
masterauth your_password  # 主节点密码

# 启动从节点
redis-server /usr/local/redis/conf/redis.conf

进行主从验证(在主节点set值,在两个从节点get)

三. 配置哨兵(所有节点）
1. 编辑配置
   vim /usr/local/redis/conf/sentinel.conf

port 26379  
daemonize yes  
logfile "/usr/local/redis/log/sentinel.log"  
sentinel monitor mymaster 172.22.4.2 6379 2  # 监控主节点，2为quorum值‌
sentinel auth-pass mymaster your_password  
sentinel down-after-milliseconds mymaster 5000  # 5秒无响应判定为故障  
sentinel failover-timeout mymaster 10000  # 故障转移超时时间

参数解释
# 哨兵监听的端口
port 26379

# 以守护进程方式运行
daemonize yes

# 日志文件路径
logfile "/var/log/redis/sentinel.log"

# 监控主节点
# mymaster 是主节点的别名
# 172.22.4.3 是主节点的 IP 地址
# 6379 是主节点的端口
# 1 是判定主节点客观下线所需的最少哨兵数量（quorum）
sentinel monitor mymaster 172.22.4.3 6379 1

# 如果主节点设置了密码，需要配置认证密码
sentinel auth-pass mymaster your_password

# 主节点多少毫秒内无响应，就认为主节点主观下线
sentinel down-after-milliseconds mymaster 5000

# 故障转移超时时间
sentinel failover-timeout mymaster 10000

# 执行故障转移时，最多可以有多少个从节点同时对新的主节点进行同步
sentinel parallel-syncs mymaster 1

2. 启动哨兵(所有节点）
   redis-sentinel /usr/local/redis/conf/sentinel.conf

redis-cli -p 26379 info sentinel  查看状态

3. 模拟主节点宕机
# 在主节点执行
redis-cli -h 172.22.4.2 shutdown  
观察哨兵日志‌：
tail -f /usr/local/redis/log/sentinel.log  # 查看选举新主过程‌

验证新主节点‌：
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster  # 查看新主IP‌

四. redis命令速查
# 查看主从信息
redis-cli -h <IP> info replication

# 动态修改从节点指向
redis-cli -h 172.22.4.3 SLAVEOF NO ONE  # 脱离主从‌

# 查看哨兵监控状态
redis-cli -p 26379 sentinel masters