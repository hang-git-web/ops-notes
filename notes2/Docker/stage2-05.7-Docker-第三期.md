# Docker 第三期：Compose、Dockerfile、Prometheus 监控与项目实战

> **对应课件**：《Docker基础到大神之路（第三期）》（千山）
> **本篇定位**：从"手工敲 docker run"升级到"**用文件描述整个系统**"。这一期是四块内容：**Compose 编排 → Dockerfile 构建 → Prometheus 监控 → Zabbix 项目实战**。
> **承接关系**：第二期学的网络和数据卷，在这一期变成 compose 文件里的 `networks:` 和 `volumes:` 两段配置。
> **环境**：Rocky Linux 8.10 + docker 26.1.3 + compose v2.27（4 核 / 3.8G）。**内存建议 4G**，因为要同时跑 MySQL + PHP + Nginx，后面还有 Zabbix 全家桶。

---

## 〇、开篇前的准备

```bash
# 1) 确认 docker 和 compose 都可用
docker version | head -5
docker compose version

# 2) 本篇所有实验都放在这个目录下，保持整洁
mkdir -p /data/docker-lab
cd /data/docker-lab

# 3) 确认能拉镜像（不行就回去配加速器，见第一期 3.3）
docker pull nginx:1.28
```

> **目录规划的习惯（很重要）**：每个 compose 工程一个独立目录，里面放 `docker-compose.yaml`、`.env`、`conf/`、`data/`。
> **好处**：compose 的"工程（project）"就是**按目录识别的** —— 同一个目录下 `up` 出来的一组容器算一个工程，`down` 的时候能整组清掉，不会误伤别的实验。

---

## 一、Docker Compose（课件第一章）

### 1.1 为什么需要 Compose（课件原话）

> **① 我们都是手工用 `docker run` 的方式来跑容器，很低效。**
> **② 实际工作中，一个服务往往需要多个容器来协同工作，并且容器之间有启动顺序。**

**把这两句翻译成具体痛点**：

```text
手工敲的痛点：
1) 一条 LNMP 要敲 3~5 条 docker run，每条都带一堆参数（端口、卷、环境变量、网络）
2) 参数全靠记忆或翻历史命令，换台机器就忘了
3) 启动顺序要自己控制 —— 必须先起 MySQL 再起 PHP，否则 PHP 连不上数据库
4) 停止/清理要一条条来，容易漏
5) 没法做版本管理 —— 这套环境配置没法提交到 git

Compose 的解法：
  把上面所有东西写进一个 YAML 文件 → docker compose up -d 一条命令全起来
  → 配置文件可以进 git、可以做 code review、可以在任何机器上重现
```

> **一句话定义**：**「Compose 是用一个 YAML 文件描述"一组容器怎么跑、怎么连、谁先起"的工具，把多条 docker run 收敛成一条命令。」**

### 1.2 Compose 文件

#### 文件命名与目录结构（课件原文）

> **文件的命名一般来说是 `docker-compose.yaml`、`compose.yaml`，也可以自己定义 compose 文件。**
> **扩展配置的文件（初始化的一些文件）、环境变量的一些文件等。**

**课件给的目录结构**：

```text
/data/qscompose
    docker-compose.yaml / compose.yaml / 自定义.yaml
    .env
    /data/init.sql
```

> **解析**：一个 compose 工程目录通常有这些东西 ——
> | 文件 | 作用 |
> | --- | --- |
> | `docker-compose.yaml` 或 `compose.yaml` | 主配置文件（`docker compose up` 默认按这个顺序找这两个名字） |
> | `.env` | 环境变量文件，compose 会**自动读取**，用于把密码等敏感信息抽出来 |
> | `conf/`、`init.sql` 等 | 初始化文件（数据库建表脚本、应用配置文件），通过挂载送进容器 |

#### 文件的三段式结构（★ 背这个骨架）

> **课件原话**：
> **第一段 `service`：服务定义，定义容器启动配置信息。**
> **第二段 `volume`：数据卷定义。**
> **第三段 `network`：网络定义，定义容器网络配置。**

```yaml
version: "3.8"                # 指定 compose 语法版本（新版不看版本号，直接校验内容）
services:                     # 以下区段定义服务列表，多个服务写多个 server_name
  server_name:                # server_name 是变量，建议起"见字知意"的名字，如 mysql57
    container_name: xxx       # 指定容器名，建议起"见字知意"的名字
    image: xxx:latest         # 指定使用的镜像名及标签
    build:                    # build 与 image 冲突，二选其一
      context: /xxx/xxx       # 指定 Dockerfile 所在目录
      dockerfile: Dockerfile  # 指定 Dockerfile 文件名（context 与 dockerfile 二选一）
    ports:
      - "00:00"               # 映射端口，格式为 本地端口:容器内端口
      - "00:00"               # 多个端口就另起一行
    volumes:
      - "contents01:/xx/xx"   # 格式为 物理机目录:容器目录，物理机目录可相对可绝对，容器必须绝对
      - "contents02:/xx/xx"   # 有多个卷映射就重复设置
    restart: always           # 设置无论遇到什么错，都重启容器
    depends_on:               # 用来解决依赖关系，这个服务启动的前置条件
      - server_name01         # 引用上面的 server_name
      - server_name02         # 多个依赖就重复设置
    links:                    # 容器之间互相通信，默认用容器名通信，不用容器 IP
      - mysql                 # 容器名称
    networks:                 # 加入指定的网络（与之前添加网卡类似）
      - app01_net             # bridge 类型的网卡名
      - app02_net             # 需要加入多个网段就重复设置
    environment:              # 定义变量，类似 Dockerfile 中的 ENV
      - TZ=Asia/Shanghai      # ★ 解决容器通过 compose 编排启动时的时区问题
    command: [                # 使用 command 可以覆盖容器启动后默认执行的命令
      '--character-set-server=utf8mb4',
      '--collation-server=utf8mb4_unicode_ci',
    ]
  server_name2:               # 开始编排第二个容器
    ...

volumes:                      # 全局数据卷，声明可以被多个服务引用的卷
  contents01:
  contents02:

networks:                     # 如果要指定 IP 网段，还是先创建好再使用；这里声明 networks
  app01_net:
    driver: bridge            # 指定网卡类型
  app02_net:                  # 需要多个独立网段就重复设置
    driver: bridge
```

> **★ 三个最容易被忽略、但最影响结果的字段**：
> | 字段 | 为什么关键 |
> | --- | --- |
> | **`TZ=Asia/Shanghai`** | 不设的话容器里是 UTC 时间，日志和数据库时间会差 8 小时（课件专门标了感叹号） |
> | **`depends_on`** | 只保证**启动顺序**，**不保证服务真的就绪** —— 要真就绪必须配 `healthcheck`（见 1.6） |
> | **`restart`** | 不设的话容器一挂就没了；`always` / `unless-stopped` 是生产常用值 |

#### 概念的层次（课件原文）

> **Project（工程）**：某个目录下所有文件的集合。
> **service（服务）**：一个服务对应一个容器。
> **container（容器）**：服务运行之后的实例。

```text
一个目录（Project 工程）
   ├── service: mysql   → 1 个容器（container）
   ├── service: php     → 1 个容器
   └── service: nginx   → 1 个容器

★ "服务"是配置里的概念，"容器"是跑起来的实体。
  所以扩容时 docker compose up -d --scale php=3 就是"一个服务，三个容器"。
```

### 1.3 MySQL 案例：docker run 方式 vs compose 方式

> 课件用同一个 MySQL 场景演示了两种做法，**对比着看最能理解 Compose 的价值**。

#### 方式一：docker run（手工方式）

**先写配置文件**：

```bash
mkdir -p /data/mysql/conf
cat >/data/mysql/conf/my.cnf <<'EOA'
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
pid-file=/var/lib/mysql/mysqld.pid
log-error=/var/lib/mysql/mysqld.log
max_connections=2000
character-set-server=utf8mb4
default-storage-engine=INNODB
lower_case_table_names=1
max_allowed_packet=256M
default-time_zone='+8:00'
symbolic-links=0
skip-name-resolve

[client]
default-character-set=utf8mb4
socket=/var/lib/mysql/mysql.sock

[mysql]
default-character-set=utf8mb4
socket=/var/lib/mysql/mysql.sock
EOA
```

> **注意课件这里有个笔误**：`log-error=//var/lib/mysql/mysqld.log` 多了一个斜杠，正确的是单个 `/`。
>
> **这份 my.cnf 里几个关键项**：
> | 配置 | 作用 |
> | --- | --- |
> | `character-set-server=utf8mb4` | 字符集，支持 emoji 和生僻字（utf8 是残缺的） |
> | `lower_case_table_names=1` | 表名不区分大小写（Windows 迁过来的库经常需要） |
> | `max_connections=2000` | 最大连接数 |
> | `default-time_zone='+8:00'` | **时区**，不然存进去的时间差 8 小时 |
> | `skip-name-resolve` | 跳过反向 DNS 解析，加快连接速度 |
>
> **另外**，官方 MySQL 镜像的配置文件读的是 `/etc/mysql/conf.d/` 目录（不是 `/etc/my.cnf`），所以下面是把它挂到 `conf.d` 上。

**启动容器**：

```bash
docker run -d -p 3307:3306 \
  -v /data/mysql/conf:/etc/mysql/conf.d:ro \
  -v /data/mysql/data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=Zhongguo@0755 \
  --name TestMysql mysql:5.7
```

**检验**：

```bash
docker ps | grep TestMysql
docker logs TestMysql 2>&1 | tail -5        # 找 "ready for connections"

# 母机连容器的端口（课件原话）
mysql -uroot -pZhongguo@0755 -h 127.0.0.1 -P 3307 -e "select version();"

# 看配置有没有生效
docker exec TestMysql mysql -uroot -pZhongguo@0755 -e "show variables like 'character_set_server';"
# 期望 utf8mb4

# 看数据目录是不是落到母机了
ls /data/mysql/data | head
```

#### 方式二：compose 方式

**步骤 1：建目录、写配置**：

```bash
mkdir -p /opt/mysql-compose/{data,conf}
cat >/opt/mysql-compose/conf/my.cnf<<'EOA'
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
pid-file=/var/lib/mysql/mysqld.pid
log-error=/var/lib/mysql/mysqld.log
max_connections=2000
character-set-server=utf8mb4
default-storage-engine=INNODB
lower_case_table_names=1
max_allowed_packet=256M
default-time_zone='+8:00'
symbolic-links=0
skip-name-resolve

[client]
default-character-set=utf8mb4
socket=/var/lib/mysql/mysql.sock

[mysql]
default-character-set=utf8mb4
socket=/var/lib/mysql/mysql.sock
EOA
```

**步骤 2：写 compose 文件**：

```bash
cat >/opt/mysql-compose/mysql57.docker.compose.yml<<'EOA'
version: '3.8'
services:
  db-mysql:
    container_name: "mysql57"
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: Zhongguo@0755
    ports:
      - 3308:3306
      - 33080:33060
    volumes:
      - ./data:/var/lib/mysql
      - ./conf:/etc/mysql/conf.d
    networks:
      - app-net
networks:
  app-net:
    driver: bridge
EOA
```

> **两个细节**：
> ① 卷写的是 **`./data`（相对路径）** —— 相对的是 **compose 文件所在目录**，也就是 `/opt/mysql-compose/data`。这就是为什么规范要求"一个工程一个目录"。
> ② 服务名叫 `db-mysql`，容器名叫 `mysql57`。**服务名是 compose 内部互相引用的名字（做 DNS 用），容器名是给人看的** —— 这两个概念要分清。

**步骤 3：启动并检验**：

```bash
cd /opt/mysql-compose

# 指定文件名启动（因为文件名不是默认的 docker-compose.yaml）
docker compose -f mysql57.docker.compose.yml up -d

docker compose -f mysql57.docker.compose.yml ps
docker ps | grep mysql57
ss -lntp | grep 3308
mysql -uroot -pZhongguo@0755 -h 127.0.0.1 -P 3308 -e "select version();"
```

**停止与清理**：

```bash
docker compose -f mysql57.docker.compose.yml stop        # 停止，容器还在
docker compose -f mysql57.docker.compose.yml down        # 停止并删除容器/网络（数据卷保留）
```

> **`down` 和 `down -v` 的区别（★ 重要）**：`down` 只删容器和网络，**数据卷保留**；`down -v` 连数据卷一起删，**数据就没了**。生产上别随手加 `-v`。

#### .env：把密码抽出来（课件第二个 MySQL demo）

> **课件原话**：**我们把 root 密码当作一个变量，再用 compose 实现。**

```bash
mkdir -p /opt/mysql-compose-v2/{data,conf}
cd /opt/mysql-compose-v2
```

**写 .env 文件**：

```bash
cat >.env <<'EOA'
MYSQL_ROOT_PASSWORD=Zhongguo@0755
EOA
more .env        # 检验
```

**写 my.cnf**（内容和上面一样，此处同前）：

```bash
cat >/opt/mysql-compose-v2/conf/my.cnf<<'EOA'
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
pid-file=/var/lib/mysql/mysqld.pid
log-error=/var/lib/mysql/mysqld.log
max_connections=2000
character-set-server=utf8mb4
default-storage-engine=INNODB
lower_case_table_names=1
max_allowed_packet=256M
default-time_zone='+8:00'
symbolic-links=0
skip-name-resolve

[client]
default-character-set=utf8mb4
socket=/var/lib/mysql/mysql.sock

[mysql]
default-character-set=utf8mb4
socket=/var/lib/mysql/mysql.sock
EOA
```

**写 compose 文件（注意 `${...}` 的写法）**：

```bash
cd /opt/mysql-compose-v2
cat >compose.yml<<'EOA'
services:
  db-mysql:
    container_name: "mysql57-v2"
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}   # 引用 .env 中的变量
    ports:
      - 3309:3306
      - 33090:33060
    volumes:
      - ./data:/var/lib/mysql
      - ./conf:/etc/mysql/conf.d
    networks:
      - app-net
networks:
  app-net:
    driver: bridge
EOA
```

**启动与检验**：

```bash
# 先看变量有没有被正确解析（★ 这一步能避免 90% 的坑）
docker compose config | grep -A2 MYSQL_ROOT_PASSWORD
# 期望看到解析后的明文密码

docker compose up -d
docker compose ps
mysql -uroot -pZhongguo@0755 -h 127.0.0.1 -P 3309 -e "select version();"
```

> **★ 为什么用 .env**：密码、密钥这类东西**不该写进 compose 文件**（因为 compose 文件要提交到 git）。抽到 `.env` 之后，把 `.env` 加进 `.gitignore`，代码仓库里就只剩 `${MYSQL_ROOT_PASSWORD}` 这样的占位符。
>
> **两个容易踩的点**：
> ① `.env` **必须和 compose 文件在同一个目录**（或者用 `--env-file` 指定），否则读不到；
> ② `.env` 里**不要写引号**（`MYSQL_ROOT_PASSWORD="xxx"` 会把引号也当成值的一部分），也不要有多余空格。
>
> **验证神器**：`docker compose config` 会把变量替换后的**最终配置**打印出来 —— 怀疑变量没生效时，先跑这一条。

### 1.4 Compose 命令集（课件 1.3）

| 命令 | 作用（课件原话） |
| --- | --- |
| `docker compose up -d` | **新建容器并启动容器** |
| `docker compose down` | **停止容器并删除容器**（还能加 `-v` 连卷一起删） |
| `docker compose exec 服务名 bash` | **进入到容器**里 |
| `docker compose logs 服务名` | **看容器日志** |
| `docker compose config` | **查看 compose 配置文件**（变量替换后的最终结果） |
| `docker compose top` | **查看母机里容器名称的进程号** |

**再补几条实战必备的**：

| 命令 | 作用 |
| --- | --- |
| `docker compose ps` | 看这个工程下的容器状态 |
| `docker compose restart 服务名` | 重启某个服务 |
| `docker compose stop` / `start` | 停止 / 启动（不删容器） |
| `docker compose build` | 按 `build:` 段构建镜像 |
| `docker compose up -d --build` | 先重新构建再启动（改完 Dockerfile 必用） |
| `docker compose up -d --scale php=3` | 把某个服务扩到 3 个容器 |
| `docker compose pull` | 拉取所有服务用的镜像 |
| `docker compose down --remove-orphans` | 顺便清理"compose 文件里已经没有的"容器 |

**完整实操（用前面写的 mysql57 工程）**：

```bash
cd /opt/mysql-compose

docker compose -f mysql57.docker.compose.yml ps
docker compose -f mysql57.docker.compose.yml top          # 看到母机上的 PID
docker compose -f mysql57.docker.compose.yml logs db-mysql | tail -5
docker compose -f mysql57.docker.compose.yml exec db-mysql bash
# 进去了就 exit 出来
docker compose -f mysql57.docker.compose.yml config | head -20
docker compose -f mysql57.docker.compose.yml down
```

> **`up` 和 `start` 的区别（高频考点）**：
> - `up` 会**按 compose 文件重新创建**容器（配置变了就重建），没起过的就新建；
> - `start` 只是把**已经存在的**容器启动起来，**不会读 compose 文件的改动**。
> - 所以：**改完 compose 文件要用 `up -d`（必要时加 `--build`），别用 `start`。**

### 1.5 Compose 网络（课件 1.4）

课件的 compose 网络部分就是一个最小示例：

```yaml
networks:
  app-net:
    driver: bridge
```

**但这里有几个必须知道的点（课件没展开，我补上）**：

```text
1) compose 会自动创建一个默认网络，名字是 <工程名>_default
   （工程名默认 = compose 文件所在目录名，可以用 -p 参数改）
2) 同一个网络里的服务，可以用【服务名】互相访问
   —— 也就是 PHP 容器里可以直接 ping 通 mysql 这个服务名
3) 显式声明 networks: 并让服务加入，是为了：
   · 起一个可读的网络名
   · 让多个 compose 工程互相隔离（不同工程默认各用各的网络）
   · 需要指定网段时（subnet）也在这里写
```

**验证"服务名就是主机名"**：

```bash
cd /opt/mysql-compose
docker compose -f mysql57.docker.compose.yml up -d
docker compose -f mysql57.docker.compose.yml exec db-mysql bash

# 在容器里：
cat /etc/hosts                      # 能看到同网络其它服务的名字和 IP
getent hosts db-mysql || nslookup db-mysql 2>/dev/null
exit
```

> **和上一期的连接**：第二期我们用 `docker network create` 手建网络，让容器能**用容器名互 ping**；Compose 把这一步自动化了 —— **同工程的服务自动进同一个网络，服务名就是 DNS 名**。

### 1.6 服务依赖与健康检查（课件 1.5，★ 本篇最重要的实验）

> **这一节是第三期的核心**：`depends_on` 只保证"启动顺序"，`healthcheck` 才保证"真的能用"。课件给的完整案例正是为了说明这件事。

#### 先理解问题：为什么光有 depends_on 不够

```text
depends_on: mysql 的意思是"先启动 mysql 容器，再启动 php 容器"。
但 MySQL 容器"启动了"不等于"能连了" —— 它启动后要初始化数据目录、
建系统表、加载配置，这个过程可能要 20~60 秒。

如果 php 在 MySQL 还没就绪时就连接，就会连接失败，
表现为：容器起来了，但网页报 "Connection refused" / "Access denied"。
```

**解决办法**：给 MySQL 加 `healthcheck`，让 php 用 `condition: service_healthy` 等它真正健康。

#### 完整实验：PHP + MySQL 的 Compose 工程（照着敲就能跑）

**步骤 1：建目录**

```bash
mkdir -p /data/compose-php-mysql-demo/mysql/initdb
cd /data/compose-php-mysql-demo
```

**步骤 2：写 compose 文件**

```bash
cat >docker-compose.yaml<<'EOA'
services:
  mysql:
    image: mysql:5.7
    container_name: demo-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: demo
      TZ: Asia/Shanghai
    ports:
      - "3306:3306"
    volumes:
      - ./mysql/initdb:/docker-entrypoint-initdb.d
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -uroot -ppassword --silent"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 30s

  php:
    build: ./php                              # 从 ./php 目录的 Dockerfile 构建
    container_name: demo-php
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy            # ★ 等 mysql 健康了才启动
    environment:
      DB_HOST: mysql                          # ★ 用服务名当主机名
      DB_NAME: demo
      DB_USER: root
      DB_PASS: password
      TZ: Asia/Shanghai
    ports:
      - "8080:80"

volumes:
  mysql_data:
EOA
```

**这块配置里的四个关键点**：

| 配置 | 为什么这么写 |
| --- | --- |
| `./mysql/initdb:/docker-entrypoint-initdb.d` | **MySQL 官方镜像的约定**：容器**第一次**启动时，会按文件名顺序执行这个目录下的 `.sh`/`.sql`。所以建表脚本放这里就能自动初始化 |
| `mysql_data:/var/lib/mysql` | **具名卷**存数据（第二期学的），容器重建数据不丢 |
| `healthcheck.test` | 用 `mysqladmin ping` 判断 MySQL 是否真的能响应（`--silent` 只关心退出码）。**注意要用 `-uroot -ppassword`**，否则认证失败会被判为 unhealthy |
| `condition: service_healthy` | **这是关键**：让 php 一直等到 mysql 健康才启动 |

**步骤 3：写初始化 SQL**

```bash
cat >mysql/initdb/01-init.sql<<'EOA'
USE demo;
CREATE TABLE IF NOT EXISTS users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL,
  age INT NOT NULL
);
INSERT INTO users (name, age) VALUES
('zhangsan', 20),
('lisi', 22),
('wangwu', 25);
EOA
```

> **注意**：`01-init.sql` 前面的数字是**执行顺序**。如果有多份脚本，按 `01-`、`02-` 排序执行。
> **★ 重要**：这个脚本**只在数据目录为空时执行一次**。如果改了脚本要重新初始化，必须先 `docker compose down -v` 把卷删掉 —— 这也是"为什么改了初始化 SQL 没生效"的标准原因。

**步骤 4：写 PHP 的 Dockerfile**

```bash
mkdir -p php
cat >php/Dockerfile<<'EOA'
FROM php:7.4-apache
RUN docker-php-ext-install mysqli
WORKDIR /var/www/html
COPY index.php /var/www/html/index.php
EOA
```

> **逐行解释**：
> | 指令 | 作用 |
> | --- | --- |
> | `FROM php:7.4-apache` | 基础镜像：PHP 7.4 + Apache（还自带 `docker-php-ext-install` 这个装扩展的脚本） |
> | `RUN docker-php-ext-install mysqli` | 装 `mysqli` 扩展 —— **PHP 默认不带 mysqli，不装的话 `new mysqli()` 会报错** |
> | `WORKDIR /var/www/html` | 设工作目录（Apache 的默认网站目录），后面 COPY 的相对路径都基于它 |
> | `COPY index.php /var/www/html/index.php` | 把当前目录的 index.php 拷进镜像 |

**步骤 5：写 index.php（连接 MySQL 并展示数据）**

```bash
cat >php/index.php<<'EOA'
<?php
date_default_timezone_set('Asia/Shanghai');

$host = getenv('DB_HOST') ?: 'mysql';
$db   = getenv('DB_NAME') ?: 'demo';
$user = getenv('DB_USER') ?: 'root';
$pass = getenv('DB_PASS') ?: 'password';

$conn = @new mysqli($host, $user, $pass, $db);
?>
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>PHP + MySQL Docker Compose 演示</title>
    <style>
        body { font-family: Arial, "Microsoft YaHei", sans-serif; margin: 40px;
               background: #f5f7fa; color: #333; }
        .box { background: #fff; padding: 24px; border-radius: 10px;
               box-shadow: 0 2px 10px rgba(0,0,0,.08); max-width: 900px; margin: auto; }
        h1 { color: #1f4e79; }
        .ok { color: green; font-weight: bold; }
        .err { color: red; font-weight: bold; }
        table { width: 100%; border-collapse: collapse; margin-top: 18px; }
        table, th, td { border: 1px solid #ccc; }
        th, td { padding: 10px; text-align: left; }
        .meta { margin-top: 15px; color: #666; }
        code { background: #f0f0f0; padding: 2px 6px; border-radius: 4px; }
    </style>
</head>
<body>
<div class="box">
    <h1>PHP + MySQL Docker Compose 演示案例</h1>
    <p>当前时间：<?php echo date('Y-m-d H:i:s'); ?></p>
    <p>PHP 容器：<span class="ok">已启动</span></p>
    <?php if ($conn->connect_error): ?>
        <p>MySQL 连接状态：<span class="err">失败</span></p>
        <p>错误信息：<?php echo htmlspecialchars($conn->connect_error); ?></p>
    <?php else: ?>
        <p>MySQL 连接状态：<span class="ok">成功</span></p>
        <p class="meta">
            连接目标：<code><?php echo htmlspecialchars($host); ?></code> /
            数据库：<code><?php echo htmlspecialchars($db); ?></code>
        </p>
        <?php
        $conn->set_charset("utf8");
        $sql = "SELECT id, name, age FROM users ORDER BY id ASC";
        $result = $conn->query($sql);
        ?>
        <h2>用户数据</h2>
        <?php if ($result && $result->num_rows > 0): ?>
            <table>
                <tr><th>ID</th><th>姓名</th><th>年龄</th></tr>
                <?php while ($row = $result->fetch_assoc()): ?>
                    <tr>
                        <td><?php echo htmlspecialchars($row['id']); ?></td>
                        <td><?php echo htmlspecialchars($row['name']); ?></td>
                        <td><?php echo htmlspecialchars($row['age']); ?></td>
                    </tr>
                <?php endwhile; ?>
            </table>
        <?php else: ?>
            <p>users 表暂无数据。</p>
        <?php endif; ?>
        <?php $conn->close(); ?>
    <?php endif; ?>
</div>
</body>
</html>
EOA
```

> **这段 PHP 里的三个关键设计**：
> ① **`$host = getenv('DB_HOST') ?: 'mysql'`** —— 从环境变量读数据库地址，而 `DB_HOST` 正是 compose 里配的 **`mysql`（服务名）**。这就是"容器间用服务名通信"的实证。
> ② **`?:` 是 PHP 的简写三元运算符**：`getenv(...) ?: 'mysql'` 表示"取到了就用它，没取到就用默认值"。
> ③ **`@new mysqli(...)`** 前面的 `@` 是抑制警告，因为我们要自己判断错误并友好显示，不想让 PHP 警告刷屏。

**步骤 6：启动并完整验证（★ 这就是"闭环"）**

```bash
cd /data/compose-php-mysql-demo

# 1) 先看解析后的配置（确认变量、路径都对）
docker compose config | head -40

# 2) 构建 + 启动
docker compose up -d --build

# 3) 看容器状态：两个都要是 Up，且 mysql 要显示 (healthy)
docker compose ps
# 期望：
# NAME         IMAGE                          STATUS
# demo-mysql   mysql:5.7                      Up (healthy)
# demo-php     compose-php-mysql-demo-php     Up

# 4) 观察"依赖等待"的效果（★ 这个观察很有价值）
docker compose logs -f mysql | head -20
# 能看到 MySQL 初始化过程；php 会一直等到 healthy 才启动
```

**用浏览器或 curl 验证最终效果**：

```bash
curl -s http://127.0.0.1:8080 | head -30
```

**期望看到**（关键的几行）：

```html
<p>当前时间：2026-xx-xx xx:xx:xx</p>
<p>PHP 容器：<span class="ok">已启动</span></p>
<p>MySQL 连接状态：<span class="ok">成功</span></p>
<p class="meta">连接目标：<code>mysql</code> / 数据库：<code>demo</code></p>
<h2>用户数据</h2>
<table>
  <tr><th>ID</th><th>姓名</th><th>年龄</th></tr>
  <tr><td>1</td><td>zhangsan</td><td>20</td></tr>
  <tr><td>2</td><td>lisi</td><td>22</td></tr>
  <tr><td>3</td><td>wangwu</td><td>25</td></tr>
</table>
```

> **★ 到了这一步才算真正跑通**：网页上出现 `MySQL 连接状态：成功` **并且**能看到三行用户数据 —— 这一条链路打通了 **compose 编排 → 自定义网络 → 服务名 DNS → 健康检查 → 数据卷 → 初始化 SQL → PHP 扩展**，是把前两期的知识全用上了。

**步骤 7：验证"数据是真的持久化了"**

```bash
# 删掉整个工程（容器+网络），但保留数据卷
docker compose down

# 再起来 —— 数据应该还在（因为 mysql_data 卷没删）
docker compose up -d
curl -s http://127.0.0.1:8080 | grep -A5 "用户数据"
# 期望：三行数据还在

# 现在连卷一起删，再起来 —— 数据会被重新初始化（回到初始的 3 行）
docker compose down -v
docker compose up -d --build
curl -s http://127.0.0.1:8080 | grep -A5 "用户数据"
```

> **这个对比实验说明了 `down` 和 `down -v` 的差别，也说明了"为什么改初始化 SQL 要先 `down -v`"。**

---

## 二、Dockerfile（课件第二章）

### 2.1 Dockerfile 构建原理

> **课件原话**：
> **Dockerfile 其实是一个文件，里面包含了一系列的指令。这一系列的指令，应该是描述"如何构建镜像"。最后我们可以用 `docker build` 生成镜像。从文件内容的最上向下执行，每执行一行，就会生成一层镜像。**

**课件用一个类比说清了三个东西的关系**：

```text
dockerfile  = 源代码
images      = 编译之后的程序
container   = 程序运行的实例
```

**"每执行一行就生成一层"是什么意思（★ 关键）**：

```text
FROM centos                    →  第 1 层（基础镜像）
RUN yum install -y vim         →  第 2 层（装了 vim 之后的变化）
RUN yum install -y net-tools   →  第 3 层（装了 net-tools 之后的变化）

所以：
· 层数越多镜像越大（所以最佳实践是合并 RUN）
· 没变的层可以复用缓存（所以构建会很快）
· 改了下层，上层缓存全部失效需要重建
```

> **一个实际影响**：如果你把 `RUN yum install` 写在前面、`COPY 代码` 写在后面，那么**只改一行代码也会让"装依赖"那层缓存失效，要重新装一遍**。所以最佳实践是：**先 COPY 依赖清单装依赖，最后再 COPY 源码**。

### 2.2 Dockerfile 指令集（课件原文，★ 背下来）

| 指令 | 作用（课件原话） |
| --- | --- |
| **`FROM`** | 指定基础镜像，一般是第一条指令 |
| **`RUN`** | 构建执行命令，**每个 RUN 都会新建一层镜像** |
| **`CMD`** | 容器启动时执行，**如果 Dockerfile 里有多个 CMD，只有最后一个生效** |
| **`COPY` / `ADD`** | 把文件拷贝到容器里。**COPY 是纯拷贝；ADD 支持 tar.gz 文件在拷贝的时候自动解压** |
| **`WORKDIR`** | 工作目录，有点类似于母机里的 `cd` 命令 |
| **`ENV`** | 定义环境变量 |
| **`EXPOSE`** | 提示监听端口 |
| **`LABEL`** | 用于标记镜像信息（作者、版本等） |

**再补几个生产常用的（课件没列但一定会用）**：

| 指令 | 作用 | 和谁容易混 |
| --- | --- | --- |
| `ENTRYPOINT` | 容器启动时的**固定入口**命令 | 和 `CMD` 的区别见下 |
| `ARG` | **构建时**的参数（`docker build --build-arg`） | 和 `ENV` 的区别见下 |
| `VOLUME` | 声明数据卷挂载点 | — |
| `USER` | 指定运行容器时的用户（**安全最佳实践**） | — |

#### ★ 三组必须分清的指令

**① `RUN` vs `CMD` vs `ENTRYPOINT`**

```text
RUN         ：在【构建镜像】时执行，结果固化到镜像层里（装软件用这个）
CMD         ：在【容器启动】时执行，可以被 docker run 后面的命令覆盖
ENTRYPOINT  ：在【容器启动】时执行，不容易被覆盖，通常和 CMD 配合
```

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
# 最终执行：nginx -g "daemon off;"
# docker run 镜像 -g "daemon on;"  → 覆盖 CMD，变成 nginx -g "daemon on;"
```

**② `COPY` vs `ADD`**

```text
COPY：纯拷贝，语义清晰，【推荐】
ADD ：除了拷贝，还会自动解压 tar.gz，还支持从 URL 下载（不推荐，行为不直观）
→ 一句话：能用 COPY 就用 COPY。
```

**③ `ENV` vs `ARG`**

```text
ENV ：构建和运行时都存在的环境变量，会留在镜像里
ARG ：只在构建时有效，构建完就没了，不会留在镜像里
```

> **格式上的一个坑**：`CMD` / `ENTRYPOINT` 有两种写法 ——
> - **exec 形式**（推荐）：`CMD ["nginx", "-g", "daemon off;"]` —— 直接启动进程，**能正确接收信号**
> - **shell 形式**：`CMD nginx -g "daemon off;"` —— 会包一层 `/bin/sh -c`，**PID 1 变成 shell，信号收不到**（这就是"容器 stop 很慢/优雅退出失效"的常见原因）

### 2.3 Dockerfile 实战 1：给 centos 装上 vim 和 net-tools

**课件内容很简短，我把它补成完整可复现的过程**：

**步骤 1：准备目录和文件**

```bash
mkdir -p /data/dockerfile-lab/mycentos
cd /data/dockerfile-lab/mycentos
cat >Dockerfile<<'EOA'
FROM centos
RUN yum install -y vim
RUN yum install -y net-tools
EOA
cat Dockerfile        # 检验
```

> **这是课件原样的写法**（两个 RUN、各装一个包）。**但生产上要改成一行**：
> ```dockerfile
> FROM centos
> RUN yum install -y vim net-tools && yum clean all
> ```
> 原因：① 两个 RUN = 两层，合一行 = 一层，镜像更小；② `yum clean all` 清掉缓存（不清的话缓存也会被固化进层里，白白占几十 MB）。
>
> **另外提醒**：`centos` 官方镜像已经 DEPRECATED（CentOS 8 停止维护）。练习可以照课件用 `centos`，新项目建议用 `rockylinux:8` 或 `almalinux:8`。

**步骤 2：构建（注意最后那个点）**

```bash
docker build -t mycentos:1.0 .
```

**预期输出**：

```text
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM centos
 ---> 5d0da3dc9764
Step 2/3 : RUN yum install -y vim
 ---> Running in 3f2a1c...
 ...
Removing intermediate container 3f2a1c...
 ---> abc123def456
Step 3/3 : RUN yum install -y net-tools
 ---> Running in 8b1d2e...
Removing intermediate container 8b1d2e...
 ---> fed654cba321
Successfully built fed654cba321
Successfully tagged mycentos:1.0
```

> **★ `docker build -t mycentos:1.0 .` 最后那个 `.` 是什么**：
> 它是**构建上下文（build context）**，表示"把当前目录打包发给 docker daemon"。
> **常见报错**：忘了写 `.` → 报 `"docker build" requires exactly 1 argument`。
> **为什么 Dockerfile 里不能用 `../xxx`**：因为只能访问上下文目录里的文件，上下文之外的拿不到。

**步骤 3：验证镜像**

```bash
docker images | grep mycentos
docker history mycentos:1.0
# 期望：能看到 FROM/RUN/RUN 各一层
```

**步骤 4：验证镜像里的东西真的能用**

```bash
docker run -it --rm mycentos:1.0 bash
# 进容器后：
#   vim --version | head -1        → 有输出
#   netstat -v                     → 有输出（这就是 net-tools 提供的）
#   which vim netstat              → 能显示路径
#   exit
```

> **`--rm` 是什么**：容器退出后**自动删除**，适合这种"跑一下就走"的一次性验证，避免留一堆 Exited 容器。
>
> **这套"构建 → 看 history → 进容器验证"的三步**，是验证任何 Dockerfile 的标准流程。

### 2.4 Dockerfile 实战 2：把 PHP 镜像构建出来

**这个实战就是 1.6 那个 demo 的镜像部分**，这里把构建过程单独讲清楚。

**Dockerfile（同 1.6）**：

```dockerfile
FROM php:7.4-apache
RUN docker-php-ext-install mysqli
WORKDIR /var/www/html
COPY index.php /var/www/html/index.php
```

**单独构建并验证**：

```bash
cd /data/compose-php-mysql-demo/php

# 1) 单独构建（不通过 compose）
docker build -t demophp:1.0 .
docker images | grep demophp
docker history demophp:1.0

# 2) 看每层做了什么
#    期望看到 FROM php:7.4-apache / RUN docker-php-ext-install mysqli /
#    WORKDIR / COPY index.php 四层

# 3) 单独跑一下，验证 mysqli 扩展确实装了
docker run --rm demophp:1.0 php -m | grep -i mysqli
# 期望输出 mysqli

# 4) 验证 index.php 确实被拷进去了
docker run --rm demophp:1.0 ls -l /var/www/html/
# 期望看到 index.php
```

> **★ 第 3 步是这类 Dockerfile 最关键的验证**：`php -m` 列出所有已加载的扩展，`grep mysqli` 能匹配到就说明 `docker-php-ext-install mysqli` 成功了。
>
> **为什么必须验证**：`RUN docker-php-ext-install mysqli` 失败时，镜像**照样能构建成功**（因为很多镜像的构建不会因为装扩展失败而中断），结果就是运行时报 `Class "mysqli" not found`。**构建成功 ≠ 功能可用**，一定要验证。

**通过 compose 一起构建**：

```bash
cd /data/compose-php-mysql-demo

# 改了 Dockerfile 或 index.php 之后，必须加 --build 才会重新构建
docker compose up -d --build

# 验证：网页能看到数据（同 1.6 步骤 6）
curl -s http://127.0.0.1:8080 | grep -i "MySQL 连接状态"
```

> **★ 最容易踩的坑**：**改了 index.php 却发现网页没变化** —— 因为 `docker compose up -d` 默认**不会重建镜像**，用的是旧镜像。
> 解决：`docker compose up -d --build`（强制重建），或者 `docker compose build && docker compose up -d`。
>
> **原理**：`COPY index.php` 这一层的内容变了，但 compose **不知道**，它只检查镜像存不存在。所以凡是改了构建输入（Dockerfile、被 COPY 的文件）之后，都要 `--build`。

---

## 三、Prometheus 监控（课件第三章）

### 3.1 四个组件各干什么（课件原文）

> **prometheus：采集和存储指标**
> **cadvisor：采集容器指标**
> **node-exporter：采集宿主机指标**
> **grafana：展示图表**

**课件的链路图**：

```text
cadvisor 和 node-exporter ：提供数据
        │
        ▼
prometheus ：存储数据
        │
        ▼
grafana ：展示数据
```

**展开成一张完整的数据流图**：

```text
【数据源】                          【存储】            【展示】
node-exporter (宿主机的 CPU/内存/磁盘) ──┐
                                       ├──► prometheus ──► grafana
cadvisor      (容器的 CPU/内存/网络)   ──┘     (拉取+存储)     (画图)
                                                    ▲
                                          prometheus 自己也算一个
                                          target（监控自己）
```

> **两个关键理解**：
> ① **Prometheus 是"拉"（pull）模型**：它按 `prometheus.yml` 里配的 targets，**定时主动去抓**这些地址的 `/metrics` 接口。所以 exporter 只需要"暴露指标"，不用管推送。
> ② **exporter 就是"翻译器"**：把宿主机的 `/proc`、docker 的统计信息翻译成 Prometheus 能读的格式。

### 3.2 完整部署（★★★ 照着敲）

#### 步骤 1：建目录

```bash
mkdir -p /opt/monitoring/{prometheus,data/prometheus,data/grafana}
```

#### 步骤 2：写 Prometheus 配置

```bash
cat >/opt/monitoring/prometheus/prometheus.yml<<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
EOF
```

> **注意 targets 写的是"服务名:端口"**，不是 IP —— 因为它们在同一个 compose 网络里，靠服务名解析（这就是 1.5 学的）。**如果这里写 IP，容器重建后 IP 变了就抓不到**。

#### 步骤 3：写 compose 文件

```bash
cat > /opt/monitoring/docker-compose.yml <<'EOF'
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - /opt/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /opt/monitoring/data/prometheus:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
    extra_hosts:
      - "host.docker.internal:host-gateway"

  cadvisor:
    image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: always
    ports:
      - "8080:8080"
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: always
    ports:
      - "9100:9100"
    pid: host
    volumes:
      - /:/host:ro,rslave
    command:
      - --path.rootfs=/host

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - /opt/monitoring/data/grafana:/var/lib/grafana
EOF
```

**这套配置里几个必须理解的参数**：

| 参数 | 为什么需要 |
| --- | --- |
| `--storage.tsdb.retention.time=15d` | **数据只留 15 天**。不设的话 Prometheus 会一直占磁盘，直到写满 |
| `--web.enable-lifecycle` | 打开这个才能用 `POST /-/reload` **热加载配置**（改了 prometheus.yml 不用重启） |
| **`cadvisor` 的 `privileged: true` + 挂 `/sys`、`/var/lib/docker`** | cAdvisor 要读宿主机的 cgroup 和 docker 数据才能统计容器指标，必须提权 + 挂宿主机路径 |
| **`node-exporter` 的 `pid: host`** | 要看到宿主机的所有进程，不能只在容器自己的 PID 命名空间里 |
| `--path.rootfs=/host` + 挂 `/:/host:ro` | 让 node-exporter 读**宿主机**的 `/proc`（否则读到的是容器自己的，数字就不对了） |
| `extra_hosts: host.docker.internal:host-gateway` | 让容器能用 `host.docker.internal` 访问宿主机 |

> **★ 安全提醒**：`privileged: true` 等于把宿主机的能力给了容器，**cAdvisor 和 node-exporter 只能在内网部署**，端口不要暴露到公网。

#### 步骤 4：启动

```bash
cd /opt/monitoring
docker compose up -d
docker compose ps
```

**预期输出**：

```text
NAME            IMAGE                STATUS
cadvisor        .../cadvisor:latest  Up
grafana         grafana/grafana      Up
node-exporter   prom/node-exporter   Up
prometheus      prom/prometheus      Up
```

#### 步骤 5：逐个验证（★ 分层验证，不要只看容器起来没）

```bash
# 1) 四个容器都在
docker compose ps

# 2) 端口都在听
ss -lntp | grep -E '9090|8080|9100|3000'

# 3) exporter 自己能出指标（这是最关键的一步）
curl -s http://127.0.0.1:9100/metrics | head -20
# 期望：能看到 node_cpu_seconds_total、node_memory_... 之类的行

curl -s http://127.0.0.1:8080/metrics | grep -m3 container_
# 期望：能看到 container_cpu_... 之类的行

# 4) Prometheus 自己活着
curl -s http://127.0.0.1:9090/-/healthy
# 期望：Prometheus Server is Healthy.
```

**步骤 6：在 Prometheus 网页上确认 target 都是 UP（★ 这一步才算联调成功）**

浏览器访问 `http://<宿主机IP>:9090`，然后：

```text
顶部菜单 Status → Targets
```

**期望看到三个 job 的 State 都是 `UP`**：

```text
prometheus     (1/1 up)     endpoint http://prometheus:9090/metrics
cadvisor       (1/1 up)     endpoint http://cadvisor:8080/metrics
node-exporter  (1/1 up)     endpoint http://node-exporter:9100/metrics
```

> **★ 这一页是 Prometheus 排障的第一站**：
> - `UP` = 能抓到最后一次指标
> - `DOWN` = 抓不到，鼠标移上去能看到报错（常见：服务名写错、端口不对、容器没起）
> - 只要这里不是全 UP，Grafana 里一定是"没数据"的

**顺便在 Prometheus 网页上手写一个查询**（Graph 页）：

```promql
# 宿主机 CPU 使用率
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 宿主机可用内存（GB）
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024

# 运行中的容器数
count(container_last_seen{name!=""})
```

#### 步骤 7：Grafana 接上 Prometheus 并出图

```text
1) 浏览器访问 http://<宿主机IP>:3000
2) 默认账号密码都是 admin / admin，首次登录会要求改密码
3) 左侧菜单 → Connections（老版本叫 Configuration）→ Data sources → Add data source
4) 选 Prometheus
5) URL 填 http://prometheus:9090     ← ★ 用服务名，不要用 localhost
6) 拉到底点 Save & test，期望提示 "Data source is working"
7) 左侧菜单 → Dashboards → New → Import
8) 填 dashboard ID，常用的：
     1860  → Node Exporter Full（宿主机总览，最经典）
     193   → Docker monitoring（容器总览）
   选好数据源点 Import
9) 回到 Dashboard 就能看到 CPU / 内存 / 磁盘 / 网络 的曲线
```

> **★ Grafana 里 URL 为什么用 `prometheus:9090`**：因为 Grafana 和 Prometheus 都在同一个 compose 网络里，Grafana 容器里访问 `localhost:9090` 是访问**它自己**，当然连不上。**这是初学者最常犯的错。**

#### 步骤 8：日常运维

```bash
cd /opt/monitoring

docker compose ps
docker compose logs -f --tail 50 prometheus

# 改了 prometheus.yml 之后热加载（因为开了 --web.enable-lifecycle）
curl -X POST http://127.0.0.1:9090/-/reload
# 或者
docker compose restart prometheus

# 停止
docker compose down
```

### 3.3 常见坑

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| Targets 里某个 job 是 DOWN | 服务名/端口写错，或容器没起 | `docker compose ps` + `docker compose logs 服务名` |
| Grafana 报 "Data source is working" 失败 | URL 用了 localhost | 改成 `http://prometheus:9090` |
| node-exporter 的数字和宿主机对不上 | 没设 `pid: host` 或没挂 `/:/host` | 按 3.2 的配置补齐 |
| cadvisor 起不来 / 看不到容器指标 | 缺 `privileged: true` 或没挂 `/sys`、`/var/lib/docker` | 补齐挂载和权限 |
| 磁盘被 Prometheus 写满 | 没设 retention | 加 `--storage.tsdb.retention.time=15d` |
| 改了 prometheus.yml 不生效 | 没 reload | `curl -X POST http://127.0.0.1:9090/-/reload` |
| Grafana 重启后要重新配数据源 | 没挂 `/var/lib/grafana` | 加卷（本配置已挂） |

---

## 四、Docker 项目实战：用容器部署 Zabbix（课件第四章）

> **课件用同一个项目做了两遍**：先手工 `docker run` 一条条起，再用 compose 一把起。**对比着看，就是"为什么要有 Compose"的最佳教材。**

### 4.1 项目说明

**要部署的东西（Zabbix 官方推荐的三容器架构）**：

```text
zabbix-web-nginx-mysql   → 前端页面（用户访问这个）
        │
zabbix-server-mysql      → 服务端（采集和计算）
        │
mysql-server             → 数据库（存历史数据）
```

| 容器 | 作用 | 端口 |
| --- | --- | --- |
| `mysql-server` | 存 Zabbix 所有数据 | 不对外映射（只给内部用） |
| `zabbix-server-mysql` | Zabbix Server，负责采集与告警 | 10051 |
| `zabbix-web-nginx-mysql` | Web 前端（Nginx + PHP） | **80** |

**三个容器之间靠什么通信**：靠**环境变量互相指认** —— web 要知道 server 在哪、server 要知道 db 在哪。这一点在两种方式里都要处理。

### 4.2 方式一：docker run 手工部署（课件原版）

#### 步骤 1：先建数据卷

```bash
docker volume create -d local mysql_data
docker volume create -d local mysql_logs
docker volume create -d local mysql_conf
docker volume create -d local zabbix_server

docker volume ls | grep -E 'mysql_|zabbix_'
```

> **为什么先建卷**：`docker run -v 卷名:路径` 时如果卷不存在，Docker 会自动创建；但**显式创建的好处是你能先想清楚要哪些卷、命名规范统一**，也方便后面 `docker volume ls` 排查。

#### 步骤 2：起 MySQL（第一个容器）

```bash
docker run --name mysql-server -t \
  -v mysql_data:/var/lib/mysql \
  -v mysql_logs:/var/log/mysql \
  -v mysql_conf:/etc/mysql \
  -e MYSQL_DATABASE="zabbix" \
  -e MYSQL_USER="zabbix" \
  -e MYSQL_PASSWORD="zabbix_pwd" \
  -e MYSQL_ROOT_PASSWORD="123456" \
  --restart=unless-stopped \
  -d mysql:8.0 \
  --character-set-server=utf8 --collation-server=utf8_bin \
  --default-authentication-plugin=mysql_native_password
```

**这里的三件事值得说清**：

| 配置 | 为什么 |
| --- | --- |
| `MYSQL_DATABASE=zabbix` | 官方镜像会自动建库（不用手工 `create database`） |
| `MYSQL_USER/MYSQL_PASSWORD` | 自动建普通用户并授权，Zabbix 就用这个账号连（**比用 root 更规范**） |
| **`--default-authentication-plugin=mysql_native_password`** | **MySQL 8 默认是 `caching_sha2_password`，Zabbix 的 PHP 客户端连不上**，必须切回 `native_password`。这是本项目最大的坑之一 |

**检验**：

```bash
docker ps | grep mysql-server
docker logs --tail 10 mysql-server        # 找 "ready for connections"

# 验证库和用户都建好了
docker exec mysql-server mysql -uroot -p123456 -e "show databases;"
docker exec mysql-server mysql -uroot -p123456 -e "select user,host from mysql.user;"
```

#### 步骤 3：起 Zabbix Server（第二个容器）

```bash
docker run --name zabbix-server-mysql -t \
  -v zabbix_server:/etc/zabbix \
  -e DB_SERVER_HOST="mysql-server" \
  -e MYSQL_DATABASE="zabbix" \
  -e MYSQL_USER="zabbix" \
  -e MYSQL_PASSWORD="zabbix_pwd" \
  -e MYSQL_ROOT_PASSWORD="123456" \
  --link mysql-server:mysql \
  --restart=unless-stopped \
  -p 10051:10051 \
  -d zabbix/zabbix-server-mysql:alpine-6.2-latest
```

> **`--link mysql-server:mysql` 是什么**：老的容器互联方式，作用是 ① 让这个容器里**能通过名字 `mysql` 解析到 mysql-server 的 IP**；② 建立启动依赖。
> **现在推荐用自定义网络代替 `--link`**（见 4.3 的 compose 版本），因为 `--link` 是单向的、不好维护，官方也标记为遗留特性。

**检验**：

```bash
docker ps | grep zabbix-server
docker logs --tail 30 zabbix-server-mysql
# ★ 重点看有没有： "database is up and running" / "Zabbix Server started"
# ★ 如果一直报数据库连不上，去看 4.5 的常见坑
ss -lntp | grep 10051
```

#### 步骤 4：起 Zabbix Web（第三个容器）

```bash
docker run --name zabbix-web-nginx-mysql -t \
  -e PHP_TZ="Asia/Shanghai" \
  -e ZBX_SERVER_HOST="zabbix-server-mysql" \
  -e DB_SERVER_HOST="mysql-server" \
  -e MYSQL_DATABASE="zabbix" \
  -e MYSQL_USER="zabbix" \
  -e MYSQL_PASSWORD="zabbix_pwd" \
  -e MYSQL_ROOT_PASSWORD="123456" \
  --link mysql-server:mysql \
  --link zabbix-server-mysql:zabbix-server \
  -p 80:8080 \
  --restart unless-stopped \
  -d zabbix/zabbix-web-nginx-mysql:alpine-6.2-latest
```

> **注意 `-p 80:8080`**：容器里的 Web 服务监听的是 **8080**（不是 80），映射到宿主机的 80。**这种"容器内端口和宿主机端口不一样"的情况很常见，一定要看镜像文档。**

**检验（完整闭环）**：

```bash
docker ps                                   # 三个容器都要 Up
ss -lntp | grep -E ':80 |10051'             # 端口在听
curl -sI http://127.0.0.1/ | head -1        # 期望 200 或 302
```

浏览器访问：

```text
http://<宿主机IP>/
默认账号：Admin（注意首字母大写）
默认密码：zabbix
```

> **登录进去后先改密码**，然后就能看到 Zabbix 的 Dashboard 了。

#### 步骤 5：手工方式的"痛点"复盘（★ 这就是为什么需要 Compose）

```text
1) 三条 docker run，每条 10+ 个参数，敲错一个就要删容器重来
2) 必须严格控制顺序：mysql → server → web（顺序错了就起不来）
3) 用的 --link 是遗留特性，不好维护
4) 密码散落在三条命令里
5) 停止/清理要一条条来：docker rm -f mysql-server zabbix-server-mysql zabbix-web-nginx-mysql
6) 这套配置没地方存，换台机器还得重新敲
```

### 4.3 方式二：Compose 部署（★ 推荐方式，课件完整版）

#### 步骤 1：建目录

```bash
mkdir /opt/zabbix-compose
cd /opt/zabbix-compose
```

#### 步骤 2：写 compose 文件

```bash
cat >compose.yaml<<'EOC'
version: '3.8'

networks:
  zabbix-net:
    driver: bridge

volumes:
  mysql_data:
  mysql_logs:
  mysql_conf:
  zabbix_server:

services:
  # MySQL 8.0 数据库
  mysql-server:
    image: mysql:8.0
    container_name: mysql-server
    restart: unless-stopped
    networks:
      - zabbix-net
    environment:
      MYSQL_DATABASE: "zabbix"
      MYSQL_USER: "zabbix"
      MYSQL_PASSWORD: "zabbix_pwd"
      MYSQL_ROOT_PASSWORD: "123456"
    volumes:
      - mysql_data:/var/lib/mysql
      - mysql_logs:/var/log/mysql
      - mysql_conf:/etc/mysql
    command:
      - --character-set-server=utf8
      - --collation-server=utf8_bin
      - --default-authentication-plugin=mysql_native_password
    expose:
      - "3306"                    # 不映射到宿主机，仅容器内部通信
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p123456"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # Zabbix Server
  zabbix-server-mysql:
    image: zabbix/zabbix-server-mysql:alpine-6.2-latest
    container_name: zabbix-server-mysql
    restart: unless-stopped
    networks:
      - zabbix-net
    depends_on:
      mysql-server:
        condition: service_healthy
    environment:
      DB_SERVER_HOST: "mysql-server"
      MYSQL_DATABASE: "zabbix"
      MYSQL_USER: "zabbix"
      MYSQL_PASSWORD: "zabbix_pwd"
      MYSQL_ROOT_PASSWORD: "123456"
    volumes:
      - zabbix_server:/etc/zabbix
    ports:
      - "10051:10051"
    expose:
      - "10051"

  # Zabbix Web (Nginx + PHP)
  zabbix-web-nginx-mysql:
    image: zabbix/zabbix-web-nginx-mysql:alpine-6.2-latest
    container_name: zabbix-web-nginx-mysql
    restart: unless-stopped
    networks:
      - zabbix-net
    depends_on:
      mysql-server:
        condition: service_healthy
      zabbix-server-mysql:
        condition: service_started
    environment:
      PHP_TZ: "Asia/Shanghai"
      ZBX_SERVER_HOST: "zabbix-server-mysql"
      DB_SERVER_HOST: "mysql-server"
      MYSQL_DATABASE: "zabbix"
      MYSQL_USER: "zabbix"
      MYSQL_PASSWORD: "zabbix_pwd"
      MYSQL_ROOT_PASSWORD: "123456"
    ports:
      - "80:8080"                 # 宿主 80 映射到容器 8080
    expose:
      - "8080"
EOC
```

**这份 compose 文件比手工方式好在哪里（逐条对照）**：

| 手工方式 | Compose 方式 |
| --- | --- |
| `--link` 互联（遗留特性） | **自定义网络 `zabbix-net`**，服务名直接当域名用 |
| 顺序靠人记 | **`depends_on` + `condition: service_healthy`** 自动控制 |
| 数据库就绪靠等 | **`healthcheck` 明确判断"能不能连"** |
| `-p 3306:3306` 把数据库暴露到宿主机（有安全风险） | **`expose: 3306`** 只给容器内部用，**不暴露到宿主机** |
| 密码散在各处 | 集中在 `environment`（进一步可以用 `.env`） |
| 停止要一条条来 | `docker compose down` 一把清 |

#### 步骤 3：启动

```bash
cd /opt/zabbix-compose
docker compose up -d
```

#### 步骤 4：验证（★ 分层验证，别只看容器起来没）

```bash
# 第 1 层：容器状态 —— mysql 必须是 (healthy)
docker compose ps
# 期望：
# NAME                     STATUS
# mysql-server             Up (healthy)
# zabbix-server-mysql      Up
# zabbix-web-nginx-mysql   Up

# 第 2 层：端口
ss -lntp | grep -E ':80 |10051'

# 第 3 层：看日志有没有报错
docker compose logs --tail 30 mysql-server
docker compose logs --tail 30 zabbix-server-mysql
# ★ 重点找 zabbix-server 有没有 "database is up and running" 和 "server #0 started"

# 第 4 层：真正访问
curl -sI http://127.0.0.1/ | head -1
```

**浏览器登录（最终验收）**：

```text
http://<宿主机IP>/
账号：Admin     密码：zabbix
```

**进去后确认这几件事，才算真的跑通**：

```text
1) 能登录进 Dashboard
2) Administration → General → 改成中文界面（可选）
3) 左侧 Monitoring → Latest data 能看到 Zabbix server 自己的数据
4) 能看到主机列表里有 "Zabbix server"
```

#### 步骤 5：日常运维

```bash
cd /opt/zabbix-compose

docker compose ps
docker compose logs -f --tail 50 zabbix-server-mysql
docker compose restart zabbix-server-mysql
docker compose stop                # 停止（保留容器和数据）
docker compose down                # 停止并删除容器/网络（数据卷保留）
```

> **★ 别随手 `down -v`**：那会把 `mysql_data` 一起删掉，Zabbix 的历史数据全没。

### 4.4 两种方式的对比总结

```text
docker run 手工方式：
  适合：临时测试、学命令、跑单个容器
  缺点：多条命令、顺序靠人记、配置没地方存、不好维护

docker compose 方式：
  适合：多容器协同、要反复部署的环境（开发/测试/生产）
  优点：配置即代码、一条命令起停、顺序和依赖声明式管理、能进 git
```

> **一句话（面试可以这么说）**：**「单容器用 `docker run` 就够了；只要超过两个容器、或者要反复部署，就应该用 Compose —— 因为 Compose 把"环境"变成了一个可以版本管理的文件。」**

### 4.5 项目实战的常见坑

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| **zabbix-server 一直报连不上数据库** | **MySQL 8 默认认证插件是 `caching_sha2_password`，Zabbix 的 PHP 客户端不支持** | 加 `--default-authentication-plugin=mysql_native_password`（本项目最大的坑） |
| web 页面报 "Zabbix server is not running" | server 容器没起 / 端口不通 | `docker compose ps` + `docker compose logs zabbix-server-mysql` |
| web 页面 502 | PHP 连不上 server 或 db | 检查 `ZBX_SERVER_HOST` / `DB_SERVER_HOST` 是否是**服务名** |
| 网页时间差 8 小时 | 没设时区 | 加 `PHP_TZ: Asia/Shanghai`；数据库侧加 `--default-time-zone='+8:00'` |
| 中文显示乱码 | 字符集不对 | MySQL 用 `--character-set-server=utf8 --collation-server=utf8_bin` |
| 三个容器起来的顺序乱 | 没有依赖控制 | 用 `depends_on` + `condition: service_healthy` |
| 宿主机 80 被占用 | 已经有 nginx/apache 在跑 | 换端口（如 `8080:8080`），或先停掉占用者（用 `ss -lntp` 看谁在听 80） |
| 数据丢了 | `down -v` 连数据卷删了 | 只用 `down`，别加 `-v` |
| 内存不够，容器反复重启 | Zabbix + MySQL 比较吃内存 | 机器至少 4G；或给 MySQL 加 `--innodb-buffer-pool-size` 限制 |

---

## 五、本期闭环自检清单

| 检查项 | 命令 | 通过标准 |
| --- | --- | --- |
| Compose 三段式结构说得清 | 看 compose 文件 | 能指出 services / volumes / networks |
| 概念分得清 | — | Project / service / container 三者关系能说清 |
| 能把 run 改写成 compose | 对比 1.3 两种方式 | 参数一一对应 |
| .env 变量生效 | `docker compose config` | 能看到替换后的值 |
| 命令集都会 | `up/down/ps/logs/exec/config/top` | 都跑过 |
| compose 网络懂了 | `docker compose exec 服务名 cat /etc/hosts` | 能看到同网络的服务名 |
| **依赖与健康检查懂了** | `docker compose ps` | mysql 显示 `(healthy)`，php 等它起来 |
| **PHP+MySQL demo 跑通** | `curl http://127.0.0.1:8080` | 显示"MySQL 连接状态：成功"+三行数据 |
| 持久化验证过 | `down` 后 `up` | 数据还在 |
| 初始化 SQL 会改 | `down -v` 后 `up` | 数据重新初始化 |
| Dockerfile 构建原理懂 | `docker history 镜像` | 能数出几层、对应哪几行指令 |
| 指令集背得出 | — | FROM/RUN/CMD/COPY/ADD/WORKDIR/ENV/EXPOSE/LABEL |
| 实战 1 跑通 | `docker run -it --rm mycentos:1.0 bash` | vim 和 netstat 都能用 |
| 实战 2 跑通 | `docker run --rm demophp:1.0 php -m`（输出里找 mysqli） | 有 mysqli |
| 构建缓存懂了 | 改 index.php 后 `up -d --build` | 网页内容更新 |
| Prometheus 四件套起来 | `docker compose ps` | 4 个都 Up |
| exporter 有指标 | `curl :9100/metrics` | 有输出 |
| Targets 全 UP | Prometheus → Status → Targets | 3 个 job 都 UP |
| Grafana 出图 | 导入 1860 面板 | 有曲线 |
| Zabbix（run 版）跑通 | 浏览器登录 | Admin/zabbix 能进 |
| Zabbix（compose 版）跑通 | `docker compose ps` | mysql healthy，web 能访问 |
| 能说清两种方式的取舍 | — | 单容器 run、多容器 compose |

---

## 六、自测问题（闭卷）

- [ ] 为什么需要 Compose？它解决了 `docker run` 的哪五个问题？
- [ ] Compose 文件的三段式结构是什么？各自写在哪一层？
- [ ] Project / service / container 三者是什么关系？
- [ ] 为什么卷路径写 `./data` 是安全的？它相对的是谁？
- [ ] `.env` 怎么用？为什么密码不该直接写在 compose 文件里？
- [ ] 怎么确认变量真的被替换了？
- [ ] `docker compose up` 和 `start` 的区别？改了 compose 文件该用哪个？
- [ ] `down` 和 `down -v` 的区别？为什么不能随手加 `-v`？
- [ ] 同一 compose 工程里的服务怎么互相访问？靠 IP 还是服务名？
- [ ] `depends_on` 保证什么？不保证什么？怎么补上？
- [ ] `healthcheck` 怎么写？`condition: service_healthy` 用在哪？
- [ ] MySQL 初始化 SQL 放在哪？为什么改了不生效？
- [ ] Dockerfile 构建时"每行一层"意味着什么？对镜像大小和构建速度有什么影响？
- [ ] `RUN` / `CMD` / `ENTRYPOINT` 三者的区别？
- [ ] `COPY` 和 `ADD` 的区别？推荐用哪个？
- [ ] `ENV` 和 `ARG` 的区别？
- [ ] `docker build` 最后那个 `.` 是什么？不写会怎样？
- [ ] 为什么改了 index.php 网页没变化？怎么解决？
- [ ] Prometheus / cAdvisor / node-exporter / Grafana 各干什么？
- [ ] Prometheus 的 Targets 页面是干什么的？DOWN 了怎么查？
- [ ] Grafana 配数据源为什么不能用 `localhost:9090`？
- [ ] Zabbix 三容器架构里，谁连谁？靠什么连？
- [ ] MySQL 8 的那个"认证插件"坑是什么？怎么解决？
- [ ] 为什么 `expose` 比 `ports` 更安全？

---

## 七、三期总回顾

```text
第一期  概念 + 安装 + 镜像基础
        ├─ Docker 是什么、核心价值、核心组件
        ├─ 在线安装 / 离线安装 / compose 安装
        ├─ Namespace（隔离）与 Cgroup（限制）
        └─ 镜像分层、镜像加速、镜像搜索与拉取

第二期  命令 + 网络 + 数据卷 + 私有仓库
        ├─ 镜像命令、容器命令（run/ps/exec/logs/inspect/cp/commit...）
        ├─ docker0 / veth-pair / 四种网络模式 / SNAT 与 DNAT
        ├─ 三种挂载（匿名 / 具名 / 路径）与数据覆盖规则
        └─ Portainer 可视化 + Harbor 私有仓库

第三期  Compose + Dockerfile + 监控 + 项目实战
        ├─ Compose 三段式、.env、命令集、网络、依赖与健康检查
        ├─ Dockerfile 构建原理与指令集、两个实战
        ├─ Prometheus + cAdvisor + node-exporter + Grafana
        └─ Zabbix 项目：docker run 与 compose 两种部署对比
```

**一条能串起三期的实验链（建议按顺序做一遍）**：

```text
1. 装 Docker + 配加速器                    （第一期）
2. 拉镜像、跑 hello-world、看 docker history（第一期）
3. 用 run 起 MySQL，练端口/卷/环境变量       （第二期）
4. 建自定义网络，两个容器用容器名互 ping      （第二期）
5. 三种挂载各做一遍，验证数据存活            （第二期）
6. 部署 Portainer，用网页管容器              （第二期）
7. 部署 Harbor，推拉一个镜像                 （第二期）
8. 用 Compose 跑通 PHP + MySQL demo         （第三期）
9. 写 Dockerfile 构建自己的镜像              （第三期）
10. 部署 Prometheus + Grafana 出图          （第三期）
11. 用 Compose 部署 Zabbix 并登录            （第三期）
```

> **下一步建议**：这套 Docker 学完之后，正好接 `notes2` 里的 K8s 笔记（`stage2-07-k8s-old.md`、`stage2-08-k8s-new.md`）—— **K8s 解决的就是"Compose 只管一台机器"的问题**：Compose 管一个工程、一台机器；K8s 管一堆服务、一堆机器。