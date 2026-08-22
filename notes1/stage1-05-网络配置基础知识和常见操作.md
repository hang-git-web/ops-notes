# 网络基础与静态 IP 配置

## 1. 网络基础概念

| 概念 | 比喻 | 作用 |
| --- | --- | --- |
| IP 地址 | 详细地址 | 设备在网络中的唯一标识 |
| 子网掩码 | 区分小区和门牌号 | 区分网络位和主机位 |
| 网关 | 小区保安亭（可以转送外卖） | 通往小区外世界的出口 |
| DNS | 外卖平台搜索框 | 域名解析：把名字翻译成 IP |

## 2. 网络配置命令

| 命令 | 说明 |
| --- | --- |
| `ifconfig` | 传统查看网卡命令（部分系统需安装 net-tools） |
| `ip addr show` | 现代推荐用法，简写为 `ip a s`，与 ifconfig 查看的信息不同 |

```bash
ifconfig        # 传统方式
ip addr show    # 现代方式
ip a s          # 简写
```

## 3. 网络诊断工具应用

| 工具 | 作用 | 示例 |
| --- | --- | --- |
| `ping` | 连通性测试 | `ping -c 4 8.8.8.8` |
| `netstat -nult` | 查看网络监听状态 | `netstat -nult` |
| `ss -nult` | netstat 的替代，更简洁高效 | `ss -nult` |

```bash
ping -c 4 8.8.8.8     # 测试到 8.8.8.8 的连通性
netstat -nult         # 查看所有监听的 TCP/UDP 端口
ss -nult              # 效果相同，更简洁高效
```

> `-n` 用数字显示地址端口，`-u` UDP，`-l` 仅显示监听状态，`-t` TCP。

## 4. 静态 IP 配置（目前是动态 IP）

### 配置文件位置

```text
/etc/sysconfig/network-scripts/ifcfg-ens33
```

```bash
cat /etc/sysconfig/network-scripts/ifcfg-ens33
```

> `BOOTPROTO="dhcp"` 表示 IP 由路由动态分配，但服务器不希望这样，要改成静态分配。

### 第一步：备份

```bash
cp /etc/sysconfig/network-scripts/ifcfg-ens33 /etc/sysconfig/network-scripts/ifcfg-ens33_bak_0818
```

### 第二步：修改配置

```bash
vim /etc/sysconfig/network-scripts/ifcfg-ens33
```



修改文件内容如下：

```ini
DEVICE=ens33
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.232.131
NETMASK=255.255.255.0
GATEWAY=192.168.232.2
```

| 配置项 | 含义 |
| --- | --- |
| `DEVICE=ens33` | 网卡设备名 |
| `BOOTPROTO=static` | 改为静态分配 |
| `ONBOOT=yes` | 开机时是否自动启用这块网卡 |
| `IPADDR=192.168.232.131` | 这台机器的固定 IP |
| `NETMASK=255.255.255.0` | 子网掩码 |
| `GATEWAY=192.168.232.2` | 网关，出网必过的大门 |

### 第三步：DNS 配置

```bash
vim /etc/resolv.conf
```

加入：

```ini
nameserver 8.8.8.8
```

> ⚠️ 小提示：`/etc/resolv.conf` 在重启网络时可能被覆盖，更推荐的做法是在 `ifcfg-ens33` 里加一行 `DNS1=192.168.232.2`，这样才永久生效。

### 第四步：重启网络生效

```bash
systemctl restart network
ip addr        # 验证 IP 是否变成 192.168.232.131
```

## 5. 完整流程：虚拟机访问 www.baidu.com

**场景：你住 192.168.232.131 这一户，想点一份"百度"外卖。**

**第一步：查地址（DNS 解析）**

你不知道百度的详细地址，于是打开外卖平台搜索框（DNS），输入 `www.baidu.com`。系统看 `/etc/resolv.conf`，知道该问哪个平台（`192.168.232.2` 或 `8.8.8.8`），问完得到结果：

```text
www.baidu.com → 103.235.46.102
```

**第二步：看是不是同小区（子网掩码判断）**

拿到地址后，先拿子网掩码 `255.255.255.0` 算一下：

```text
你的地址：192.168.232.131  →  网络位是 192.168.232（小区：192.168.232）
百度地址：103.235.46.102   →  网络位是 103.235.46（小区：103.235.46）
```

不是同一个小区！不同小区不能直接送，必须出小区。

> 如果目标和你同小区，比如 `192.168.232.1`，就直接送，不需要经过保安亭——这也是为什么 ping 同网段不需要网关。

**第三步：交给保安亭（网关）**

既然要出小区，数据包就交给保安亭 `192.168.232.2`。你的电脑查路由表，发现默认路由 `default via 192.168.232.2`，于是把所有"寄往外地的快递"统一交给它。

**第四步：保安亭接力转送（路由）**

保安亭（VMware NAT 网关）把快递转出去，经过 VMnet8 → Windows → 你的路由器 → 运营商，一路上的每一个"保安亭"（路由器）都只干一件事：看快递上写的目的地，把它送到下一站。最终到达百度服务器 `103.235.46.102`。

**第五步：百度回包**

百度收到请求，把回复按原路寄回来：还是经过那一串保安亭，最后从 `192.168.232.2` 送进小区，交到你手上 `192.168.232.131`。

**第六步：你看到结果**

浏览器显示网页，或者 `ping` 打出 `64 bytes from 103.235.46.102`。


# 06 - Ubuntu 静态 IP 配置（VMware 虚拟机）

**日期**：2026-08-20

---

## 目标
为 VMware 中的 Ubuntu Server 22.04 虚拟机配置静态 IP，保证 NFS 项目中的 163（服务端）和 164（客户端）IP 固定不变，避免 DHCP 重启后地址漂移。

**为什么需要静态 IP？**
- NFS 的访问控制（`/etc/exports`）和客户端挂载（`/etc/fstab`）都依赖 IP/主机名。
- DHCP 分配的地址重启后会变，一变所有挂载全部失效。
- 生产环境中存储服务器 IP 还会被写进 DNS、监控、备份白名单，从来都是静态。

---

## 前置检查

登录虚拟机后，先确认三件事：

```bash
ip addr                     # 当前 IP 和网卡名（如 ens33）
ip route | grep default     # 默认网关（本次实验为 192.168.232.2）
ls /etc/netplan/            # netplan 配置文件（本次为 50-cloud-init.yaml）
```

> ⚠️ 注意：Ubuntu 不用 CentOS 的 `/etc/sysconfig/network-scripts/ifcfg-ens33`，网络配置在 `/etc/netplan/` 下，且文件名可能不是 `00-installer-config.yaml`，以 `ls` 实际看到为准（本次是 `50-cloud-init.yaml`）。

---

## 操作步骤

### 1. 禁用 cloud-init 网络接管

`50-cloud-init.yaml` 是 cloud-init 自动生成的，直接改的话**重启后会被还原**。先关掉 cloud-init 的网络配置：

```bash
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

验证：

```bash
cat /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

### 2. 备份原配置

```bash
sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
```

`.bak` 结尾不会被 netplan 加载，放在同目录安全。

### 3. 写入静态 IP 配置

163（服务端）示例：

```bash
sudo tee /etc/netplan/50-cloud-init.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.232.130/24
      routes:
        - to: default
          via: 192.168.232.2
      nameservers:
        addresses: [192.168.232.2, 114.114.114.114]
EOF
```

164（客户端）只需把 IP 换成 `.131`：

```bash
sudo tee /etc/netplan/50-cloud-init.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.232.131/24
      routes:
        - to: default
          via: 192.168.232.2
      nameservers:
        addresses: [192.168.232.2, 114.114.114.114]
EOF
```

> 网关 `via:` 必须填第 1 步 `ip route` 里看到的实际地址，不要照抄。
> `tee` 整段在命令行提示符下粘贴，不要在编辑器里粘，否则 `EOF` 会被写进文件。

### 4. 应用配置

```bash
sudo netplan apply
```

如果出现 `WARNING:root:Cannot call Open vSwitch`，忽略即可，只是提示系统没装 OVS，不影响配置。

### 5. 验证

```bash
ip addr show ens33
ping -c 3 192.168.232.2     # 网关
ping -c 3 baidu.com         # 外网
```

预期：ens33 显示 `inet 192.168.232.130/24` 且**没有 `dynamic` 字样**。

> ⚠️ `netplan apply` 后 Xshell 会断线（IP 变了），用新 IP 重新连接即可。

---

## 验证结果

- [x] 163 ens33 静态 IP `192.168.232.130/24`，无 `dynamic`
- [x] 164 ens33 静态 IP `192.168.232.131/24`，无 `dynamic`
- [x] 网关 `192.168.232.2` 可 ping 通
- [x] 外网可访问（`ping baidu.com` 正常）
- [x] 163 与 164 互通（`ping 192.168.232.163`）
- [x] 重启后 IP 不变（cloud-init 已禁用）

---

## 关键知识点

- **CentOS vs Ubuntu 网卡配置**：CentOS 用 `ifcfg-ens33`（`BOOTPROTO=dhcp/static`），Ubuntu 用 netplan YAML（`dhcp4: true/false`），意思相同、语法不同。
- **cloud-init 会覆盖手动修改**：必须写 `99-disable-network-config.cfg` 关掉接管，否则重启还原 DHCP。
- **静态 IP 必须和网关同网段**：直接改成别的网段（如 `172.25.250.x`）会导致宿主机连不上、外网也断；要换网段得在 VMware 里加"仅主机模式"网卡单独承载。
- **netplan 可多文件合并**：`/etc/netplan/*.yaml` 都会生效，备份用 `.bak` 后缀避免被加载。
- **heredoc 写法**：`tee 文件 <<'EOF' ... EOF` 是把整段内容写进文件的一行式写法，适合避免在 nano/vim 里手滑。

---

## 常见坑与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| `No such file or directory` 找不到 `ifcfg-ens33` | Ubuntu 不用 CentOS 路径 | 用 `/etc/netplan/` 下的文件 |
| 文件是 `50-cloud-init.yaml` 不是 `00-installer-config.yaml` | Ubuntu Server 用 cloud-init 生成 | 以 `ls /etc/netplan/` 实际为准 |
| 改完重启 IP 又变回 DHCP | cloud-init 重新生成配置 | 先执行第 1 步禁用 cloud-init |
| `netplan apply` 后 Xshell 断线 | IP 变了 | 用新 IP 重连 |
| 文件末尾混进 `EOF` 行 | 把 tee 命令粘贴进了编辑器 | `sudo sed -i '$d' 文件` 删末行，或重新整段写入 |
| `Cannot call Open vSwitch` 警告 | 系统未装 OVS | 忽略，不影响配置 |
| 改成 `172.25.250.x` 后连不上 | 跨网段、无路由 | 留在当前网段，或加仅主机网卡 |


