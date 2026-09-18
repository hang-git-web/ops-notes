# Linux 基础与安全加固笔记

## 一、快捷键

### Bash 常用快捷键

| 快捷键 | 作用 |
| --- | --- |
| `ctrl+L` | 清屏，但是在上面还是能翻回来 |
| `ctrl+c` | 强制停止 |
| `Tab` | 自动补全命令，文件名，目录名 |
| `ctrl+a` | 光标跳至行首 |
| `ctrl+e` | 光标跳至行尾 |
| `ctrl+z` | 暂停当前前台进程放入后台 |
| `ctrl+w` | 删除单词，以空格间隔 |
| `ctrl+y` | 快捷键删除的内容恢复 |

### Vim 快捷键

启动参数：

- `vim -R` 只读不修改
- `vim +行号 filename`
- `vim -r` 恢复机制

其他：

- 没有上下左右，HJKL
- 命令行模式的批量替换

快捷键：

- 光标移动到行首 `^`，行尾 `$`，最后一行 `G`，第一行 `gg`，888 行 `888G` 或 `999gg`
- `o` 当前行下方添加一行，删除当前行 `dd`，撤销到上一步：`u`，取消撤销：`ctrl+r`
- 删除当前行往后的 14 行：`14dd`，粘贴回来：`p`
- 查找关键字：命令行模式，`/关键字`，`n`：到下一个匹配关键字，`N` 向上，`n` 向下翻
- 打开行号：命令行模式输入 `:set nu`

---

## 二、Linux 安全加固——修复 bug，打补丁

### 1. 更新系统与打补丁

```bash
apt update
apt install unattended-upgrades  # 安装安全补丁
```

### 2. 防火墙配置

```bash
sudo ufw status       # 查看防火墙状态
sudo ufw allow http   # 相当于 80/tcp
```

查看所有规则（带序号）：

```bash
sudo iptables -L -n --line-number
```

`-L` 列出所有规则，`-n` 显示 IP 和端口数字，`--line-number`

### 3. 禁止 root 用户远程登录——防止密码泄漏的情况

a. 修改 ssh 配置文件 `vim etc/ssh/sshd_config`

b. `PermitRootLogin` 项

c. 重启 ssh 服务

### 4. SELinux

SELinux 给程序装上紧箍咒，限制访问无语

a. 查看服务状态 `sestatus` 或者 `getenforce`

b. 通过 `sestatus 1` 和 `sestatus 0` 改变开始或关闭状态

或者通过配置文件来改 `vim /etc/selinux/config`

### 5. 关闭不必要的服务——节省与维护

a. 先查看服务 `sytemctrl list`

b. 关闭