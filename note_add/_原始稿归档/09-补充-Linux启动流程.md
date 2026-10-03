# 09 补充：Linux 启动流程（面经第 1 题，你挂掉的那题）

> **面经原题**：Linux 启动有哪几个阶段？
> 这题在原笔记里完全没有，属于"不补必挂"。
> **30 秒答法**：**"固件自检 → 引导程序加载内核 → 内核初始化并挂载根文件系统 → 启动 init（systemd）拉起服务 → 进入登录界面。"**
> 下面每一层都准备一句"它具体做了什么"，追问就不会卡。

---

## 一、5 个阶段（背这张表）

| 阶段 | 谁在干活 | 做了什么 | 出错会怎样 |
| --- | --- | --- | --- |
| ① 固件（BIOS / UEFI） | 主板上的固件 | **上电自检 POST**（CPU、内存、显卡），然后按启动顺序找到启动盘，读 MBR / GPT / ESP 分区 | 找不到启动盘 → 提示 `No bootable device` |
| ② 引导程序（GRUB2） | `/boot/grub2/grub.cfg` | 显示菜单、倒计时；**把内核（`vmlinuz`）和 `initramfs` 加载进内存**，然后把控制权交给内核 | 菜单都进不去 → `grub rescue>` |
| ③ 内核初始化 | 内核自己 | **自解压 `vmlinuz`** → 初始化 CPU、内存管理、驱动 → 执行 `initramfs` 里的 `/init` → **加载挂根分区所需的驱动（RAID/LVM/加密）** → 挂载真正的 `/` | 内核 panic（`Kernel panic - not syncing`）|
| ④ 启动 init / systemd（PID 1） | `/sbin/init` → systemd | systemd 作为 **PID 1**，按 `default.target` **并行**拉起服务、执行 `/etc/fstab` 挂载、启动网络与日志 | 根挂不上 → emergency mode |
| ⑤ 登录 | `getty` / `sshd` / 显示管理器 | 给出登录界面（本地 tty 或 SSH），登录后进 shell，启动流程结束 | 服务没起 → 但系统"看起来是起来的" |

> **一句话串起来**：
> **"固件把控制权交给 GRUB，GRUB 把内核和 initramfs 装进内存，内核挂上根文件系统后把 PID 1 交给 systemd，systemd 拉起 target 里定义的服务，最后进登录界面。"**

### 常见口述错误（别踩）

| 错误说法 | 正确说法 |
| --- | --- |
| 引导程序"解压内核" | 引导程序**只负责加载**内核到内存，内核是**自己解压**自己的 |
| 内核"直接启动服务" | 内核交给 **PID 1（systemd）**，服务由 systemd 拉起 |
| 启动顺序是"一个接一个" | systemd 是**并行**启动的（这也是 CentOS 7 比 6 开机快的原因）|

---

## 二、必须知道的 3 个概念

### 2.1 initramfs 是干什么的（★ 追问高频）

```text
问题：内核挂根分区时，可能要用到 LVM / RAID / 加密的驱动，
      但这些驱动又装在根分区上 → 鸡生蛋问题
解法：bootloader 额外加载一个很小的内存文件系统 initramfs（临时根），
      里面有挂根分区所需的驱动和脚本，挂上真根后再 switch_root 切过去
```

> **答法**：**"initramfs 是一个临时的内存文件系统，解决'要用驱动才能挂根分区、但驱动又在根分区上'的问题。"**

### 2.2 为什么叫 PID 1

- 内核启动后运行的第一个用户态进程就是 **PID 1（systemd）**，其它所有进程都是它的子孙。
- 它负责：**按依赖启动服务、收养孤儿进程（收尸）、响应 `systemctl` 的请求**。
- **推论**：`kill -9 1` 会搞死整个系统；容器里那个 PID 1 也是同样的角色（所以容器里 "1 号进程不收信号" 是个经典坑）。

### 2.3 target 和旧版 runlevel 的对应（CentOS 7+）

| target | 旧 runlevel | 含义 |
| --- | --- | --- |
| `poweroff.target` | 0 | 关机 |
| `rescue.target` | 1 | 单用户 / 救援 |
| `multi-user.target` | 3 | 多用户命令行（**服务器默认**）|
| `graphical.target` | 5 | 图形界面 |
| `reboot.target` | 6 | 重启 |

```bash
systemctl get-default                       # 看默认进哪个 target
systemctl set-default multi-user.target     # 设默认（服务器就别开图形了）
systemctl list-units --type=target          # 看当前有哪些 target
```

---

## 三、怎么验证 / 怎么看启动过程（面试加分）

| 命令 | 看什么 |
| --- | --- |
| `dmesg -T` | 内核启动日志（带可读时间），看驱动、磁盘、内存识别情况 |
| `journalctl -b` | **本次开机**的所有日志（`-b -1` 看上次开机，排查"上次为什么重启"很有用）|
| `journalctl -b -p err` | 只看本次开机的错误级别日志 |
| **`systemd-analyze`** | 开机总耗时：`firmware + loader + kernel + userspace` 四段 |
| **`systemd-analyze blame`** | **哪个服务最拖开机速度**（★ 面试很爱问"开机慢怎么办"）|
| `systemd-analyze critical-chain` | 关键路径：是谁在等谁 |
| `cat /proc/cmdline` | 当前内核启动参数（改 grub 后可以在这里确认生效）|
| `uptime -s` / `who -b` | 本次开机时间（判断机器是不是刚重启过）|
| `lsblk -f` / `blkid` | 启动后挂载的盘和文件系统 |
| `systemctl --failed` | **有哪些服务启动失败**（排障第一站）|

```bash
# 面试常问："服务器开机慢，怎么查？"
systemd-analyze blame | head -10        # 找出最慢的服务
systemctl --failed                      # 找失败的服务
journalctl -b -p warning | tail -50     # 看警告和报错
```

---

## 四、启动失败怎么救（★ 运维岗真正的高分点）

### 4.1 先定位卡在哪一层

```text
屏幕停在 No bootable device                 → 阶段①：固件找不到盘（启动顺序/硬盘故障）
屏幕进 grub rescue>                         → 阶段②：GRUB 配置或 /boot 丢了
屏幕刷一堆内核日志后 Kernel panic            → 阶段③：根文件系统挂不上（驱动/盘/UUID 变了）
屏幕出现 emergency mode 或 "Give root password" → 阶段④：fstab 有错 / 服务挂载失败
能登录但某些服务没起                          → 阶段⑤：服务自身的问题（看 journalctl）
```

### 4.2 常用救援入口（记住两个）

**① 救援/单用户模式**：GRUB 菜单上按 `e` 编辑启动项，在 `linux16 /vmlinuz...` 那行末尾加参数：

```text
systemd.unit=rescue.target     # 进救援模式（几乎所有服务都不起，但根是挂好的）
rd.break                       # 更早一步，在 switch_root 之前打断（CentOS 7/8 改密码标配）
init=/bin/bash                 # 直接绕过 systemd 起一个 bash（最低限度环境）
```

改完按 `Ctrl+X` 启动。**`rd.break` 之后的标准动作**：

```bash
mount -o remount,rw /sysroot      # 此时 /sysroot 是只读挂载的
chroot /sysroot
passwd root                        # 改密码 / 注释掉 fstab 里写错的那一行
touch /.autorelabel                # 改过密码要重打 SELinux 标签，否则登录失败
exit
reboot
```

**② fstab 写错导致起不来**（★ 最经典的运维事故，你笔记 `03` 里已经提过）：

```bash
# 进 emergency mode 或 rd.break 后：
mount -o remount,rw /              # 根分区重新挂成可写
vim /etc/fstab                     # 注释掉写错的行，或改成正确的 UUID
mount -a                           # 验证不报错
reboot
```

> **防御性习惯**（面试说出来就是加分）：**改 fstab 前先备份**、用 **UUID** 而不是 `/dev/sdb1`、加上 **`nofail`** 选项让挂不上也不阻塞启动、改完必须 `mount -a` 验证。

### 4.3 其它启动故障

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| GRUB 菜单没有新内核 | `grub2-mkconfig` 没生成 | `grub2-mkconfig -o /boot/grub2/grub.cfg` |
| 换盘/扩盘后起不来（UUID 变了） | fstab 里写的是旧 UUID | `blkid` 查新 UUID，改 fstab |
| 服务启动失败但系统能起 | 端口占用、配置错、权限错 | `systemctl status` + `journalctl -u`（见 `02`） |
| 开机慢 | 某个服务等超时（网络、挂载） | `systemd-analyze blame` |
| 内核升级后没生效 | 没重启，或默认内核没切 | `uname -r` 对比，重启生效（见 `notes1/stage1-10`）|

---

## 五、面试可能这么问（追问链）

**Q：Linux 启动有哪几个阶段？**
A：五阶段——固件自检 → GRUB 加载内核和 initramfs → 内核初始化并挂载根 → systemd（PID 1）拉起 target 里的服务 → 登录。核心是"控制权交接"：固件 → GRUB → 内核 → systemd。

**Q：initramfs 是干什么的？**
A：临时内存文件系统，解决"要用驱动才能挂根分区、但驱动又存在根分区里"的问题。

**Q：开机慢怎么查？**
A：`systemd-analyze` 看总耗时，`systemd-analyze blame` 找最慢的服务，`systemctl --failed` 找失败的，再 `journalctl -b -p err` 看报错。常见原因是网络等待、挂载超时、服务互相等。

**Q：系统起不来，你怎么救？**
A：先看卡在哪一层——进不去 GRUB 是引导问题，Kernel panic 是根挂载问题，emergency mode 一般是 fstab。救援入口是 GRUB 菜单加 `systemd.unit=rescue.target` 或 `rd.break`；fstab 写错就重挂成可写后改回来。**同时说明你会怎么防：用 UUID、加 `nofail`、改完 `mount -a` 验证。**

**Q：怎么让服务开机自启？**
A：`systemctl enable 服务`（本质是在 target 的 `.wants` 目录建软链接），配合 `systemctl start` 才是"现在起来 + 开机也起"，标准写法 `systemctl enable --now 服务`。临时任务也可以用 `crontab @reboot`。

**Q：内核升级完为什么重启才生效？**
A：因为运行中的内核在内存里，新内核文件只是写到了 `/boot`，只有重新走一遍引导流程才会加载它。`uname -r` 能看到当前跑的内核版本。

---

## 六、3 分钟自测

- [ ] 能按顺序说出 5 个阶段，并说出每个阶段"谁在干活"
- [ ] 能说出 initramfs 解决什么问题
- [ ] 知道 PID 1 是 systemd，并知道它负责什么
- [ ] 能说出 `rescue.target` / `multi-user.target` 和 runlevel 3 的对应
- [ ] 会用 `systemd-analyze blame` 和 `journalctl -b` 排查开机问题
- [ ] 能说出 GRUB 菜单加 `rd.break` 后改 root 密码的完整步骤
- [ ] 能说出 fstab 写错怎么救、以及怎么预防
