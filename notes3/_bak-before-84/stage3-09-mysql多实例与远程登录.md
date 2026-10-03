# MySQL 多实例

**介绍**：在一台物理服务器上运行多个独立的 MySQL 服务。

每个实例拥有：

- 独立的数据目录、配置文件、端口号、日志文件
- 独立的进程和系统资源分配（CPU、内存）

---

## 一、多实例部署

### 1. 安装 MySQL 8 源

```bash
# 下载 MySQL 8 的 Yum 源
wget http://dev.mysql.com/get/mysql84-community-release-el7-1.noarch.rpm

# 安装 Yum 源
sudo yum localinstall mysql84-community-release-el7-1.noarch.rpm

# 安装 MySQL 服务器
sudo yum install mysql-community-server
```

### 2. 启动第一个 MySQL 实例并进行初始配置

```bash
# 启动第一个 MySQL 实例
sudo systemctl start mysqld

# 设置第一个 MySQL 实例开机自启
sudo systemctl enable mysqld

# 获取第一个实例的初始临时密码
sudo grep 'temporary password' /var/log/mysqld.log

# 登录第一个 MySQL 实例
mysql -u root -p
```

修改 root 用户密码：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Qiubai@123';
```

### 3. 创建第二个实例

```bash
# 创建第二个实例的数据目录
sudo mkdir -p /var/lib/mysql2
sudo chown -R mysql:mysql /var/lib/mysql2

# 创建第二个实例的日志目录
sudo mkdir -p /var/log/mysql2
sudo chown -R mysql:mysql /var/log/mysql2
```

创建第二个实例的配置文件：

```bash
sudo vi /etc/my.cnf.d/mysql2.cnf
```

添加以下内容：

```ini
[mysqld2]

# 第二个实例的端口，避免与第一个实例冲突
port = 3307

# 第二个实例的数据目录
datadir = /var/lib/mysql2

# 第二个实例的日志文件路径
log-error = /var/log/mysql2/error.log

# 第二个实例的套接字文件路径
socket = /var/lib/mysql2/mysql.sock

# 服务器 ID，需要唯一
server-id = 2
```

初始化第二个 MySQL 实例：

```bash
sudo mysqld --defaults-file=/etc/my.cnf.d/mysql2.cnf --initialize --user=mysql
```

执行该命令后，会生成一个临时密码，使用以下命令获取：

```bash
sudo grep 'temporary password' /var/log/mysql2/error.log
```

创建第二个 MySQL 实例的服务文件，使用 systemd 管理：

```bash
sudo vi /etc/systemd/system/mysqld2.service
```

添加以下内容：

```ini
[Unit]
Description=MySQL Server 2
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql
ExecStart=/usr/sbin/mysqld --defaults-file=/etc/my.cnf.d/mysql2.cnf
LimitNOFILE = 5000
Restart=on-failure
RestartPreventExitStatus=1
PrivateTmp=false
```

启动第二个 MySQL 实例并设置开机自启：

```bash
# 重新加载 systemd 管理器配置
sudo systemctl daemon-reload

# 启动第二个 MySQL 实例
sudo systemctl start mysqld2

# 设置第二个 MySQL 实例开机自启
sudo systemctl enable mysqld2
```

登录第二个 MySQL 实例并修改密码：

```bash
mysql -u root -p -P 3307 -S /var/lib/mysql2/mysql.sock
```

输入之前获取的临时密码，登录成功后，修改 root 用户密码：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Qiubai@123';
```

---

## 二、验证

### 1. 验证两个实例是否可用

```bash
# 验证第一个实例
mysql -u root -p -P 3306

# 验证第二个实例
mysql -u root -p -P 3307 -S /var/lib/mysql2/mysql.sock
```

两个实例设置开机自启：

```bash
systemctl enable mysqld
systemctl enable mysqld2
```

### 2. 验证实例运行状态

```bash
# 查看进程
ps aux | grep mysqld | grep -v grep

# 检查端口监听
netstat -tulnp | grep mysql
```

### 3. 配置账号远程访问

```sql
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'Remote@2023';
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%';
```

### 4. 远程连接要过的五关

| 关卡 | 检查什么 | 常见问题 |
| --- | --- | --- |
| 1. 账号 | `remote_user` 在目标实例里存在且来源匹配 | 账号建在了实例1，你连的是 3307 |
| 2. 实例在监听 | `bind-address` 是否允许对外 | 被设成 `127.0.0.1` 或 `skip-networking` 就只许本机 |
| 3. 端口被监听 | 3307 是否真的在 listen | 实例没启动、端口冲突 |
| 4. 系统防火墙 / SELinux | 3307 是否放行、SELinux 是否允许 | 只放行了 3306，漏掉 3307 |
| 5. 网络与云安全组 | 路由是否可达、云平台入方向规则 | 云服务器安全组没放行 |

---

## 三、熟悉实例操作的常用命令

| 操作 | 命令 |
| --- | --- |
| 启动指定实例 | `systemctl start mysqld` |
| 停止指定实例 | `systemctl stop mysqld2` |
| 查看日志 | `tail -f /var/log/mysql2/error.log` |
| 连接指定实例 | `mysql -uroot -p -P 3307 -h 127.0.0.1` |
| 备份指定实例 | `mysqldump -uroot -p -S /var/lib/mysql2/mysql.sock --all-databases > backup.sql` |

---

## 四、多实例注意事项与故障排查

| 问题 | 解决方案 |
| --- | --- |
| 端口冲突 | 检查 `netstat -tulnp` 确认端口未被占用 |
| 数据目录权限错误 | `chown -R mysql:mysql /var/lib/mysql2` |
| 服务启动失败 | 查看日志 `/var/log/mysql2/error.log` |