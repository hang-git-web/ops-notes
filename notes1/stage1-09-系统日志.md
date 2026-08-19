# 日志管理

## 1. 日志的重要性

| 场景 | 作用 |
| --- | --- |
| 故障排错 | 服务出问题，日志是第一现场 |
| 安全审计 | 记录登录、提权等敏感操作 |
| 监控分析 | 观察访问量、错误率、性能指标 |
| 磁盘管理 | 日志不管理会撑爆磁盘 |

## 2. 日志文件概述（/var/log）

系统日志统一存放在 `/var/log` 目录下。

| 文件 | 内容 |
| --- | --- |
| `/var/log/messages` | 系统主日志（内核、服务运行信息） |
| `/var/log/secure` | 安全日志（登录、sudo 操作） |
| `/var/log/cron` | 计划任务日志 |
| `/var/log/dmesg` | 内核启动日志 |
| `/var/log/boot.log` | 开机启动日志 |
| `/var/log/nginx/` | 服务/应用日志（如 access.log） |

## 3. 日志文件分类

| 分类 | 代表文件 |
| --- | --- |
| 系统日志 | `/var/log/messages` |
| 安全日志 | `/var/log/secure` |
| 任务日志 | `/var/log/cron` |
| 内核日志 | `/var/log/dmesg` |
| 服务/应用日志 | `/var/log/nginx/access.log` 等 |

## 4. 查看日志命令

### cat —— 查看全部内容

```bash
cat /var/log/messages        # 直接输出全部（适合小文件）
cat -n /var/log/messages     # 带行号查看
```

### more / less —— 分页查看

```bash
more /var/log/messages       # 分页，只能往下翻
less /var/log/messages       # 分页，支持上下翻、搜索
```

> **less 性能更强**，大文件不卡，支持 `/关键词` 搜索、`q` 退出，推荐使用。

### tail —— 尾部查看（最常用）

| 命令 | 作用 |
| --- | --- |
| `tail messages` | 尾部查看（默认最后 10 行） |
| `tail -n10 messages` | 查看最后 10 行 |
| `tail -f messages` | 实时监控（新日志实时滚动） |
| `tail -fn50 messages \| grep "error"` | 实时过滤错误关键字 |

```bash
tail /var/log/messages
tail -n10 /var/log/messages
tail -f /var/log/messages
tail -fn50 /var/log/messages | grep " error "
```

> `-f` 用于实时监控，按 `Ctrl + C` 退出；排查服务异常时 `tail -f` + `grep` 是黄金组合。

## 5. 自动管理日志（logrotate）

"自动化日志管理"在 Linux 里通常就是指 **logrotate（日志轮转）**——自动压缩、切割、清理旧日志，防止日志把磁盘塞满。

### 配置文件位置

| 位置 | 作用 |
| --- | --- |
| `/etc/logrotate.conf` | 全局主配置（默认规则） |
| `/etc/logrotate.d/` | 每个服务的独立规则写在这里（推荐） |

### 配置示例（nginx）

```bash
vim /etc/logrotate.d/nginx
```

```ini
/var/log/nginx/*.log {
    daily
    rotate 7
    missingok
    compress
    delaycompress
    notifempty
    create 644 nginx adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

### 关键选项含义

| 选项 | 含义 |
| --- | --- |
| `daily` | 每天轮转一次（也可 weekly/monthly） |
| `rotate 7` | 保留最近 7 份，更旧的自动删除 |
| `compress` | 轮转后压缩成 .gz |
| `delaycompress` | 上一份不立即压缩（等下一次） |
| `missingok` | 日志文件不存在也不报错 |
| `notifempty` | 空日志不轮转 |
| `postrotate ... endscript` | 轮转后执行的操作（让服务重新打开日志文件） |

### 它怎么自动跑起来？

logrotate 本身不会定时，靠 cron 每天触发一次：

```bash
cat /etc/cron.daily/logrotate
```

默认每天自动执行，所以写完配置文件就不用手动管了，这就是"自动化"。

### 验证配置

```bash
logrotate -d /etc/logrotate.d/nginx    # 调试模式，只预览不执行
logrotate -f /etc/logrotate.d/nginx    # 强制立即执行一次
```

## 6. 实战场景：Nginx 日志管理

### 场景设定

你负责一台 Nginx 网站服务器，每天几十万次访问，`/var/log/nginx/access.log` 每天增长约 500MB。日志文件一直在变大——这就是你的"定时炸弹"。

### 事故版：没配自动化日志管理

三个月后，你收到监控报警：**磁盘使用率 100%**。

```bash
df -h
```

```text
/dev/sda2   19G   19G   0G  100% /
```

再一看，罪魁祸首：

```bash
du -sh /var/log/nginx/access.log
```

```text
87G    /var/log/nginx/access.log
```

网站开始出问题：写不了日志、页面报错、数据库也连不上——**磁盘满引发的连锁故障**。

你手动删日志还踩了个坑：

```bash
rm -f /var/log/nginx/access.log
df -h        # 咦？空间还是没释放！
```

因为 **nginx 进程还握着这个文件**，文件虽然删了名字，但空间要等进程释放句柄才回收。真实运维里这叫"删了不释放"，最后只能重启 nginx 才解决。

这就是日志不自动管理的下场：**磁盘被日志撑爆，业务中断，手动清还麻烦。**

### 自动化版：配了 logrotate 之后

```ini
/var/log/nginx/access.log /var/log/nginx/error.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 644 nginx adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
```

之后每天凌晨，cron 自动触发 logrotate，磁盘里发生这样的"交接班"：

```text
第 1 步：access.log 改名为 access.log.1（旧日志归档）
第 2 步：发 USR1 信号给 nginx，让它重新打开一个新的空 access.log
        （不通知的话，nginx 还会往改完名的旧文件里继续写）
第 3 步：把 access.log.1 压缩成 access.log.1.gz（1GB → 约 100MB）
第 4 步：检查是否超过 7 份，更旧的自动删除
```

磁盘里永远只有最近 7 天的日志，占用空间从"87G"变成"最多 7 个 1GB 文件"——**磁盘再也不怕被日志撑爆**，这就是"自动化日志管理"的价值。

### 运维日常验证

```bash
ls -lh /var/log/nginx/         # 查看轮转后的文件列表
zcat /var/log/nginx/access.log.3.gz | grep "某个IP"   # 查旧日志
logrotate -f /etc/logrotate.d/nginx                   # 手动强制轮转
logrotate -d /etc/logrotate.d/nginx                   # 调试检查配置
```