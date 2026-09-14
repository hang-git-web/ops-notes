# Nginx 源码编译安装

## 一、安装依赖包

在编译 Nginx 之前，必须确保你的系统已经安装了编译所需的开发工具和依赖库。执行以下命令安装它们：

```bash
# 安装开发工具包
sudo yum groupinstall "Development Tools" -y

# 安装编译 Nginx 时需要的库文件
sudo yum install -y pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

**解释：**

- **pcre**：正则表达式库，Nginx 使用它来处理 URL 匹配。
- **zlib**：用于文件压缩的库，Nginx 用它来支持 gzip 压缩。
- **openssl**：提供加密功能，Nginx 通过它来支持 SSL/TLS。

## 二、下载 Nginx 源码包

Nginx 的源代码可以从官方网站下载。执行以下命令进入你希望存放源码的目录：

```bash
# 进入源码存放的目录
cd /usr/local/src

# 下载最新版本的 Nginx 源码包
sudo wget http://nginx.org/download/nginx-1.24.0.tar.gz
```

如果需要其他版本，可以去 [Nginx 官方下载页面](http://nginx.org/en/download.html) 查找相应版本的下载链接。

## 三、解压源码包

```bash
# 解压下载的 Nginx 源码包
sudo tar -zxvf nginx-1.24.0.tar.gz
```

执行后，源码包将会被解压到当前目录。

## 四、配置 Nginx 编译选项

解压后，进入 Nginx 源码目录并开始配置编译选项。这里可以指定安装路径、启用模块等功能。

```bash
# 进入解压后的目录
cd nginx-1.24.0

# 配置编译选项
sudo ./configure \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module \   # 启用 SSL 模块
    --with-http_v2_module \    # 启用 HTTP/2 模块
    --with-pcre                # 启用 PCRE 正则库
```

**解释：**

- **--prefix=/usr/local/nginx**：指定 Nginx 安装目录，默认是 `/usr/local/nginx`。
- **--with-http_ssl_module**：启用 SSL 模块，支持 HTTPS 加密。
- **--with-http_v2_module**：启用 HTTP/2 模块，提高性能。
- **--with-pcre**：启用 PCRE 库，支持正则表达式功能。

执行此命令时，系统会检查依赖库的存在，确保可以编译。

## 五、编译安装 Nginx

```bash
# 编译 Nginx
sudo make

# 安装 Nginx
sudo make install
```

此命令会将 Nginx 安装到你在配置时指定的路径 `/usr/local/nginx`。如果没有指定，默认会安装到 `/usr/local/nginx`。

## 六、配置启动脚本

```bash
# 创建 Nginx 启动脚本
sudo vim /etc/init.d/nginx
```

在打开的编辑器中，输入以下内容：

```bash
#!/bin/bash

# Nginx 启动脚本

case "$1" in
start)
    /usr/local/nginx/sbin/nginx
    ;;
stop)
    /usr/local/nginx/sbin/nginx -s stop
    ;;
restart)
    /usr/local/nginx/sbin/nginx -s reload
    ;;
*)
    echo "Usage: $0 {start|stop|restart}"
    exit 1
    ;;
esac
exit 0
```

保存并退出编辑器，然后给脚本添加执行权限：

```bash
# 添加执行权限
sudo chmod +x /etc/init.d/nginx

# 启动 Nginx
sudo /etc/init.d/nginx start
```

> 提示：上面的 `restart` 实际执行的是 `reload`。如果你希望脚本真正做到“先停再启”，可以把 restart 分支改成：
>
> ```bash
> restart)
>     /usr/local/nginx/sbin/nginx -s stop
>     sleep 1
>     /usr/local/nginx/sbin/nginx
>     ;;
> ```

## 七、验证 Nginx 是否启动

确认 Nginx 是否成功启动，可以使用以下命令查看进程：

```bash
# 查看 Nginx 进程
ps -ef | grep nginx
```

Nginx 启动后，打开浏览器，访问你服务器的 IP 地址。如果你看到 Nginx 的欢迎页面，表示 Nginx 已经成功安装。

可以通过访问 `http://your_server_ip` 来查看是否成功。

## 八、卸载 Nginx

### 1. 停止 Nginx 服务

```bash
sudo /etc/init.d/nginx stop
```

### 2. 删除安装目录

```bash
sudo rm -rf /usr/local/nginx
```

### 3. 删除启动脚本

```bash
sudo rm -f /etc/init.d/nginx
```