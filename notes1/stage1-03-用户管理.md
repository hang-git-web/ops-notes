# 用户与用户组管理

## 1. 创建新用户

```bash
useradd 名字     # 创建新用户
adduser 名字     # 创建新用户（CentOS 下与 useradd 相同）
```

> 创建后还需要设置密码才能登录；把用户分到合适的用户组在后面讲解。

## 2. 设置密码

| 命令 | 作用 |
| --- | --- |
| `passwd 名字` | 设置用户密码 |
| `passwd -l tom` | 锁定密码（禁止登录） |
| `passwd -u tom` | 解锁密码 |
| `passwd -e tom` | 让密码过期（下次登录强制改密） |

```bash
passwd tom        # 为 tom 设置密码
passwd -l tom     # 锁定 tom 的密码
passwd -u tom     # 解锁 tom 的密码
passwd -e tom     # 使 tom 的密码立即过期
```

## 3. 设置账号过期时间

| 命令 | 作用 |
| --- | --- |
| `chage -l 用户名` | 查看账号过期时间 |
| `chage -E 0 用户名` | 账号立即过期 |
| `chage -E 2099-02-26 用户名` | 设置账号 2099 年过期 |

```bash
chage -l tom                 # 查看 tom 的账号过期信息
chage -E 0 tom               # 让 tom 账号立即过期
chage -E 2099-02-26 tom      # 让 tom 账号 2099-02-26 过期
```

## 4. 用户组管理

| 命令 | 作用 |
| --- | --- |
| `groupadd 组名` | 创建用户组 |
| `usermod -aG 组名 账号名` | 把用户添加到组 |
| `gpasswd -d 账号名 组名` | 把用户从组中移除 |
| `groupdel 组名` | 注销（删除）用户组，删完可用 `/etc/group` 查询验证 |
| `getent group 组名` | 查看组里有哪些用户 |
| `id -nG 账号名` | 查看账号属于哪些组 |

```bash
groupadd dev                 # 创建 dev 组
usermod -aG dev tom          # 把 tom 加入 dev 组
gpasswd -d tom dev           # 把 tom 从 dev 组移除
groupdel dev                 # 删除 dev 组
getent group dev             # 查看 dev 组成员
id -nG tom                   # 查看 tom 所属的所有组
```

## 5. 总结：修改用户属性命令

| 操作 | 命令 |
| --- | --- |
| 修改账户属性 | `usermod` |
| 更改密码过期时间 | `chage` |
| 修改用户密码过期时间 | `passwd -e` / `chage` |
| 添加用户到用户组 | `usermod -aG 组名 用户名` |
| 从组中移除用户 | `gpasswd -d 用户名 组名` |

## 6. 添加用户到 sudo 组

CentOS 的 sudo 组叫 **wheel**：

```bash
usermod -aG wheel 账号名    # 加入 wheel 组，获得半个管理员权限（可 sudo）
```

> 加入 wheel 组后，该用户登录即可使用 `sudo` 执行管理员命令。