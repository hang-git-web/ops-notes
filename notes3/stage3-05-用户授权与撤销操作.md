# MySQL 用户与权限管理笔记

## 一、创建用户

语法:

```sql
CREATE USER '用户名'@'主机地址' IDENTIFIED BY '密码';
```

示例:

```sql
-- 创建本地登录用户
CREATE USER 'dev'@'localhost' IDENTIFIED BY 'P@ssw0rd!';

-- 创建远程登录用户(允许任意 IP 访问)
CREATE USER 'admin'@'%' IDENTIFIED BY 'Admin!2025';
```

**主机地址的三种写法**

| 写法 | 含义 |
| --- | --- |
| `localhost` | 仅允许本机登录 |
| `%` | 允许任意 IP 登录 |
| `192.168.1.100` | 只允许该 IP 登录(可用 `192.168.1.%` 表示网段) |

**注意事项**

- 密码需符合安全策略:8 位以上,含大小写字母、数字、符号
- MySQL 8 默认认证插件是 `caching_sha2_password`;若老客户端(PHP 5.x 等)连接失败,可改为:
  ```sql
  CREATE USER 'dev'@'localhost' IDENTIFIED WITH mysql_native_password BY 'P@ssw0rd!';
  ```

## 二、授予用户权限

语法:

```sql
GRANT 权限列表 ON 数据库.表 TO '用户名'@'主机地址';
```

### 权限层级

| 层级 | 写法示例 | 作用范围 |
| --- | --- | --- |
| 全局 | `GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%';` | 所有数据库和表 |
| 数据库级 | `GRANT SELECT, INSERT ON mydb.* TO 'dev'@'localhost';` | 指定库的所有表 |
| 表级 | `GRANT UPDATE, DELETE ON mydb.orders TO 'user1'@'192.168.1.100';` | 指定单张表 |
| 列级 | `GRANT SELECT (id, name), UPDATE (price) ON mydb.products TO 'audit'@'%';` | 指定表的某些列 |

### 常用权限

| 权限 | 含义 |
| --- | --- |
| SELECT | 查询数据 |
| INSERT | 插入数据 |
| UPDATE | 更新数据 |
| DELETE | 删除数据 |
| CREATE | 创建表/数据库 |
| ALTER | 修改表结构 |
| DROP | 删除表/数据库 |
| INDEX | 管理索引 |
| ALL PRIVILEGES | 除 GRANT OPTION 外的全部权限 |
| GRANT OPTION | 允许用户把自己的权限再授权给他人(**高风险,慎用**) |
| USAGE | “无权限”,仅表示账号可登录 |

**注意事项**

- 授权后执行 `FLUSH PRIVILEGES;`(详见下方说明)
- `GRANT` 使用 `WITH GRANT OPTION` 时,被授权者可以继续转授权,生产环境一般不给

> 补充说明:用 `GRANT` / `REVOKE` 命令授权时,权限表会自动更新,**不需要**手动 `FLUSH PRIVILEGES`。只有直接用 `INSERT`/`UPDATE` 改 `mysql` 库的权限表时才需要刷新。

## 三、撤销权限

语法:

```sql
REVOKE 权限列表 ON 数据库.表 FROM '用户名'@'主机地址';
```

示例:

```sql
-- 撤销全局权限
REVOKE ALL PRIVILEGES ON *.* FROM 'admin'@'%';

-- 撤销特定权限
REVOKE DELETE, DROP ON mydb.* FROM 'dev'@'localhost';

-- 撤销列级权限
REVOKE SELECT, UPDATE ON hr.employees FROM 'audit'@'%';
```

**注意事项**

- 撤销的权限范围必须与授权时完全匹配,否则操作无效
- 撤销后如需立即生效,可执行 `FLUSH PRIVILEGES;`

## 四、用户管理

### 1. 重命名用户

```sql
RENAME USER 'old_user'@'localhost' TO 'new_user'@'localhost';
```

### 2. 修改密码

```sql
-- 方式一(推荐,MySQL 5.7/8.0 通用)
ALTER USER 'dev'@'localhost' IDENTIFIED BY 'NewPass!2025';

-- 方式二
SET PASSWORD FOR 'admin'@'%' = 'Admin#2025';
```

> 版本提示:MySQL 8.0 已移除 `PASSWORD()` 函数,不能再写 `SET PASSWORD ... = PASSWORD('xxx')`;直接写明文密码字符串即可,或统一使用 `ALTER USER`。

### 3. 删除用户

```sql
DROP USER '用户名'@'主机地址';
```

**操作建议**:先撤销该用户的所有权限,再删除用户,避免残留权限造成安全隐患。

## 五、查看用户权限

```sql
-- 查看某个用户的权限
SHOW GRANTS FOR 'dev'@'localhost';

-- 查询权限表(需较高权限)
SELECT * FROM mysql.user WHERE User = 'dev';
```

## 六、安全注意事项

1. **最小权限原则**:只授予完成工作所需的最小权限
2. **定期审查权限**:用 `SHOW GRANTS` 检查权限分配是否合理
3. **避免使用 root 日常操作**:root 仅用于管理任务,应用连接使用独立账号
4. **限制远程来源**:远程用户尽量限定到具体 IP 或网段(如 `192.168.1.%`),避免直接使用 `%`
5. **慎用 `WITH GRANT OPTION`**:避免权限被层层转授,失去控制

## 七、完整实操示例

```sql
-- 1. 创建应用账号(仅允许内网网段登录)
CREATE USER 'app_user'@'192.168.1.%' IDENTIFIED BY 'App@2025';

-- 2. 只授予业务库的读写权限
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'app_user'@'192.168.1.%';

-- 3. 查看授权结果
SHOW GRANTS FOR 'app_user'@'192.168.1.%';

-- 4. 撤销其中的删除权限
REVOKE DELETE ON shop.* FROM 'app_user'@'192.168.1.%';

-- 5. 修改密码
ALTER USER 'app_user'@'192.168.1.%' IDENTIFIED BY 'App@2026';

-- 6. 删除用户(先撤销全部权限)
REVOKE ALL PRIVILEGES ON shop.* FROM 'app_user'@'192.168.1.%';
DROP USER 'app_user'@'192.168.1.%';
```