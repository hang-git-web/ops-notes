# Nginx 配置文件详解

Nginx 的配置文件通常位于 `/etc/nginx/nginx.conf`，这是 Nginx 的主配置文件。Nginx 配置文件采用模块化结构，主要分为以下几个部分：

1. 全局块（Global Block）
2. 事件块（Events Block）
3. HTTP 块（HTTP Block）

## 1. 全局块（Global Block）

全局块位于配置文件的最顶部，用于设置 Nginx 的全局配置选项。它通常包括日志配置、用户权限、工作进程等。

### 全局块示例

```nginx
# 全局块示例
user nginx;            # 设置运行 Nginx 的用户
worker_processes 1;    # 设置工作进程数，通常设置为 CPU 核心数
pid /var/run/nginx.pid;  # 存储进程 ID 的文件位置
error_log /var/log/nginx/error.log warn;  # 设置错误日志的路径和日志级别
```

## 2. 事件块（Events Block）

事件块主要控制 Nginx 的工作进程如何处理连接。这里设置了 `worker_connections`，它表示每个工作进程能够同时处理的最大连接数。

### 事件块示例

```nginx
# 事件块示例
events {
    worker_connections 1024;  # 每个工作进程的最大连接数
}
```

## 3. HTTP 块（HTTP Block）

HTTP 块是 Nginx 配置中最重要的部分，它包含了服务器的核心配置。HTTP 块内部可以包含多个服务器块，定义了虚拟主机的配置。HTTP 块中的配置影响整个 HTTP 服务的行为。

### 3.1 配置虚拟主机

在 HTTP 块中，我们可以配置多个虚拟主机，每个虚拟主机用 `server` 块表示。一个 `server` 块可以包含 `listen`（监听端口）、`server_name`（域名）、`location`（请求路径）等配置。

```nginx
http {
    # 默认配置
    server {
        listen 80;              # 监听 80 端口
        server_name example.com; # 设置虚拟主机的域名

        location / {
            root /usr/share/nginx/html;  # 设置网站根目录
            index index.html index.htm;  # 设置默认首页文件
        }

        # 错误页面配置
        error_page 404 /404.html;
        location = /404.html {
            root /usr/share/nginx/html;
        }
    }
}
```

### 3.2 反向代理配置

Nginx 经常用作反向代理服务器，将客户端的请求转发到后台的应用服务器。下面是反向代理的配置示例：

```nginx
http {
    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend_server;  # 转发请求到后台服务器
            proxy_set_header Host $host;       # 保持客户端原始请求头
            proxy_set_header X-Real-IP $remote_addr;  # 传递客户端 IP 地址
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

这里，`proxy_pass` 用于指向后台服务器，所有的客户端请求都会通过 Nginx 转发到 `http://backend_server`。

### 3.3 负载均衡配置

Nginx 还可以配置为负载均衡器，将流量分配到多个后端服务器上。Nginx 支持多种负载均衡算法，比如轮询、最少连接等。

```nginx
http {
    upstream backend_servers {
        server backend1.example.com;
        server backend2.example.com;
        server backend3.example.com;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend_servers;  # 将请求转发到负载均衡池
        }
    }
}
```

这里，`upstream` 用于定义一个服务器池，Nginx 会根据配置的负载均衡算法将请求转发到池中的服务器。

### 3.4 HTTPS 配置

Nginx 还支持 HTTPS 协议，需要配置 SSL 证书和相关的加密设置。

```nginx
http {
    server {
        listen 443 ssl;  # 监听 443 端口，并启用 SSL
        server_name example.com;

        ssl_certificate /etc/ssl/certs/example.com.crt;       # SSL 证书路径
        ssl_certificate_key /etc/ssl/private/example.com.key; # SSL 私钥路径

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
```

此配置启用 SSL，并指向 SSL 证书和私钥。

---

# Nginx 常见命令

1. **启动 Nginx**

```bash
systemctl start nginx
```

2. **停止 Nginx**

```bash
systemctl stop nginx
```

3. **重启 Nginx**

```bash
systemctl restart nginx
```

4. **检查 Nginx 配置是否正确**

```bash
nginx -t
```

5. **重新加载配置（无需停止服务）**

```bash
systemctl reload nginx
```