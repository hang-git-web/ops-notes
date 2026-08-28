# 日志管理（含作业二与复盘）

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

### 关键选项含义

| 选项 | 含义 |
| --- | --- |
| `daily` | 每天轮转一次（也可 weekly/monthly） |
| `rotate 7` | 保留最近 7 份，更旧的自动删除 |
| `compress` | 轮转后压缩成 .gz |
| `delaycompress` | 上一份不立即压缩（等下一次） |
| `missingok` | 日志文件不存在也不报错 |
| `notifempty` | 空日志不轮转 |
| `create 644 nginx adm` | 轮转后新建日志文件并设置权限属主 |
| `postrotate ... endscript` | 轮转后执行的操作（让服务重新打开日志文件） |

### 它怎么自动跑起来？

logrotate 本身不会定时，靠 cron 每天触发一次：

```bash
cat /etc/cron.daily/logrotate
```

默认每天自动执行，所以写完配置文件就不用手动管了，这就是"自动化"。

---

# 实验：Nginx 日志自动轮转（logrotate）

## 一、实验目的

1. 理解 logrotate 的作用：自动压缩、切割、清理旧日志，防止日志撑爆磁盘
2. 掌握 logrotate 配置文件的编写方法（`/etc/logrotate.d/` 下的规则文件）
3. 实际验证各参数的效果：`daily`、`rotate`、`compress`、`delaycompress`、`missingok`、`notifempty`、`create`、`postrotate`
4. 理解"自动化"机制：logrotate 本身不定时，靠 cron 每天触发
5. 建立运维意识：日志管理不能靠手动，必须自动化

## 二、实验环境

- 系统：CentOS 7（yum 源已切换阿里云归档镜像）
- 应用：Nginx
- 工具：logrotate + cron

## 三、实验步骤

### 步骤 1：启动服务并制造访问日志

```bash
systemctl start nginx
for i in {1..100}; do curl -s http://127.0.0.1 > /dev/null; done
wc -l /var/log/nginx/access.log
```

**输出：** 101 行——这就是轮转的"原料"。

### 步骤 2：编写 logrotate 配置文件

```bash
vim /etc/logrotate.d/nginx
```

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
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

### 步骤 3：测试加载（debug 模式）

```bash
logrotate -d /etc/logrotate.d/nginx
```

看到 `considering log /var/log/nginx/access.log` 说明配置没问题。

### 步骤 4：第一次强制轮转

```bash
logrotate -f /etc/logrotate.d/nginx
ls -lh /var/log/nginx/
```

（此处插入图 1：第一次轮转后的 ls 输出）

| 现象 | 对应参数 | 说明 |
| --- | --- | --- |
| 新的 access.log 属主是 nginx:adm、权限 -rw-r--r-- | `create 644 nginx adm` | 轮转后按指定权限属主重建日志 ✅ |
| access.log.1 还是 9.0K、没变 .gz | `delaycompress` | 等下一次轮转才压缩 ✅ |
| error.log 是空的、没有 error.log.1 | `notifempty` | 空日志不轮转 ✅ |

### 步骤 5：第二次轮转（意外收获：验证 notifempty）

```bash
logrotate -f /etc/logrotate.d/nginx
ls -lh /var/log/nginx/
```

（此处插入图 2：输出完全一样的 ls）

**现象：** 输出一模一样，什么都没发生。

**原因：** 第一次轮转后新 access.log 是空的（0 字节），`notifempty` 检查到空日志直接跳过——**再次验证 notifempty 在工作**。

### 步骤 6：制造新日志再轮转（验证 compress）

```bash
for i in {1..50}; do curl -s http://127.0.0.1 > /dev/null; done
wc -l /var/log/nginx/access.log     # 输出：50

logrotate -f /etc/logrotate.d/nginx
ls -lh /var/log/nginx/
```

（此处插入图 3：轮转后的 ls 输出）

**效果：** 之前 9.0K 的日志被压缩成 **158 字节**（压缩率 98%），`compress` 实锤生效。

| 文件 | 说明 |
| --- | --- |
| access.log（空，nginx:adm） | create 新建的当前日志 |
| access.log.1（4.5K，未压缩） | delaycompress：等下次才压缩 |
| access.log.2.gz（158 字节） | compress：上上次的已压缩 |
| error.log（空） | notifempty：一直跳过 |

### 步骤 7：模拟 8 天，验证 rotate 7 上限

```bash
for i in {1..8}; do
  echo "log line $i" >> /var/log/nginx/access.log
  logrotate -f /etc/logrotate.d/nginx
done

ls /var/log/nginx/ | grep access
```

（此处插入图 4：只有 7 份历史文件）

**预期：** 只看到 access.log + access.log.1 ~ access.log.7.gz，**没有 .8**——最旧的一份被自动删除，`rotate 7` 生效。

### 步骤 8：验证 postrotate（nginx 正常写新日志）

```bash
curl -s http://127.0.0.1 > /dev/null
tail -n 2 /var/log/nginx/access.log
```

能看到新访问记录写入 access.log，说明 `kill -USR1` 让 nginx 重新打开了新日志文件。

### 步骤 9：理解自动化机制

```bash
cat /etc/cron.daily/logrotate
```

logrotate 每天凌晨由 cron 自动执行——配置写好就不用管了。

## 四、实验现象汇总

| 参数 | 验证证据 |
| --- | --- |
| `daily` | 配置中声明，cron 每天触发 |
| `rotate 7` | 循环 8 次后只有 7 份历史 |
| `compress` | 9.0K → 158 字节 |
| `delaycompress` | .1 保持未压缩 |
| `missingok` / `notifempty` | 空日志跳过（error.log） |
| `create 644 nginx adm` | 新日志属主 nginx:adm、权限 644 |
| `postrotate` | 轮转后 curl 仍写入 access.log |

## 五、实验总结

**1. logrotate 的本质：**

日志轮转 = **触发 → 检查 → 改名 → 压缩 → 通知服务 → 清理**，每天自动循环一次，让磁盘只保留最近 N 份日志。

**2. 参数记忆口诀：**

- `rotate 7` 管"留几份"，`compress` 管"压不压"，`delaycompress` 管"晚点压"
- `missingok` 管"没有也不报错"，`notifempty` 管"空的就不切"
- `create` 管"切完建新文件"，`postrotate` 管"切完通知服务"

**3. 踩过的两个坑（也是知识点）：**

- **空日志不轮转**：`notifempty` 生效时，连续 `-f` 看似"没反应"，要先有日志才能轮转
- **delaycompress 的副作用**：历史中最新一份永远是 `.1` 未压缩，压缩的是更早的

**4. 运维意义：**

日志不管理的下场是磁盘被撑爆、业务中断；配置一次 logrotate，就能永久自动化——**用最小的成本，避免最大的事故**。

## 六、延伸思考（解答版）

### Q1：把 daily 改成 size 100M 会怎样？

**含义：** 不再按"时间"轮转，改成"日志文件长到 100M 就切"。

- 日志一天没到 100M → 不切（哪怕过了凌晨）
- 日志一天涨到 300M → 一天切 3 次
- **注意：logrotate 规则里 `size` 会覆盖 `daily/weekly/monthly`**，两者不要同时写，写了也以 size 为准

**适合场景：** 访问量大的服务（nginx、java 应用），日志增长快，用大小控制单个日志文件体积，避免"一天一个文件，打开卡死"。

**验证方法：**

```bash
dd if=/dev/zero of=/var/log/nginx/access.log bs=1M count=120   # 造一个 120M 日志
logrotate -d /etc/logrotate.d/nginx     # debug 里会显示 log needs rotating
logrotate -f /etc/logrotate.d/nginx     # 强制轮转，access.log.1 就是 120M 切出来的
```

### Q2：生产环境还有哪些"自动清理"策略？

| 策略 | 命令/工具 | 清理对象 |
| --- | --- | --- |
| 日志轮转 | logrotate | 应用日志按天/大小切，保留 N 份 |
| 超期文件清理 | `find /backup -type f -mtime +30 -delete` | 30 天前的备份、临时文件 |
| 系统日志上限 | `journalctl --vacuum-size=100M` / `--vacuum-time=7d` | systemd 日志 |
| 临时文件清理 | `systemd-tmpfiles --clean`（或 tmpwatch） | /tmp、/var/tmp |
| yum 缓存清理 | `yum clean all` | 下载缓存 |
| 旧内核清理 | `package-cleanup --oldkernels --count=2` | 无用内核，释放 /boot |
| Docker 垃圾清理 | `docker system prune -af` | 悬空镜像、停止的容器 |
| 磁盘空间监控 | 脚本 + df 告警 | 提前发现，别等满了才处理 |

**核心原则：**

1. 全部交给 cron：写个 cleanup.sh，`crontab -e` 里每天凌晨跑
2. 清理前确认是"垃圾"不是"数据"：日志、缓存、旧内核可以删，数据库、业务文件不能乱动
3. 可验证：清理脚本加日志输出，定期看执行结果

```bash
#!/bin/bash
# 一个简单的清理脚本示例
find /var/log -name "*.log" -mtime +30 -delete
find /backup -type f -mtime +30 -delete
journalctl --vacuum-size=100M
yum clean all > /dev/null 2>&1
```

### Q3：为什么 postrotate 用 USR1，不用 HUP 或重启？

因为 **USR1 是 nginx 专门用来"重新打开日志文件"的信号**，精准、轻量、零副作用：

| 信号/操作 | nginx 的行为 | 副作用 |
| --- | --- | --- |
| **USR1** | 只重新打开日志文件 | ✅ 最小：连接不断、配置不重载 |
| **HUP** | 重载全部配置 + 优雅重启 worker | ⚠️ 重新读配置，配置有语法错误会失败，影响面大 |
| **systemctl restart** | 停止再启动整个 nginx | ❌ 服务中断、连接全断、开销大 |

**逻辑：** 日志轮转只是"换了个文件写"，nginx 其他一切都没变——那就只告诉它"去开新文件"，**不要做任何多余的事**。用 HUP 等于"为了换个本子，把整个办公室重新装修了一遍"；用 restart 等于"为了换个本子，把整栋楼电闸拉了"。

**扩展知识：** 信号含义是每个服务自己定义的，不是通用标准：

- nginx：`USR1` = 重开日志，`HUP` = 重载配置，`QUIT` = 优雅退出
- rsyslog：`HUP` = 重开日志（和 nginx 相反）
- Apache：`USR1` = 优雅重启

所以写 postrotate 前，先查那个服务的文档确认它认哪个信号。**nginx 用 USR1，别记反**。