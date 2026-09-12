# Nginx 优化点(高并发)

## 0. 优化前的准备

先确认 Nginx 是 yum 安装还是源码编译安装,两者配置文件路径不同:

```bash
nginx -V 2>&1 | grep -E "conf-path|prefix"
```

- yum 安装:`/etc/nginx/nginx.conf`
- 源码编译安装:`/usr/local/nginx/conf/nginx.conf`

```bash
cd /etc/nginx
vim nginx.conf
```

## 一、优化点速查

| # | 优化项 | 配置 | 作用 | 生效位置 |
| --- | --- | --- | --- | --- |
| 1 | 工作进程数 | `worker_processes auto;` | 等同 CPU 核心数,充分利用多核 CPU | 全局块 |
| 2 | CPU 亲和性 | `worker_cpu_affinity auto;` | 进程绑定固定 CPU,减少缓存失效和上下文切换 | 全局块 |
| 3 | 最大连接数 | `worker_connections 65535;` | 单个 worker 允许的最大并发连接数 | events 块 |
| 4 | 网络模型 | `use epoll;` | Linux 下高效处理事件,优于 select | events 块 |
| 5 | 文件描述符 | `worker_rlimit_nofile 65535;` | 提高 worker 可打开的文件句柄上限 | 全局块 |
| 6 | 静态缓存 | `location ~* \.(jpg\|png\|css)$ { expires 30d; }` | 设置浏览器缓存过期时间,减少重复请求 | location 块 |
| 7 | Gzip 压缩 | `gzip on; gzip_comp_level 5; gzip_types ...;` | 压缩文本传输,节省带宽;级别 4~5,过高耗 CPU | http 块 |
| 8 | 文件读取 | `sendfile on; tcp_nopush on;` | 高效读取文件,减少网络小包 | http 块 |
| 9 | 隐藏版本 | `server_tokens off;` | 不暴露版本号,减少被针对性攻击 | http 块 |

## 二、优化后的 nginx.conf

```nginx
# ==================== 全局块 ====================
user  nginx;

worker_processes      auto;      # 1. 工作进程数 = CPU 核心数
worker_cpu_affinity   auto;      # 2. CPU 亲和性
worker_rlimit_nofile  65535;     # 5. 文件描述符上限

error_log  /var/log/nginx/error.log warn;
pid        /run/nginx.pid;

# ==================== events 块 ====================
events {
    use epoll;                   # 4. Linux 高效事件模型
    worker_connections 65535;    # 3. 单 worker 最大并发连接数
}

# ==================== http 块 ====================
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    # 8. 文件读取优化
    sendfile    on;
    tcp_nopush  on;

    # 9. 隐藏版本号
    server_tokens off;

    # 7. Gzip 压缩
    gzip              on;
    gzip_comp_level   5;
    gzip_types        text/plain text/css application/json;

    server {
        listen       80;
        server_name  www.example.com;

        root   /data/www/site1;
        index  index.php index.html;

        # 6. 静态资源缓存
        location ~* \.(jpg|jpeg|png|gif|css|js|ico)$ {
            expires 30d;
        }

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

        access_log /var/log/nginx/site1_access.log main;
        error_log  /var/log/nginx/site1_error.log warn;
    }
}
```

## 三、检查并生效

```bash
nginx -t          # 检查语法
nginx -s reload   # 平滑重载,不中断连接
# 或
systemctl reload nginx
```