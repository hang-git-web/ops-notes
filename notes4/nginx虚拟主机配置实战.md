# Nginx 虚拟主机实战

## 一、什么是虚拟主机？

如果把一台服务器比作一座公寓楼，那“虚拟主机”就像是不同的房间号。

- 一台服务器跑多个网站（多个“房间”）
- 每个网站都有自己门牌（域名）、装修（配置）、访问路径（目录）

## 二、环境准备

前提：成功安装并启动了 Nginx

Nginx 默认的网站配置文件路径如下：

```text
/etc/nginx/nginx.conf
```

子配置文件位置：

```text
/etc/nginx/conf.d/*.conf
```

## 三、虚拟主机配置的结构

在 `/etc/nginx/conf.d/` 目录下新建一个 `.conf` 文件，Nginx 会自动加载。

## 四、实战演练：配置两个虚拟主机网站

创建两个自定义网站：

| 域名 | 根目录 |
| --- | --- |
| www.site1.com | /data/nginx/site1 |
| www.site2.com | /data/nginx/site2 |

### 1. 创建目录并放上首页

```bash
# 创建网站根目录
mkdir -p /data/nginx/site1
mkdir -p /data/nginx/site2

# 创建简单首页
echo "这是 Site1 首页" > /data/nginx/site1/index.html
echo "这是 Site2 首页" > /data/nginx/site2/index.html
```

### 2. 创建虚拟主机配置文件

**编辑第一个虚拟主机配置文件：**

```bash
vim /etc/nginx/conf.d/site1.conf
```

内容如下：

```nginx
server {
    listen       80;
    server_name  www.site1.com;

    root   /data/nginx/site1;
    index  index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**创建第二个网站配置：**

```bash
vim /etc/nginx/conf.d/site2.conf
```

内容如下：

```nginx
server {
    listen       80;
    server_name  www.site2.com;

    root   /data/nginx/site2;
    index  index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 3. 修改本地 hosts 文件（测试用）

```bash
echo "127.0.0.1 www.site1.com" >> /etc/hosts
echo "127.0.0.1 www.site2.com" >> /etc/hosts
```

### 4. 检查配置并重启 Nginx

```bash
nginx -t                # 检查配置文件是否正确
systemctl restart nginx # 重启生效
```

## 五、访问测试

打开浏览器访问域名或用 curl：

```bash
curl http://www.site1.com
# 输出：这是 Site1 首页

curl http://www.site2.com
# 输出：这是 Site2 首页
```

## 六、知识回顾

| 参数 | 含义 |
| --- | --- |
| server | 定义一个虚拟主机 |
| listen 80 | 监听端口 80（HTTP 默认端口） |
| server_name | 指定域名，用来区分哪个网站 |
| root | 网站根目录 |
| location / | 定义访问规则 |
| try_files | 按顺序尝试访问，404 表示找不到页面 |









