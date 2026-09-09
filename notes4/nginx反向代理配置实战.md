# Nginx 反向代理实战

## 一、什么是反向代理？

用户请求 --> Nginx --> 后端服务

## 二、实战准备

- Nginx 是网关
- 后端用 Python 启动一个 Web 服务

### 1. 安装 Python（如果没有）

```bash
yum install -y python3
```

### 2. 启动一个测试服务（监听 5000 端口）

```bash
cat << EOF > /data/testapp.py
from http.server import BaseHTTPRequestHandler, HTTPServer

class HelloHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(b"<h1>Hello from backend server!</h1>")

server = HTTPServer(('0.0.0.0', 5000), HelloHandler)
print("Serving on port 5000...")
server.serve_forever()
EOF
```

后台运行：

```bash
nohup python3 /data/testapp.py &
```

## 三、配置反向代理

### 1. 新建配置文件

```bash
vim /etc/nginx/conf.d/proxy_demo.conf
```

写入以下内容：

```nginx
server {
    listen 80;
    server_name www.proxytest.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**参数解释：**

- `proxy_pass`：把请求转发到这个地址
- `proxy_set_header`：保留用户原始信息（如 IP）

### 2. 修改 hosts 文件测试（仅测试使用）

```bash
echo "127.0.0.1 www.proxytest.com" >> /etc/hosts
```

### 3. 检查并重启 Nginx

```bash
nginx -t
systemctl reload nginx
```

## 四、测试效果

浏览器访问：http://www.proxytest.com

看到后端 Python 返回的 `Hello from backend server!`，说明反向代理配置成功。

## 五、更多反向代理技巧

### 1. 反向代理多个后端（负载均衡）

```nginx
upstream myapp {
    server 127.0.0.1:5000;
    server 127.0.0.1:5001;
}

server {
    listen 80;
    server_name www.balancer.com;

    location / {
        proxy_pass http://myapp;
    }
}
```

### 2. 反向代理静态 + 动态混合站点

```nginx
server {
    listen 80;
    server_name www.mixsite.com;

    # 静态资源由 Nginx 直接返回
    location /static/ {
        alias /data/site/;
        autoindex on;   # 开启目录列表，可视化效果更明显
    }

    # 其他动态请求转发给后端
    location / {
        proxy_pass http://127.0.0.1:5000;
    }
}
```

这里使用的是 **alias** 而不是 root。

**root 与 alias 的区别：**

- `root` 会在后面拼上请求的整个路径（包含 location 匹配部分）
- `alias` 不会拼接 location 前缀，而是完全替换掉

例如请求 `/static/test.png`：

- `root /data/site;` → 实际查找 `/data/site/static/test.png`
- `alias /data/site/;` → 实际查找 `/data/site/test.png`




