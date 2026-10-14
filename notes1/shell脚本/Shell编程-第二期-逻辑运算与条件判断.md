# Shell 编程 第二期：逻辑运算与条件判断
---

## 〇、开篇：先建实验目录

```bash
mkdir -p /data/shell-lab
cd /data/shell-lab
bash --version | head -1        # 确认是 bash
echo $SHELL                     # 确认当前 shell
```

> **建议**：每个案例建一个独立 `.sh` 文件，名字用案例含义命名（如 `check_root.sh`），学到后面翻回来还能直接跑。
> **每次写完先做两个动作**：
> ```bash
> bash -n 脚本.sh     # 1) 语法检查（-n = 只检查不执行）
> bash -x 脚本.sh     # 2) 追踪执行（每一条命令都打印出来）
> ```
> 这两条是 Shell 排错的标准起手式（第四期还会详细讲）。

---

## 一、Shell 逻辑运算（课件第一章）

### 1.1 三个逻辑运算符（课件原话）

| 运算符 | 说明（课件原话） |
| --- | --- |
| `!` | **非**，取反的意思 |
| `-o` | **或**，2 个以上的表达式，只要一个为真（true），则返回为真（true） |
| `-a` | **与**，2 个以上的表达式，都要为真（true），才返回为真（true） |

> **记忆**：`!` 是"反过来"，`-o` 是"or 或"，`-a` 是"and 与"。
> **注意**：`-a` / `-o` 是 **`test` / `[ ]` 内部**的逻辑运算符；`&&` / `||` 是**命令之间**的逻辑运算符。两者不能混用位置。

### 1.2 先搞懂 `$?`：逻辑运算的底层（课件实验）

课件的第一个实验就是证明 `!` 的作用：

```bash
false
echo $?          # 1
! false
echo $?          # 0
true
echo $?          # 0
! true
echo $?          # 1
```

**读法**：

```text
false 命令的退出码是 1（失败）
! false 把失败"取反"成成功 → 退出码 0
true  命令的退出码是 0（成功）
! true  把成功"取反"成失败 → 退出码 1
```

> **★ 这是理解一切条件判断的地基**：
> **Shell 判断"真假"，看的不是布尔值，而是前面那条命令的退出码 —— `0` 是真，非 `0` 是假。**
> 所以 `if` 后面跟的其实不是"表达式"，而是"**一条命令**"，它靠这条命令的退出码决定走不走 then 分支。

再看课件里 `-o` / `-a` 的实验：

```bash
[ $a -lt 30 -o $b -gt 100 ]
echo $?          # 0
[ $a -lt 30 -a $b -gt 100 ]
echo $?          # 1
```

> **注意**：这个实验里 `$a`、`$b` 是**没赋值的空变量**，空值在数值比较里会被当成 0，所以第一条（或）为真、第二条（与）为假。
> **生产中不要这样写**：变量一定要先赋值，并且加引号/加判断，否则会出 `integer expression expected` 这类报错。

**自己动手验证一遍（推荐）**：

```bash
a=20
b=200
[ $a -lt 30 -o $b -gt 100 ]; echo $?     # 20<30 成立 → 0（真）
[ $a -lt 30 -a $b -gt 100 ]; echo $?     # 200>100 不成立 → 1（假）
```

### 1.3 企业案例一：判断脚本是否由 root 执行（课件原话）

**需求**：只有 root 才能跑这个脚本，否则提示并退出。

```bash
#!/bin/bash
user=$(whoami)

if [ ! "${user}" = "root" ];then
    echo "非root用户,不能执行"
else
    dnf install -y nginx
fi
```

**逐句拆解**：

| 代码 | 含义 |
| --- | --- |
| `user=$(whoami)` | **命令替换**：把 `whoami` 的执行结果赋给变量 `user`（也可以直接 `user=$USER`） |
| `[ ! "${user}" = "root" ]` | 字符串比较：`!` 取反，`"${user}"` **必须加双引号**（见下方坑） |
| `;then` | `then` 和 `if` 写同一行时，前面**必须加分号** |
| `fi` | `if` 语句结束 |

**完整可跑版本（建议这样写）**：

```bash
#!/bin/bash
# check_root.sh
if [ "$(id -u)" -ne 0 ]; then
    echo "错误：请用 root 执行（或 sudo）"
    exit 1
fi
echo "以 root 身份运行，继续执行后续操作..."
```

```bash
chmod +x check_root.sh
bash check_root.sh            # 普通用户执行 → 提示错误
sudo bash check_root.sh       # root 执行 → 通过
```

> **★ 为什么推荐用 `$UID` / `id -u` 而不是 `whoami`**：
> `whoami` 是靠**用户名**判断的，而 `id -u` 直接看 **UID 是不是 0**，更可靠（用户名可以被改、sudo 环境下 `whoami` 也可能有歧义）。
> **这是本项目里最实用的一个模板，后面所有需要 root 的脚本都可以套。**

### 1.4 企业案例二：判断服务器的压力（课件原话）

```bash
#!/bin/bash
cpu=85
mem=65
if [ ${cpu} -gt 80 -o $mem -gt 80 ];then
   echo "服务器资源异常，触发告警！"
else
   echo "服务器压力正常"
fi
```

**检验**：

```bash
bash pressure.sh
# 期望输出：服务器资源异常，触发告警！   （因为 cpu=85 > 80）
```

**改成真实取值（生产写法）**：

```bash
#!/bin/bash
# pressure.sh —— 用真实 CPU / 内存数据判断
cpu=$(top -bn1 | awk -F'[ %]+' '/%Cpu/ {print int(100 - $8 - $10)}')   # 简化取值
mem=$(free | awk 'NR==2 {printf "%d", $3/$2*100}')

echo "当前 CPU 使用率: ${cpu}%   内存使用率: ${mem}%"

if [ "${cpu}" -gt 80 ] || [ "${mem}" -gt 80 ]; then
   echo "服务器资源异常，触发告警！"
else
   echo "服务器压力正常"
fi
```

> **从"写死 85/65"到"真实取值"是这一步的关键**：课件的 85/65 是为了让你专注在 `if` 逻辑上；实战里必须采真实数据。

### 1.5 企业案例三：判断服务（课件原话）

```bash
#!/bin/bash
ps -ef |grep nginx |grep -v grep >/dev/null
p1=$?
netstat -lntp |grep 80 >/dev/null
p2=$?
if [ $p1 -eq 0 -a $p2 -eq 0 ];then
    echo "服务正常"
else
    echo "服务异常"
fi
```

**这段里的三个技巧（★ 都是干货）**：

| 技巧 | 为什么 |
| --- | --- |
| `grep -v grep` | `ps -ef 管道符 grep nginx` 会把**grep 自己这条命令**也匹配到，必须把它排除掉 |
| `>/dev/null` | 我们只关心**退出码**，不关心输出，所以丢掉 |
| `p1=$?` **马上接住** | **`$?` 只保留上一条命令的退出码**，中间再执行任何命令都会覆盖它。所以必须紧接着赋值 |

> **★ `p1=$?` 这个"接住退出码"的写法，是 Shell 脚本里最容易出错的地方之一**。作业里经常写成：
> ```bash
> ps -ef | grep nginx > /dev/null
> netstat -lntp | grep 80 > /dev/null
> p1=$?          # ❌ 这里已经是 netstat 的退出码了！
> ```
> 结果判断逻辑全错，而且很难查。

**课件的"改良版"（并列命令，不需要接退出码）**：

```bash
#!/bin/bash
if ps -ef |grep nginx |grep -v grep >/dev/null && netstat -lntp |grep 80 >/dev/null;then
    echo "服务正常"
else
    echo "服务异常"
fi
```

> **这个写法更好**：`&&` 直接连接两条命令，`if` 判断的是整组的退出码，不用手工接 `$?`，也就不存在"接错"的问题。

**用 systemd 的推荐写法（生产更常用）**：

```bash
#!/bin/bash
if systemctl is-active --quiet nginx; then
    echo "服务正常（systemd 视角）"
else
    echo "服务异常（systemd 视角）"
fi
```

> **`systemctl is-active --quiet` 只返回退出码、不打印**，正好配合 `if` 使用。比 `ps + grep` 更准确（能识别"进程活着但服务是 failed 状态"的情况）。

### 1.6 企业规范：别用 `-o` / `-a`，用 `||` 和 `&&`（★ 课件重点）

> **课件原话**：**在实际企业里，一般我们不用 `-o` 跟 `-a`，`-o` 用 `||` 来代替，`-a` 用 `&&` 来代替。**

**三种写法对照（同一个"服务是否正常"的案例）**：

```bash
# 写法一：用 -a（课件说生产不推荐）
if [ $p1 -eq 0 -a $p2 -eq 0 ];then
    echo "服务正常"
else
    echo "服务异常"
fi

# 写法二：拆成两个 [ ]，用 && 连接（推荐）
if [ $p1 -eq 0 ] && [ $p2 -eq 0 ];then
    echo "服务正常"
else
    echo "服务异常"
fi

# 写法三：直接用命令的退出码 + &&（最推荐）
if ps -ef |grep nginx |grep -v grep >/dev/null && netstat -lntp |grep 80 >/dev/null;then
    echo "服务正常"
else
    echo "服务异常"
fi
```

**为什么生产不用 `-o` / `-a`（★ 面试可能问）**：

```text
1) 可读性差：条件一多，[ a -o b -a c ] 的优先级容易看错（-a 比 -o 优先）
2) 兼容性差：POSIX 已经把它标记为"过时"（obsolescent），
   在有些实现里行为不确定；bash 的 [[ ]] 里也不支持 -a/-o
3) 语义混淆：-a/-o 是 test 内部运算，&&/|| 是命令之间的运算，
   两者混在一起写（比如 [ a -o b ] && [ c ]）非常容易出 bug
4) 用 && / || 可以连接"命令"而不只是"表达式"，
   比如 [ -f a ] && [ -f b ] && [ -f c ]，还能短路求值
```

**注意一个易错点**：`[ ]` 里**不能**写 `&&` / `||`

```bash
[ $a -gt 1 && $b -lt 2 ]     # ❌ 语法错误
[ $a -gt 1 ] && [ $b -lt 2 ] # ✅ 正确：用两个 [ ]
[[ $a -gt 1 && $b -lt 2 ]]   # ✅ 正确：[[ ]] 里可以写
```

> **`[ ]` 和 `[[ ]]` 的区别（先说一句，第三期会细讲）**：
> - `[ ]` 是**内置命令**（等价于 `test`），里面对 `&&` 不友好，变量不加引号还容易出错；
> - `[[ ]]` 是 **bash 的关键字**，支持 `&&`、`||`、`=~`（正则）、`==` 通配符匹配，**更安全**。
> - 结论：**写 bash 脚本优先用 `[[ ]]`。**

---

## 二、IF 条件判断（课件第二章）

### 2.0 条件判断的本质和场景（课件原话）

> **课件原文**：在作备份的时候，只有文件/目录存在的情况下。只有在服务没有启动的时候，才执行启动命令。只有磁盘使用率超过某一个值，才会报警。只有我们用户输入了某个指令/选项，程序才执行相应的命令。包含上面的一些场景，我们就会用到条件判断语句。
>
> **条件判断的本质，就是让我们的 Shell 脚本有条件的执行一些命令。**

**把课件说的四类场景整理成一句话**：

```text
顺序执行 = 不管发生什么都往下跑（脚本最脆弱的地方）
条件判断 = 先"看一眼情况"，再决定跑不跑

四类典型场景：
  ① 前提检查 ：备份前先确认目录存在
  ② 幂等操作 ：服务没起才启动（起了就别重复启动）
  ③ 阈值告警 ：磁盘超 80% 才报警（没超就别吵人）
  ④ 交互分支 ：用户输入 y 才执行（防误操作）
```

> **整个 `if` 条件判断有单分支、双分支、多分支三种**（课件原话），下面逐个来。

### 2.1 单分支 if/then（课件 2.1）

> **课件原文**：当条件成立的时候，执行某段代码，当条件不成立的时候，不操作（什么都不做）。
> 它以 `if` 开头，中间有关键字 `then`，最后以 `fi` 结尾；**如果 `then` 与 `if` 在同一行，那么 `then` 前面要加 `;` 号**；如果 `fi` 与代码在同一行，前面要加 `;` 号。

**语法（两种写法，等价）**：

```bash
# 写法一：then 另起一行（可读性好，推荐）
if [ 条件判断表达式 ]
then
    执行程序代码
fi

# 写法二：then 跟 if 同一行（必须加分号）
if [ 条件判断表达式 ];then
    执行程序代码
fi

# 写法三：一行写完（简单判断时才这么写）
if [ -f /etc/passwd ];then echo "文件存在";fi
```

**最基础的一个例子（课件给的）**：

```bash
if [ -f /etc/passwd ];then echo "文件存在";fi
```

> **`-f` 是常用的文件测试符**，完整的常用清单见下表（★ 背下来）：

| 测试符 | 含义 |
| --- | --- |
| `-f 文件` | 存在且是**普通文件** |
| `-d 目录` | 存在且是**目录** |
| `-e 路径` | 存在（不管是文件还是目录） |
| `-r` / `-w` / `-x` | 可读 / 可写 / 可执行 |
| `-s 文件` | 存在且**非空**（size > 0） |
| `-z "字符串"` | 字符串**为空** |
| `-n "字符串"` | 字符串**非空** |
| `a -eq b` | 数值相等（`-ne` 不等、`-gt` 大于、`-lt` 小于、`-ge` `-le`） |
| `"a" = "b"` | 字符串相等（`!=` 不等） |

> **★ 一个必须记住的坑**：**字符串比较用 `=`，数值比较用 `-eq`**，两者不能混。
> ```bash
> [ "abc" = "abc" ]    # ✅ 字符串相等
> [ 10 -eq 10 ]        # ✅ 数值相等
> [ "10" = "10" ]      # ✅ 也能用，但语义是"字符串相等"
> [ "abc" -eq "abc" ]  # ❌ 报错 integer expression expected
> ```
#### 单分支企业案例一：打包时执行判断（课件原话）

```bash
#!/bin/bash
Date_Time=$(date +%F)
if [ -d /data/www ];then
    tar -zcf /opt/www_${Date_Time}.tar.gz /data/www
    echo "备份完成"
fi
```

**这里有两个知识点**：

| 知识点 | 说明 |
| --- | --- |
| `Date_Time=$(date +%F)` | 命令替换取日期，`%F` = `YYYY-MM-DD` |
| `/opt/www_${Date_Time}.tar.gz` | **变量后面紧跟字符时，必须用 `${}` 把变量名框起来**，否则 `$Date_Time.tar` 会被理解成变量名 `Date_Time.tar` |

**课件顺带提醒**：`date --help` —— **了解并掌握常见的时间格式**。

| 常用格式 | 输出示例 | 用途 |
| --- | --- | --- |
| `date +%F` | 2026-05-07 | 日期（做文件名最常用） |
| `date +%T` | 14:30:25 | 时间 |
| `date +%Y%m%d` | 20260507 | 紧凑日期 |
| `date +%Y-%m-%d-%T` | 2026-05-07-14:30:25 | 精确到秒（备份文件名推荐） |
| `date +%s` | 1778000000 | 时间戳（比较时间先后用） |
| `date -d "-1 day" +%F` | 昨天 | 算日期 |

**完整可跑版本（含检验）**：

```bash
mkdir -p /data/www
echo "test" > /data/www/index.html

cat > backup_www.sh <<'EOF'
#!/bin/bash
Date_Time=$(date +%Y-%m-%d-%T)
if [ -d /data/www ];then
    tar -zcf "/opt/www_${Date_Time}.tar.gz" /data/www
    echo "备份完成：/opt/www_${Date_Time}.tar.gz"
else
    echo "目录 /data/www 不存在，跳过备份"
fi
EOF

bash backup_www.sh
ls -lh /opt/www_*.tar.gz                    # 检验：能看到生成的包
tar -tzf /opt/www_*.tar.gz | head -3        # 检验：包内容能列出来（★ 备份必须验证）
```

> **★ "备份完成"不等于"备份可用"** —— 一定要 `tar -tzf`（或 `-tf`）验证一次，这是运维的基本素养，也是面试加分点。

#### 单分支企业案例二：服务重启判断（课件原话）

课件给了两版，**第二版是改良版**：

```bash
#!/bin/bash
if [ -f /etc/nginx/nginx.conf ];then
    nginx -s reload
    echo "Nginx 已重载"
fi
```

```bash
# 改良版（课件原话：改良）
#!/bin/bash
if [ -f /etc/nginx/nginx.conf ] && pgrep nginx >/dev/null;then
    nginx -s reload
    echo "Nginx 已重载"
fi
```

**两版差在哪（★ 这就是"想清楚前置条件"的思维）**：

| 版本 | 判断条件 | 问题 |
| --- | --- | --- |
| 第一版 | 只判断**配置文件存在** | 配置存在，但 nginx **没在跑** → `nginx -s reload` 会报错 `nginx: [error] open() "/run/nginx.pid" failed` |
| **改良版** | 配置文件存在 **并且** nginx 进程在跑 | 两个前提都满足才 reload，**不会误报错** |

> **`pgrep nginx >/dev/null`** 的用法和前面 `grep -v grep` 是一个道理：`pgrep` 只返回退出码，输出丢掉。
> **`pgrep` 比 `ps 管道符 grep` 更好**：它天然不会匹配到自己，也不用 `grep -v grep`。

**顺带记住几个 nginx 的操作**：

```bash
nginx -t                 # ★ 改配置后第一件事：校验语法
nginx -s reload          # 平滑重载（不断连接）
systemctl reload nginx   # 同上，systemd 视角
```

> **★ 完整的"安全 reload"应该是三步**：`nginx -t` 通过 → 再 reload → 失败就别动。所以更好的写法是：
> ```bash
> if [ -f /etc/nginx/nginx.conf ] && pgrep -x nginx >/dev/null; then
>     if nginx -t >/dev/null 2>&1; then
>         nginx -s reload && echo "Nginx 已重载"
>     else
>         echo "配置有语法错误，已中止 reload"
>     fi
> fi
> ```

#### 单分支企业案例三：二次确认（课件原话，★ 最实用）

```bash
#!/bin/bash
read -p "确认是否备份?(yes/no):" confirm
if [ "$confirm" != "yes" ] && [ "$confirm" != "y" ];then
    echo "操作取消"
fi
```

**这里引入了 `read`（课件专门讲了）**：

> **课件原文**：`read`，接受从键盘输入的字符（变量）。

```bash
read [option] [varname]
   -p ：提示信息
   -t ：读秒（超时时间）
   -s ：隐藏输入信息（输密码用）
```

**`read` 的三个常用姿势（课件都给了）**：

```bash
# 1) 基础用法
#!/bin/bash
read -p "plese input you name:" name
echo "name is $name"

# 2) 带超时（5 秒不输入就继续往下走）
read -t 5 -p "please input you age:" age
echo "age is $age"

# 3) 隐藏输入（密码/金额这类敏感信息）
read -s -p "please input you income:" shouru
echo
echo "shouru is $shouru"
```

> **`read -s` 后面为什么要单独写一个 `echo`**：因为 `-s` 把用户输入的回显关掉了，回车符也不会显示，所以下一行输出会**粘在同一行**，加一个 `echo` 就是为了**换行**，让界面整齐。这个细节很能体现你会不会写"给人用的"脚本。

**把二次确认写成完整可跑版本**：

```bash
cat > confirm_backup.sh <<'EOF'
#!/bin/bash
read -p "确认是否备份?(yes/no): " confirm

if [ "$confirm" != "yes" ] && [ "$confirm" != "y" ]; then
    echo "操作取消"
    exit 0
fi

echo "开始备份..."
Date_Time=$(date +%Y-%m-%d-%T)
tar -zcf "/opt/confirm_${Date_Time}.tar.gz" /data/www
echo "备份完成"
EOF

bash confirm_backup.sh        # 输入 no → 期望"操作取消"
bash confirm_backup.sh        # 输入 yes → 期望"备份完成"
```

**更好的写法：用 case 做确认（见第三章）**

```bash
#!/bin/bash
read -p "确认是否备份?(yes/no): " confirm
case "$confirm" in
    yes|y|Y|YES) echo "开始备份..." ;;
    *)           echo "操作取消"; exit 0 ;;
esac
```

> **为什么 case 更适合"多值匹配"**：`[ "$a" != "yes" ] && [ "$a" != "y" ]` 要写两遍很啰嗦；而且用户可能输入 `Y`、`YES`、`Yes`，用 `yes|y|Y|YES` 一行就覆盖了。
> **这就是第三章 case 存在的意义。**

### 2.2 双分支 if/then/else（课件 2.2）

> **课件原文**：增加了 `else` 一个关键词。

```bash
if [ 条件判断式 ];then
    当条件判断成立的时候，执行本代码
else
    当条件判断不成立的时候，执行本代码
fi
```

> **单分支和双分支的区别（一句话）**：单分支是"**满足就做事，不满足就什么都不做**"；双分支是"**满足做 A，不满足做 B**"。
> **实战里双分支更常用**，因为你总得告诉用户"为什么没做"。

**企业小案例：进程巡检（课件原话）**

```bash
#!/bin/bash
if pgrep -x nginx >/dev/null;then
    echo "nginx服务正在运行"
else
    echo "nginx服务未运行，尝试启动它"
    systemctl start nginx
fi
```

**逐句拆解**：

| 代码 | 含义 |
| --- | --- |
| `pgrep -x nginx` | **`-x` 表示精确匹配进程名**（不加 `-x` 时 `nginx` 会匹配到 `nginx-helper` 之类的）；输出丢掉，只取退出码 |
| `>/dev/null` | 丢弃标准输出（`pgrep` 会打印 PID，我们不需要） |
| `systemctl start nginx` | 服务没起就拉起来 |

**完整可跑 + 闭环检验**：

```bash
cat > check_nginx.sh <<'EOF'
#!/bin/bash
if pgrep -x nginx >/dev/null;then
    echo "nginx服务正在运行"
else
    echo "nginx服务未运行，尝试启动它"
    systemctl start nginx
fi
EOF

# 检验 A：服务本来就在跑
systemctl start nginx
bash check_nginx.sh            # 期望：nginx服务正在运行

# 检验 B：把服务停掉再跑
systemctl stop nginx
bash check_nginx.sh            # 期望：nginx服务未运行，尝试启动它
systemctl is-active nginx      # 期望：active（确认真的把服务拉起来了）
```

> **★ 检验 B 才是真检验**：只在"服务本来就在跑"的情况下测，你根本不知道 else 分支对不对。**每个分支都要构造条件跑一遍** —— 这是写脚本的好习惯，也是面试时"你怎么测试脚本"的标准答法。

**升级版：带重试和告警的巡检（更贴近生产）**

```bash
#!/bin/bash
if ! pgrep -x nginx >/dev/null; then
    echo "$(date '+%F %T') nginx 未运行，尝试启动" >> /var/log/check_nginx.log
    systemctl start nginx
    sleep 2
    if pgrep -x nginx >/dev/null; then
        echo "$(date '+%F %T') 启动成功" >> /var/log/check_nginx.log
    else
        echo "$(date '+%F %T') 启动失败，请人工介入" >> /var/log/check_nginx.log
    fi
fi
```

> **`if ! 命令` 的写法**：`!` 把退出码取反，读作"如果命令**失败了**"。比 `if [ $? -ne 0 ]` 更简洁（而且不会有"接错 $?" 的问题）。

### 2.3 多分支 if/then/elif（课件 2.3）

> **课件原文**：多分支 `(if/then/elif/then...)`

```bash
if [ 条件判断式1 ];then
    当条件判断式1为真时，执行本代码
elif [ 条件判断式2 ];then
    当条件判断式2为真时，执行本代码
else
    当所有条件不成立时，执行本代码
fi
```

> **执行逻辑（★ 关键）**：**从上往下逐个判断，命中一个就执行并跳出整个 if，后面的条件不再看。** 所以**顺序很重要** —— 范围大的条件要写在后面。

#### 案例 1：成绩判断（课件原话，★ 体现"顺序很重要"）

```bash
#!/bin/bash
read -p "input 0-100 num:" nums
if [ $nums -ge 90 ];then
    echo "优秀"
elif [ $nums -ge 80 ];then
    echo "良好"
elif [ $nums -ge 60 ];then
    echo "及格"
else
    echo "加油"
fi
```

**为什么这个顺序能工作（★ 想清楚这个就懂了 elif）**：

```text
输入 95 → 第一个条件 95>=90 成立 → 输出"优秀"，跳出（后面不看）
输入 85 → 95>=90 不成立 → 看 85>=80 成立 → 输出"良好"，跳出
输入 70 → 90/80 都不成立 → 70>=60 成立 → 输出"及格"
输入 50 → 都不成立 → else → "加油"

★ 假如把"及格"写在最前面（nums>=60）：
   输入 95 也会命中"及格"，因为 95>=60 成立且排在最前 → 结果全错
★ 所以：条件要按"从严格到宽松"排。
```

**完整检验（四个分支都要跑）**：

```bash
cat > score.sh <<'EOF'
#!/bin/bash
read -p "input 0-100 num:" nums
if [ "$nums" -ge 90 ];then
    echo "优秀"
elif [ "$nums" -ge 80 ];then
    echo "良好"
elif [ "$nums" -ge 60 ];then
    echo "及格"
else
    echo "加油"
fi
EOF

for n in 95 85 70 50; do echo "$n" | bash score.sh; done
# 期望依次输出：优秀、良好、及格、加油
```

> **`echo "$n" 管道符 bash score.sh`** 这个技巧很有用：`read` 是从**标准输入**读的，所以可以用管道喂数据，实现**自动化测试交互式脚本**（不用手工敲）。
>
> **注意课件原版写的是 `[ $nums -ge 90 ]`（不加引号）**，如果用户什么都不输入直接回车，会变成 `[ -ge 90 ]` → 报 `unary operator expected`。**加引号就安全了**（`[ "" -ge 90 ]` 会报 `integer expression expected`，但至少不会语法错误）。更严谨的写法是先判断是否为空：
> ```bash
> if [ -z "$nums" ]; then echo "没有输入"; exit 1; fi
> if ! [[ "$nums" =~ ^[0-9]+$ ]]; then echo "请输入数字"; exit 1; fi
> ```

#### 案例 2：判断文件类型（课件原话，★ 综合运用）

```bash
#!/bin/bash
read -t 20 -p "please input a filename:" file
if [ -z "$file" ];then
      echo "error"
elif [ ! -e "$file" ];then
      echo "$file is not exist!"
elif [ -f "$file" ];then
      echo "$file is file"
elif [ -d "$file" ];then
      echo "$file is directory"
else
  echo "$file is other file(device.)"
fi
```

**这段的判断顺序设计得很好（★ 值得学）**：

```text
第 1 层：-z "$file"       → 用户什么都没输（空）      → 报 error
第 2 层：! -e "$file"     → 路径不存在                → 报 not exist
第 3 层：-f "$file"       → 是普通文件                → is file
第 4 层：-d "$file"       → 是目录                    → is directory
第 5 层：else             → 剩下的是特殊文件          → 设备、socket、管道等

★ 为什么要按这个顺序？
   如果先判 -f，再判 -e，那"不存在的文件"会走到 else 报"other file"，提示就错了。
   → 先排除"空"和"不存在"，再判断具体类型，逻辑才严密。
```

**完整检验**：

```bash
cat > check_type.sh <<'EOF'
#!/bin/bash
read -t 20 -p "please input a filename:" file
if [ -z "$file" ];then
      echo "error: 没有输入"
elif [ ! -e "$file" ];then
      echo "$file is not exist!"
elif [ -f "$file" ];then
      echo "$file is file"
elif [ -d "$file" ];then
      echo "$file is directory"
else
      echo "$file is other file(device.)"
fi
EOF

# 五种情况各测一次
printf '\n'            | bash check_type.sh    # 空输入 → error
echo /no/such/file     | bash check_type.sh    # 不存在 → not exist
echo /etc/passwd       | bash check_type.sh    # 文件 → is file
echo /etc              | bash check_type.sh    # 目录 → is directory
echo /dev/sda          | bash check_type.sh    # 块设备 → other file
```

> **注意 `-t 20`**：20 秒不输入就超时（`read` 返回非 0，`$file` 为空）→ 走到第一个分支报 error。**给交互脚本加超时是生产习惯**，避免脚本挂在那儿等人等到天荒地老。

---

## 三、Case 条件判断（课件第三章）

### 3.1 case 语法（课件原话）

```bash
case $变量 in
    "值1")
        如果变量的值等于值1，就执行本程序代码
        ;;
    "值2")
        如果变量的值等于值2，就执行本程序代码
        ;;
    "值3")
        如果变量的值等于值3，就执行本程序代码
        ;;
    "值4")
        如果变量的值等于值4，就执行本程序代码
        ;;
    *)
        如果变量的值都不是上面的值，就执行本程序代码
        ;;
esac
```

**语法要点（★ 每个符号都不能省）**：

| 元素 | 作用 |
| --- | --- |
| `case ... in` | 开始 |
| `"值N")` | 每个分支的匹配模式，**以 `)` 结尾** |
| `;;` | **每个分支的结束符**（相当于 break，跳出 case） |
| `*)` | **默认分支**（相当于 if 的 else），匹配所有其他情况 |
| `esac` | case 语句结束（**esac = case 倒着写**，好记） |

> **case vs if/elif 怎么选（★ 面试可能问）**：
> | | case | if/elif |
> | --- | --- | --- |
> | 适合 | **一个变量和多个具体值比较**（相等判断） | **范围判断、复杂条件**（大于小于、多条件组合） |
> | 优势 | 结构清晰、支持通配符和正则、匹配多个值写一起 | 灵活、能做任意表达式 |
> | 例子 | 菜单选择、字符类型判断、参数分发 | 成绩分段、磁盘阈值告警 |
>
> **一句话**：**「等值匹配多用 case，范围判断用 if。」**

### 3.2 案例 1：判断输入字符的类型（课件原话）

```bash
#!/bin/bash
read -p "请输入一个字符，并回车:" MyKey
case "$MyKey" in
    [a-z])
        echo "你输入的是小写字母:$MyKey"
        ;;
    [A-Z])
        echo "你输入的daxie字母:$MyKey"
        ;;
    [0-9])
        echo "你输入的是shuzi:$MyKey"
        ;;
    *)
        echo "you input shi qi ta zi hu"
        ;;
esac
```

**这里的 `[a-z]` 是"通配符"不是正则（★ 容易搞混）**：

```text
case 的模式匹配用的是 shell 的【通配符（glob）】：
   [a-z]   匹配任意一个小写字母（一次只匹配一个字符）
   [0-9]   匹配任意一个数字
   ?       匹配任意一个字符
   *       匹配任意字符串

★ 注意：它一次只匹配一个字符！
  "ab" 不会命中 [a-z]（因为 [a-z] 只匹配单个字符）
  要匹配"多个小写字母"，得写 [a-z]* 或 [[:lower:]]* 或用正则的 [[ ]]
```

**完整检验（四种情况都要测）**：

```bash
cat > check_char.sh <<'EOF'
#!/bin/bash
read -p "请输入一个字符，并回车:" MyKey
case "$MyKey" in
    [a-z])
        echo "你输入的是小写字母:$MyKey"
        ;;
    [A-Z])
        echo "你输入的是大写字母:$MyKey"
        ;;
    [0-9])
        echo "你输入的是数字:$MyKey"
        ;;
    *)
        echo "其他字符:$MyKey"
        ;;
esac
EOF

echo a  | bash check_char.sh     # 小写字母
echo A  | bash check_char.sh     # 大写字母
echo 7  | bash check_char.sh     # 数字
echo '@'| bash check_char.sh     # 其他字符
```

> **顺带一提**：如果想匹配"一个或多个小写字母"，用 `[a-z]*`；如果要更强的匹配能力（正则），就用 `if [[ "$x" =~ ^[a-z]+$ ]]`。

### 3.3 案例 2：DOS 功能菜单（课件原话，★ 最实用的 case 案例）

```bash
#!/bin/bash
echo "##########################"
echo "# 1.打印磁盘空间利用情况 #"
echo "# 2.打印系统inode情况    #"
echo "# 3.打印系统负载情况     #"
echo "# 4.quit退出             #"
echo "##########################"
read -p "请输入你要执行的任务，以序号为索引:" action
case "$action" in
    1)
        df -h
        ;;
    2)
        df -ih
        ;;
    3)
        uptime
        ;;
    4)
        echo "程序退出"
        ;;
    *)
        echo "输入错误，请输入1-4"
        ;;
esac
```

**逐项对应的命令及其意义**：

| 选项 | 命令 | 看什么 |
| --- | --- | --- |
| 1 | `df -h` | 磁盘**空间**使用率 |
| 2 | `df -ih` | 磁盘 **inode** 使用率（空间没满但写不进时查它） |
| 3 | `uptime` | 系统负载 + 运行时长 |
| 4 | — | 退出 |

> **★ 这个菜单只做一次就退出了**，是"单次执行"。**要循环显示菜单（执行完还能再选）**，需要外面套一层 `while true`（第三期会讲）。

**完整可跑 + 检验**：

```bash
cat > menu.sh <<'EOF'
#!/bin/bash
echo "##########################"
echo "# 1.打印磁盘空间利用情况 #"
echo "# 2.打印系统inode情况    #"
echo "# 3.打印系统负载情况     #"
echo "# 4.quit退出             #"
echo "##########################"
read -p "请输入你要执行的任务，以序号为索引:" action
case "$action" in
    1)  df -h ;;
    2)  df -ih ;;
    3)  uptime ;;
    4)  echo "程序退出" ;;
    *)  echo "输入错误，请输入1-4" ;;
esac
EOF

echo 1 | bash menu.sh    # 期望：打印磁盘使用率
echo 3 | bash menu.sh    # 期望：打印负载
echo 9 | bash menu.sh    # 期望：输入错误，请输入1-4
```

> **提示**：课件原版把每个分支的 `;;` 单独写了一行，其实 `命令 ;;` 写一行更紧凑。**两种写法都对**，`;;` 只要在分支末尾即可。

---

## 四、本篇闭环自检清单

| 检查项 | 命令或位置 | 通过标准 |
| --- | --- | --- |
| 理解"退出码才是真假" | `false; echo $?` / `! false; echo $?` | 输出 1、0 |
| 会用 `!` `-a` `-o` | `[ $a -lt 30 -o $b -gt 100 ]; echo $?` | 能解释为什么是 0 |
| 知道生产为什么用 `&&` 和 `\|\|` | 三种写法对照 | 能说出 4 条理由 |
| root 判断脚本能跑 | `bash check_root.sh` | 普通用户报错、root 通过 |
| `read` 三种用法都会 | `-p` / `-t` / `-s` | 都跑过 |
| 单分支语法两种写法都会 | `then` 换行 / 同行加 `;` | 都能跑 |
| 会判断文件/目录 | `-f` `-d` `-e` `-z` `-s` | 各写一个例子 |
| 双分支能覆盖两个分支 | `check_nginx.sh` | 停服务前后各跑一次，结果不同 |
| 多分支顺序会排 | `score.sh` | 95/85/70/50 → 优秀/良好/及格/加油 |
| 会写文件类型判断 | `check_type.sh` | 五种输入都有正确输出 |
| case 语法完整 | `case ... esac` | 四个元素（`in` `)` `;;` `esac`）一个不漏 |
| case 支持多值匹配 | `yes\|y\|Y\|YES)` | 能跑通 |
| 会写功能菜单 | `menu.sh` | 选 1/3/9 输出正确 |

---

## 五、常见坑速查

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `syntax error near unexpected token 'then'` | `then` 和 `if` 写在同一行但**忘了分号** | 写成 `if [ ... ];then` 或让 `then` 另起一行 |
| `syntax error: unexpected end of file` | 忘了写 `fi` / `esac` / `done` | 补齐结尾关键字 |
| `[: ==: unary operator expected` | 变量**没加引号**且为空，`[ = "root" ]` 变成只有一个操作数 | 所有变量引用都加双引号：`[ "$user" = "root" ]` |
| `integer expression expected` | 用 `-eq` `-gt` 去比较**非数字** | 数值用 `-eq/-gt`，字符串用 `=` |
| 判断总是"服务异常" | `p1=$?` **接错了**（中间又执行了别的命令） | 紧接着接住退出码，或改用 `&&` 连接 |
| `ps aux 管道符 grep nginx` 结果多一行 | 匹配到 grep 自己 | 加 `grep -v grep`，或改用 `pgrep -x nginx` |
| 用户不输入就回车，脚本报错 | 空变量参与比较 | 先 `[ -z "$var" ]` 判断空 |
| 变量后面紧跟字符被吃掉 | 写成 `$Date_Time.tar` | 用 `${Date_Time}.tar` |
| `case` 里 `;;` 漏了一个 | 少写结束符 | 每个分支末尾都要 `;;`（最后一个分支也要） |
| `read -s` 之后输出粘在一起 | `-s` 关掉了回显 | 下一行单独 `echo` 换行 |
| 备份"完成"了但恢复不了 | 没验证 | 用 `tar -tzf` 列出内容确认 |

---

## 六、自测问题（闭卷）

- [ ] Shell 判断"真假"看的是什么？`0` 是真还是假？
- [ ] `!` `-a` `-o` 分别是什么？为什么生产不用它们？
- [ ] `[ ]` 和 `[[ ]]` 有什么区别？
- [ ] `[ "$a" -eq 5 ]` 和 `[ "$a" = "5" ]` 有什么区别？
- [ ] 变量为什么要加双引号？不加会出什么错？
- [ ] 条件判断的四类典型场景是什么？
- [ ] 单分支、双分支、多分支的语法分别怎么写？
- [ ] `then` 和 `if` 写同一行为什么要加分号？
- [ ] 多分支为什么顺序很重要？举例说明。
- [ ] `read` 的 `-p` `-t` `-s` 分别是什么？`-s` 后为什么要补一个 `echo`？
- [ ] 判断"文件是否存在、是文件、是目录"分别用什么测试符？
- [ ] `case` 的四个必备元素是什么？`esac` 是什么的倒写？
- [ ] `case` 和 `if/elif` 分别适合什么场景？
- [ ] `case` 里的 `[a-z]` 是正则还是通配符？它一次匹配几个字符？
- [ ] 怎么让交互式脚本能自动测试？（提示：管道喂输入）

---

## 七、与第三期的衔接

```text
第二期（本篇）学到的           第三期会怎么用
──────────────────────────────────────────────────
if / elif / else        →    套在 for / while 循环里，做批量判断
case 菜单               →    外面套 while true，变成"选完还能再选"的循环菜单
read 交互               →    批量建用户时读密码、循环里读文件每一行
判断"服务是否在跑"       →    写成守护进程：while true 里不断检查、挂了就拉起
`$?` 和退出码           →    循环里用 break / continue / exit 精确控制流程
```

> **下一步**：先把本篇的 `check_root.sh`、`check_nginx.sh`、`score.sh`、`check_type.sh`、`menu.sh` 五个脚本亲手敲一遍并跑通（**每个分支都要构造条件测到**），再进第三期学循环。
