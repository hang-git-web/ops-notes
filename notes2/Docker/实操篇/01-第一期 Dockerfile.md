# 第一期 · Dockerfile（课件第二章）

| 项目 | 内容 |
| --- | --- |
| **你在哪** | 第一期 / 共三期 |
| **上一期产出** | 无 —— 第一期从这里开始 |
| **本期产出** | `demophp:1.0` 镜像 —— **第二期 Compose 会直接用它** |
| **预计耗时** | 1–2 小时 |
| **验收标准** | `docker history 镜像` 能数出层、并对上 Dockerfile 的每一行；`docker run --rm demophp:1.0 php -m` 输出里有 `mysqli` |
| **开工前** | 先跑一遍 `00-三期总览.md` 里的「开工前检查」 |

---

## 一、Dockerfile 构建原理（课件 2.1）

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

## 二、Dockerfile 指令集（课件 2.2，★ 背下来）

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

### ★ 三组必须分清的指令

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

## 三、Dockerfile 实战 2：把 PHP 镜像构建出来（课件 2.4）

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

---

## 本期自检清单（原第五章摘出）

| 检查项 | 命令 | 通过标准 |
| --- | --- | --- |
| Dockerfile 构建原理懂 | `docker history 镜像` | 能数出几层、对应哪几行指令 |
| 指令集背得出 | — | FROM/RUN/CMD/COPY/ADD/WORKDIR/ENV/EXPOSE/LABEL |
| 实战 2 跑通 | `docker run --rm demophp:1.0 php -m`（输出里找 mysqli） | 有 mysqli |
| 构建缓存懂了 | 改 index.php 后 `up -d --build` | 网页内容更新 |

## 本期自测题（闭卷）

- [ ] Dockerfile 构建时"每行一层"意味着什么？对镜像大小和构建速度有什么影响？
- [ ] `RUN` / `CMD` / `ENTRYPOINT` 三者的区别？
- [ ] `COPY` 和 `ADD` 的区别？推荐用哪个？
- [ ] `ENV` 和 `ARG` 的区别？
- [ ] `docker build` 最后那个 `.` 是什么？不写会怎样？
- [ ] 为什么改了 index.php 网页没变化？怎么解决？
