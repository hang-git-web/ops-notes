MySQL 两种登录方式：socket 与 TCP

## 一、先记住一句话

localhost 在本机连 MySQL 时，**默认不是走网络的，而是走一个本机文件**。

所以：

- `mysql -u root -p -S /path/mysql.sock` —— 走这个文件（叫 socket）
- `mysql -u root -p -h 127.0.0.1 -P 3307` —— 走网络（叫 TCP）

两者都能连上同一个实例，但不是同一条路。

---

## 二、为什么会有两种连法

MySQL 客户端在连服务器时，会自己判断走哪条路：

| 你写的命令 | 客户端实际走的路 |
| --- | --- |
| `mysql -u root -p` | socket（用默认路径） |
| `mysql -u root -p -S /var/lib/mysql2/mysql.sock` | socket（用你指定的路径） |
| `mysql -u root -p -h localhost` | socket（`localhost` 是暗号，意思是走 socket） |
| `mysql -u root -p -h 127.0.0.1 -P 3307` | TCP |

关键点：**用了 socket 的时候，`-P` 端口参数是被忽略的**，因为根本没走网络。

---

## 三、打个比方

想象一栋办公楼里有两个办公室：

```
        ┌──────────────────────────────────────────────┐
        │  本机（一台服务器）                          │
        │                                              │
        │   实例1  门牌 3306  内部通道 /var/lib/mysql/mysql.sock
        │   实例2  门牌 3307  内部通道 /var/lib/mysql2/mysql.sock
        │                                              │
        └──────────────────────────────────────────────┘
               ↑                        ↑
        走内部通道(socket)        走电话线(TCP)
        不出这栋楼                经过网络协议栈
```

- **socket 像楼里的内部通道**：只要知道通道在哪（路径），推门就进，不出楼、不需要电话号码。
- **TCP 像打电话**：要先有电话线路，还要拨对分机号（端口）。

两个办公室的**内部通道必须各自不同**，否则谁也不知道你在敲哪扇门——这就是为什么第二个实例要在配置里单独指定 `socket = /var/lib/mysql2/mysql.sock`。

---

## 四、两种方式的具体差别

| 对比项 | socket 方式 | TCP 方式 |
| --- | --- | --- |
| 走什么路 | 本机文件，不经过网卡 | 网络协议栈 |
| 靠什么找到实例 | socket 文件的路径 | IP + 端口 |
| 没写对会怎样 | 报 socket 不存在 | 报连接被拒绝 |
| 速度 | 更快 | 稍慢 |
| 谁能尝试连接 | 电脑上有权限读这个文件的用户 | 任何本机用户都能试着连 |
| 多实例会不会连错 | 路径唯一，写对就进对 | 端口记错会连到隔壁实例 |
| 远程能连吗 | 不能，只能在服务器本机 | 能（前提是监听对外、防火墙放行） |
| 各种客户端都支持吗 | 少数图形工具不支持 | 几乎所有工具都支持 |

**速度**：socket 省掉了网络收发包的开销，大量小查询、批量导入导出、备份时差距明显；偶尔手动登进去敲两条命令，感觉不到差别。

**安全**：socket 更严。socket 文件有属主和权限，权限不对的用户连试都试不了。TCP 的本机端口则是本机任何用户都能尝试连接，只能靠密码挡住。

---

## 五、最容易踩的三个坑

### 坑 1：以为写了 `-P 3307` 就连的是第二个实例

`mysql -u root -p -P 3307`（没写 `-S`）里，3307 被忽略，实际连的是**默认 socket**，也就是第一个实例。

如果两个实例的 root 密码恰好一样，你会毫无察觉地登进了错误的实例，做的改动全落在隔壁。

进错实例会有多尴尬，一句话就能验证——登录后看：

```sql
SELECT @@port, @@socket, @@datadir;
```

或者直接在客户端里敲 `status`，看这行：

```
Connection: Localhost via UNIX socket
Connection: 127.0.0.1 via TCP/IP
```

### 坑 2：TCP 连 127.0.0.1 提示 Access denied，但密码明明是对的

因为走 TCP 时，服务器看到的来源是 IP `127.0.0.1`；走 socket 时，服务器认为是 `localhost`。这是两个不同的来源。

MySQL 里的账号是「用户名 + 来源」配对的：

- `'root'@'localhost'` 只管 socket 来的连接
- `'root'@'127.0.0.1'` 只管从 127.0.0.1 来的 TCP 连接
- `'user'@'%'` 匹配所有来源

所以只建了 `root@localhost`，改用 TCP 就可能登不进去；反过来也一样。

### 坑 3：`-h localhost` 并不是走 TCP

写 `localhost` 时客户端会自动退回 socket 模式。想强制走 TCP，要写 IP 或者加参数：

```bash
mysql -u root -p -h 127.0.0.1 -P 3307 --protocol=TCP
```

---

## 六、什么时候用哪个

**用 socket 的场景**

- 在服务器本机上做运维、排错
- 写脚本、做自动化任务
- `mysqldump` 备份、导数据
- 本机压测、对性能敏感的应用连接

**用 TCP 的场景**

- 从别的机器连过来
- 容器、Kubernetes 等跨环境的连接
- Navicat、DataGrip、DBeaver 这类图形工具
- 想验证端口监听、防火墙规则是否配好

**一个顺手的建议**：如果某个实例只打算给本机用，就配上一句 `skip-networking`（或 `bind-address=127.0.0.1`），让它干脆不监听端口，全部走 socket。这样即使防火墙没配好，外面也连不进来。

---

## 七、命令速查

```bash
# 连第一个实例（默认 socket）
mysql -u root -p

# 连第二个实例（指定它自己的 socket）
mysql -u root -p -S /var/lib/mysql2/mysql.sock

# 连第二个实例（走 TCP）
mysql -u root -p -h 127.0.0.1 -P 3307

# 强制走 TCP（即使写的是 localhost）
mysql -u root -p -h 127.0.0.1 -P 3307 --protocol=TCP

# 查看某个实例当前的端口与 socket 路径
mysql -u root -p -S /var/lib/mysql2/mysql.sock -e "SELECT @@port, @@socket, @@datadir;"
```

---

## 八、常见报错对照

| 报错 | 含义 | 处理方向 |
| --- | --- | --- |
| `Can't connect to local MySQL server through socket '/xxx/mysql.sock'` | socket 路径不对或实例没启动 | 核对 `-S` 路径、确认实例在运行 |
| `Can't connect to MySQL server on '127.0.0.1' (111)` | 端口没监听或被拒绝 | 检查实例是否监听该端口、防火墙 |
| `Access denied for user 'root'@'127.0.0.1'` | 来源不匹配 | 建对应来源的账号，或用 socket 连接 |
| 连上了但数据不对 | 连错实例 | 用 `SELECT @@port, @@socket;` 确认 |
