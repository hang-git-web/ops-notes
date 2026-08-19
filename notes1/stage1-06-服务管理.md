1:Systemd定义与作用
看stage1-07-服务管理 systemd.md

2：主要功能与管理命令
看stage1-07-服务管理 systemd.md
例子：systemctl start nginx，启动nginx


3：查看服务日志
看stage1-07-服务管理 systemd.md
补充：
journalctl -u ssh 查看日志
journalctl -u cron -f 实时跟踪服务日志
journalctl --vacuum-time=2weeks  清理超过最近两周的日志
journalctl --since "2025-02-27" --until "2025-02-27" 查看当天日志

4：自定义服务单元
服务单元位置：/etc/systemd/system/，以.service结尾

实战：
1：安装nginx
2：启动、停止、重启Nginx服务
3：将nginx服务设置开机自启动，并查看是否成功
4：创建一个简单的自定义服务器单元
5：将自定义服务器启动并设置自动开机