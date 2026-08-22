# NFS 网络文件系统配置

> 架构：服务端（CentOS 7）共享目录，客户端（Ubuntu）挂载使用。以下 IP 按 `192.168.232.0/24` 网段、服务端 `192.168.232.131` 示例，实际替换为你自己的 IP。

## 一、服务端配置

### 1. 安装 NFS 安装包

```bash
yum -y install nfs-utils
```

### 2. 创建共享文件夹

```bash
mkdir -p /export/share
```

### 3. 设置 NFS 配置文件（/etc/exports）

```bash
yum -y install vim    # 没有 vim 先装
vim /etc/exports
```

配置格式：`共享目录 允许访问的网段(权限)`，写入：

```ini
/export/share 192.168.232.0/24(rw,sync,no_root_squash)
```

| 权限选项 | 含义 |
| --- | --- |
| `rw` | 可读可写 |
| `sync` | 同步写入磁盘 |
| `no_root_squash` | 客户端 root 拥有管理员权限 |

### 4. 启动服务

```bash
systemctl restart nfs-server
systemctl status nfs-server
systemctl enable nfs-server      # 开机自启
```

> 修改 /etc/exports 后也可以不重启服务，用 `exportfs -rv` 重新加载。

### 5. 给共享目录添加权限

```bash
chmod 755 /export/share
```

### 6. 准备测试文件

```bash
touch /export/share/file1.txt /export/share/file2.txt
echo "hello nfs" >> /export/share/file1.txt
```

## 二、客户端配置

### 1. 安装 NFS 安装包（客户端装 nfs-common 即可）

```bash
apt update
apt -y install nfs-common
```

检查服务：

```bash
systemctl status rpcbind    # 或 systemctl status nfs-common
```

### 2. 创建挂载点

```bash
mkdir -p /mnt/nfs
```

### 3. 挂载

```bash
mount -t nfs 192.168.232.131:/export/share /mnt/nfs
```

### 4. 卡住了？检查防火墙

```bash
systemctl status firewalld      # CentOS 客户端
sudo ufw status                 # Ubuntu 客户端用 ufw
```

防火墙未关则关闭：

```bash
systemctl stop firewalld        # CentOS
systemctl disable firewalld
sudo ufw disable                # Ubuntu
```

### 5. 检查是否挂载成功

```bash
df -h | grep nfs
ls /mnt/nfs          # 能看到服务端的 file1.txt、file2.txt
```

### 6. 永久挂载（开机自动挂载）

```bash
vim /etc/fstab
```

在底部追加：

```ini
192.168.232.131:/export/share  /mnt/nfs  nfs  defaults  0 0
```

```bash
mount -a      # 按 fstab 挂载
df -h         # 检查是否挂载成功
```

卸载：

```bash
umount /mnt/nfs
```

> ⚠️ 不能在挂载目录**内部**执行卸载（会报 target is busy），要先 `cd /` 再 umount。

### 7. 检查 NFS 共享是否生效

```bash
showmount -e 192.168.232.131
```

能看到共享的 `/export/share` 列表即配置成功。

## 三、口诀与注意事项

| 口诀 | 含义 |
| --- | --- |
| 白名单大于黑名单 | 用允许列表（/etc/exports 白名单），别用拒绝逻辑 |
| 客户能读就不要给他通 | 权限最小化：只读就不给写 |
| 能普通不 root | 尽量不给 root 权限，默认保留 `root_squash` |

其他要点：

- 防火墙、SELinux 是 NFS 挂载卡住的两大常见原因（CentOS 还可用 `setenforce 0` 临时验证）
- 客户端卸载前确认没有进程占用挂载目录
- 生产环境建议给 NFS 加权限限制（如 `ro` 只读、指定具体客户端 IP 而非整个网段）