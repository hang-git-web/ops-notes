# Systemd 服务管理

## 1. Systemd 定义与作用

- **Systemd** 是 Linux 的系统和服务管理器，也是系统的 **1 号进程（PID 1）**
- 作用：管理服务的启动、停止、开机自启、依赖关系，并收集服务日志

> 详细内容可看 `stage1-07-服务管理 systemd.md`

## 2. 主要功能与管理命令

| 命令 | 作用 |
| --- | --- |
| `systemctl start 服务名` | 启动服务 |
| `systemctl stop 服务名` | 停止服务 |
| `systemctl restart 服务名` | 重启服务 |
| `systemctl status 服务名` | 查看服务状态 |
| `systemctl enable 服务名` | 设置开机自启 |
| `systemctl disable 服务名` | 取消开机自启 |
| `systemctl is-enabled 服务名` | 查看是否开机自启 |

例子：

```bash
systemctl start nginx      # 启动 nginx
```

> 详细内容可看 `stage1-07-服务管理 systemd.md`

## 3. 查看服务日志（journalctl）

| 命令 | 作用 |
| --- | --- |
| `journalctl -u ssh` | 查看 ssh 服务的日志 |
| `journalctl -u cron -f` | 实时跟踪 cron 服务日志 |
| `journalctl --vacuum-time=2weeks` | 清理超过最近两周的日志 |
| `journalctl --since "2025-02-27" --until "2025-02-27"` | 查看当天日志 |

```bash
journalctl -u ssh                        # 查看 ssh 日志
journalctl -u cron -f                    # 实时跟踪 cron 日志
journalctl --vacuum-time=2weeks          # 清理 2 周前的日志
journalctl --since "2025-02-27" --until "2025-02-27"   # 查某天日志
```

## 4. 自定义服务单元

- **位置**：`/etc/systemd/system/`
- **后缀**：以 `.service` 结尾

基本结构：

```ini
[Unit]
Description=服务描述
After=network.target

[Service]
Type=simple
ExecStart=要执行的命令

[Install]
WantedBy=multi-user.target
```

创建后生效：

```bash
systemctl daemon-reload        # 重新加载服务单元
systemctl start 服务名
systemctl enable 服务名        # 开机自启
```

> 详细内容可看 `stage1-07-服务管理 systemd.md`