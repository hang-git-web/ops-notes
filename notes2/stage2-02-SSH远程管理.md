# SSH 登录原理与安全加固

## 1. TCP 三次握手（连接建立的"打招呼"）

TCP 三次握手是**建立连接**的过程，与密码/密钥验证无关，只有三次，每次一个标志：

```text
① 客户端 → 服务器：SYN（我想和你建立连接）
② 服务器 → 客户端：SYN-ACK（收到，我同意）
③ 客户端 → 服务器：ACK（确认，连接建立）
```

```text
客户端 ── SYN ──────► 服务器
客户端 ◄─ SYN-ACK ─── 服务器
客户端 ── ACK ──────► 服务器    ← 连接建立，开始传输数据
```

> 之后才进入 SSH 认证阶段（密码或密钥验证）。三次握手少一次都连不上，这就是"三次"的意义。

## 2. 密钥登录（公钥/私钥方式，代替密码）

### 概念回顾

| 名词 | 比喻 | 作用 |
| --- | --- | --- |
| 公钥 | 锁 / 验章卡 | 放到服务器，验证身份 |
| 私钥 | 钥匙 / 印章 | 留在本机，签名证明身份 |

### 命令行方式

**① 本机生成密钥对（Win+R → cmd）：**

```bash
ssh-keygen -t ed25519
```

生成地址在 `C:\Users\用户名\.ssh\`，其中 `.pub` 结尾的是公钥：

```bash
type %USERPROFILE%\.ssh\id_ed25519.pub     # 查看公钥内容
```

**② 把公钥放进服务器：**

```bash
cd /root/.ssh
vim authorized_keys
```

把公钥粘贴进去保存，并改权限：

```bash
chmod 600 authorized_keys
chmod 700 /root/.ssh
```

**③ Xshell 配置公钥登录：**

1. 会话属性 → 用户身份验证
2. 方法改为 **Public Key**
3. 右侧点"浏览"选择私钥文件（`id_ed25519`）
4. 用户名填 `root`，连接即可免密登录

## 3. SSH 安全加固实战

### ① 禁止 root 登录

```bash
vim /etc/ssh/sshd_config
```

```ini
PermitRootLogin no
```

> 需要先用普通用户登录，再用 `sudo` 提权；否则把自己锁在门外。

### ② 限制并发登录数

```bash
vim /etc/security/limits.conf
```

底部追加：

```ini
root   hard   maxlogins   2      # root 最多同时 2 个登录会话
```

> 说明：`maxlogins` 限制的是**并发会话数量**，密码和密钥登录都会计入。

### ③ 连接超时调整

**SSH 会话保活（防挂死连接）：**

```bash
vim /etc/ssh/sshd_config
```

```ini
ClientAliveInterval 300      # 每 300 秒发一次心跳
ClientAliveCountMax 3        # 3 次无响应则断开
```

**全局会话超时（闲置自动退出）：**

```bash
vim /etc/profile
```

底部追加：

```bash
export TMOUT=3600     # 闲置 3600 秒（1小时）自动退出
```

```bash
source /etc/profile   # 立即生效
```

### ④ 密码复杂度

```bash
vim /etc/security/pwquality.conf
```

```ini
minlen = 12        # 密码最短 12 位
dcredit = -1       # 至少 1 个数字
ucredit = -1       # 至少 1 个大写字母
lcredit = -1       # 至少 1 个小写字母
ocredit = -1       # 至少 1 个特殊符号
```

### ⑤ 密码时效性

```bash
chage -M 90 用户名      # 密码 90 天有效
chage -W 7 用户名       # 过期前 7 天提醒
chage -l 用户名         # 查看密码有效期
```

### ⑥ 端口安全与 fail2ban 联动

**修改 SSH 端口（避开默认 22 的扫描）：**

```bash
vim /etc/ssh/sshd_config
```

```ini
Port 2222
```

**安装 fail2ban，自动封禁暴力破解 IP：**

```bash
yum -y install fail2ban
systemctl enable --now fail2ban
```

fail2ban 会监控登录失败日志，连续失败 N 次后自动封禁该 IP。

### ⑦ 用户访问白名单机制

只允许指定用户通过 SSH 登录：

```bash
vim /etc/ssh/sshd_config
```

```ini
AllowUsers yuanhang root    # 白名单：只允许这两个用户 SSH 登录
```

> 白名单优先于黑名单：配置了 AllowUsers，名单外用户一律拒绝。

### ⑧ 全部改完后重启服务

```bash
systemctl restart sshd
systemctl status sshd
```

## 4. 高级应用场景

| 场景 | 说明 |
| --- | --- |
| Ansible | 基于 SSH 的批量任务调度，免密登录后一次管理成百上千台机器 |
| Zabbix | 利用 SSH 隧道实时监控服务器，安全传输监控数据 |

## 5. 一句话总结

SSH 加固三板斧：**改端口 + 禁 root + 白名单**；钥匙（密钥）比密码更安全，配合 fail2ban 封暴力破解，再结合 Ansible/Zabbix 就能大规模自动化运维。