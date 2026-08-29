按数据模型分类：
关系型数据库：支持sql查询，mysql(轻量级项目),oracle(高可用，不差钱),postgreSQL(复杂查询需求多)
非关系型数据库：mangoDB(文档型数据库),redis(键值型)
时间序列数据库：适合服务器监控传感器
图数据库：节点和关系表示数据

决策树分析
是否需要强一致性和复杂事物
是：关系型数据

数据结构是否固定
是：关系型数据库

是否需要高速读写和低延迟
是：内存数据库(redis)

是否需要处理时间相关数据
是->时间序列数据库

是否涉及复杂关系网络
是->图数据库


Mysql安装
CentOs
1：安装
启用Mysql官方仓库
yum -y install https://repo.mysql.com/mysql80-community-release-el7-3.noarch.rpm

可切换到mysql5.7，默认8.0
[root@host1 ~]# yum install yum-utils-1.1.31-54.el7_8.noarch -y
禁用8.0
yum-config-manager --disable mysql80-community
启动5.7
yum-config-manager --enable mysql57-community

yum install mysql-server -y  #安装Mysql服务器
# 启动服务并设置开机自启
systemctl start mysqld  
systemctl enable mysqld

查看mysql初始密码
grep -i password /var/log/mysqld.log

修改密码
ALTER USER root@localhost identified by '新密码';

用新密码登录
mysql -uroot -p 

最终进入数据库，输入show databases；测试效果
如图images3下1.png


2：初始化安全配置
运行安全脚本-交互式脚本
退出数据库后输入：mysql_secure_installation  

是否启用密码强度检查？建议新手选择 N（避免复杂度要求）。
设置 root 密码‌：输入并确认密码（需牢记！）。
删除匿名用户‌：选择 Y（提高安全性）。
禁止 root 远程登录‌：选择 Y（仅允许本地登录）。
删除测试数据库‌：选择 Y（减少冗余）。
重新加载权限表‌：选择 Y（使配置生效）

3：配置远程访问
  修改 MySQL 绑定地址‌
   vim /etc/mysql/mysql.conf.d/mysqld.cnf  	# Ubuntu/Debian
   vim /etc/my.cnf							# CentOS/RHEL
   找到/添加 bind-address 并修改为：
   bind-address = 0.0.0.0  # 允许所有 IP 访问
   保存后重启服务：
   systemctl restart mysql    # Ubuntu/Debian  
   systemctl restart mysqld   # CentOS/RHEL

 创建远程访问用户
# 登录 MySQL（使用 root 密码）
mysql -u root -p

# 创建用户（示例：用户名为 remote_user，密码为 123456）
CREATE USER 'remote_user'@'%' IDENTIFIED BY '123456';  
如图images3下2.png  会提示密码复杂度不够

#授权
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%' WITH GRANT OPTION;  
FLUSH PRIVILEGES;    #刷新权限表
EXIT;
如图images3下3.png  会提示密码复杂度不够

然后别的安装了mysql的服务器就可以通过
执行 mysql -h部署服务器IP -uremote_user -pMysql@123命令来连接



