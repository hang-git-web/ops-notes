# Tomcat 运维与性能优化知识点

> 本文覆盖 Tomcat 的安装、目录结构、核心配置文件、Web 应用部署实战、HTTPS 配置,并把性能优化单独展开成 JVM、连接器、应用、架构、系统五层,方便当复习手册和调优参考。

## 一、Tomcat 是什么

Tomcat 是 Apache 基金会开发的开源 **Java Web 容器**,支持 Servlet 和 JSP,是运行 Java Web 项目最常见的平台。

### 1.1 名词解释

| 名词 | 含义 |
| --- | --- |
| Servlet | Java 编写的后端逻辑组件,相当于 PHP、Python 脚本 |
| JSP | Java Server Pages,嵌入 Java 代码的网页模板 |
| WAR | Web ARchive,Java Web 应用的压缩包 |
| JAR | Java 类库/应用包 |

### 1.2 工作原理

```text
浏览器 → HTTP 请求 → Tomcat → 调用 Servlet/JSP → 返回 HTML → 浏览器
```

Tomcat 本身也内置了一个 HTTP 服务器(Connector),所以可以独立运行,不像 PHP 必须依赖外部 Web 服务器。

## 二、安装 Tomcat

### 2.1 安装 JDK

```bash
sudo yum install -y java-1.8.0-openjdk
java -version
```

Tomcat 依赖 Java 运行环境,先装 JDK 再装 Tomcat。

### 2.2 下载并解压

```bash
cd /usr/local/src
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.107/bin/apache-tomcat-9.0.107.tar.gz
tar -xvzf apache-tomcat-9.0.107.tar.gz
sudo mv apache-tomcat-9.0.107 /usr/local/tomcat
```

> 版本号以官网当前版本为准,本文路径统一用 `/usr/local/tomcat`。

### 2.3 启动与验证

```bash
cd /usr/local/tomcat/bin
./startup.sh
```

Tomcat 默认监听 **8080** 端口。浏览器访问 `http://服务器IP:8080/`,看到 Apache Tomcat 欢迎页即成功。

停止 Tomcat:

```bash
./shutdown.sh
```

## 三、Tomcat 目录结构(重点)

| 目录 | 作用 |
| --- | --- |
| `bin/` | 启动、关闭脚本,如 `startup.sh`、`shutdown.sh`、`setenv.sh` |
| `conf/` | 核心配置文件,如 `server.xml`、`web.xml` |
| `webapps/` | 网站项目目录,默认部署路径 |
| `logs/` | 启动与运行日志 |
| `lib/` | Java 类库依赖 `.jar` 文件 |
| `temp/` | 临时文件目录 |
| `work/` | 编译后的 JSP、缓存文件 |

记忆口诀:**bin 管开关,conf 管配置,webapps 放项目,logs 看日志,lib 装依赖,work 放编译缓存。**

## 四、核心配置文件(重点)

| 配置文件 | 说明 |
| --- | --- |
| `server.xml` | 最重要的主配置文件,定义端口与服务 |
| `web.xml` | 全局 Web 应用配置,如默认欢迎页 |
| `tomcat-users.xml` | 配置后台管理界面的用户名和密码 |
| `context.xml` | 应用级配置,如数据库连接池 |
| `logging.properties` | 日志级别与输出方式 |
| `setenv.sh` | 自定义 JVM 参数(需手动创建,性能调优关键) |

### 4.1 server.xml 结构详解

```xml
<Server port="8005" shutdown="SHUTDOWN">
  <Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1" />
    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true" />
    </Engine>
  </Service>
</Server>
```

各层作用:

| 组件 | 作用 |
| --- | --- |
| `<Server>` | 整个 Tomcat 实例,监听关闭端口(默认 8005) |
| `<Service>` | 一个实例可含多个服务,包含 Connector 和 Engine |
| `<Connector>` | 监听请求的端口(默认 8080),可增加 HTTPS 支持 |
| `<Engine>` | 请求处理引擎,可包含多个 Host |
| `<Host>` | 虚拟主机,支持绑定多个域名,`appBase` 是部署目录 |
| `<Context>` | 具体的 Web 应用 |

端口约定:**8005 关闭端口、8080 HTTP、8443 HTTPS、8009 AJP**。

修改访问端口:

```xml
<Connector port="9090" protocol="HTTP/1.1" />
```

### 4.2 多网站部署(虚拟主机)

```xml
<Host name="www.a.com" appBase="/webs/a" unpackWARs="true" autoDeploy="true" />
<Host name="www.b.com" appBase="/webs/b" unpackWARs="true" autoDeploy="true" />
```

不同域名访问不同项目,实现多站共存,思路和 Nginx 虚拟主机一致。

### 4.3 web.xml 常见配置

- 默认欢迎页列表(`<welcome-file-list>`:index.html、index.jsp)
- MIME 类型
- session 过期时间
- 错误页面定义

### 4.4 配置管理后台账号

```xml
<role rolename="manager-gui"/>
<role rolename="admin-gui"/>
<user username="admin" password="123456" roles="manager-gui,admin-gui"/>
```

重启后访问 `http://服务器IP:8080/manager/html` 登录管理界面。

> 安全提醒:生产环境不要保留弱密码管理账号,建议限制管理页面的访问来源 IP,或直接停用 manager/host-manager 应用。

### 4.5 日志文件

| 文件 | 内容 |
| --- | --- |
| `catalina.out` | Tomcat 标准输出日志,启动报错首选 |
| `localhost_access_log.*.txt` | 访问日志 |
| `localhost.*.log` | 网站应用运行日志 |
| `manager.log` / `host-manager.log` | 管理应用日志 |

## 五、应用部署实战(详细流程)

### 5.1 标准部署流程

1. 准备应用包(`.war` 或 `.jar`)
2. 上传到服务器
3. 放入 `webapps/`,Tomcat 自动解压部署
4. 访问服务

```bash
cp helloworld.war /usr/local/tomcat/webapps/
cd /usr/local/tomcat/bin
./startup.sh
```

访问 `http://服务器IP:8080/helloworld` 即可。**上下文路径(context path)默认就是 war 包的文件名**。

### 5.2 自动部署与热部署配置

```xml
<Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true">
    <Context path="/helloworld" docBase="helloworld" reloadable="true"/>
</Host>
```

| 属性 | 含义 |
| --- | --- |
| `unpackWARs="true"` | 自动解压 war 包 |
| `autoDeploy="true"` | 支持自动部署 |
| `reloadable="true"` | 支持热部署,代码/WAR 变化时自动重新加载 |

**注意:修改后需重启 Tomcat 生效。**

生产建议:`autoDeploy="false"`、`reloadable="false"`,用可控的发版流程替代热部署,避免运行时自动重载带来性能和稳定性风险。

### 5.3 日志排查技巧

```bash
tail -f /usr/local/tomcat/logs/catalina.out
tail -f /usr/local/tomcat/logs/localhost_access_log.*.txt
```

`catalina.out` 用于定位启动失败、应用异常;访问日志用于确认请求是否真正到达。

### 5.4 常见部署故障对照

| 现象 | 排查方向 |
| --- | --- |
| 访问 404 | war 文件名/context path 不对;应用未解压成功 |
| 访问 503 | Connector 线程耗尽,检查线程池与并发 |
| 页面 500 | 应用报错,看 catalina.out 与 localhost.log |
| OutOfMemoryError | JVM 堆内存不足,调整 -Xmx |
| 端口被占用 | `ss -tlnp` 查冲突进程 |
| 启动很慢 | 随机数熵不足,可加 `-Djava.security.egd=file:/dev/./urandom` |
| Java 找不到 | 检查 JAVA_HOME 与 JDK 是否安装 |

## 六、Tomcat 配置 HTTPS(8443)

### 6.1 用 keytool 生成自签名 Keystore

```bash
cd /usr/local/tomcat
keytool -genkeypair -alias tomcat \
  -keyalg RSA -keysize 2048 \
  -keystore conf/tomcat.keystore \
  -validity 365
```

生成后得到 `conf/tomcat.keystore`,过程中设置的密码后面要用到。

### 6.2 修改 server.xml 开启 HTTPS

```xml
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="150" SSLEnabled="true" scheme="https" secure="true"
           keystoreFile="conf/tomcat.keystore"
           keystorePass="123456"
           clientAuth="false" sslProtocol="TLS" />
```

| 参数 | 说明 |
| --- | --- |
| `port="8443"` | HTTPS 默认端口 |
| `SSLEnabled="true"` | 启用 SSL |
| `keystoreFile` | 证书文件路径(相对 CATALINA_BASE 或绝对路径) |
| `keystorePass` | 证书密码 |
| `sslProtocol="TLS"` | 使用 TLS 协议 |
| `clientAuth="false"` | 不要求客户端证书 |

### 6.3 启动验证

```bash
cd /usr/local/tomcat/bin
./startup.sh
```

浏览器访问 `https://服务器IP:8443`,自签名证书会提示不受信任,选择继续访问即可。

### 6.4 正式证书接入思路

生产环境要把 CA 证书接入 Tomcat,常用做法是先转成 PKCS#12,再导入 Java 密钥库:

```bash
# crt + key → PKCS12
openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem \
  -out tomcat.p12 -name tomcat -passout pass:你的密码

# PKCS12 → JKS(旧版 Tomcat 需要)
keytool -importkeystore -srckeystore tomcat.p12 -srcstoretype PKCS12 \
  -destkeystore tomcat.jks -deststoretype JKS -deststorepass 你的密码
```

新版 Tomcat 可直接使用 PKCS12,配置 `keystoreType="PKCS12"` 即可。

### 6.5 HTTPS 常见问题

| 问题 | 原因与解决 |
| --- | --- |
| 浏览器提示不安全 | 自签名证书,测试阶段忽略 |
| 访问 8443 报错 | Tomcat 未启动或配置错误,查 `catalina.out` |
| keystore 路径错误 | 用绝对路径或相对于 Tomcat 目录的路径 |
| 页面打不开或超时 | 防火墙未放行 8443 端口 |
| 证书链不完整 | 补充中间证书或使用 PKCS12 完整包 |

## 七、Tomcat 性能优化(重点)

性能优化可以从**五层**来看,从上到下依次是:JVM → 连接器 → 应用 → 架构 → 系统。

### 7.1 JVM 层(最基础、最有效)

在 `bin/setenv.sh` 中配置(没有该文件就新建):

```bash
export CATALINA_OPTS="-server \
  -Xms2g -Xmx2g \
  -Xss512k \
  -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
  -XX:+UseG1GC -XX:MaxGCPauseMillis=200 \
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/tomcat/logs \
  -Djava.security.egd=file:/dev/./urandom"
```

| 参数 | 作用 |
| --- | --- |
| `-Xms` / `-Xmx` | 初始与最大堆内存,建议设置成相同值,避免动态扩缩容开销 |
| `-Xss` | 单线程栈大小 |
| `MetaspaceSize` | 元空间大小(替代老版本永久代) |
| `UseG1GC` | 使用 G1 垃圾回收器,停顿更可控 |
| `HeapDumpOnOutOfMemoryError` | 内存溢出时自动 dump,便于事后分析 |
| `java.security.egd` | 使用非阻塞随机数源,加快启动 |

调优原则:堆内存不要超过物理内存的一半;先监控 GC 再调参数,常用 `jstat -gcutil <pid> 1000` 观察。

### 7.2 连接器与线程池层(高并发关键)

```xml
<Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
          maxThreads="500" minSpareThreads="50" maxIdleTime="60000"/>

<Connector port="8080" protocol="org.apache.coyote.http11.Http11NioProtocol"
           executor="tomcatThreadPool"
           acceptCount="300"
           maxConnections="10000"
           connectionTimeout="20000"
           keepAliveTimeout="15000"
           maxKeepAliveRequests="200"
           compression="on"
           compressionMinSize="2048"
           compressibleMimeType="text/html,text/css,application/json,application/javascript"
           URIEncoding="UTF-8" />
```

| 参数 | 含义 | 调优思路 |
| --- | --- | --- |
| `maxThreads` | 最大工作线程数 | 太小会 503,太大会频繁上下文切换,按 CPU 与压测结果定 |
| `minSpareThreads` | 最小空闲线程 | 保证突发流量时能立即响应 |
| `acceptCount` | 等待队列长度 | 队列满后新连接被拒绝,适当加大 |
| `maxConnections` | 最大瞬时连接数(NIO) | 非阻塞模型下可设较大 |
| `keepAliveTimeout` | 长连接保持时间 | 减少重复握手,但会占用连接资源 |
| `maxKeepAliveRequests` | 单连接最大请求数 | 适当调大可提升静态资源场景效率 |
| `compression` | 开启响应压缩 | 省带宽,但增加 CPU 消耗 |
| `protocol` | 连接器协议 | `Http11NioProtocol` 非阻塞高并发;APR 性能更高但需装本地库 |

经验规律:**线程数不是越大越好**。过多线程会导致 CPU 疯狂切换,反而降低吞吐;正确做法是压测后找到瓶颈(CPU/内存/IO/GC),再针对性调整。

### 7.3 应用与部署层

- 删除用不到的 `webapps`:`docs`、`examples`、`manager`、`host-manager`,减少资源占用和安全隐患
- 生产关闭 `autoDeploy` 和 `reloadable`,用可控发布替代自动重载
- JSP 预编译,避免首次访问时的编译延迟
- 使用数据库连接池(`context.xml` 中配置 `Resource`),避免每次请求新建连接
- 合理设置 session 超时,避免内存长期被无效会话占用

### 7.4 架构层(收益最大的优化)

- **动静分离**:静态资源交给 Nginx/Apache 处理,Tomcat 专注动态请求
- **反向代理 + 负载均衡**:Nginx 在前端分发流量,后端部署多个 Tomcat 实例
- **TLS 卸载**:HTTPS 在 Nginx 层终止,Tomcat 内部走 HTTP,减少加密计算
- **会话共享**:多实例场景把 session 放到 Redis,避免用户来回登录
- **水平扩展**:单机性能有上限,集群才是应对大规模并发的根本手段

Nginx 负载均衡示例:

```nginx
upstream tomcat_pool {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
    listen 80;
    server_name www.app.com;

    location / {
        proxy_pass http://tomcat_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 7.5 系统层

- 提高文件句柄上限:`ulimit -n 65535`
- 调整内核 TCP 参数(高并发场景)
- 监控手段:`jstat`(GC)、`jmap`(堆)、`jstack`(线程)、`jconsole`/VisualVM、JMX 接入 Prometheus
- 日志:生产环境日志级别用 INFO,访问日志按天切割,避免磁盘被写满

### 7.6 调优原则(必记)

1. **先压测,后调参**,不要凭感觉改数字
2. **一次只改一个变量**,改完复测,否则无法判断效果
3. 关注真正瓶颈:CPU、内存、磁盘 IO、GC、线程池、数据库
4. 参数之间会互相影响,`maxThreads` 加大可能压垮数据库
5. 单机调优收益有限,架构优化(集群、缓存、动静分离)才是量级提升

## 八、常用命令速查

| 场景 | 命令 |
| --- | --- |
| 启动 / 停止 Tomcat | `/usr/local/tomcat/bin/startup.sh`、`shutdown.sh` |
| 查看主日志 | `tail -f /usr/local/tomcat/logs/catalina.out` |
| 查看访问日志 | `tail -f /usr/local/tomcat/logs/localhost_access_log.*.txt` |
| 部署应用 | `cp xx.war /usr/local/tomcat/webapps/` |
| 生成 Java 密钥库 | `keytool -genkeypair -alias tomcat -keyalg RSA -keystore conf/tomcat.keystore -validity 365` |
| 查看 GC 情况 | `jstat -gcutil <pid> 1000` |
| 查看端口占用 | `ss -tlnp \| grep 8080` |
| 查看 Java 版本 | `java -version` |