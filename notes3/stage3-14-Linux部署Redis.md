# Redis

Redis 是一个开源的内存数据结构存储系统。它支持多种数据类型，如字符串、哈希、列表、集合、有序集合等，并且提供了强大的功能如持久化、分布式支持等。Redis 主要用于缓存、消息队列、任务调度等场景，具有非常高的性能

## Redis 特点

- **高性能**：Redis 的操作大多数都是在内存中进行的，速度非常快。它能够在单台服务器上处理每秒数百万次的操作。
- **多种数据类型**：支持字符串、哈希、列表、集合、有序集合等多种数据结构。
- **持久化机制**：通过 RDB 快照和 AOF 追加文件支持数据持久化，保证数据不会丢失。
- **高可用性**：通过 Redis Sentinel 和 Redis 集群提供高可用性和分布式部署。
- **支持发布/订阅（Pub/Sub）**：允许客户端订阅特定的消息，并接收发布的消息。
- **简洁的命令行操作**：Redis 提供了简洁且高效的命令行工具 `redis-cli` 来与 Redis 服务交互

## Redis 的应用场景

- **缓存**：减少数据库查询次数，提高访问速度。
- **Session 管理**：通过 Redis 保存用户会话信息。
- **实时数据处理**：如实时统计、消息队列等。
- **排行榜**：利用 Redis 的有序集合来实现实时排行榜。
- **任务队列**：Redis 支持列表数据结构，非常适合做任务队列

---

## 一、CentOS 部署 Redis

### 1. 更新软件包

```bash
sudo yum update
```

### 2. 安装 `epel-release` 仓库

```bash
yum install epel-release
```

### 3. 安装 Redis

```bash
yum -y install redis
```

### 4. 修改 Redis 配置文件

默认配置文件位于 `/etc/redis/redis.conf` 或 `/etc/redis.conf`

配置绑定地址：默认情况下，Redis 只允许本地访问。要允许外部访问，打开 Redis 配置文件并更改 `bind` 配置。

```bash
vim /etc/redis/redis.conf
```

```ini
bind 0.0.0.0
```

Redis 设置一个密码，启用密码保护：

```ini
requirepass MyStrongPassw0rd!
```

设置 Redis 开机自启：

```bash
systemctl start redis && systemctl enable redis
```

检查 Redis 服务状态，通过以下命令检查 Redis 服务的状态：

```bash
systemctl status redis
```

验证 Redis 是否启动成功：

使用 `redis-cli` 连接 Redis，验证其是否正常工作：

```bash
redis-cli
```

如果启用了密码，请使用 `auth` 命令输入密码：

```bash
auth your_redis_password
```

执行以下命令，检查 Redis 是否可以正常读写数据：

```
ping
pong
```

### 5. Redis 部署遇到的问题

**Redis 无法启动**

- 检查端口是否被占用：Redis 默认使用端口 6379，检查是否有其他应用占用了该端口：

  ```bash
  lsof -i :6379
  ```

- 检查配置文件：查看 `/var/log/redis/redis-server.log` 文件，查看是否有错误信息

**Redis 客户端无法连接**

- 确保防火墙已允许访问 Redis 端口。
- 确保 Redis 服务正在运行，可以通过 `systemctl` 检查 Redis 状态。
- 如果启用了密码，确保在连接时使用正确的密码