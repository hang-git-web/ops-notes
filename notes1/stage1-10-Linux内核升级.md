# 内核升级

## 1. 内核定义与功能

- **内核是操作系统的核心**（相当于电脑的"发动机"）
- 主要功能：进程管理、内存管理、文件系统、设备驱动、网络协议栈、安全控制

## 2. 升级内核的必要性

| 原因 | 说明 |
| --- | --- |
| 安全漏洞修复 | 内核漏洞影响整台机器，升级是最重要的理由 |
| 新硬件/驱动支持 | 新 CPU、网卡、显卡需要新内核支持 |
| 性能与稳定性优化 | 新内核修复旧问题，性能更好 |
| 新软件兼容性 | 部分新软件要求较新的内核版本 |

## 3. 升级前准备

### ① 快照（最重要）

在 VMware 里给虚拟机做一个快照，出问题一键回退，比任何备份都省事。

### ② 系统备份

```bash
cp -r /boot/ /boot_backup_2026_08_19
cp -r /etc/ /etc_backup_2026_08_19
```

> `/boot` 存放内核和引导文件，`/etc` 存放系统配置，这两个是最容易出问题的目录。

## 4. 两种方法更新内核

### CentOS

**方法一：yum 直接更新（官方仓库，稳定保守）**

```bash
yum update -y
reboot
uname -r          # 验证新内核是否生效
```

**方法二：通过 ELRepo 升级主线最新内核**

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm

yum --enablerepo=elrepo-kernel install -y kernel-ml
reboot
uname -r
```

> `kernel-ml` = 最新主线版内核；想要更稳定可以用 `kernel-lt`（长期支持版）。

### Ubuntu / Debian

```bash
apt update              # 更新软件源
apt upgrade -y          # 升级系统（包含内核）
reboot                  # 重启生效
uname -r                # 查看当前内核版本
```

## 5. 清理旧内核文件

### Ubuntu / Debian

```bash
apt autoremove --purge    # 删除不再需要的旧内核
```

### CentOS

```bash
yum remove 旧内核包名      # 手动删除
# 或用 yum-utils 的清理工具，只保留最近 2 个
yum install -y yum-utils
package-cleanup --oldkernels --count=2
```

## 6. 注意事项

1. **升级后必须重启**，新内核才会生效
2. **旧内核保留**：开机 GRUB 菜单可以选择旧内核，新内核出问题可以回退
3. **生产环境谨慎升级**：重启=业务中断，除非有安全漏洞否则不轻易动内核
4. **升级后验证**：`uname -r` 查看内核版本号是否变化