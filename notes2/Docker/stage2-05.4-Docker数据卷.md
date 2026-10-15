# Docker 数据卷（课件第六章）

> **（为什么需要数据卷）**：**数据需要保存，需要持久化，需要方便编辑。**

## 6.1 数据覆盖（★ 两条规则，必须记准）

> **① 如果是 Linux 母机的【空目录】挂载到容器里，容器中的目录数据不会复制到母机里，并且容器目录下现有的数据会被隐藏。**
> **② 如果是 Linux 母机的【有数据目录】挂载到容器里，容器中的目录数据不会复制到母机里，并且容器目录下的现有数据会被隐藏，会显示母机当前数据目录下的数据。**

**翻译成人话**：

```text
挂载 = 用母机的目录【盖住】容器里的目录。
· 母机目录是空的  → 容器里那个目录原来的内容【看不见了】，而且母机的空目录里也不会多出东西
· 母机目录有数据  → 显示母机的数据，容器里原来的内容同样【被盖住】
· 共同点：方向永远是"母机盖住容器"，容器里的东西【不会】被复制到母机
```

**实操验证（非常值得做一遍）**：

```bash
# 1) 准备一个空目录
mkdir -p /data/empty

# 2) 把空目录挂到 nginx 的网站目录上（容器里本来有 index.html）
docker run -d --name web1 -p 8081:80 -v /data/empty:/usr/share/nginx/html nginx

# 3) 看容器里那个目录 —— 原来的 index.html 被"盖"没了
docker exec web1 ls -l /usr/share/nginx/html
# 期望：空的（没有 index.html）

# 4) 访问会看到 403 Forbidden（因为 index.html 不存在）
curl -sI http://127.0.0.1:8081 | head -1

# 5) 母机目录里也不会凭空多出容器原来的文件
ls -l /data/empty
# 期望：还是空的
```

> **这个规则的实际意义**：**部署 MySQL 时，如果挂一个空目录到 `/var/lib/mysql`，MySQL 会以为"数据目录是空的"而执行初始化**（这正好是我们要的）；但如果你把空目录挂到 Nginx 的 `/usr/share/nginx/html`，网站就"打不开"了 —— 这不是坏了，是**被盖住了**。

## 6.2 三种挂载方式（★★ 面试必问）

课件原话：**一个叫具名挂载，一个是匿名挂载，一个是路径挂载。**

### 方式一：匿名挂载（不写母机路径）

```bash
docker run -d -P --name=nginxt001 -v /etc/nginx nginx:1.28
```

> **课件批注**：`/etc/nginx` 指的是**容器的目录**，不是母机的目录。数据会挂载到 `/var/lib/docker/volumes` 目录下。

**检验（看它到底挂到哪了）**：

```bash
docker inspect -f '{{json .Mounts}}' nginxt001 | python -m json.tool
# 期望看到 Source 形如 /var/lib/docker/volumes/<一长串随机ID>/_data

docker volume ls
# 期望看到一个名字是一长串随机十六进制字符的卷（没有可读名字）
```

> **特点**：卷名是**随机字符串**，没有可读性，回头很难认出这个卷是谁的 —— 所以生产上基本不用匿名挂载。

### 方式二：具名挂载（指定卷名，不指定母机路径）

```bash
docker run -d -P --name nginx002 -v nginx-config:/etc/nginx nginx:1.28
```

> **课件批注**：数据会挂载到 `/var/lib/docker/volumes` 目录下；**它与匿名挂载的区别是有"具体的目录名"**。

**检验**：

```bash
docker volume ls
# 期望看到 nginx-config 这个卷（名字可读！）

docker volume inspect nginx-config
# 期望看到 Mountpoint: /var/lib/docker/volumes/nginx-config/_data
```

> **怎么区分匿名和具名（一句话）**：`-v` 后面**只有容器路径**、没有冒号 → 匿名；`-v` 后面是 **`卷名:容器路径`**（卷名不以 `/` 开头）→ 具名。

### 方式三：路径挂载 / 绑定挂载（指定母机具体路径）

```bash
docker run -d -P --name=nginx003 -v /data/nginx:/etc/nginx nginx:1.28
```

**检验**：

```bash
docker inspect -f '{{json .Mounts}}' nginx003 | python -m json.tool
# 期望 Source 就是 /data/nginx（我们自己指定的），Type 是 bind
ls -l /data/nginx        # 母机目录里能看到容器 /etc/nginx 的内容吗？答案是：看不到（被盖住的是容器那一侧）
```

> **这三种到底怎么选（★ 面试答法）**：
>
> | 方式 | 写法 | 母机路径 | 适合 | 生产推荐度 |
> | --- | --- | --- | --- | --- |
> | 匿名挂载 | `-v /容器路径` | 随机 ID，在 `/var/lib/docker/volumes/` | 临时测试 | 不推荐（名字不可读） |
> | **具名挂载** | `-v 卷名:/容器路径` | `/var/lib/docker/volumes/卷名/_data` | **数据库、需要 Docker 管理的数据** | **推荐** |
> | **路径挂载** | `-v /母机路径:/容器路径` | 你指定的任意路径 | **配置文件、网站代码、日志** | **推荐** |
>
> **一句话总结**：**「数据交给 Docker 管就用具名卷；需要自己在母机上直接编辑、看日志的，就用路径挂载。」**

### 常用卷管理命令

```bash
docker volume ls                    # 列出所有卷
docker volume inspect 卷名          # 看卷详情（含 Mountpoint）
docker volume create 卷名           # 手动创建卷
docker volume rm 卷名               # 删除卷（有容器在用会报错）
docker volume prune                 # 清理所有没被使用的卷（★ 会删数据，生产慎用）
```

> **★ 重要提醒**：**`docker volume prune` 会删掉未被引用卷里的所有数据，且不可恢复。** 生产上执行前一定要先 `docker volume ls` 确认，或者干脆不用它。
>
> **一个高频面试题**：「容器删了，数据怎么才能不丢？」
> 答：**用 `-v` 挂载 —— 具名卷或母机路径都在容器之外，容器删了数据还在；只有放在容器可写层里的数据才会跟着容器消失。**

### 综合实验：把三种挂载的存活情况一次验证完

```bash
# A. 不挂载（数据在可写层）
docker run -d --name t-a -p 8081:80 nginx
docker exec t-a bash -c 'echo A > /usr/share/nginx/html/a.txt'
docker rm -f t-a
docker run -d --name t-a2 -p 8081:80 nginx
docker exec t-a2 ls /usr/share/nginx/html       # a.txt 不见了 → 数据丢了

# B. 具名卷（数据在卷里）
docker rm -f t-a2
docker run -d --name t-b -p 8081:80 -v myhtml:/usr/share/nginx/html nginx
docker exec t-b bash -c 'echo B > /usr/share/nginx/html/b.txt'
docker rm -f t-b
docker run -d --name t-b2 -p 8081:80 -v myhtml:/usr/share/nginx/html nginx
docker exec t-b2 cat /usr/share/nginx/html/b.txt    # 输出 B → 数据还在

# C. 母机路径（数据在母机目录里）
docker rm -f t-b2
mkdir -p /data/myhtml && echo C > /data/myhtml/c.txt
docker run -d --name t-c -p 8081:80 -v /data/myhtml:/usr/share/nginx/html nginx
curl -s http://127.0.0.1:8081/c.txt                 # 输出 C → 母机直接改文件、容器立即可见

# 清理
docker rm -f t-c
docker volume rm myhtml
```

> **★ 实验 C 是路径挂载最大的优势**：改配置/改页面**不用进容器**，在母机上 `vim` 一下、`docker restart` 一下就行 —— 这也是"方便编辑"的意思。
