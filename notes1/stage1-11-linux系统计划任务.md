# 定时任务：Cron 与 At

## 1. Cron 定时任务（周期性任务）

### 基本命令

| 命令 | 作用 |
| --- | --- |
| `crontab -e` | 编辑定时任务 |
| `crontab -l` | 查看定时任务 |
| `crontab -r` | 删除所有定时任务 |

### 语法：五颗星 + 要执行的命令

```text
分  时  日  月  周   命令
*   *   *   *   *   /path/to/command
```

| 位置 | 含义 | 取值范围 |
| --- | --- | --- |
| 1 | 分钟 | 0-59 |
| 2 | 小时 | 0-23 |
| 3 | 日期 | 1-31 |
| 4 | 月份 | 1-12 |
| 5 | 星期几 | 0-7（0 和 7 都代表周日） |

### 常用示例

```bash
crontab -e
```

```cron
0 3 * * * /usr/local/bin/backup.sh     # 每天凌晨 3 点执行备份
*/5 * * * * /usr/local/bin/check.sh    # 每 5 分钟执行一次
30 8 1 * * /usr/local/bin/report.sh    # 每月 1 号 8:30 执行
0 9 * * 1-5 /usr/local/bin/work.sh     # 每周一到周五 9 点执行
```

> `*` 表示任意值，`*/5` 表示每隔 5 个单位，`1-5` 表示范围。

## 2. At 命令（一次性任务）

**概述：特定时间只执行一次的命令行工具**（与 cron 周期性执行互补）。

### 创建任务

```bash
at now + 1 minute      # 1 分钟后执行
at 23:00               # 今天 23:00 执行
```

进入 `at>` 提示符输入命令，`Ctrl + D` 提交。

### 管理任务

| 命令 | 作用 |
| --- | --- |
| `atq` | 查看等待执行的任务 |
| `at -l` | 同 atq，查看任务 |
| `at -d 编号` | 删除对应编号的任务 |
| `atrm 编号` | 同 at -d，删除任务 |

```bash
atq              # 查看任务列表，记住任务编号
at -d 3          # 删除编号为 3 的任务
```


# 作业复盘：定时任务与日志轮转

## 作业一：定时任务（cron + at）

### 任务要求

1. 编写一个定时任务，每五分钟输出一个"远航最帅"到 `~/data/hello.txt` 里面
2. 用 `at` 设置服务器今天 11 点自动关机，并查看定时任务是否存在

### 任务 1：cron 定时任务

**① 创建目录并手动测试写入：**

```bash
mkdir -p ~/data
echo "远航最帅" >> /root/data/hello.txt
```

**② 编辑定时任务：**

```bash
crontab -e
```

（第一次提示选编辑器，输入 `2` 选 vim）加入一行：

```cron
*/5 * * * * echo "远航最帅" >> /root/data/hello.txt
```

保存退出（vim 里 `:wq`）。

**③ 验证：**

```bash
crontab -l                 # 查看任务是否存在
cat /root/data/hello.txt   # 现在 1 行（手动写的）
```

等 5 分钟后再次 `cat`，多出"远航最帅"即成功。

**④ 排错：**

```bash
systemctl status crond     # crond 服务是否运行
tail -n 20 /var/log/cron   # cron 执行日志
```

**知识点：cron 五颗星语法**

```text
分  时  日  月  周   命令
*   *   *   *   *   /path/to/command
```

| 位置 | 含义 | 范围 |
| --- | --- | --- |
| 1 | 分钟 | 0-59 |
| 2 | 小时 | 0-23 |
| 3 | 日期 | 1-31 |
| 4 | 月份 | 1-12 |
| 5 | 星期 | 0-7 |

`*/5` = 每 5 分钟一次。常用管理命令：`crontab -l` 查看、`crontab -r` 删除、`crontab -e` 编辑。

### 任务 2：at 定时关机

**① 确认 atd 服务运行：**

```bash
systemctl status atd
```

未运行则：`systemctl start atd && systemctl enable atd`

**② 创建任务：**

```text
[root@localhost ~]# at 11:00
at> shutdown -h now        ← 输入要执行的命令，回车
at>                        ← 空行处按 Ctrl + D 提交
job 2 at Thu Aug 20 11:00:00 2026   ← 提交成功
```

> `Ctrl + D` 是键盘快捷键（按住 Ctrl 再按 D），表示输入结束提交，不是文字。

**③ 查看任务是否存在：**

```bash
atq        # 或 at -l
```

**④ 删除任务（练习完建议删）：**cron

```bash
atq
atrm 编号
```

**注意：** 已过当天 11 点的话 `at 11:00` 会自动排到明天；任务到点真会执行关机。




### 三个问题答案

**Q1：如何设置 logrotate 配置文件的路径？**

- 主配置：`/etc/logrotate.conf`
- 子配置（推荐）：`/etc/logrotate.d/` 目录下的文件会被自动加载（主配置里有 `include /etc/logrotate.d` 一行）
- 手动测试可指定任意路径：`logrotate /任意路径/xxx.conf --debug`

**Q2：compress 参数有什么作用？**

轮转后把旧日志压缩成 `.gz` 格式（如 `custom.log.1.gz`），体积缩小 80% 以上，节省磁盘空间；常搭配 `delaycompress` 延迟压缩，避免影响正在写日志的进程。

**Q3：除了 logrotate，还有哪些工具可以做日志轮转？**

| 工具/方案 | 说明 |
| --- | --- |
| journald | systemd 自带，`journalctl --vacuum-size=100M` 限制体积 |
| Docker | 容器日志 `max-size` / `max-file` 参数 |
| 应用自带 | 如 Python 的 `RotatingFileHandler` |
| newsyslog | BSD 系统默认轮转工具 |
| 手写 cron 脚本 | mv + gzip + 清理，定时执行 |

## 今日知识点速查

| 工具 | 场景 | 管理命令 |
| --- | --- | --- |
| cron | 周期性任务 | `crontab -e/-l/-r` |
| at | 一次性任务 | `atq` 查看、`atrm`/`at -d` 删除 |
| logrotate | 日志轮转 | 配置放 `/etc/logrotate.d/`，`logrotate -d/-f` 测试 |

**关键对比：** cron 管"反复做"，at 管"只做一次"；logrotate 管"日志自动归档清理"，三者是日常运维的"定时三件套"。