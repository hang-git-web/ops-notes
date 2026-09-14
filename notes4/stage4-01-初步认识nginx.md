# Nginx 学习笔记

## 1. Nginx 是什么

Nginx 是一款**超快、超轻量级**的 Web 服务器，同时也是一个功能强大的**反向代理服务器**和**负载均衡器**。

## 2. Nginx 有什么作用

让网站**更快、更稳、更省资源**。

## 3. Nginx 的优点

- **并发高**：能轻松应对大量并发连接
- **配置灵活**：模块化设计，按需启用功能
- **支持反向代理**：请求转发给后端服务处理
- **支持负载均衡**：将压力分摊到多台服务器

## 4. Nginx 的常见使用场景

| 场景 | 说明 |
| --- | --- |
| 静态资源服务 | 图片、HTML、CSS 等直接由 Nginx 返回 |
| 反向代理 | 请求由 Nginx 中转给后端（如 PHP、Node.js） |
| 负载均衡 | 一个后端吃不消？Nginx 帮你平均分担压力 |
| 动静分离 | 动态请求交给后端，静态资源由 Nginx 自己处理 |
| HTTPS | 配置 SSL 证书，实现网站加密访问 |

## 5. Nginx 的架构组件

1. **Worker Processes（工人进程）**：真正处理请求的进程，数量越多并发能力越强
2. **Master Process（主控进程）**：负责管理"工人"
3. **配置文件**：`/etc/nginx/nginx.conf`，相当于 Nginx 的"指挥大纲"

## 6. Nginx 的安装

### 安装（以 CentOS 为例）

```bash
# 安装 EPEL 源
yum install -y epel-release

# 安装 Nginx
yum install -y nginx
```

### 启动服务

```bash
# 启动 Nginx
systemctl start nginx

# 设置开机自启
systemctl enable nginx
```

### 验证

浏览器访问服务器 IP，看到 Nginx 欢迎页面即安装成功。

## 7. nginx.conf 配置文件讲解

```nginx
worker_processes 1;          # 启动的工人数量，越多并发越强

events {
    worker_connections 1024; # 每个工人最多同时处理的连接数
}

http {
    include       mime.types;

    server {
        listen       80;              # 监听端口，默认 80
        server_name  localhost;       # 服务器域名或 IP

        location / {                  # location /：设置访问路径
            root   /usr/share/nginx/html;   # 静态资源的根目录
            index  index.html index.htm;    # 默认首页文件
        }
    }
}
```

**关键配置项说明：**

| 配置项 | 作用 |
| --- | --- |
| `worker_processes` | 启动多少个 worker 进程，决定并发处理能力 |
| `worker_connections` | 单个 worker 最多同时处理的连接数 |
| `listen` | 监听端口，默认 80 |
| `server_name` | 配置的域名或服务器标识 |
| `location` | 设置不同访问路径的匹配规则 |
| `root` | 指定静态资源存放的根目录 |
| `index` | 指定默认首页文件 |

## 8. 常用 Nginx 命令总结

| 用途 | 命令 |
| --- | --- |
| 启动服务 | `systemctl start nginx` |
| 强制停止 | `nginx -s stop` / `systemctl stop nginx` |
| 优雅退出 | `nginx -s quit` |
| 重新加载配置 | `nginx -s reload` |
| 检查配置是否有语法错误 | `nginx -t` |
| 重启服务 | `systemctl restart nginx` |
| 查看运行状态 | `systemctl status nginx` |

> 小提示：
> - `nginx -s quit` 会等当前请求处理完再退出，比 `stop` 更温柔；
> - 修改配置文件后建议先 `nginx -t` 检查语法，再用 `nginx -s reload` 热加载，而不是直接重启；
> - 生产环境优先使用 `systemctl` 系列命令管理服务。

## 9. Nginx 小实战：部署一个静态网站

### 第 1 步：创建页面

```bash
echo "<h1>Hello Nginx!</h1>" > /usr/share/nginx/html/index.html
```

### 第 2 步：重启服务

```bash
systemctl restart nginx
```

### 第 3 步：验证

在浏览器中访问服务器 IP，成功看到页面即部署完成。

## 10. Nginx 实战：创建自己的虚拟主机

**虚拟主机的作用**：一台 Nginx 服务器只用一个 IP，却能同时跑多个网站。常用方式有两种：**基于域名的虚拟主机**（不同域名 → 不同网站）和**基于端口的虚拟主机**（不同端口 → 不同网站）。下面先做最常用的“基于域名”。

### 第 1 步：创建自己的站点目录和页面

```bash
# 建一个属于你自己的站点目录
mkdir -p /var/www/mysite

# 写入测试页面
echo "<h1>这是我的第一个虚拟主机!</h1>" > /var/www/mysite/index.html

# 给目录设置权限，避免 Nginx 读不到
chown -R nginx:nginx /var/www/mysite
```

### 第 2 步：编写虚拟主机配置文件

在 Nginx 的 `conf.d` 目录里新建一个配置文件，一个文件代表一个站点：

```bash
vim /etc/nginx/conf.d/mysite.conf
```

```nginx
server {
    listen       80;                    # 监听 80 端口
    server_name  mysite.com www.mysite.com;   # 你的域名，多个用空格隔开

    location / {
        root   /var/www/mysite;         # 指向第 1 步建的目录
        index  index.html index.htm;    # 默认首页
    }
}
```

> 提示：CentOS 的 `/etc/nginx/nginx.conf` 里默认有
> `include /etc/nginx/conf.d/*.conf;`
> 所以只要文件放进 `conf.d`，Nginx 就会自动加载，不用改动主配置文件。

### 第 3 步：本机测试域名（改 hosts）

如果暂时没有真实域名，先把域名指向自己这台服务器：

- Linux：`vim /etc/hosts`
- Windows：编辑 `C:\Windows\System32\drivers\etc\hosts`（需要管理员权限）

在文件末尾加一行：

```text
192.168.1.100  mysite.com www.mysite.com
```

`192.168.1.100` 换成你服务器的实际 IP。

### 第 4 步：校验配置并生效

```bash
# 1. 测试配置文件语法是否正确
nginx -t

# 2. 语法正确后热加载配置（不用重启，不影响现有服务）
systemctl reload nginx
```

### 第 5 步：验证虚拟主机是否生效

```bash
# 方式一：命令行测试
curl http://mysite.com

# 方式二：浏览器直接访问
# http://mysite.com
```

能看到“这是我的第一个虚拟主机!”就说明配置成功。

### 附加：一个服务器建多个虚拟主机

继续重复上面步骤，在 `conf.d` 里再建第二个文件：

```nginx
# /etc/nginx/conf.d/shop.conf
server {
    listen       80;
    server_name  shop.com;

    location / {
        root   /var/www/shop;
        index  index.html;
    }
}
```

只要访问的域名不同，Nginx 就会自动把它送到对应的站点目录。

### 快速验证：没有域名时用“端口虚拟主机”

如果不方便改 hosts，也可以让第二个站点监听其他端口：

```nginx
# /etc/nginx/conf.d/shop.conf
server {
    listen       8080;
    server_name  localhost;

    location / {
        root   /var/www/shop;
        index  index.html;
    }
}
```

```bash
mkdir -p /var/www/shop
echo "<h1>第二个站点(8080端口)</h1>" > /var/www/shop/index.html
nginx -t && systemctl reload nginx
```

浏览器访问 `http://服务器IP:8080` 即可看到第二个站点。

### 常见问题

| 现象 | 原因与解决 |
| --- | --- |
| 访问域名却显示 Nginx 欢迎页 | 你的域名没匹配到 server_name，检查 hosts 和配置文件；或把默认欢迎页的 server 块停用 |
| `nginx -t` 报错 | 配置文件语法有问题，按提示检查分号、大括号是否完整 |
| 外网访问 8080 不通 | 防火墙未放行：`firewall-cmd --add-port=8080/tcp --permanent && firewall-cmd --reload` |