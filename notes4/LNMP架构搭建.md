# LNMP 架构实战

## LNMP 架构概述

LNMP 是一种非常流行的 Web 服务架构，它包括 Linux 操作系统、Nginx Web 服务器、MySQL 数据库和 PHP（通过 PHP-FPM 运行）。在此架构中，Nginx 负责处理 HTTP 请求，PHP-FPM 用于动态内容的解析，MySQL 作为数据库管理系统用于存储数据。

- **Linux**：提供操作系统支持，建议使用 CentOS 或 Ubuntu。
- **Nginx**：高性能 Web 服务器，负责 HTTP 请求的处理。
- **MySQL**：关系型数据库管理系统，负责存储和管理数据。
- **PHP**：动态脚本语言，负责处理与数据库交互的动态内容。

## 环境准备

### 1. 安装和配置 Nginx

**安装 EPEL 仓库和 Nginx**

```bash
yum install -y epel-release
yum install -y nginx
```

**启动 Nginx 服务**

```bash
systemctl start nginx
```

**设置 Nginx 开机自启动**

```bash
systemctl enable nginx
```

**配置 Nginx**

编辑 Nginx 配置文件，以支持 PHP 文件的处理。我们需要修改 `server` 配置块，添加 PHP 解析的相关配置：

```bash
vim /etc/nginx/nginx.conf
```

在 `server` 块中添加如下配置，确保 Nginx 能与 PHP 配合工作：

```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;  # PHP-FPM 监听的端口
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME /usr/share/nginx/html$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

**重启 Nginx 服务使配置生效**

```bash
systemctl restart nginx
```

### 2. 安装和配置 MySQL

**安装 MySQL 8.0 的仓库**

```bash
yum localinstall https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
```

**安装 MySQL 8.0**

```bash
yum install -y mysql-community-server
```

**启动 MySQL 服务**

```bash
systemctl start mysqld
```

**设置 MySQL 开机自启动**

```bash
systemctl enable mysqld
```

**获取 MySQL 临时密码并登录**

```bash
grep 'temporary password' /var/log/mysqld.log
```

使用该临时密码登录 MySQL：

```bash
mysql -u root -p
```

**修改 root 密码并配置安全性**

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Qiubai@123';
```

执行安全配置，去除测试用户、关闭远程 root 登录等：

```bash
mysql_secure_installation
```

### 3. 安装和配置 PHP

PHP 需要与 Nginx 配合使用，使用 PHP-FPM（FastCGI Process Manager）处理 PHP 请求。

**安装 PHP、PHP-FPM 和 MySQL 扩展**

```bash
yum install -y php php-fpm php-mysqlnd
```

**配置 PHP-FPM**

PHP-FPM 配置文件在 `/etc/php-fpm.d/www.conf`，打开该文件并编辑以下配置：

```bash
vim /etc/php-fpm.d/www.conf
```

在配置文件中，找到并修改 `user` 和 `group` 为 `nginx`，确保 PHP-FPM 进程以 Nginx 用户运行：

```ini
user = nginx
group = nginx
```

**启动 PHP-FPM 服务**

```bash
systemctl start php-fpm
systemctl enable php-fpm
```

### 4. 测试 LNMP 环境

**创建 PHP 测试文件**

我们将在 `/usr/share/nginx/html/` 目录下创建一个 `info.php` 文件，用来测试 PHP 是否能正常工作：

```bash
# 创建 info.php 文件
echo "<?php phpinfo(); ?>" > /usr/share/nginx/html/info.php
```

**访问测试页面**

在浏览器中输入你的服务器的 IP 地址或域名，访问：

```text
http://your_server_ip/info.php
```

如果 PHP 配置成功，你应该能够看到 PHP 的配置信息页面。