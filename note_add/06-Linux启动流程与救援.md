# 06 Linux 启动流程与救援

> **本章考什么**：你知不知道「**机器从按下电源到出现登录界面，中间发生了什么**」，以及**起不来的时候怎么救**。
> **关键词**：五阶段、initramfs、PID 1、target vs runlevel、开机慢、`rd.break`、fstab 写错怎么救。
> **来源**：面经①第 1 题（**原稿完全空白，是挂掉的那题**）。

---

## 一、五个阶段（★ 背这张表）

### Q1 面试官：Linux 启动有哪几个阶段？

> **考察点**：启动流程 · 控制权交接 · 每一步"谁在干活"
> **30 秒答法**：**「固件自检 → 引导程序加载内核 → 内核初始化并挂载根文件系统 → 启动 init（systemd）拉起服务 → 进入登录界面。」**

| 阶段 | 谁在干活 | 做了什么 | 出错会怎样 |
| --- | --- | --- | --- |
| 1 固件（BIOS / UEFI） | 主板上的固件 | **上电自检 POST**（CPU、内存、显卡），然后按启动顺序找到启动盘，读 MBR / GPT / ESP 分区 | 找不到启动盘 → 提示 `No bootable device` |
| 2 引导程序（GRUB2） | `/boot/grub2/grub.cfg` | 显示菜单、倒计时；**把内核（`vmlinuz`）和 `initramfs` 加载进内存**，然后把控制权交给内核 | 菜单都进不去 → `grub rescue>` |
| 3 内核初始化 | 内核自己 | **自解压 `vmlinuz`** → 初始化 CPU、内存管理、驱动 → 执行 `initramfs` 里的 `/init` → **加载挂根分区所需的驱动（RAID/LVM/加密）** → 挂载真正的 `/` | 内核 panic（`Kernel panic - not syncing`） |
| 4 启动 init / systemd（PID 1） | `/sbin/init` → systemd | systemd 作为 **PID 1**，按 `default.target` **并行**拉起服务、执行 `/etc/fstab` 挂载、启动网络与日志 | 根挂不上 → emergency mode |
| 5 登录 | `getty` / `sshd` / 显示管理器 | 给出登录界面（本地 tty 或 SSH），登录后进 shell，启动流程结束 | 服务没起 → 但系统"看起来是起来的" |

> **一句话串起来（★ 背这句）**：
> **「固件把控制权交给 GRUB，GRUB 把内核和 initramfs 装进内存，内核挂上根文件系统后把 PID 1 交给 systemd，systemd 拉起 target 里定义的服务，最后进登录界面。」**

**常见口述错误（别踩）**：

| 错误说法 | 正确说法 |
| --- | --- |
| 引导程序"解压内核" | 引导程序**只负责加载**内核到内存，内核是**自己解压**自己的 |
| 内核"直接启动服务" | 内核交给 **PID 1（systemd）**，服务由 systemd 拉起 |
| 启动顺序是"一个接一个" | systemd 是**并行**启动的（这也是 CentOS 7 比 6 开机快的原因） |

---

## 二、必须知道的三个概念

### Q2 面试官：initramfs 是干什么的？（★ 追问高频）

> **考察点**：鸡生蛋问题 · 内核模块加载时机

```text
问题：内核挂根分区时，可能要用到 LVM / RAID / 加密的驱动，
      但这些驱动又装在根分区上 → 鸡生蛋问题
解法：bootloader 额外加载一个很小的内存文件系统 initramfs（临时根），
      里面有挂根分区所需的驱动和脚本，挂上真根后再 switch_root 切过去
```

> **答法**：**「initramfs 是一个临时的内存文件系统，解决『要用驱动才能挂根分区、但驱动又在根分区上』的问题。」**

---

### Q3 面试官：为什么叫 PID 1？它负责什么？

> **考察点**：进程树根 · 孤儿收养 · 容器 PID 1 的坑

- 内核启动后运行的**第一个用户态进程**就是 **PID 1（systemd）**，其它所有进程都是它的子孙。
- 它负责：**按依赖启动服务、收养孤儿进程（收尸）、响应 `systemctl` 的请求**。
- **推论**：`kill -9 1` 会搞死整个系统；容器里那个 PID 1 也是同样的角色（所以容器里「1 号进程不收信号」是个经典坑，见 `05-Q7`）。

---

### Q4 面试官：target 和旧的 runlevel 怎么对应？

> **考察点**：systemd 运行级别 · 服务器默认级别

| target | 旧 runlevel | 含义 |
| --- | --- | --- |
| `poweroff.target` | 0 | 关机 |
| `rescue.target` | 1 | 单用户 / 救援 |
| `multi-user.target` | 3 | 多用户命令行（**服务器默认**） |
| `graphical.target` | 5 | 图形界面 |
| `reboot.target` | 6 | 重启 |

```bash
systemctl get-default                       # 看默认进哪个 target
systemctl set-default multi-user.target     # 设默认（服务器就别开图形了）
systemctl list-units --type=target          # 看当前有哪些 target
```

> **一句话**：**`rescue.target` = 单用户/救援 = runlevel 1；`multi-user.target` = runlevel 3（服务器默认）；`graphical.target` = runlevel 5。**

---

## 三、怎么验证 / 怎么看启动过程

### Q5 面试官：开机慢怎么查？怎么知道上次为什么重启？

> **考察点**：`systemd-analyze` · `journalctl -b` · 排障工具

| 命令 | 看什么 |
| --- | --- |
| `dmesg -T` | 内核启动日志（带可读时间），看驱动、磁盘、内存识别情况 |
| `journalctl -b` | **本次开机**的所有日志（`-b -1` 看上次开机，排查"上次为什么重启"很有用） |
| `journalctl -b -p err` | 只看本次开机的错误级别日志 |
| **`systemd-analyze`** | 开机总耗时：`firmware + loader + kernel + userspace` 四段 |
| **`systemd-analyze blame`** | **哪个服务最拖开机速度**（★ 面试很爱问"开机慢怎么办"） |
| `systemd-analyze critical-chain` | 关键路径：是谁在等谁 |
| `cat /proc/cmdline` | 当前内核启动参数（改 grub 后可以在这里确认生效） |
| `uptime -s` / `who -b` | 本次开机时间（判断机器是不是刚重启过） |
| `systemctl --failed` | **有哪些服务启动失败**（排障第一站） |

```bash
# 面试常问："服务器开机慢，怎么查？"
systemd-analyze blame | head -10        # 找出最慢的服务
systemctl --failed                      # 找失败的服务
journalctl -b -p warning | tail -50     # 看警告和报错
```

> **常见原因**：网络等待（`NetworkManager-wait-online`）、某个挂载超时（`nfs`、`fstab` 里的盘没插）、服务互相等、磁盘 fsck 太慢。

---

## 四、启动失败怎么救（★ 运维岗真正的高分点）

### Q6 面试官：系统起不来，你怎么判断卡在哪一层、怎么救？

> **考察点**：救援模式 · fstab 事故 · 防御性习惯
> **这是本章最实用的一节**，也是面试官判断"你有没有真上过手"的地方。

#### 第一步：先定位卡在哪一层

```text
屏幕停在 No bootable device                      → 阶段1：固件找不到盘（启动顺序 / 硬盘故障）
屏幕进 grub rescue>                              → 阶段2：GRUB 配置或 /boot 丢了
屏幕刷一堆内核日志后 Kernel panic                 → 阶段3：根文件系统挂不上（驱动/盘/UUID 变了）
屏幕出现 emergency mode 或 "Give root password"   → 阶段4：fstab 有错 / 服务挂载失败
能登录但某些服务没起                              → 阶段5：服务自身的问题（看 journalctl）
```

#### 第二步：常用救援入口（记住两个）

**① 救援 / 单用户模式**：GRUB 菜单上按 `e` 编辑启动项，在 `linux16 /vmlinuz...` 那行末尾加参数：

```text
systemd.unit=rescue.target     # 进救援模式（几乎所有服务都不起，但根是挂好的）
rd.break                       # 更早一步，在 switch_root 之前打断（CentOS 7/8 改密码标配）
init=/bin/bash                 # 直接绕过 systemd 起一个 bash（最低限度环境）
```

改完按 `Ctrl+X` 启动。**`rd.break` 之后的标准动作（★ 背下来）**：

```bash
mount -o remount,rw /sysroot      # 此时 /sysroot 是只读挂载的
chroot /sysroot
passwd root                        # 改密码 / 或注释掉 fstab 里写错的那一行
touch /.autorelabel                # 改过密码要重打 SELinux 标签，否则登录失败
exit
reboot
```

> **`touch /.autorelabel` 这步最容易漏** —— 漏了的话改完密码还是登不进去（SELinux 标签不对）。这个细节能说出来非常加分。

**② fstab 写错导致起不来（★ 最经典的运维事故）**：

```bash
# 进 emergency mode 或 rd.break 后：
mount -o remount,rw /              # 根分区重新挂成可写
vim /etc/fstab                     # 注释掉写错的行，或改成正确的 UUID
mount -a                           # 验证不报错
reboot
```

> **防御性习惯（★ 面试说出来就是加分）**：
> **改 fstab 前先备份**、用 **UUID** 而不是 `/dev/sdb1`、加上 **`nofail`** 选项让挂不上也不阻塞启动、改完必须 `mount -a` 验证。

#### 第三步：其它启动故障速查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| GRUB 菜单没有新内核 | `grub2-mkconfig` 没生成 | `grub2-mkconfig -o /boot/grub2/grub.cfg` |
| 换盘/扩盘后起不来（UUID 变了） | fstab 里写的是旧 UUID | `blkid` 查新 UUID，改 fstab |
| 服务启动失败但系统能起 | 端口占用、配置错、权限错 | `systemctl status` + `journalctl -u`（见 `04-Q6`） |
| 开机慢 | 某个服务等超时（网络、挂载） | `systemd-analyze blame` |
| 内核升级后没生效 | 没重启，或默认内核没切 | `uname -r` 对比，重启生效 |

---

### Q7 面试官：怎么让服务开机自启？

> **考察点**：target 与 enable 的本质

```bash
systemctl enable 服务          # 正规做法：在 target 的 .wants 目录建软链接
systemctl enable --now 服务    # 现在起来 + 开机也起
```

其他做法：

| 做法 | 说明 |
| --- | --- |
| `crontab -e` 里写 `@reboot /path/script.sh` | 临时任务，但**环境很干净**（见 `03-Q8`） |
| `/etc/rc.local` | 老系统，**要确认有执行权限**（`chmod +x`），新系统默认没启用 |

---

### Q8 面试官：内核升级完为什么重启才生效？

> **考察点**：运行中的内核在内存里

> **答**：因为**运行中的内核在内存里**，新内核文件只是写到了 `/boot`，只有重新走一遍引导流程才会加载它。`uname -r` 能看到当前跑的内核版本。
> **顺带补一句**：CentOS 7 升级内核后，如果默认还是旧内核，要用 `grub2-set-default 0`（或 `grubby`）切换，再重启验证 —— 这也是"升级了却没生效"的常见原因。

---

## 五、本章追问链

```text
五阶段        →  每一阶段"谁在干活"？
              →  initramfs 解决什么问题？
              →  为什么说是"控制权交接"？
PID 1         →  它负责什么？（收尸 / 启服务 / 响应 systemctl）
              →  容器里的 PID 1 有什么坑？（05-Q7）
target        →  和 runlevel 怎么对应？
              →  enable 本质做了什么？（.wants 软链接）
起不来        →  怎么定位卡在哪一层？（屏幕现象 → 对应阶段）
              →  rd.break 之后要做什么？（remount / chroot / autorelabel）
fstab         →  为什么用 UUID？（盘符会变）
              →  加 nofail 有什么用？（挂不上不阻塞开机）
开机慢        →  systemd-analyze blame / systemctl --failed / journalctl -b
```

---

## 六、3 分钟自测（闭卷）

- [ ] 能按顺序说出 5 个阶段，并说出每个阶段"谁在干活"
- [ ] 能说出 initramfs 解决什么问题
- [ ] 知道 PID 1 是 systemd，并知道它负责什么
- [ ] 能说出 `rescue.target` / `multi-user.target` 和 runlevel 的对应
- [ ] 会用 `systemd-analyze blame` 和 `journalctl -b` 排查开机问题
- [ ] 能说出 GRUB 菜单加 `rd.break` 后改 root 密码的完整步骤（含 `touch /.autorelabel`）
- [ ] 能说出 fstab 写错怎么救、以及怎么预防（UUID / nofail / mount -a / 备份）
- [ ] 能说出内核升级完为什么要重启才生效

---

## 关联章节

| 想深挖 | 去哪 |
| --- | --- |
| 服务起不来的排查顺序 | `04-Q6` + `00` 场景题 4 |
| fstab 六字段、UUID、nofail | `08-Q4` |
| 磁盘换了 UUID 变了怎么办 | `08-Q4` + 本章 Q6 |
| 内核升级的操作过程 | `notes1` 里对应阶段笔记 |
