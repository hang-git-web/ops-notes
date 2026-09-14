Redis Cluster集群
Redis Cluster‌：
‌分布式数据库‌：数据分散存储在多个节点，支持横向扩展和高可用性‌。
‌节点（Node）‌：每个Redis实例，分为‌主节点（Master）‌和‌从节点（Slave）‌‌。
‌哈希槽（Slot）‌：
数据分片的最小单位，共16384个槽，每个主节点负责部分槽（类比“蛋糕分块”）‌。
‌Gossip协议‌：
节点间通过PING/PONG消息交换状态信息，实现自发现和故障检测（类似“微信群聊”）。
‌故障转移（Failover）‌：
主节点宕机时，从节点自动升级为新主节点，保障服务连续性

Redis Cluster集群部署准备环境
需要三台centos服务器,每台服务器上启动两个Redis实例，端口分别为7000和7001
一 . 所有节点执行：安装依赖和配置redis
1.安装依赖
yum install -y gcc tcl wget

2. 安装并配置Redis(所有节点)
# 下载源码包
cd /data/redis
wget http://download.redis.io/releases/redis-7.2.0.tar.gz
tar -zxvf /opt/redis-7.2.0.tar.gz  
cd /data/redis/redis-7.2.0
make && make install

# 创建集群配置文件目录（以172.22.4.2为例）
mkdir -p /usr/local/redis-cluster/{7000,7001}  
cp /data/redis/redis-7.2.0/redis.conf /usr/local/redis-cluster/7000/  
cp /data/redis/redis-7.2.0/redis.conf /usr/local/redis-cluster/7001/

修改配置文件,每台机器都需修改(以7000端口为例修改,.修改7001,将7000的配置文件里面的7000改为7001即可,可通过scp方式传给另外两台)
vim /usr/local/redis-cluster/7000/redis.conf
修改内容
port 7000
bind 0.0.0.0
daemonize	 yes
pidfile 	 /var/run/redis_7000.pid
logfile 	"/usr/local/reids-cluster/7000/redis.log"
dir	/usr/local/redis-cluster/7000/
protected-mode  no
cluster-enabled 	yes
cluster-config-file  nodes-7000.conf
cluster-node-timeout  15000
cluster-require-full-coverage no
cluster-migration-barrier 1

启动6台redis
redis-server  /usr/local/redis-cluster/7000/redis.conf
redis-server  /usr/local/redis-cluster/7001/redis.conf

二. 创建集群并验证
1.使用redis-cli创建集群
/usr/local/bin/redis-cli --cluster create 172.22.4.2:7000 172.22.4.2:7001 172.22.4.3:7000 172.22.4.3:7001 172.22.4.4:7000 172.22.4.4:7001 --cluster-replicas 1
# 每个主节点带1个从节点（提示输入yes确认槽分配）

2. 验证集群状态
# 连接任意节点查看集群信息
/usr/local/bin/redis-cli -h 172.22.4.2 -p 7000 cluster info

# 查看节点角色
/usr/local/bin/redis-cli -h 172.22.4.2 -p 7000 cluster nodes

# 写入数据，观察自动重定向
/usr/local/bin/redis-cli -c -h 172.22.4.2 -p 7000 set user:1 "Alice"  
/usr/local/bin/redis-cli -c -h 172.22.4.2 -p 7000 get user:1  
如果返回MOVED错误，说明数据被分配到其他节点，试试用-c参数启动客户端！

3. 模拟主节点故障
# 关闭一个主节点（如172.22.4.2:7000）
/usr/local/bin/redis-cli -h 172.22.4.2 -p 7000 shutdown

# 观察从节点是否自动升级为主节点
/usr/local/bin/redis-cli -h 172.22.4.2 -p 7001 cluster nodes

三. redis集群常用命令
# 查看集群槽分配
redis-cli -h <IP> -p <PORT> cluster slots

# 动态添加新节点
redis-cli --cluster add-node <新节点IP:PORT> <现有节点IP:PORT>

# 修复槽分配（节点重启后）
redis-cli --cluster fix <问题节点IP:PORT>




