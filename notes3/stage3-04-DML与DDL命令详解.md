# 数据库 SQL 语句笔记:DML 与 DDL

## 一、DML:数据操作语言

DML(Data Manipulation Language)用于操作数据库中的**数据(表内记录)**。

常见语句:

| 语句 | 作用 |
| --- | --- |
| SELECT | 查询表中的数据 |
| INSERT | 向表中插入新数据 |
| UPDATE | 修改表中已存在的数据 |
| DELETE | 删除表中的数据 |

> 说明:严格按分类,SELECT 属于 DQL(数据查询语言),但在日常学习和面试中通常和 DML 一起讲。

### 1. SELECT 查询

```sql
SELECT 列1, 列2 FROM 表名
WHERE 条件
ORDER BY 列名 [ASC|DESC]   -- 排序,默认 ASC 升序
LIMIT 行数;
```

示例:查询年龄大于 18 的用户,按注册时间倒序

```sql
SELECT * FROM users
WHERE age > 18
ORDER BY created_at DESC;
```

### 2. INSERT 插入

```sql
INSERT INTO 表名 (列1, 列2, ...)
VALUES (值1, 值2, ...);
```

批量插入:

```sql
INSERT INTO 表名 (列1, 列2)
VALUES (值1, 值2), (值3, 值4);
```

示例:

```sql
-- 插入单条数据
INSERT INTO users (username, email, age)
VALUES ('alice', 'alice@example.com', 25);

-- 插入多条数据
INSERT INTO products (name, price)
VALUES ('Book', 29.9), ('Pen', 5.5);
```

### 3. UPDATE 更新

```sql
UPDATE 表名
SET 列1 = 值1, 列2 = 值2
WHERE 条件;
```

示例:修改用户邮箱

```sql
UPDATE users
SET email = 'alice_new@example.com'
WHERE id = 1;
```

### 4. DELETE 删除

```sql
DELETE FROM 表名
WHERE 条件;
```

示例:删除特定用户

```sql
DELETE FROM users
WHERE id = 100;
```

删除表中所有数据(表结构保留):

```sql
DELETE FROM 表名;
```

> **安全提示**:`UPDATE` 和 `DELETE` 不写 `WHERE` 会影响全表数据,执行前先用相同条件 `SELECT` 确认影响范围,重要操作前先备份或开启事务。

## 二、DDL:数据定义语言

DDL(Data Definition Language)用于定义和管理数据库的**结构**。

常见语句:

| 语句 | 作用 |
| --- | --- |
| CREATE | 创建新的数据库对象(库、表、索引) |
| ALTER | 修改已存在对象的结构 |
| DROP | 删除数据库对象 |
| TRUNCATE | 删除表中所有数据,但保留表结构 |

### 1. CREATE 创建

```sql
CREATE TABLE 表名 (
    列1 数据类型 [约束],
    列2 数据类型 [约束],
    PRIMARY KEY (列1)
);
```

示例:

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 常用数据类型

| 类型 | 说明 | 适用场景 |
| --- | --- | --- |
| INT | 4 字节有符号整数,范围 -2147483648 ~ 2147483647 | 用户数量、商品库存 |
| VARCHAR(n) | 可变长度字符串,n 最大 65535 | 姓名、地址等长度不定的文本 |
| CHAR(n) | 固定长度字符串,n 为 0~255,不足会用空格补齐 | 身份证号等长度固定的数据 |
| DATE | 日期,格式 YYYY-MM-DD | 生日、入职日期 |
| DATETIME | 日期时间,格式 YYYY-MM-DD HH:MM:SS | 订单创建时间等具体时间点 |
| DECIMAL | 精确小数,可指定精度和标度 | 商品价格、账户余额等财务数据,避免浮点误差 |
| ENUM | 枚举,从预定义列表中取值 | 如 `ENUM('男','女')` 限制取值范围 |
| TEXT | 大文本,最大 65535 字节 | 文章内容、产品描述 |

### 2. ALTER 修改表结构

```sql
-- 添加列
ALTER TABLE 表名 ADD COLUMN 列名 数据类型;

-- 修改列类型
ALTER TABLE 表名 MODIFY COLUMN 列名 新数据类型;

-- 删除列
ALTER TABLE 表名 DROP COLUMN 列名;

-- 添加索引
ALTER TABLE 表名 ADD INDEX 索引名 (列名);
```

### 3. DROP 删除对象

```sql
-- 删除表
DROP TABLE 表名;

-- 删除数据库
DROP DATABASE 数据库名;

-- 删除索引
ALTER TABLE 表名 DROP INDEX 索引名;
```

### 4. TRUNCATE 清空表

```sql
TRUNCATE TABLE 表名;
```

## 三、DDL 与 DML 的区别

| 对比项 | DML | DDL |
| --- | --- | --- |
| 作用对象 | 数据(表内记录) | 数据库结构(表、索引) |
| 事务支持 | 支持,可回滚 | 隐式提交事务,不可回滚 |
| 执行速度 | 逐行操作,较慢 | 直接操作结构,较快 |
| 典型命令 | SELECT、INSERT、UPDATE、DELETE | CREATE、ALTER、DROP、TRUNCATE |

## 四、DELETE / TRUNCATE / DROP 速查

| 操作 | 删除内容 | 表结构 | 可回滚 | 速度 |
| --- | --- | --- | --- | --- |
| DELETE | 按条件删除行 | 保留 | 可以 | 慢(逐行删并写日志) |
| TRUNCATE | 全部数据 | 保留 | 不可以 | 快 |
| DROP | 整张表(数据+结构) | 删除 | 不可以 | 快 |