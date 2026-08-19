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

## 6. 作业二：logrotate 实战

### ① 创建配置文件

在 `/etc/logrotate.d/` 下创建 `custom_log` 文件（注意：是文件，不是目录）：

```bash
vim /etc/logrotate.d/custom_log
```

### ② 加入日志配置内容

```ini
/var/log/custom.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}
```

配置文件同时保存在 GitHub 仓库中：[custom_log 配置文件](../scripts/custom_log)

### ③ 准备测试日志

```bash
touch /var/log/custom.log
echo "这是一条测试日志" >> /var/log/custom.log
```

### ④ 手动测试

```bash
logrotate /etc/logrotate.conf --debug         # debug 模式，只预览不执行
logrotate -f /etc/logrotate.d/custom_log      # 强制轮转，看真实效果
ls -lh /var/log/custom.log*                   # 验证：出现 custom.log.1.gz 即成功
```

debug 输出出现 `considering log /var/log/custom.log` 即配置加载成功。

### ⑤ 小结

| 步骤 | 命令 |
| --- | --- |
| 创建配置 | `vim /etc/logrotate.d/custom_log` |
| 加载测试 | `logrotate /etc/logrotate.conf --debug` |
| 强制执行 | `logrotate -f /etc/logrotate.d/custom_log` |
| 验证结果 | `ls -lh /var/log/custom.log*` |

## 7. 日志轮转流程（六步）

```text
触发 → 检查 → 改名 → 压缩 → 通知服务 → 清理
```

| 步骤 | 动作 |
| --- | --- |
| 触发 | cron 每天执行 `/etc/cron.daily/logrotate` |
| 检查 | daily、notifempty、missingok 判断是否轮转 |
| 改名 | custom.log → .1 → .2 → ...（rotate 7 最多 7 份） |
| 压缩 | 旧日志压成 .gz，省磁盘空间 |
| 通知服务 | postrotate 发信号（如 kill -USR1 nginx）重开日志文件 |
| 清理 | 删除超过 rotate 份数的旧日志 |

## 8. 踩过的坑

1. `custom_log` 被建成了**目录**（vim 打开变成文件浏览器）→ 要用 `vim` 直接建文件，别先 `mkdir`
2. 参数后跟**行尾注释**会报 `unknown option` → 注释单独一行
3. 配置文件在 Windows 仓库里 ≠ 虚拟机里有 → 两边都要放
4. 不通知服务（缺 postrotate）→ 新日志继续写进改名的旧文件，轮转白做

## 9. 一句话总结

日志轮转 = 每天自动"切一刀 + 压缩归档 + 清旧的"，让磁盘只保留最近 N 份日志；配置放 `/etc/logrotate.d/`，`--debug` 测加载、`-f` 看效果。