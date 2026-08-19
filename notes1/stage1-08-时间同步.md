# 时间同步：NTP 与 Chrony

## 1. 为什么要时间同步

| 原因 | 说明 |
| --- | --- |
| 日志排错 | 各服务器时间不一致，日志顺序错乱没法查 |
| 分布式系统 | 节点时间不同步会导致数据、任务调度异常 |
| 安全协议 | HTTPS 证书、Kerberos 认证对时间敏感 |
| 审计合规 | 需要统一准确的时间戳 |

## 2. NTP 协议基本原理

- **NTP（Network Time Protocol）**：传统、成熟的时间同步协议，**可靠，适合服务器**
- 通过 UDP 123 端口与时间服务器通信，按层级（stratum）分级，同步精度通常到毫秒级

## 3. Chrony 简介

- **Chrony**：比较新的时间同步工具，**同步速度快，适合虚拟机、网络不稳定环境**
- 对网络抖动容忍度高，即使网络波动也能较快校准

## 4. NTP 服务配置

### ① 安装

```bash
yum -y install ntp
```

> ⚠️ CentOS 7 已停止维护（EOL，2024 年 6 月）：官方镜像源下线，`mirrorlist.centos.org` 不再提供服务。解决办法是把 yum 源切换到"归档源"（vault），国内推荐用阿里云归档镜像：
>
> ```bash
> sed -i 's/^mirrorlist=/#mirrorlist=/' /etc/yum.repos.d/CentOS-Base.repo
> sed -i 's|^#baseurl=http://mirror.centos.org/centos/$releasever/|baseurl=http://mirrors.aliyun.com/centos-vault/7.9.2009/|' /etc/yum.repos.d/CentOS-Base.repo
> yum clean all && yum makecache
> ```

### ② 服务启动与自启配置

```bash
systemctl restart ntpd
systemctl enable ntpd
systemctl status ntpd     # 检查是否执行成功
```

### ③ 补充：怎样确认服务名

**第一招：记住"d"的规律（最快）**

Linux 里很多服务程序是"守护进程"（daemon），命名规律是 **命令 + d**：

| 包名 | 服务名 | 规律 |
| --- | --- | --- |
| ntp | ntpd | ntp + d |
| httpd | httpd | 本来就带 d |
| openssh-server | sshd | ssh + d |
| vsftpd | vsftpd | 带 d |
| crontabs | crond | cron + d |

> 但不是所有服务都这样（nginx、mysql、docker 就不带 d），这招只算"大概率"，不能保证。

**第二招：让系统告诉你（最可靠）**

```bash
systemctl list-unit-files | grep ntp
```

### ④ 修改 NTP 配置（/etc/ntp.conf）

```bash
vim /etc/ntp.conf
```

注释掉原有的 CentOS 时间服务器，加上下面一段：

```ini
server ntp.aliyun.com iburst
server time.apple.com iburst
server ntp.ntsc.ac.cn iburst
```

### ⑤ 验证

```bash
ntpq -p        # 查看服务器时间同步状态以及 IP 地址（实时状态）
date           # 查看现在时间
```

> ⚠️ 提示：如果 `date` 发现差一天，通常**不是网络延迟，而是时区问题**（例如系统时区是 PDT 而你在上海）。修复：
>
> ```bash
> timedatectl set-timezone Asia/Shanghai
> date
> ```

## 5. Chrony 时间同步

### ① 安装（先停掉 ntp 服务，避免端口冲突）

```bash
systemctl stop ntpd
yum -y install chrony
```

### ② 修改配置（可选）

```bash
vim /etc/chrony.conf
```

在 `server` 区域加国内时间服务器：

```ini
server ntp.aliyun.com iburst
server ntp.ntsc.ac.cn iburst
```

### ③ 启动服务

```bash
systemctl restart chronyd
systemctl enable chronyd
```

### ④ 验证

```bash
chronyc tracking     # 查看实时同步状态
chronyc sources      # 判断服务器网络稳定性
```

## 6. 方案对比与选型建议

| 对比项 | NTP | Chrony |
| --- | --- | --- |
| 定位 | 传统可靠 | 比较新 |
| 精度 | 毫秒级 | 毫秒级（同步更快） |
| 适合场景 | 物理服务器 | 虚拟机、网络不稳定环境 |
| 特点 | 成熟稳定、生态广泛 | 启动快、抗网络抖动、云环境友好 |

**选型建议：**

- 物理机、传统机房环境 → **NTP**（成熟可靠）
- 虚拟机、云主机、网络不稳定 → **Chrony**（更快更稳）
- 学习环境建议直接用 **Chrony**，省心