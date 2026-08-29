# MySQL 8 安全加固完整流程（CentOS 7）

## 环境

- 系统：CentOS 7
- 数据库：MySQL 8.0.46
- 端口计划：默认 3306 → 修改为 3307

## 前置确认：先登录

```bash
mysql -uroot -p 
```

## 一、检查默认配置

```bash
mysql -uroot -p'Yuanhang@2026' -e "SELECT user, host, authentication_string FROM mysql.user;"
```

**风险点：** 默认可能存在匿名用户（`''@localhost`）或测试数据库（test）。

**删除匿名用户和测试库：**

```sql
DELETE FROM mysql.user WHERE User='';
DROP DATABASE IF EXISTS test;
FLUSH PRIVILEGES;
```

## 二、安全初始化配置

### 1. 运行安全初始化脚本

```bash
mysql_secure_installation
```

选项说明：

| 选项 | 推荐设置 | 作用 |
| --- | --- | --- |
| 密码强度校验插件（VALIDATE PASSWORD） | 新手选 0 | 禁用复杂度要求，避免密码设置失败 |
| 设置 root 密码 | 自定义强密码 | 至少 8 位，含大小写字母、数字、符号 |
| 删除匿名用户 | Y | 防止无密码用户访问 |
| 禁止 root 远程登录 | Y | 仅允许 root 本地登录 |
| 删除测试数据库（test） | Y | 测试库无权限控制，存在隐患 |
| 刷新权限表 | Y | 使配置立即生效 |


### 2. 修改配置文件（CentOS 路径：/etc/my.cnf）

```bash
vim /etc/my.cnf
```

在 `[mysqld]` 段下添加：

```ini
[mysqld]
port = 3307                    # 修改默认端口（避免扫描攻击）
bind-address = 127.0.0.1       # 只允许本机访问（如需远程改为指定 IP，勿用 0.0.0.0）
local_infile = 0               # 禁用 LOCAL INFILE（防止文件读取漏洞）
```

重启生效：

```bash
systemctl restart mysqld
```

> ⚠️ **改端口后的三个坑：**
>
> **1. 登录必须加 `-P 3307`**
>
> ```bash
> mysql -uroot -p -P 3307
> ```
>
> 忘记加端口会报 `Can't connect to MySQL server on 'localhost' (111)`。
>
> **2. CentOS 的 SELinux 可能拦截非标准端口**
>
> 如果启动失败或连不上，放行端口：
>
> ```bash
> yum -y install policycoreutils-python
> semanage port -a -t mysqld_port_t -p tcp 3307
> ```
>
> **3. 防火墙也要放行新端口**（如果开了防火墙）
>
> ```bash
> firewall-cmd --permanent --add-port=3307/tcp
> firewall-cmd --reload
> ```

## 三、用户权限最小化

```bash
mysql -uroot -p -P 3307
```

```sql
-- ① 创建专用用户（替代 root）
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'StrongPass!2026';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;

-- ② 撤销危险权限
REVOKE FILE ON *.* FROM 'admin'@'localhost';    -- 禁止文件操作
REVOKE SUPER ON *.* FROM 'admin'@'localhost';   -- 禁止修改系统变量

FLUSH PRIVILEGES;
```

## 四、密码策略强化

```sql
-- ① 安装密码强度组件（MySQL 8 需要手动安装）
INSTALL COMPONENT 'file://component_validate_password';

-- ② 配置密码规则
SET GLOBAL validate_password.policy = 1;    -- 强度级别 0~2，2 最强
SET GLOBAL validate_password.length = 10;   -- 最小长度 10
```

> MySQL 5.x 版本默认已安装密码插件，无需第一步。

## 五、日志与监控

```bash
vim /etc/my.cnf
```

`[mysqld]` 段追加：

```ini
[mysqld]
log-error = /var/log/mysqld.log    
general_log = 1
general_log_file = /var/log/mysql/general.log
log-bin = /var/lib/mysql/mysql-bin
server-id = 1
```

重启并查看：

```bash
systemctl restart mysqld
tail -f /var/log/mysql-general.log
```

> ⚠️ `general_log = 1` 会记录所有 SQL，**文件增长极快**——建议排查问题时开启，平时改为 `general_log = 0`。

## 六、安全配置复查（对照检查表）

```bash
mysql -uroot -p -P 3307
```

| 检查项 | 命令 | 预期结果 |
| --- | --- | --- |
| 匿名用户是否删除 | `SELECT user FROM mysql.user WHERE user='';` | 空结果 |
| root 能否远程登录 | `SELECT user,host FROM mysql.user WHERE user='root' AND host='%';` | 无记录 |
| 测试库是否存在 | `SHOW DATABASES LIKE 'test%';` | 无记录 |
| 默认端口是否修改 | 退出后 `netstat -tulnp \| grep mysql` | 显示 3307 |
| 本地文件读取是否禁用 | `SHOW VARIABLES LIKE 'local_infile';` | local_infile = OFF |

全部符合预期 = 安全配置完成 ✅

## 七、运维常见问题及解决

### 问题 1：忘记 root 密码（MySQL 8 修正版）

> ⚠️ 网上很多教程用 `UPDATE mysql.user SET authentication_string=PASSWORD('密码')`——**MySQL 8 已移除 PASSWORD() 函数，会报错**，必须用 ALTER USER。

```bash
systemctl stop mysqld
echo "skip-grant-tables" >> /etc/my.cnf    # 或 vim /etc/my.cnf 添加 skip-grant-tables=1
systemctl start mysqld
mysql -uroot                               # 无密码登录
```

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
FLUSH PRIVILEGES;
EXIT;
```

```bash
sed -i '/skip-grant-tables/d' /etc/my.cnf   # 删除跳过验证配置
systemctl restart mysqld
mysql -uroot -p -P 3307                      # 用新密码登录
```

### 问题 2：外部无法访问

检查顺序：

```bash
# ① 防火墙是否放行端口（CentOS）
firewall-cmd --permanent --add-port=3307/tcp
firewall-cmd --reload

# ② 配置里的 bind-address 是否限制为本机
grep bind-address /etc/my.cnf    # 如果是 127.0.0.1，外部当然连不上

# ③ SELinux 是否拦截
semanage port -l | grep mysqld   # 看 3307 在不在列表里

# ④ MySQL 是否真的在监听新端口
netstat -tulnp | grep mysql
```

## 八、核心记忆

- 改端口三件套：**`-P 3307` 登录 + 防火墙放行 + SELinux 放行**
- MySQL 8 重置密码：**ALTER USER**（不是 UPDATE + PASSWORD()）
- 权限最小化：**建专用用户 + REVOKE FILE/SUPER**
- 日志注意：general_log 排查时开，平时关


/etc/my.cnf配置如图1.png所示
