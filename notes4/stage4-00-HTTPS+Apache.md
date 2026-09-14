# HTTPS 与 Apache 实战知识点(SSL 证书重点)

> HTTP 是明文传输,HTTPS 通过 SSL/TLS 为通信加密。本文从 HTTPS 原理讲起,重点梳理 SSL 证书的类型、格式、生成与验证,最后完成 Apache(httpd)部署 HTTPS、强制跳转以及 TLS 相关的安全与性能优化。

## 一、为什么需要 HTTPS

| 对比项 | HTTP | HTTPS |
| --- | --- | --- |
| 传输方式 | 明文 | 加密(HTTP + SSL/TLS) |
| 安全性 | 数据可被窃听、篡改 | 加密传输,防窃听、防篡改 |
| 默认端口 | 80 | 443 |
| 证书 | 不需要 | 需要 SSL 证书 |

一句话结论:**HTTPS = HTTP + SSL/TLS**,在 HTTP 之外套了一层加密协议,让浏览器和服务器之间的通信内容别人看不到、改不了。

## 二、HTTPS 的工作过程(简化理解)

1. 浏览器请求 `https://www.test1.com`,服务器返回自己的 **SSL 证书**
2. 浏览器验证证书是否可信(是否由信任的 CA 签发、域名是否匹配、是否过期)
3. 验证通过后,双方协商出一把临时会话密钥
4. 后续通信全部用这把密钥加密传输

证书解决两个问题:**证明“你是你”(身份认证)** 和 **安全地交换密钥**。

## 三、SSL 证书详解(重点掌握)

### 3.1 证书是什么

SSL 证书是服务器的“数字身份证 + 加密钥匙”,里面包含:

- 证书持有者的域名(如 www.test1.com)
- 服务器公钥
- 签发机构(CA)信息
- 有效期
- 数字签名(证明内容未被篡改)

### 3.2 证书类型对比

| 类型 | 说明 | 适用场景 |
| --- | --- | --- |
| 自签名证书 | 自己签发,浏览器不信任,会提示“不安全” | 测试、学习 |
| Let's Encrypt | 免费、自动续期,被主流浏览器信任 | 个人站点、正式上线 |
| 商业证书(CA) | 阿里云、腾讯云等签发,分 DV/OV/EV 等级 | 企业正式业务 |

记忆点:**自签名 = 自己给自己发身份证,只有自己认;CA 证书 = 官方机构发的身份证,大家都认。**

### 3.3 常见证书文件格式

| 后缀 | 含义 |
| --- | --- |
| `.key` | 私钥文件,绝对不能泄露 |
| `.crt` / `.pem` | 证书文件(公钥部分) |
| `.csr` | 证书签名请求,申请正式证书时提交给 CA |
| `.pfx` / `.p12` | PKCS#12 格式,把证书和私钥打包在一起 |
| `.jks` | Java KeyStore,Tomcat/Java 使用的密钥库格式 |

### 3.4 生成自签名证书(openssl)

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/pki/tls/private/test.key \
  -out /etc/pki/tls/certs/test.crt
```

参数逐条解释:

| 参数 | 作用 |
| --- | --- |
| `req` | 证书请求/生成工具 |
| `-x509` | 直接生成自签名证书 |
| `-nodes` | 私钥不加密(方便服务启动时免密码) |
| `-days 365` | 有效期 1 年 |
| `-newkey rsa:2048` | 生成 2048 位 RSA 密钥 |
| `-keyout` | 私钥保存路径 |
| `-out` | 证书保存路径 |

执行过程中会提示填写国家、组织、域名等信息,自签名证书随意填写即可。

> 进阶提醒:现代浏览器校验证书时会看 SAN(Subject Alternative Name)字段。新版 OpenSSL 可加 `-addext "subjectAltName=DNS:www.test1.com"`;CentOS 7 自带的 OpenSSL 1.0.2 不支持该参数,需要用配置文件方式声明 SAN。

### 3.5 验证证书

```bash
# 查看证书内容(颁发者、有效期、SAN 等)
openssl x509 -in /etc/pki/tls/certs/test.crt -noout -text

# 模拟客户端连接,查看服务器实际返回的证书
openssl s_client -connect www.test1.com:443 -servername www.test1.com
```

### 3.6 申请正式证书的思路

- 免费:安装 certbot 后 `certbot --apache`,自动签发并配置,支持自动续期
- 商业/云证书:自己生成 CSR → 提交给 CA/云平台 → 审核通过后下载证书 → 替换配置中的证书路径
- 证书链:正式证书常需要用 `SSLCertificateChainFile` 指定中间证书,否则部分浏览器会提示证书不完整

## 四、Apache 部署 HTTPS 实战

### 4.1 安装 SSL 模块

```bash
sudo yum install -y mod_ssl
```

`mod_ssl` 让 Apache 具备 SSL/TLS 能力,同时会提供 443 端口的默认配置。

### 4.2 配置 HTTPS 虚拟主机

```bash
vim /etc/httpd/conf.d/test1-ssl.conf
```

```apache
<VirtualHost *:443>
    ServerName www.test1.com
    DocumentRoot /var/www/test1

    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/test.crt
    SSLCertificateKeyFile /etc/pki/tls/private/test.key

    ErrorLog logs/test1-ssl-error.log
    CustomLog logs/test1-ssl-access.log combined
</VirtualHost>
```

| 参数 | 作用 |
| --- | --- |
| `*:443` | 监听 HTTPS 默认端口 |
| `SSLEngine on` | 开启 SSL 加密引擎 |
| `SSLCertificateFile` | 证书文件路径 |
| `SSLCertificateKeyFile` | 私钥文件路径 |
| `ErrorLog` / `CustomLog` | 独立的错误日志和访问日志,方便排查 |

**注意:证书路径必须和生成时保存的路径完全一致,路径写错 Apache 会启动失败。**

### 4.3 重启并验证

```bash
sudo systemctl restart httpd
```

浏览器访问 `https://www.test1.com`。使用自签名证书时会提示“连接不安全”,点击“高级 → 继续访问”即可。

### 4.4 强制跳转到 HTTPS

```apache
<VirtualHost *:80>
    ServerName www.test1.com
    Redirect permanent / https://www.test1.com/
</VirtualHost>
```

`Redirect permanent` 返回 301 永久重定向,用户访问 http 地址会自动跳到 https,实现强制加密。

### 4.5 进阶:开启 HSTS

```apache
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
```

HSTS 会告诉浏览器“以后这个域名只能用 HTTPS 访问”,能有效抵御降级攻击。需要先加载 `mod_headers` 模块。

## 五、Apache 的 SSL 安全与性能优化(重点)

### 5.1 只保留安全协议与加密套件

```apache
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
SSLCipherSuite HIGH:!aNULL:!MD5
SSLHonorCipherOrder on
```

- `SSLProtocol` 禁用已不安全的 SSLv3、TLS 1.0/1.1,只留 TLS 1.2/1.3
- `SSLHonorCipherOrder on` 由服务器决定加密套件优先级,优先使用更安全的算法

### 5.2 开启 SSL 会话缓存(减少重复握手)

```apache
SSLSessionCache shmcb:/run/httpd/sslcache(512000)
SSLSessionCacheTimeout 300
```

TLS 握手的计算开销很大,会话缓存让客户端重连时复用已协商好的密钥,显著降低 CPU 消耗、加快响应。

### 5.3 开启 OCSP Stapling

```apache
SSLUseStapling on
SSLStaplingCache shmcb:/run/httpd/stapling(128000)
```

由服务器代替客户端去查询证书吊销状态,减少客户端等待时间,同时提升隐私性。

### 5.4 启用 HTTP/2

```apache
Protocols h2 http/1.1
```

HTTP/2 支持多路复用、头部压缩,在 HTTPS 场景下能明显减少页面加载时间(需要 `mod_http2`)。

### 5.5 开启压缩

```apache
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/css application/javascript application/json
</IfModule>
```

### 5.6 调整 MPM 工作模式

- `prefork`:一个连接一个进程,稳定但吃内存
- `worker` / `event`:多线程模型,并发更高、更省内存

高并发场景推荐 `event` 模式,并调优 `StartServers`、`MinSpareThreads`、`ThreadsPerChild`、`MaxRequestWorkers` 等参数。

### 5.7 生产架构建议

- 证书用 certbot 或云平台自动续期,避免忘记续费导致站点打不开
- 高并发架构下,常把 TLS 卸载到 Nginx 或负载均衡器,后端用 HTTP 通信,减轻应用服务器压力

## 六、常见问题排查

| 现象 | 原因与解决 |
| --- | --- |
| 浏览器提示“不安全” | 自签名证书不被信任,测试阶段可忽略;上线需换 CA 证书 |
| Apache 启动失败 | 证书/私钥路径写错,或证书文件权限不足 |
| 访问 https 无响应 | 防火墙未放行 443 端口、`mod_ssl` 未安装或未重启服务 |
| 部分浏览器提示证书不完整 | 缺少中间证书,配置 `SSLCertificateChainFile` |
| http 访问没跳转 | 80 端口的 `Redirect` 配置未生效,检查配置文件并重启 |

## 七、常用命令速查

| 场景 | 命令 |
| --- | --- |
| 生成自签名证书 | `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout x.key -out x.crt` |
| 查看证书信息 | `openssl x509 -in x.crt -noout -text` |
| 测试 HTTPS 连接 | `openssl s_client -connect 域名:443 -servername 域名` |
| 安装 SSL 模块 | `yum install -y mod_ssl` |
| 检查配置语法 | `apachectl configtest` |
| 重启 Apache | `systemctl restart httpd` |
| 查看端口监听 | `ss -tlnp \| grep 443` |