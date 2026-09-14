redis 的持久化
什么是持久化?
持久化（Persistence）‌：
定义‌：将内存中的数据保存到磁盘，防止服务宕机后数据丢失。
类比‌：游戏存档（内存是游戏进程，磁盘是存档文件）
redis持久化的两种方案:
RDB（Redis Database）‌：
原理‌：定时生成内存快照（Snapshot），保存为.rdb文件。
优点‌：文件小、恢复快；‌缺点‌：可能丢失最近一次快照后的数据。
AOF（Append Only File）‌：
原理‌：记录所有写操作命令，重启时重放命令恢复数据。
优点‌：数据完整性高；‌缺点‌：文件大、恢复慢。

持久化操作:
一. 安装Redis（新手必做）
# 安装依赖
sudo yum install -y gcc tcl wget
# 下载并编译Redis
wget http://download.redis.io/releases/redis-7.2.0.tar.gz  
tar -zxvf redis-7.2.0.tar.gz  
cd redis-7.2.0 && make && sudo make install
# 创建配置文件目录
sudo mkdir -p /etc/redis  
sudo cp redis.conf /etc/redis/
输入redis-server --version，看版本号确认安装成功！

二. 修改redis持久化的配置文件

RDB方式(不要把注释复制到配置文件中,会出现报错)
1. 修改配置文件
   sudo vim /etc/redis/redis.conf
# 启用RDB（默认已开启）
save 900 1       # 900秒内至少1次修改则触发快照  
save 300 10      # 300秒内至少10次修改  
save 60 10000    # 60秒内至少10000次修改  
dir /var/lib/redis  # 持久化文件存储目录  
dbfilename dump.rdb # RDB文件名

2. 创建数据目录并启动服务
   sudo mkdir -p /var/lib/redis  
   sudo chown -R $USER:$USER /var/lib/redis  # 赋权给当前用户

# 启动Redis（指定配置文件）
redis-server /etc/redis/redis.conf

# 测试写入数据
redis-cli set test_rdb "Hello, RDB!"

3. 手动触发RDB快照
# 执行SAVE命令（阻塞式，生产慎用）
redis-cli save

# 或执行BGSAVE命令（后台异步执行）
redis-cli bgsave

# 检查RDB文件
ls -l /var/lib/redis/dump.rdb


-----AOF持久化配置
1. 修改配置文件
   sudo vim /etc/redis/redis.conf  
   appendonly yes              # 开启AOF  
   appendfilename "appendonly.aof"  # AOF文件名  
   appendfsync everysec        # 每秒同步一次（平衡性能与安全）

2. 重启Redis并测试

# 重启服务
redis-cli shutdown  
redis-server /etc/redis/redis.conf

# 写入数据
redis-cli set test_aof "Hello, AOF!"

# 检查AOF文件
tail -f /var/lib/redis/appendonly.aof

三. 模拟持久化故障和恢复
1. 模拟宕机恢复

# 强制关闭Redis
sudo pkill redis-server

# 重新启动
redis-server /etc/redis/redis.conf

# 检查数据是否存在
redis-cli get test_rdb  
redis-cli get test_aof

2. 修复损坏的持久化文件

# 使用redis-check工具
redis-check-rdb /var/lib/redis/dump.rdb  
redis-check-aof --fix /var/lib/redis/appendonly.aof

高级配置‌：
# 混合持久化（Redis 4.0+）
aof-use-rdb-preamble yes