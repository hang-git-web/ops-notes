# Shell 编程 第四期：数组、组合命令与巡检实战

> **对应课件**：《Shell从入门到精通技能进阶（第四期）》（千山 2026-05）
> **本篇定位**：Shell 的"数据能力"和"实战收口"。**数组**让脚本能像程序一样管理"一批数据"；**组合命令**（grep/awk/sort/uniq/wc/cut）是日志分析的利器；最后用**综合巡检脚本**把三期学的全串起来。
> **环境**：Rocky Linux 8 / CentOS 7，`bash` 即可（关联数组需要 **bash 4.0+**，CentOS 7 自带的 4.2 满足）。
> **前置**：第二期（if/case）、第三期（for/while/函数）。
> **系列关系**：第二期（判断）→ 第三期（循环+函数）→ **第四期（数组+组合命令+巡检实战）**。

---

## 〇、开篇：建实验目录 + 确认 bash 版本

```bash
mkdir -p /data/shell-lab4
cd /data/shell-lab4

bash --version | head -1     # ★ 关联数组需要 4.0+，这里确认一下
# 期望类似：GNU bash, version 4.2.46(2)-release (x86_64-redhat-linux-gnu)
```

> **为什么要先看 bash 版本**：本篇的**关联数组**（`declare -A`）是 **bash 4.0 才引入**的。CentOS 7 是 bash 4.2、CentOS 6 是 4.1、有些老系统是 3.x —— 如果版本低于 4.0，关联数组直接报错。
> **这也是为什么脚本首行一定要写 `#!/bin/bash`**，而不是 `#!/bin/sh`。

---

## 一、数组（课件第一章）

### 1.1 数组是什么（课件原话）

> **课件原文**：**数组是一种数据结构，用于存入多个相同类型的数据集合。在 shell 中，用 `()` 表示数组，括号中间的元素用空格进行分隔。**

**为什么要用数组（对照第三期看一下）**：

```text
第三期我们这样写：
   for i in {200..210}                   ← 列表是"写死在代码里"的

有了数组：
   IPS=(192.168.1.10 192.168.1.11 ...)   ← 列表变成"数据"，可以增删改查
   for ip in "${IPS[@]}"; do ...         ← 循环遍历它

★ 数组最大的价值：**把"一堆同类数据"变成一个变量**，
  可以传参、可以追加、可以统计个数、可以从命令输出动态生成。
```

### 1.2 数组的两种类型（★ 课件开头的对比表）

| 类型 | 索引类型 | 是否需要明确声明 | 声明示例 | 适用场景（课件原话） |
| --- | --- | --- | --- | --- |
| **普通数组** | **数字** | 否 | `arr=(a b c)` | **顺序**（IP 列表、用户列表、服务列表） |
| **关联数组** | **字符串** | **是** | `declare -A arr2` | **键值统计**（IP 统计、TCP 分类统计） |

> **一句话区分**：
> **普通数组 = 一排编了号的格子**（`arr[0]`、`arr[1]`…），适合放"一个列表"；
> **关联数组 = 一本字典**（`arr["changsha"]`），适合做"计数、映射、归类"。

### 1.3 普通数组（课件 1.3）

#### 1.3.1 语法（课件原话）

```bash
数组名=(值1 值2 值3 ...)          # 一次性定义

# 或者逐个赋值（下标从 0 开始）
数组名[索引0]=值1
数组名[索引1]=值2
数组名[索引2]=值3
```

#### 1.3.2 取值（课件原话 + 实操输出）

```bash
arr=(a b c)

echo ${arr[1]}     # 取数组的单个值   → b
echo ${arr[0]}     #                  → a
echo ${arr[2]}     #                  → c
```

**课件里的实际输出**：

```text
[root@r810-201 ~]# arr=(a b c)
[root@r810-201 ~]# echo ${arr[1]}   #取数组的单个值
b
[root@r810-201 ~]# echo ${arr[0]}
a
[root@r810-201 ~]# echo ${arr[2]}
c
```

> **★ 第一个易错点：数组下标从 0 开始**（和大多数编程语言一样），所以 `arr[0]` 是第一个元素。

#### 1.3.3 四个"必会"的取值符号（★ 课件给了，最容易记混）

| 写法 | 含义 | 课件实际输出（`arr=(a b c)`） |
| --- | --- | --- |
| `${arr[0]}` | 取**某一个**元素 | `a` |
| **`${arr[@]}`** | 取**所有元素**（数组的值） | `a b c` |
| **`${#arr[@]}`** | 取**数组长度**（有几个人元素） | `3` |
| **`${!arr[@]}`** | 取**所有下标**（索引值） | `0 1 2` |

> **★ 记忆法**：
> ```text
> ${arr[@]}   → @ 像"all"，全部的值
> ${#arr[@]}  → # 像"number"，个数（长度）
> ${!arr[@]}  → ! 像"index"，下标
> ```
> **这三个符号（`@` `#` `!`）是数组最核心的东西，背下来。**

#### 1.3.4 增删（课件原话）

```bash
arr+=(d)              # 追加一个元素   → a b c d
unset arr[3]          # 删除指定下标   → a b c
unset arr             # 删除整个数组   → （空了）
```

**课件里的实际操作与输出**：

```text
[root@r810-201 ~]# echo ${arr[@]}   #数组的值
a b c
[root@r810-201 ~]# echo ${#arr[@]}  #数组的长度
3
[root@r810-201 ~]# echo ${!arr[@]}  #下标，或者是索引值
0 1 2
[root@r810-201 ~]# arr+=(d)
[root@r810-201 ~]# echo ${arr[@]}
a b c d
[root@r810-201 ~]# unset arr[3]
[root@r810-201 ~]# echo ${arr[@]}
a b c
[root@r810-201 ~]# unset arr
[root@r810-201 ~]# echo ${arr[@]}
（无输出）
```

> **注意 `unset arr[3]` 之后，数组长度变成 3，但下标是 `0 1 2`**（不是 `0 1 2 4` 那种"空洞"）—— 因为删的正好是最后一个。
> **如果删的是中间**（`unset arr[1]`），数组会变成**稀疏数组**：`${#arr[@]}` 还是 3（`a c d`），但 `${!arr[@]}` 是 `0 2 3`（**下标不连续**）。**遍历时用 `${!arr[@]}` 就不会漏**，这也是为什么推荐"按下标遍历"。

#### 1.3.5 三种遍历写法（★ 课件给的，重点掌握前两种）

```bash
# 写法一：值遍历 —— 只关心每个值是什么
for i in ${arr[@]}
do
    echo $i
done

# 写法二：索引遍历 —— 需要知道"第几个"时用
for i in ${!arr[@]}
do
    echo $i
done

# 写法三：C 风格遍历
for ((i=0;i<${#arr[@]};i++))
do
    echo ${arr[$i]}
done
```

> **★ 课件的记录里有个笔误**：`for ((i=0 i<${#arr[@]};i++))` 少了分号，正确是 `for ((i=0;i<${#arr[@]};i++))`。

#### 1.3.6 实操一下（课件原话 + 完整输出）

**课件里的实测记录**：

```text
[root@r810-201 ~]# for i in ${arr[@]};do echo $i;done
1
2
3
4
[root@r810-201 ~]# for i in  ${!arr[@]};do echo $i;done
0
1
2
3
```

**★ 注意这两个结果的关键差别**：

| 遍历方式 | 输出的东西 | 什么时候用 |
| --- | --- | --- |
| `for i in ${arr[@]}` | **元素的值**（1 2 3 4） | "我要对每个 IP 做 ping" |
| `for i in ${!arr[@]}` | **元素的下标**（0 1 2 3） | "我要打印编号"、"我要按下标操作" |

> **★ 一个容易忽略但很关键的细节：`${arr[@]}` 一定要加双引号**
> ```bash
> # 假设数组里有个元素含空格
> files=("my file.txt" "b.txt")
>
> for f in ${files[@]}; do  echo "[$f]"; done     # ❌ 会被拆成 my / file.txt / b.txt
> for f in "${files[@]}"; do echo "[$f]"; done     # ✅ 每一项都完整
> ```
> **规则**：**数组展开一律写成 `"${arr[@]}"`（带双引号）**，否则含空格或通配符的元素会出问题。
> 这一条是第三期"for 按空格切清单"那个坑的**正解**。

#### 1.3.7 企业小案例：MySQL 多库备份（课件原话，★ 最实用）

```bash
#!/bin/bash
DATABASES=( mysql qianshan sys )

for db in ${DATABASES[@]};do
    echo "正在备份: $db"
    mysqldump -uroot -p"Zhongguo@0755" ${db} >/opt/${db}-$(date +%Y-%m-%d-%T).sql
done
```

**逐句拆解**：

| 代码 | 含义 |
| --- | --- |
| `DATABASES=( mysql qianshan sys )` | **数组定义**：三个库名，空格分隔 |
| `for db in ${DATABASES[@]}` | 遍历数组的每一个值 |
| `mysqldump ... >/opt/${db}-$(date +%Y-%m-%d-%T).sql` | 每个库导出成一个带**时间戳**的文件 |

**这段的价值（★ 为什么比"写三条 mysqldump"好）**：

```text
要备份 3 个库 → 写 3 条 mysqldump
要备份 30 个库 → 还是写 3 条（把库名塞进数组即可），代码行数不变
要加一个库   → 只改数组那一行
★ 这就是"数据"和"代码"分离。列表变长，代码不变。
```

**完整可跑版本（含闭环验证，★ 加了三处生产必备的东西）**：

```bash
cat > mysql_backup.sh <<'EOF'
#!/bin/bash
set -euo pipefail                       # ★ 1) 出错就退出，避免"备份失败了还在往下跑"

DATABASES=( mysql qianshan sys )
BACKUP_DIR=/opt/mysql-backup
PASS='Zhongguo@0755'

mkdir -p "$BACKUP_DIR"                  # ★ 2) 目录不存在就建，别让脚本因路径报错

for db in "${DATABASES[@]}";do
    ts=$(date +%Y-%m-%d-%T)
    out="${BACKUP_DIR}/${db}-${ts}.sql"
    echo "正在备份: $db -> $out"
    mysqldump -uroot -p"$PASS" --single-transaction "$db" > "$out"

    # ★ 3) 验证：备份文件不为空才算成功
    if [ -s "$out" ]; then
        echo "  备份成功，大小: $(du -h "$out" | cut -f1)"
    else
        echo "  【错误】备份文件为空，请检查！"
        exit 1
    fi
done

echo "全部备份完成，共 ${#DATABASES[@]} 个库"
EOF

bash mysql_backup.sh
ls -lh /opt/mysql-backup/          # 检验：能看到三个带时间戳的 .sql
```

> **★ 加的三样东西，每一样都是"生产差异"**：
> | 加的 | 为什么 |
> | --- | --- |
> | `set -euo pipefail` | 任何一条命令失败就**立刻停下**。备份脚本最怕"库 1 失败、库 2 库 3 继续跑"，最后以为全备份了 |
> | `[ -s "$out" ]` 判空 | **`mysqldump` 失败时也会创建一个空文件**，不验证的话你会得到几个 0 字节的"备份" |
> | `${#DATABASES[@]}` | 用**数组长度**报总数，加/减库时这行不用改 |
>
> **`--single-transaction`**：InnoDB 下用来拿到**一致性快照而不锁表**，是备份在线数据库的标准做法。

### 1.4 关联数组（课件 1.4）

#### 1.4.1 语法（课件原话）

```bash
declare -A 数组名                    # ★ 必须先声明（这一句不能少）

# 然后两种赋值方式
数组名=( [索引1]=值1 [索引2]=值2 [索引3]=值3 ... )
# 或者
数组名[索引1]=值1
数组名[索引2]=值2
数组名[索引3]=值3
```

> **★ `declare -A` 是关联数组和普通数组的**唯一分水岭**：
> 不写 `declare -A`，`user=( [changsha]=wo )` 会被当成**普通数组**，`[changsha]` 会被解析成算术表达式（`changsha` 未定义 → 当 0），结果全乱。
> **记住：关联数组第一句永远是 `declare -A 名字`。**

#### 1.4.2 取值与遍历（课件实操）

```bash
declare -A user
user=([changsha]=wo [beijing]=ni [shangsha]=ta)

echo ${user[changsha]}      # wo
echo ${user[beijing]}       # ni
```

**课件里的实际输出**：

```text
[root@r810-201 ~]# declare -A user
[root@r810-201 ~]# user=([changsha]=wo [beijing]=ni [shangsha]=ta)
[root@r810-201 ~]# echo ${user[changsha]}
wo
[root@r810-201 ~]# echo ${user[beijing]}
ni
```

**遍历（课件给的两种）**：

```bash
# 值遍历：for i in ${user[@]}
[root@r810-201 ~]# for i in ${user[@]};do echo $i;done
ta
ni
wo

# 索引遍历：for i in ${!user[@]}
[root@r810-201 ~]# for i in ${!user[@]};do echo $i;done
shangsha
beijing
changsha
```

> **★ 三个必须注意的点**：
> ① **关联数组的"下标"是字符串**（就是我们自己定义的键名），所以 `${!user[@]}` 输出的是 `shangsha beijing changsha`。
> ② **输出顺序是乱序的！** 不是定义时的 `changsha beijing shangsha`，而是 `shangsha beijing changsha` —— **因为关联数组内部是哈希表，遍历顺序不保证**。**需要有序输出必须自己再 `sort`**。
> ③ **课件里有一行写错了**：`for i in ${#user[@]}` 输出 `3`，那是**长度**，不是遍历下标 —— 正确写法是 `${!user[@]}`（`#` 是长度、`!` 才是下标，和第二期普通数组的规则一样）。

**打印"键 → 值"的正确姿势（★ 生产里最常用）**：

```bash
for k in "${!user[@]}";do
    echo "$k -> ${user[$k]}"
done | sort

# 期望输出（排序后）：
# beijing -> ni
# changsha -> wo
# shangsha -> ta
```

### 1.5 数组两种类型的选型速查

| 需求 | 用哪个 | 写法 |
| --- | --- | --- |
| 一批 IP / 用户 / 服务名，只关心"都有谁" | **普通数组** | `arr=(a b c)` + `"${arr[@]}"` |
| 要按下标取第几个 | **普通数组** | `${arr[2]}` |
| 要统计"每个值出现几次" | **关联数组** | `declare -A cnt` + `cnt[$k]++` |
| 要做"名字 → 值"的映射（如服务名→端口） | **关联数组** | `declare -A port` + `port[nginx]=80` |

---

## 二、Shell 常用组合命令（课件第二章）

> **课件原文**：`grep，awk，sort，uniq，wc......`

这一章是**日志分析和数据统计**的看家本领。核心思想是：**每个命令只做一件小事，用管道把它们串起来。**

### 2.1 grep —— 过滤（挑出符合条件的行）

> **课件原文**：`ps aux|grep nginx|grep -v grep`

```bash
ps aux | grep nginx | grep -v grep     # 找 nginx 进程（排除 grep 自己）
```

**高频用法（★ 背这几个）**：

```bash
grep "ERROR" app.log                    # 找出含 ERROR 的行
grep -i "error" app.log                 # -i 忽略大小写
grep -v "DEBUG" app.log                 # -v 反选（不含 DEBUG 的行）
grep -c "ERROR" app.log                 # -c 只输出匹配的行数
grep -n "ERROR" app.log                 # -n 显示行号
grep -r "listen" /etc/nginx/            # -r 递归搜目录
grep -E "ERROR|WARN" app.log            # -E 用扩展正则（或的关系）
grep -q "ok" file && echo "找到了"      # -q 静默，只用退出码（脚本里最常用）
```

### 2.2 awk —— 按列处理（挑列、统计、格式化）

> **课件原文**：`ss -ant | awk '{print $1}'`、`more /etc/passwd | awk -F: '{print $1}'`

```bash
ss -ant | awk '{print $1}'                    # 取第一列（连接状态）
more /etc/passwd | awk -F: '{print $1}'       # -F 指定分隔符，取第一列（用户名）
```

**★ awk 最核心的四个概念**：

| 概念 | 说明 | 例子 |
| --- | --- | --- |
| `$1 $2 ...` | 第 1、2…列（默认按**空白**分列） | `awk '{print $1}'` |
| `$0` | 整行 | `awk '{print $0}'` |
| `-F` | 指定分隔符 | `awk -F: '{print $1}'` |
| `NR` | 当前行号（`NR>1` 就跳过表头） | `awk 'NR>1 {print $1}'` |

**高频用法**：

```bash
awk 'NR>1 {print $1}'            # 跳过表头，只取第一列
awk '{print $1, $9}' access.log  # 同时打印两列（用空格隔开）
awk '$9 == 404 {print $1}'       # 只打印状态码是 404 的行的 IP
awk '{sum+=$10} END {print sum}' # 求和（END 在所有行处理完后执行）
```

### 2.3 sort —— 排序

> **课件原文**：`more access.log | awk '{print $1}' | sort`、`| sort -r  #反序（倒序）`

```bash
... | sort          # 升序（默认按字典序）
... | sort -r       # 反序（倒序）
... | sort -n       # 按数值排序（★ 不加 -n 时 "10" 会排在 "9" 前面！）
... | sort -rn      # 按数值倒序（最常用：从大到小）
... | sort -k2      # 按第 2 列排序
... | sort -u       # 排序并去重
```

> **★ `sort -n` 是最容易被忽略的一个**：
> ```bash
> printf "9\n10\n2\n" | sort        # 输出 10 2 9   ← 字典序，"10" 比 "2" 小
> printf "9\n10\n2\n" | sort -n     # 输出 2 9 10  ← 数值序，正确
> ```
> **凡是"数字排序"，一律加 `-n`。**

### 2.4 uniq —— 去重（★ 必须配合 sort）

> **课件原文**：`more access.log | awk '{print $1}' | sort | uniq`、`| sort | uniq -c`

```bash
... | sort | uniq        # 去重（只保留每种一行）
... | sort | uniq -c     # ★ 去重并计数（每行前面加出现次数）
... | sort | uniq -d     # 只显示重复的行
... | sort | uniq -u     # 只显示不重复的行
```

> **★ `uniq` 只能去掉"相邻"的重复行**，所以**前面必须 `sort`**。
> **这是新手最常犯的错**：直接 `uniq` 而不 `sort`，结果去不干净。
> **标准写法**：`sort | uniq -c`（先排序，再统计）—— 这个组合出场率极高，背下来。

### 2.5 wc —— 统计

> **课件原文**：`more access.log | awk '{print $1}' | sort | uniq -c | wc -l`

```bash
wc -l        # 统计行数（最常用）
wc -w        # 统计单词数
wc -c        # 统计字节数
```

> **注意 `wc -l` 到底在数什么**：在 `sort | uniq -c | wc -l` 这一串里，它数的是**"一共有几个不同的 IP"**（去重之后的行数），不是访问次数。**想清楚每一段管道在"数什么"，是能不能写对统计命令的关键。**

### 2.6 cut —— 按分隔符截列

> **课件原文**：`cut -d: -f1 /etc/passwd`

```bash
cut -d: -f1 /etc/passwd        # -d 指定分隔符，-f 指定第几列（这里取用户名）
cut -d: -f1,3 /etc/passwd      # 取第 1 和第 3 列
cut -c1-5 file                 # 按字符位置取（前 5 个字符）
```

> **`cut` 和 `awk` 的区别**：`cut` 只能按**单个字符**分隔符取列，简单快速；`awk` 更强大（能按多个连续空格分列、能计算、能条件判断）。
> **经验**：分隔符规整（如冒号、逗号）用 `cut`；以空格/多空格分列（如命令输出）**用 `awk`**（`cut` 处理不了"连续多个空格"）。

### 2.7 实战：日志分析流水线（★ 把六个命令串起来）

**先造一份测试日志**（方便你照着练）：

```bash
cat > /data/shell-lab4/access.log <<'EOF'
192.168.1.10 - - [07/May/2026:10:00:01] "GET /index.html HTTP/1.1" 200 1024
192.168.1.11 - - [07/May/2026:10:00:02] "GET /login HTTP/1.1" 200 2048
192.168.1.10 - - [07/May/2026:10:00:03] "POST /api HTTP/1.1" 500 128
192.168.1.12 - - [07/May/2026:10:00:04] "GET /index.html HTTP/1.1" 200 1024
192.168.1.10 - - [07/May/2026:10:00:05] "GET /admin HTTP/1.1" 403 64
10.0.0.9     - - [07/May/2026:10:00:06] "GET /index.html HTTP/1.1" 200 1024
EOF
```

**★ 六个必备的日志分析命令（背下来，面试和日常都用）**：

```bash
cd /data/shell-lab4

# ① 访问次数最多的 IP（Top10）
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# ② 状态码分布
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# ③ 只看 5xx 错误，并列出是哪些 IP
awk '$9 ~ /^5/ {print $1, $9}' access.log | sort | uniq -c | sort -rn

# ④ 一共有多少个不同的 IP（先 sort|uniq 去重，再 wc -l 数行）
awk '{print $1}' access.log | sort -u | wc -l

# ⑤ 总请求数（= 日志行数）
wc -l access.log

# ⑥ 统计每个 IP "访问了几次"，并只挑出超过 2 次的（异常访问告警）
awk '{print $1}' access.log | sort | uniq -c | awk '$1 > 2 {print "【告警】" $2 " 访问 " $1 " 次"}'
```

**预期输出（对照着看）**：

```text
① 输出：
      3 192.168.1.10
      1 10.0.0.9
      1 192.168.1.11
      1 192.168.1.12

② 输出：
      4 200
      1 403
      1 500

⑥ 输出：
【告警】192.168.1.10 访问 3 次
```

> **★ 记住这个"万能管道"**：
> ```bash
> awk '{print $N}' 文件 | sort | uniq -c | sort -rn | head
> ```
> **统计任何"某一列的出现次数排行"都是这个套路** —— 换成 `$9` 就是状态码排行，换成别的列就是别的排行。
>
> **★ 每一段的职责（理解了这个，就不会写错）**：
> ```text
> awk '{print $N}'   → 挑出我要的那一列
> sort               → 排序（为 uniq 做准备，uniq 只能去相邻重复）
> uniq -c            → 去重并计数，输出格式是 "次数 值"
> sort -rn           → 按【次数】从大到小排（-n 数值序，-r 倒序）
> head               → 只看前几行
> ```

---

## 三、Shell 脚本排错（课件第三章）

> **课件原文（两步法 + 一个小技巧）**：
> **第一步：先检查语法　`bash -n tcp_count.sh`**
> **第二步：追踪执行　`bash -x tcp_count.sh`**
> **`${var}` 分段定位**

### 3.1 第一步：语法检查 `bash -n`（★ 写完脚本的第一件事）

```bash
bash -n tcp_count.sh
```

**两个结果**：

```text
没有任何输出          → 语法 OK（-n 只检查语法，不执行）
报错并指出行号        → 语法有问题，例如：
  tcp_count.sh: line 8: syntax error near unexpected token `done'
  tcp_count.sh: line 8: `done'
```

> **★ `bash -n` 好在哪里**：它**不执行脚本**，所以对"备份脚本、删文件脚本"这类**执行起来有风险**的脚本，先 `-n` 检查一遍是最安全的做法。
> **它查不出什么**：`-n` 只管**语法**（比如少了 `done`、少了 `fi`），**不管逻辑**（比如变量名拼错、路径不存在）。逻辑要靠下面两步。

### 3.2 第二步：追踪执行 `bash -x`（★ 定位逻辑错误的主力）

```bash
bash -x tcp_count.sh
```

**它会打印每一条命令，并在行首加 `+` 号标记"这是执行到的命令"**，输出形如：

```text
+ declare -A status
+ ss -ant
+ awk 'NR>1 {print $1}'
+ Connection_Type=LISTEN
ESTAB
TIME-WAIT
+ for a in ${Connection_Type}
+ let 'status[LISTEN]++'
+ let 'status[ESTAB]++'
+ let 'status[TIME-WAIT]++'
+ for b in ${!status[@]}
+ echo 'TIME-WAIT: 1'
TIME-WAIT: 1
...
```

> **★ 怎么用 `bash -x` 排错（关键在第 3 点）**：
> ```text
> ① 看"有没有执行到你觉得该执行的那一行" → 判断是不是 if 条件判错了
> ② 看"变量的实际值" → 比如会发现 $Connection_Type 只有一个词，
>    而不是预期的三个词（说明命令替换那里出问题了）
> ③ 看"执行的顺序" → 循环轮次对不对、有没有提前退出
>
> ★ 技巧：输出太长时，只看中间一段
>    bash -x 脚本.sh 2>&1 | sed -n '20,40p'     # 只看第 20~40 行
>    bash -x 脚本.sh 2>&1 | grep '变量名'         # 只看和某个变量有关的
> ```

**关于 `2>&1`**：`bash -x` 的追踪信息是**打到标准错误（stderr）**的，所以要 `2>&1` 才能被管道接住。

### 3.3 课件提到的 `${var}` 分段定位（★ 很实用的小技巧）

> **课件原文**：`${var} 分段定位`

**是什么意思**：在脚本里**临时插入 echo，把可疑变量打出来看**，快速缩小问题范围。

```bash
# 原脚本（出问题了）
for a in ${Connection_Type}
do
   let status[$a]++
done

# 加两行 echo 定位（★ 这就是"分段定位"）
echo "DEBUG: Connection_Type=[${Connection_Type}]"      # 值对不对？
echo "DEBUG: 元素个数=[$(echo ${Connection_Type} | wc -w)]"  # 有几个？
for a in ${Connection_Type}
do
   echo "DEBUG: 本轮 a=[$a]"                             # 每一轮拿到什么？
   let status[$a]++
done
```

> **★ 为什么要用 `${var}` 加花括号**：
> ```bash
> var=abc
> echo "DEBUG: $var123"      # ❌ 会被当成变量 $var123（不存在），输出 DEBUG: 
> echo "DEBUG: ${var}123"    # ✅ 输出 DEBUG: abc123
> ```
> **变量后面紧跟数字/字母/下划线时，必须用 `${}` 把变量名框起来** —— 这也是第二期备份脚本里 `${Date_Time}.tar.gz` 那一条的同一个知识点。

**排查顺序总结（★ 建议按这个顺序排，效率最高）**：

```text
1) bash -n 脚本.sh          语法有没有问题？（先排除低级错误）
2) 插 echo 看关键变量值       数据对不对？（最常见的问题都是"变量不是你以为的值"）
3) bash -x 脚本.sh          执行到哪一步了、顺序对不对？（逻辑问题）
4) 单独把可疑的那条命令拿出来在命令行跑一遍    （排除是"命令本身"还是"脚本环境"的问题）
```

---

## 四、综合巡检脚本（课件第四章，★ 三期的收口实战）

> **课件这一节先列了"巡检要查什么"，再给了一个完整脚本。** 我们把它整理成"清单 → 脚本 → 优化版"三步。

### 4.1 巡检清单（课件原文，★ 这就是"运维日常要看什么"）

| 分类 | 检查项 | 命令（课件原话） |
| --- | --- | --- |
| **主机** | 取主机名 | `echo $(hostname)` |
| | 取 IP 地址 | `ip -4 add \| awk '/inet / && $2 !~ "^127" {print $2}'` |
| | 取日期 | `date` |
| **系统** | 操作系统版本 | `more /etc/redhat-release` |
| | 查内核 | `uname -r` |
| **资源** | CPU 压力 | `uptime` |
| | 查看内存 | `free -h` |
| | 磁盘使用情况 | `df -h` |
| **安全** | 查看 selinux | `getenforce` |
| | 查看防火墙 | `systemctl is-active firewalld` |
| **网络** | 查看监听状态 | `ss -lntup \| awk '{print $2,$5}' \| column -t` |
| | TCP 状态统计 | `ss -ant \| awk 'NR >1 {print $1}' \| sort \| uniq -c` |
| | 登录用户 | `w` |
| | 系统运行时间 | `uptime -p` |
| **进程** | 占 CPU 多的进程 | `ps aux --sort=-%cpu \| head -3` |
| | 占内存多的进程 | `ps aux --sort=-%mem \| head -3` |

> **★ 注意课件里的一处笔误**：`ip -4 add` 应该是 **`ip -4 addr`**（`addr` 是 address 的缩写，`add` 是增加的意思）—— 不过在 `ip` 命令里 `addr` 可以简写成 `a`，所以 `ip -4 a` 也能用。
> **`more` 也可以换成 `cat`**：它们的区别是 `more` 会分页（适合人看），`cat` 一次性输出（适合脚本）。

### 4.2 课件原版脚本

```bash
#!/bin/bash
#-----------------------
# Linux 巡检脚本
#----------------------
echo "======================"
echo "hostname: $(hostname)"
echo "ip      : $(ip -4 add |awk '/inet / && $2 !~ "^127" {print $2}')"
echo "Date    : $(date)"
echo "======================"

#--------------------------
#系统信息
#--------------------------
echo
echo ">>>>>>>>>>>系统版本<<<<<<<<<<"
more /etc/redhat-release
echo
echo ">>>>>>>>>>>内核版本<<<<<<<<<<"
uname -r
echo
echo ">>>>>>>>>>>CPU负载<<<<<<<<<<"
uptime
echo
echo ">>>>>>>>>>>内存负载<<<<<<<<<<"
free -h
echo
echo ">>>>>>>>>>>磁盘负载<<<<<<<<<<"
df -h
echo
echo ">>>>>>>>>>>SELINUX<<<<<<<<<<"
getenforce
echo
echo ">>>>>占CPU的前2个进程><<<<<<<<"
ps aux --sort=-%cpu|head -3
echo
echo ">>>>>占内存的前2个进程<<<<<<<<"
ps aux --sort=-%mem|head -3
echo
echo "======================"
echo "巡检完成！"
echo "======================"
```

> **★ 课件原文把两处写错了，我这里已修正**：
> ① `ip -4 add` → 应为 `ip -4 addr`；
> ② `占NC的前2个进程` → 应为 `占内存的前2个进程`。

**跑起来看看**：

```bash
cat > linux_health_check.sh <<'SCRIPT'
#!/bin/bash
echo "======================"
echo "hostname: $(hostname)"
echo "ip      : $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}')"
echo "Date    : $(date)"
echo "======================"
echo
echo ">>>>>>>>>>>系统版本<<<<<<<<<<"
cat /etc/redhat-release 2>/dev/null || cat /etc/os-release | head -2
echo
echo ">>>>>>>>>>>内核版本<<<<<<<<<<"
uname -r
echo
echo ">>>>>>>>>>>CPU负载<<<<<<<<<<"
uptime
echo
echo ">>>>>>>>>>>内存负载<<<<<<<<<<"
free -h
echo
echo ">>>>>>>>>>>磁盘负载<<<<<<<<<<"
df -h
echo
echo ">>>>>>>>>>>SELINUX<<<<<<<<<<"
getenforce
echo
echo ">>>>>占CPU的前2个进程<<<<<<<<"
ps aux --sort=-%cpu | head -3
echo
echo ">>>>>占内存的前2个进程<<<<<<<<"
ps aux --sort=-%mem | head -3
echo
echo "======================"
echo "巡检完成！"
echo "======================"
SCRIPT

bash -n linux_health_check.sh && bash linux_health_check.sh
```

### 4.3 优化版巡检脚本（★ 把三期知识全用上）

课件版脚本能跑，但只能"人看着屏幕读"。**生产上需要的是"能自动判断 + 能留痕"**，所以下面这个版本用到了：

- **第二期**：`if` 判阈值
- **第三期**：`for` 循环遍历检查项、**函数**封装
- **第四期**：**数组**管理"要检查的挂载点和服务名"、组合命令做统计

```bash
cat > health_check_v2.sh <<'SCRIPT'
#!/bin/bash
#=========================================
# Linux 巡检脚本（优化版）
# 用法: bash health_check_v2.sh
# 建议挂 cron: 0 8 * * * /shell/health_check_v2.sh >> /var/log/health_check.log 2>&1
#=========================================

set -uo pipefail
WARN=0                                  # 告警计数器

info()  { echo "[信息] $*"; }
warn()  { echo "[告警] $*"; WARN=$((WARN+1)); }
ok()    { echo "[正常] $*"; }
title() { echo; echo "=============== $* ==============="; }

# ---------- 1. 基本信息 ----------
title "基本信息"
info "主机名   : $(hostname)"
info "IP 地址  : $(ip -4 addr | awk '/inet / && $2 !~ "^127" {print $2}' | paste -sd, -)"
info "巡检时间 : $(date '+%F %T')"
info "运行时长 : $(uptime -p)"
info "系统版本 : $(cat /etc/redhat-release 2>/dev/null || head -1 /etc/os-release)"
info "内核版本 : $(uname -r)"

# ---------- 2. 负载检查（CPU） ----------
title "CPU 与负载"
info "核数 : $(nproc)"
info "负载 : $(uptime | awk -F'load average:' '{print $2}')"
info "占 CPU 前 3 的进程:"
ps aux --sort=-%cpu | head -4

# ---------- 3. 内存检查 ----------
title "内存"
free -h
MEM_USED=$(free | awk 'NR==2 {printf "%d", $3/$2*100}')
if [ "$MEM_USED" -gt 90 ]; then
    warn "内存使用率 ${MEM_USED}%，超过 90%"
else
    ok "内存使用率 ${MEM_USED}%"
fi

# ---------- 4. 磁盘检查（用数组管理挂载点） ----------
title "磁盘"
df -hT | grep -v tmpfs

MOUNTS=(/ /var /data)                    # ★ 数组：要重点检查的挂载点
for m in "${MOUNTS[@]}";do
    if [ -d "$m" ]; then
        use=$(df -P "$m" | awk 'NR==2 {gsub("%",""); print $5}')
        if [ -n "$use" ] && [ "$use" -gt 80 ]; then
            warn "$m 使用率 ${use}%，超过 80%，请清理"
        else
            ok "$m 使用率 ${use}%"
        fi
    fi
done

inode_use=$(df -Pi / | awk 'NR==2 {gsub("%",""); print $5}')
if [ "$inode_use" -gt 80 ]; then
    warn "/ inode 使用率 ${inode_use}%"
else
    ok "/ inode 使用率 ${inode_use}%"
fi

# ---------- 5. 服务检查（数组 + 函数） ----------
title "服务状态"
check_service() {
    local svc=$1
    if systemctl is-active --quiet "$svc"; then
        ok "服务 $svc 正在运行"
    else
        warn "服务 $svc 未运行！"
    fi
}

SERVICES=(sshd firewalld)                # ★ 数组：按自己环境增减
for s in "${SERVICES[@]}";do
    check_service "$s"
done

# ---------- 6. 网络与安全 ----------
title "网络与安全"
info "监听端口:"
ss -lntup | awk 'NR>1 {print "   " $1, $5}' | head -10
info "TCP 连接状态统计:"
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn | sed 's/^/   /'
info "登录用户: $(w -h | wc -l) 个"
info "SELinux : $(getenforce)"
info "防火墙  : $(systemctl is-active firewalld 2>/dev/null || echo inactive)"

# ---------- 7. 汇总 ----------
title "巡检结果"
if [ "$WARN" -eq 0 ]; then
    ok "全部检查通过，没有发现异常"
else
    warn "共发现 $WARN 项异常，请及时处理"
fi
echo "巡检完成时间: $(date '+%F %T')"
exit 0
SCRIPT

bash -n health_check_v2.sh && bash health_check_v2.sh
```

**这个版本比课件版多出来的东西（★ 每一条都是"生产差异"）**：

| 增加的 | 为什么 |
| --- | --- |
| **函数**（`info`/`warn`/`ok`/`title`/`check_service`） | 主流程一眼看懂；输出格式统一；改格式只改一处 |
| **`WARN` 计数器 + 结尾汇总** | 从"让人读一屏输出"变成"**机器可判断**"：结尾一句"共发现 N 项异常" |
| **阈值判断**（内存 90%、磁盘 80%、inode 80%） | **巡检的核心不是"打印"，是"判断"**。没有阈值的巡检等于没有巡检 |
| **数组管理挂载点和服务名** | 加一个挂载点/服务，只改数组那一行，代码不变 |
| **`df -hT` 加 `grep -v tmpfs`** | `-T` 显示文件系统类型；tmpfs 是内存盘不用管，过滤掉避免误报 |
| **检查 inode** | 空间没满但 inode 满了照样写不进文件（只有 `df -i` 看得到） |
| **`set -uo pipefail`** | 变量写错立刻报错、管道任一环失败也算失败（不写 `-e` 是为了让巡检尽量跑完全部项） |

> **★ 挂到定时任务（这一步才是"运维自动化"）**：
> ```bash
> crontab -e
> 0 8 * * * /data/shell-lab4/health_check_v2.sh >> /var/log/health_check.log 2>&1
>
> # 检验：手动跑一次 crontab 里的那条命令，确认能正常输出
> /data/shell-lab4/health_check_v2.sh >> /var/log/health_check.log 2>&1
> tail -30 /var/log/health_check.log
> ```
>
> **★ cron 的两个经典坑**（前面提过，这里再强调一次）：
> ① **cron 的环境变量很干净**，脚本里**必须用绝对路径**；
> ② **cron 不会把输出显示给你**，所以**必须自己重定向**（`>> 日志 2>&1`），否则报错你根本看不到。

---

## 五、本篇闭环自检清单

| 检查项 | 命令 | 通过标准 |
| --- | --- | --- |
| bash 版本够 | `bash --version` | 4.0+（关联数组需要） |
| 普通数组会定义取值 | `arr=(a b c); echo ${arr[1]}` | 输出 b |
| 四个符号分得清 | `${arr[@]}` `${#arr[@]}` `${!arr[@]}` `${arr[0]}` | 值 / 长度 / 下标 / 单个值 |
| 会追加和删除 | `arr+=(d)`、`unset arr[1]` | 能说清"稀疏数组"的下标 |
| 三种遍历都会 | 值遍历 / 索引遍历 / C 风格 | 都跑过 |
| 知道数组要加双引号 | `"${arr[@]}"` | 能说出不加会怎样 |
| MySQL 备份脚本能跑 | `mysql_backup.sh` | 每库一个文件，且非空 |
| 关联数组会声明 | `declare -A x` | 知道不声明会怎样 |
| 关联数组会遍历 | `${!x[@]}` + `${x[$k]}` | 能打印"键 -> 值" |
| 知道关联数组是乱序的 | 遍历两次对比 | 能说出要排序得自己 `sort` |
| TCP 状态统计能跑 | `tcp_count.sh` | 输出各状态及条数 |
| 六个组合命令都会用 | grep/awk/sort/uniq/wc/cut | 各写出一个例子 |
| 会写"万能统计管道" | awk + sort + uniq -c + sort -rn + head | 能统计 IP Top10 |
| 知道 `sort -n` 的必要性 | `printf "9\n10\n" \| sort -n` | 能说出不加会错 |
| 知道 `uniq` 要先 `sort` | — | 能说出为什么 |
| 会用 `bash -n` | `bash -n 脚本.sh` | 无输出即语法 OK |
| 会用 `bash -x` | `bash -x 脚本.sh` | 能看到 `+` 开头的追踪 |
| 会"分段定位" | 插 `echo "DEBUG: ${var}"` | 能说出为什么要加 `{}` |
| 课件版巡检脚本能跑 | `linux_health_check.sh` | 输出各段信息 |
| 优化版巡检脚本能跑 | `health_check_v2.sh` | 末尾有"共发现 N 项异常" |
| 会挂定时任务 | `crontab -e` | 能说出 cron 的两个坑 |

**本篇要能独立写出来的 4 个脚本**：

```text
1. mysql_backup.sh       数组 + for + 备份验证
2. tcp_count.sh          关联数组 + 计数
3. log_top.sh            组合命令统计 IP / 状态码 Top10
4. health_check_v2.sh    函数 + 数组 + 阈值判断的综合巡检
```

---

## 六、常见坑速查

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `declare: -A: invalid option` | bash 版本低于 4.0 | `bash --version` 确认；老系统改用普通数组 + 其他方式绕开 |
| 关联数组取值全是空的 | **忘了 `declare -A`** | 定义前先 `declare -A 名字` |
| 数组元素含空格被拆开 | `for f in ${arr[@]}` **没加引号** | 写成 `for f in "${arr[@]}"` |
| 遍历关联数组顺序乱了 | 关联数组是哈希结构，**顺序不保证** | 用 `for k in "${!arr[@]}"; do ...; done` 再接 `sort` |
| 想遍历下标却用了 `${#arr[@]}` | `#` 是长度、`!` 才是下标 | 改成 `${!arr[@]}` |
| 删了中间元素后遍历漏掉 | 数组变"稀疏"，下标不连续 | 用 `"${!arr[@]}"` 按下标遍历 |
| `sort` 出来的数字顺序不对 | 没加 `-n`（按字典序排了） | 加 `-n`（倒序 `-rn`） |
| `uniq` 去不干净 | **没先 `sort`**（uniq 只能去相邻重复） | 固定写成 `sort` 再接 `uniq` |
| `cut` 切不出想要的列 | 分隔符是"多个连续空格" | 改用 `awk`（`cut` 不支持多空格） |
| `awk` 把表头也统计进去了 | 没跳过第一行 | 用 `awk 'NR>1 {...}'` |
| 脚本语法没问题但结果不对 | 逻辑错 / 变量值不对 | `bash -x` 追踪 + 插 `echo "DEBUG: ${var}"` |
| `$var123` 输出为空 | 变量后面紧跟字符 | 用 `${var}123` |
| 手动跑正常，cron 里失败 | **cron 环境变量干净、不读 `.bashrc`** | 脚本里全用绝对路径；必须自己重定向输出 |
| 巡检脚本跑完才知道有问题 | 没有阈值判断 | 加 `if [ "$use" -gt 80 ]` 之类的判断和结尾汇总 |
| 磁盘还有空间却写不进文件 | **inode 用尽** | 巡检里加 `df -i` |

---

## 七、自测问题（闭卷）

- [ ] 普通数组和关联数组的区别是什么？各自适合什么场景？
- [ ] 关联数组为什么必须先 `declare -A`？
- [ ] `${arr[@]}`、`${#arr[@]}`、`${!arr[@]}`、`${arr[1]}` 分别是什么？
- [ ] 遍历数组为什么推荐加双引号？不加会怎样？
- [ ] 数组删了中间一个元素后，`${!arr[@]}` 会输出什么？
- [ ] 关联数组遍历的顺序是定义顺序吗？为什么？
- [ ] 用关联数组计数为什么不用先初始化？
- [ ] TCP 状态统计的命令链路是怎样的？每一段在做什么？
- [ ] `grep` 的 `-v`、`-c`、`-n`、`-q`、`-E` 分别是什么？
- [ ] `awk` 里 `$1`、`$0`、`-F`、`NR` 分别是什么？
- [ ] `sort` 为什么要加 `-n`？不加会怎样？
- [ ] `uniq` 为什么必须先 `sort`？
- [ ] `cut` 和 `awk` 什么时候用哪个？
- [ ] 统计"访问次数最多的 IP"的完整管道怎么写？
- [ ] `bash -n` 和 `bash -x` 分别解决什么问题？哪个查不出逻辑错误？
- [ ] `bash -x` 的输出为什么有时要 `2>&1`？
- [ ] 什么是"分段定位"？为什么要用 `${var}` 而不是 `$var`？
- [ ] 一份"合格"的巡检脚本应该包含哪几类检查？
- [ ] 巡检脚本里为什么必须有"阈值判断"和"结尾汇总"？
- [ ] cron 里跑脚本的两个经典坑是什么？

---

## 八、四期总回顾

```text
第二期  逻辑运算与条件判断
        ├─ ! -a -o 与退出码 $?
        ├─ if 单分支 / 双分支 / 多分支（elif）
        ├─ read 的 -p / -t / -s
        └─ case 语法与 DOS 功能菜单

第三期  循环与函数
        ├─ for（5 种清单写法 + C 风格）
        ├─ while（test / [ ] / [[ ]] 三种判断）
        ├─ until（等就绪场景 + 超时保护）
        ├─ break / continue / exit
        └─ 函数（三种定义、参数、local、return vs echo）

第四期  数组、组合命令与巡检实战
        ├─ 普通数组（定义 / 取值 / 长度 / 下标 / 遍历）
        ├─ 关联数组（declare -A、计数统计）
        ├─ 组合命令 grep / awk / sort / uniq / wc / cut
        ├─ 脚本排错（bash -n、bash -x、分段定位）
        └─ 综合巡检脚本（课件版 + 优化版）
```

**一条能串起三期的练习链（建议按顺序做一遍）**：

```text
1.  check_root.sh        判断是不是 root 在执行          （第二期）
2.  score.sh             用 if/elif 做成绩分段            （第二期）
3.  menu.sh              用 case 做功能菜单               （第二期）
4.  ping_net.sh          for 循环批量 ping                （第三期）
5.  guard_nginx.sh       while 守护进程                   （第三期）
6.  until_wait.sh        等端口就绪（带超时保护）          （第三期）
7.  ping_func.sh         把 ping 逻辑封成函数              （第三期）
8.  mysql_backup.sh      用数组批量备份                    （第四期）
9.  tcp_count.sh         用关联数组统计 TCP 状态            （第四期）
10. log_top.sh           用组合命令统计日志 Top10          （第四期）
11. health_check_v2.sh   函数 + 数组 + 阈值的综合巡检       （第四期）
12. 挂 crontab，让巡检每天自动跑                            （第四期）
```

> **下一步建议**：这四期学完，Shell 的"语法层"就齐了。接下来可以往三个方向走 ——
>
> ① **运维实战方向**：把巡检、备份、日志清理这些脚本真正挂到机器上跑起来（重点体会 cron 的两个坑）；
> ② **文本处理方向**：深入 `awk`（数组、函数、多文件处理）和 `sed`，这是日志分析的进阶利器（`notes1` 里已有 `stage1-15-awk使用指南.md`、`stage1-15-sed使用指南.md` 可以接着看）；
> ③ **自动化方向**：接 `notes2` 里的 **Ansible** 笔记 —— 它本质上是"把 Shell 脚本批量推到一堆机器上执行并统一管理"，学完 Shell 正好能理解它为什么存在。
