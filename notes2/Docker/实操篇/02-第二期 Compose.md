# 第二期 · Compose（课件第一章）

| 项目 | 内容 |
| --- | --- |
| **你在哪** | 第二期 / 共三期 |
| **上一期产出** | `demophp:1.0` 镜像（第一期的成果，本期直接拿来用） |
| **本期产出** | 一个能跑的 PHP + MySQL 双容器工程 |
| **预计耗时** | 1–2 小时 |
| **验收标准** | `docker compose ps` 里 mysql 显示 `(healthy)` 且 php 等它起来；`curl http://127.0.0.1:8080` 出「连接成功」+ 三行数据 |
| **开工前** | 先跑一遍 `00-三期总览.md` 里的「开工前检查」；`ss -lntp` 确认 8080 没被占 |

---

## 一、为什么需要 Compose（课件 1.1）

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

## 二、Compose 文件（课件 1.2）

### 文件命名与目录结构（课件原文）

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

### 文件的三段式结构（★ 背这个骨架）

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

### 概念的层次（课件原文）

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

## 三、Compose 命令集（课件 1.3）

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

## 四、Compose 网络（课件 1.4）

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

## 五、服务依赖与健康检查（课件 1.5，★ 本篇最重要的实验）

> **这一节是第三期的核心**：`depends_on` 只保证"启动顺序"，`healthcheck` 才保证"真的能用"。课件给的完整案例正是为了说明这件事。

### 先理解问题：为什么光有 depends_on 不够

```text
depends_on: mysql 的意思是"先启动 mysql 容器，再启动 php 容器"。
但 MySQL 容器"启动了"不等于"能连了" —— 它启动后要初始化数据目录、
建系统表、加载配置，这个过程可能要 20~60 秒。

如果 php 在 MySQL 还没就绪时就连接，就会连接失败，
表现为：容器起来了，但网页报 "Connection refused" / "Access denied"。
```

**解决办法**：给 MySQL 加 `healthcheck`，让 php 用 `condition: service_healthy` 等它真正健康。

### 完整实验：PHP + MySQL 的 Compose 工程（照着敲就能跑）

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

---

## 本期自检清单（原第五章摘出）

| 检查项 | 命令 | 通过标准 |
| --- | --- | --- |
| Compose 三段式结构说得清 | 看 compose 文件 | 能指出 services / volumes / networks |
| 概念分得清 | — | Project / service / container 三者关系能说清 |
| 命令集都会 | `up/down/ps/logs/exec/config/top` | 都跑过 |
| compose 网络懂了 | `docker compose exec 服务名 cat /etc/hosts` | 能看到同网络的服务名 |
| **依赖与健康检查懂了** | `docker compose ps` | mysql 显示 `(healthy)`，php 等它起来 |
| **PHP+MySQL demo 跑通** | `curl http://127.0.0.1:8080` | 显示"MySQL 连接状态：成功"+三行数据 |
| 持久化验证过 | `down` 后 `up` | 数据还在 |
| 初始化 SQL 会改 | `down -v` 后 `up` | 数据重新初始化 |
| 能说清两种方式的取舍 | — | 单容器 run、多容器 compose |

## 本期自测题（闭卷）

- [ ] 为什么需要 Compose？它解决了 `docker run` 的哪五个问题？
- [ ] Compose 文件的三段式结构是什么？各自写在哪一层？
- [ ] Project / service / container 三者是什么关系？
- [ ] 为什么卷路径写 `./data` 是安全的？它相对的是谁？
- [ ] `docker compose up` 和 `start` 的区别？改了 compose 文件该用哪个？
- [ ] `down` 和 `down -v` 的区别？为什么不能随手加 `-v`？
- [ ] 同一 compose 工程里的服务怎么互相访问？靠 IP 还是服务名？
- [ ] `depends_on` 保证什么？不保证什么？怎么补上？
- [ ] `healthcheck` 怎么写？`condition: service_healthy` 用在哪？
- [ ] MySQL 初始化 SQL 放在哪？为什么改了不生效？
- [ ] 为什么 `expose` 比 `ports` 更安全？
