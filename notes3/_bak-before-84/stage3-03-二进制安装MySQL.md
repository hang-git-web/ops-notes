一. 先检查一下是否已经安装mysql
如果安装，需要卸载
yum remove mysql-server

安装mysql的依赖包
# Ubuntu/Debian
sudo apt update  
sudo apt install libaio1 libnuma-dev -y
# CentOS/RHEL
sudo yum install libaio numactl-libs -y  
作用说明‌：
libaio：支持异步 I/O 操作，MySQL 运行必需。
numactl：优化多核 CPU 内存分配（非必需，但建议安装）

二. 创建专用用户和组
sudo groupadd mysql  
sudo useradd -r -g mysql -s /bin/false mysql
参数解释‌：
-r：创建系统用户（无登录权限）
-g mysql：指定用户组为 mysql
-s /bin/false：禁止该用户登录系统

三. 下载mysql的二进制安装包
1.下载二进制包
进入临时目录  
cd /tmp
下载（以 5.7.42 为例，可从官网替换最新版本链接）  
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.42-linux-glibc2.12-x86_64.tar.gz
下载完成之后,验证文件的完整性,不然有可能会存在病毒等内容，存在一定的风险
# 验证文件完整性（可选）
md5sum mysql-5.7.42-linux-glibc2.12-x86_64.tar.gz
# 对比官网提供的 MD5 值：https://dev.mysql.com/downloads/mysql/5.7.html

2.解压并移动到安装目录
解压  
tar -zxvf mysql-5.7.42-linux-glibc2.12-x86_64.tar.gz
# 创建安装目录（通常为 /usr/local/mysql）
sudo mkdir -p /usr/local/mysql
# 移动文件
sudo mv mysql-5.7.42-linux-glibc2.12-x86_64/* /usr/local/mysql
# 清理临时文件
rm -rf mysql-5.7.42-linux-glibc2.12-x86_64*

四. 初始化MySQL数据库
1. 配置环境变量‌

# 临时生效（当前会话）
export PATH=/usr/local/mysql/bin:$PATH

# 永久生效（添加到 ~/.bashrc 或 /etc/profile）
echo 'export PATH=/usr/local/mysql/bin:$PATH' | sudo tee -a /etc/profile  
source /etc/profile

2. 创建数据目录并设置权限‌
   sudo mkdir -p /data/mysql  
   sudo chown -R mysql:mysql /data/mysql  
   sudo chmod 750 /data/mysql

3. 初始化数据库‌
   cd /usr/local/mysql  
   sudo bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql

关键输出‌：
[Note] A temporary password is generated for root@localhost: Xq3&k9o!l2+_  # 保存此临时密码！

4. 创建配置文件my.cnf
   sudo tee /etc/my.cnf <<EOF  
   [mysqld]  
   basedir = /usr/local/mysql  
   datadir = /data/mysql  
   socket = /tmp/mysql.sock  
   port = 3306  
   log-error = /data/mysql/mysql-error.log  
   pid-file = /data/mysql/mysql.pid
# 安全配置
skip-name-resolve = 1  
symbolic-links = 0  
explicit_defaults_for_timestamp = 1
# 内存优化（根据服务器配置调整）
key_buffer_size = 256M  
max_allowed_packet = 64M  
EOF

5. 对mysql进行服务管理
   创建 Systemd 服务文件‌
   sudo tee /etc/systemd/system/mysql.service <<EOF  
   [Unit]  
   Description=MySQL Server  
   After=network.target  
   [Service]  
   User=mysql  
   Group=mysql  
   ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my.cnf  
   Restart=on-failure  
   [Install]  
   WantedBy=multi-user.target  
   EOF

6. 启动mysql服务器
   sudo systemctl daemon-reload  
   sudo systemctl start mysql  
   sudo systemctl enable mysql

五. 安全初始化
1. 修改 root 密码
   使用临时密码登录  
   mysql -u root -p
   输入临时密码后执行  
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewStrongPassword!';  
   FLUSH PRIVILEGES;  
   EXIT;

2. 运行安全脚本‌
   sudo mysql_secure_installation
   删除匿名用户：Y
   禁止 root 远程登录：Y
   删除测试数据库：Y
   刷新权限表：Y

3. 测试连接‌
   mysql -u root -p -e "SHOW DATABASES;"


六. 二进制安装mysql常见问题及解决方式
启动失败
查看错误日志：sudo tail -n 100 /data/mysql/mysql-error.log

忘记临时密码‌
删除数据目录 /data/mysql 重新初始化（慎用！会丢失数据）

权限不足‌		
检查目录权限：sudo chown -R mysql:mysql /usr/local/mysql /data/mysql

端口冲突‌		
修改 my.cnf 中的 port 并重启服务

