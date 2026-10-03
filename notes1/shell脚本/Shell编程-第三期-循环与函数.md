# Shell 编程 第三期：循环与函数

> **对应课件**：《Shell从入门到精通(三期)》（千山 2026-5-7）
> **本篇定位**：Shell 的"重复能力"和"封装能力"。**循环**让脚本自动处理"一批"东西，**函数**让脚本不再复制粘贴。
> **环境**：Rocky Linux 8 / CentOS 7，`bash` 即可。
> **前置**：第二期（逻辑运算 + if/case）。
> **系列关系**：第二期（判断）→ **第三期（循环 + 函数）** → 第四期（数组 + 组合命令 + 巡检实战）。
> **本篇所有脚本都可直接跑**，每个案例都给了「预期结果」。

---

## 〇、开篇：建实验目录

```bash
mkdir -p /data/shell-lab3
cd /data/shell-lab3
```

> 依然用第二期学的两条排错命令：
> ```bash
> bash -n 脚本.sh     # 语法检查
> bash -x 脚本.sh     # 追踪执行（循环里尤其有用，能看到每一轮变量的值）
> ```

---

## 一、Shell 循环（课件第一章）

### 1.0 为什么要循环（课件原话：三类场景）

> **背景：经常会碰到一些重复的任务，对目标多次执行某个操作。**
>
> **批量处理**：同时我们去 ping 100 台机器，创建 50 个用户。
> **周期性检查**：每 5 秒钟检查一次 CPU 的使用率，每 10 分钟分析一下日志中是否有错误。
> **遍历数据**：读取配置文件中的 IP 列表；处理某个目录下的所有 `*.log` 文件。

**三种循环怎么选（★ 课件开头的对比表）**：

| 循环类型 | 适用场景（课件原话） |
| --- | --- |
| **for** | **已知清单（列表）** |
| **while** | **条件为真的时候，它会持续执行**（可能需要设置一些中止条件） |
| **until** | **条件为假的时候，它会持续执行**（它与 while 相反） |

> **一句话选型口诀**：**「有清单用 for，有条件用 while，等某事发生用 until。」**

### 1.1 for 循环（课件 1.1）

#### 语法一：for 变量 in 清单

```bash
# 标准写法
for 变量 in 清单
do
    程序代码
done

# 紧凑写法（do 提到同一行，前面必须加分号）
for 变量 in 清单 ;do 程序代码;done
```

> **★ 四个关键字一个都不能少**：`for` → `in` → `do` → `done`。
> **`do` 写在同一行时必须加 `;`**，和第二期 `then` 的规则一样。

#### 清单可以怎么写（★ 课件给了 5 种，全都很实用）

| 类型 | 示例（课件原话） | 结果 |
| --- | --- | --- |
| 直接写 | `for i in a b c` 或 `for i in 1 2 3` | a b c 或 1 2 3 |
| **大括号扩展（区间）** | `{1..5}` | 1 2 3 4 5 |
| **大括号（逗号）** | `{1,2,3}` 或 `{a,b,c}` | 1 2 3 或 a b c |
| **命令替换** | `$(ls /opt)` | 列出 /opt 目录下的所有文件和目录 |
| **通配符** | `*.sh` | 匹配当前目录下所有 Shell 脚本 |

**逐个动手验证一遍（这一组很值得跑）**：

```bash
# 1) 直接写
for i in a b c; do echo "直接写: $i"; done

# 2) 大括号区间
for i in {1..5}; do echo "区间: $i"; done

# 2b) 区间还能带步长（课件没提，很实用）
for i in {1..10..2}; do echo "步长2: $i"; done     # 1 3 5 7 9

# 3) 大括号逗号
for i in {a,b,c}; do echo "逗号: $i"; done

# 4) 命令替换
for f in $(ls /opt); do echo "命令替换: $f"; done

# 5) 通配符
cd /data/shell-lab3
touch t1.sh t2.sh t3.sh
for f in *.sh; do echo "通配符: $f"; done
```

> **★ 通配符这一种在生产里用得最多**：比如"把当前目录下所有 `.log` 都压缩一遍"、"把所有 `.sh` 都加上执行权限"。

#### 案例：打印星期（课件原话）

```bash
#!/bin/bash
for MyDay in S7 M1 T2 W3 T4 F5 S6
do
    echo "This time is: $MyDay!"
done
```

**检验**：`bash week.sh`，期望输出 7 行（S7 到 S6）。

> **观察**：`for` 是按**空格**把清单切成一项一项的，所以 `S7 M1 T2...` 被切成了 7 项。
> **这也是一个坑**：如果某一项**本身含空格**（比如文件名 `my file.txt`），循环会把它拆成两项。解决办法是改用 `while read` 逐行读（见 1.2 后面的用法）。

#### 语法二：C 风格 for（课件原话）

```bash
for ((初始值；循环控制条件；变量变化))
do
    程序代码
done
```

| 部分 | 课件说明 |
| --- | --- |
| 初始值 | `i=1` |
| 控制条件 | `i<=10` |
| 变量变化 | `i=i+1`（也可以写 `i++`、`++i`） |

> **★ 注意写法细节**：三个部分之间用**半角分号**分隔 —— `for ((i=1;i<=10;i++))`。
> 课件里写成了全角分号 `；`，**实际写脚本必须用半角 `;`**，否则语法报错。

#### 案例：1 到 10 求和（课件原话）

```bash
#!/bin/bash
a=0
for ((i=1;i<=10;i=i+1))
do
   a=$(($a+$i))
done
echo "The Sum is: $a"
```

**检验**：`bash sum.sh`，期望输出 `The Sum is: 55`。

**这段里三个知识点**：

| 写法 | 说明 |
| --- | --- |
| `i=i+1` | C 风格 for 里的变量变化，也可以写 `i++` |
| `a=$(($a+$i))` | **算术运算**，`$(( ))` 里可以直接写算式；里面的 `$` 可省（`a=$((a+i))` 也对） |
| `a=0` 初始化 | **必须初始化！** 不初始化的话 `$a` 是空，`$(($a+$i))` 会算错 |

**三种累加写法（对比记忆）**：

```bash
sum=$((sum + i))      # 推荐：$(( )) 算术展开，最直观
let sum+=i            # let 命令
sum=$(($sum+$i))      # 课件写法，等价
```

#### 企业小案例：批量 ping 网段（课件原话，★ 最实用）

```bash
#!/bin/bash
for i in {200..210}
do
   ping -c 2 172.22.4.$i &> /dev/null
   if [ $? -eq 0 ];then
       echo "172.22.4.$i is alive!"
   fi
done
```

**逐句拆解**：

| 代码 | 含义 |
| --- | --- |
| `{200..210}` | 大括号区间，展开成 200 201 ... 210（共 11 个） |
| `ping -c 2` | **只发 2 个包**（不加 `-c` 会一直 ping，脚本永远不结束） |
| `&> /dev/null` | 正确和错误输出都丢掉（我们只关心退出码） |
| `if [ $? -eq 0 ]` | ping 成功的退出码是 0 |

**完整可跑 + 带"不通"分支的版本**：

```bash
cat > ping_net.sh <<'EOF'
#!/bin/bash
for i in {200..210}
do
   ping -c 2 -W 1 172.22.4.$i &> /dev/null
   if [ $? -eq 0 ];then
       echo "172.22.4.$i is alive!"
   else
       echo "172.22.4.$i is down"
   fi
done
EOF

bash ping_net.sh
# 期望：11 行输出，每行要么 alive 要么 down
```

> **★ 加了 `-W 1`（等 1 秒就超时）**：不加的话，ping 不通的地址会**卡很久**（默认超时十几秒），11 个地址可能要等好几分钟。**批量 ping 一定要加 `-W`** —— 这是实践里最容易忽略、又最影响速度的参数。
>
> **注意**：课件用的是 `172.22.4.x` 网段，**要换成自己环境的网段**（比如 `192.168.171.x`），否则全是 down。

> **★ 更贴近生产的写法**：带上日志，方便事后查。
> ```bash
> for ip in 172.22.4.{200..210}; do
>     if ping -c2 -W1 "$ip" &>/dev/null; then
>         echo "OK   $ip" | tee -a /var/log/ping_check.log
>     else
>         echo "DOWN $ip" | tee -a /var/log/ping_check.log
>     fi
> done
> ```

### 1.2 while 循环（课件 1.2）

> **课件原文**：
> **while 循环：只要条件判断式为真（成立），循环就会一直继续，直到条件判断式为假（不成立），循环就停止。**
> **它循环的次数可能是固定的，也可能是无限的（死循环）。**
> **它一般用于哪个场景：守护进程。**
> **它用于弥补计划任务中分钟级的缺陷。**

**把最后两句展开（★ 这是 while 最值钱的用法）**：

```text
① 守护进程：while true 里不断检查"服务是不是活着"，挂了就立刻拉起。
   —— 计划任务（cron）最小粒度是 1 分钟，服务挂掉后最长要等 1 分钟才被拉起；
      而 while 循环可以 sleep 3 秒检查一次，把恢复时间从"分钟级"压到"秒级"。

② 弥补 cron 的分钟级缺陷：
   cron 是"到点执行一次"，而 while 是"持续盯着"。
   典型的对应关系：
     cron    → 每天凌晨 3 点跑一次备份
     while   → 每 3 秒检查一次 nginx 在不在
```

#### 语法

```bash
while [ 条件判断式 ]
do
    程序代码
done
```

#### 三种条件写法（课件重点，★ 必须搞清区别）

> **课件原文**：
> `test`：`test $i -le 10`　# 最原始
> `[]`：`[ $i -le 10 ]`　# 前后要有空格
> `[[ ]]`：`[[ $i -le 10 ]]`　# 前后要有空格，它支持正则或扩展判断

| 写法 | 本质 | 特点 |
| --- | --- | --- |
| `test 表达式` | **外部命令**（`/usr/bin/test`） | 最原始，现在基本不直接用 |
| `[ 表达式 ]` | **内置命令**（等价 `test`） | 前后**必须有空格**；变量不加引号容易出错；**不支持 `&&` 和 `\|\|`** |
| `[[ 表达式 ]]` | **bash 关键字** | 前后必须有空格；**支持 `&&`、`\|\|`、`=~`（正则）、`==` 通配符**；**推荐** |

**课件给的对比实验（★ 一眼看出差别）**：

```bash
#!/bin/bash
name="abc123"

if [ "$name" == a* ];then
    echo "不执行"
fi

if [[ $name == a* ]];then
    echo "模式匹配"
fi

if [[ $name =~ ^a* ]];then
    echo "正则匹配"
fi
```

**运行结果（★ 关键）**：

```text
第 1 个 if：不输出
  —— [ ] 里 "abc123" == a* 是按【字符串字面量】比较的，
     它不会把 a* 当通配符，所以不相等，条件为假。

第 2 个 if：输出"模式匹配"
  —— [[ ]] 里右边的 a* 会被当成【通配符（glob）】，
     abc123 以 a 开头，匹配成功。

第 3 个 if：输出"正则匹配"
  —— [[ ]] 的 =~ 右边是【正则表达式】，^a* 表示"以 a 开头"。
```

> **★ 一句话记住**：**`[ ]` 只做字面比较，`[[ ]]` 才支持通配符和正则。** 写 bash 脚本时**优先用 `[[ ]]`**。
> **但要注意**：`[[ ]]` 是 **bash 专有**的，`sh`（dash）里没有。所以脚本首行必须写 `#!/bin/bash`，别写 `#!/bin/sh`。

#### 案例：1 到 10 求和（课件给了三种写法，★ 对照着看）

```bash
# 写法一：test
#!/bin/bash
i=1
while test $i -le 10
do
     echo "num is: $i "
     i=$(($i+1))
done
```

```bash
# 写法二：[ ]（课件主推）
#!/bin/bash
i=1
while [ $i -le 10 ]
do
     echo "num is: $i "
     i=$(($i+1))
done
```

```bash
# 写法三：[[ ]]
#!/bin/bash
i=1
while [[ $i -le 10 ]]
do
     echo "num is: $i "
     i=$(($i+1))
done
```

**三个脚本的输出完全一样**：`num is: 1` 到 `num is: 10`。

> **★ `i=$(($i+1))` 这一行绝对不能少** —— 少了就是**死循环**（条件永远是 `1 <= 10`）。
>
> **while 写错最常见的两个后果**：
> ① 忘了改变量 → 死循环，把 CPU 跑满；
> ② 忘了 `sleep` → 检查太频繁，把目标服务打挂（后面的守护进程案例会讲）。

**死循环了怎么救**：

```bash
# 前台跑着：直接 Ctrl+C 中断

# 已经挂到后台了：
ps -ef | grep 脚本名        # 找到 PID
kill PID                    # 先温和地杀（发 SIGTERM）
kill -9 PID                 # 不行再强杀（SIGKILL）
```

#### 案例：批量创建用户（课件原话，★ 批量处理的代表）

```bash
#!/bin/bash
for user in user{1..10}
do
    useradd $user
    echo "Qianshan@2026" | passwd --stdin $user
    echo "用户$user 创建成功"
    sleep 2
done
```

**逐句拆解**：

| 代码 | 含义 |
| --- | --- |
| `user{1..10}` | **大括号扩展**：展开成 user1 user2 ... user10（注意这里没有 `$`，是纯文本展开，不是变量） |
| `useradd $user` | 建用户 |
| `echo "密码" 管道符 passwd --stdin $user` | **批量设密码的关键**：`--stdin` 让 `passwd` 从标准输入读密码，就不用交互两次了 |
| `sleep 2` | 每建一个用户停 2 秒（防止瞬时创建太多进程） |

**检验（★ 必须验证，不能只看"创建成功"的打印）**：

```bash
# 1) 确认用户真的建出来了
id user1 ; id user10
tail -3 /etc/passwd

# 2) 确认密码真的设上了（能切换过去）
su - user1 -c 'whoami'      # 期望输出 user1

# 3) 批量清理（做实验别留垃圾）
for u in user{1..10}; do userdel -r "$u"; done
id user1                    # 期望：no such user
```

> **★ 平台差异的坑**：`passwd --stdin` 是 **RedHat / CentOS / Rocky 系**才有的参数，**Ubuntu / Debian 不支持**。
> Ubuntu 上要改用 `chpasswd`：
> ```bash
> echo "$user:Qianshan@2026" | chpasswd
> ```
> **一个跨平台都能用的写法**：
> ```bash
> if command -v passwd >/dev/null && passwd --help 2>&1 | grep -q -- '--stdin'; then
>     echo "$PASS" | passwd --stdin "$user"
> else
>     echo "$user:$PASS" | chpasswd
> fi
> ```
> **能说出这个差异，面试时显得你真在多套系统上跑过。**

#### 案例：守护进程（课件原话，★★ while 最经典的用法）

```bash
#!/bin/bash
while true;do
        if pgrep -x "nginx" >/dev/null;then
                echo "nginx is runnning"
        else
            echo "nginx is down,restarting。。。"
            systemctl start nginx
        fi
        sleep 3
done
```

**逐句拆解**：

| 代码 | 含义 |
| --- | --- |
| `while true` | **永远为真** → 死循环（这里的"死循环"是**设计意图**，不是 bug） |
| `pgrep -x "nginx"` | 精确匹配进程名 nginx，输出丢掉只看退出码 |
| `systemctl start nginx` | 挂了就拉起来 |
| **`sleep 3`** | ★ **绝对不能少** |

> **★ 为什么 `sleep 3` 是必须的**：
> 如果不 sleep，这个循环会**每秒检查几百上千次**：
> ① CPU 白白被吃掉（`pgrep` 也是要创建进程的）；
> ② `nginx` 挂掉时，会在 1 秒内被 `systemctl start` 疯狂拉起，可能直接把机器打死。
> **检查类循环的 sleep 间隔，就是"恢复速度"和"系统开销"的权衡**：3 秒 ≈ 故障 3 秒内被发现，代价几乎为零 —— 这也是课件选 3 秒的原因。

**完整跑一遍 + 检验（★ 这个检验很有说服力）**：

```bash
cat > guard_nginx.sh <<'EOF'
#!/bin/bash
while true;do
    if pgrep -x "nginx" >/dev/null;then
        echo "$(date '+%T') nginx is running"
    else
        echo "$(date '+%T') nginx is down, restarting..."
        systemctl start nginx
    fi
    sleep 3
done
EOF

# 1) 后台跑起来（放后台 + 日志留痕）
nohup bash guard_nginx.sh >> /var/log/guard_nginx.log 2>&1 &

# 2) 看它在正常工作
sleep 5; tail -3 /var/log/guard_nginx.log

# 3) ★ 关键检验：故意把 nginx 停掉，看它能不能自动拉起来
systemctl stop nginx
sleep 8
tail -5 /var/log/guard_nginx.log     # 期望看到 "nginx is down, restarting..."
systemctl is-active nginx            # 期望：active（真的被拉起来了）

# 4) 收工：停掉守护脚本（否则它会一直跑）
ps -ef | grep guard_nginx | grep -v grep
kill $(pgrep -f guard_nginx.sh)
```

> **★ 想清楚这个脚本的"生产版本"该怎么改（面试加分点）**：
> ① **别用 `nohup ... &`**，应该写成 **systemd 服务**（能开机自启、能自动重启、日志进 journald）；
> ② 应该加**次数限制**：如果 5 分钟内重启超过 3 次，就发告警而不是继续无脑重启（否则服务本身有问题时会陷入"重启风暴"）；
> ③ 重启后应该**验证是否真的起来了**（`systemctl start` 返回成功不代表服务健康）；
> ④ 真正生产上优先用 **systemd 的 `Restart=always`** 或 **K8s 的探针**，而不是自己写 while 守护进程 —— 这个脚本更适合"临时顶一下"。

#### 案例：循环菜单（课件原话，★ 和第二期的菜单合起来看）

```bash
#!/bin/bash
while true;do
echo "##########################"
echo "# 1.打印磁盘空间利用情况 #"
echo "# 2.打印系统inode情况    #"
echo "# 3.打印系统负载情况     #"
echo "# 4.quit退出             #"
echo "##########################"
read -p "请输入你要执行的任务，以序号为索引:" action
case "$action" in
    1)  df -h  ;;
    2)  df -ih ;;
    3)  uptime ;;
    4)  echo "程序退出" ;;
    *)  echo "输入错误，请输入1-4" ;;
esac
done
```

> **★ 这一节把第二期和第三期连起来了**：
> **第二期的 case 菜单**只执行一次就结束；**外面套一层 `while true`** 之后，选完还能再选，变成一个"交互式工具"。
> **`case + while true + read`** 这三个组合，是写"运维小工具"的标准套路。

**改进：让选项 4 真正能退出（★ 课件原版有个 bug）**：

```bash
#!/bin/bash
while true;do
    echo "##########################"
    echo "# 1.打印磁盘空间利用情况 #"
    echo "# 2.打印系统inode情况    #"
    echo "# 3.打印系统负载情况     #"
    echo "# 4.quit退出             #"
    echo "##########################"
    read -p "请输入你要执行的任务，以序号为索引:" action
    case "$action" in
        1)  df -h  ;;
        2)  df -ih ;;
        3)  uptime ;;
        4)  echo "程序退出"; break ;;      # ★ 加 break 才会跳出 while
        *)  echo "输入错误，请输入1-4" ;;
    esac
done
```

> **★ 课件原版的选项 4 只打印了"程序退出"，但外层是 `while true`，所以打印完又会重新显示菜单** —— 这是"忘记 break"的经典 bug。
> **`break` 后面马上讲。**

### 1.3 until 循环（课件 1.3）

> **课件原文**：**跟 while 相反，它循环时只要条件判断式不成立就进行循环，一旦条件判断式成立就终止循环。**

**一句话对照（★ 记住这张图）**：

```text
while [ 条件 ]  →  条件为真就循环，为假就停
until [ 条件 ]  →  条件为假就循环，为真就停

同一个事情的两种说法：
  while [ $i -le 10 ]   等价于   until [ $i -gt 10 ]
```

#### 语法

```bash
until [ 条件判断式 ]
do
    程序体
done
```

#### 案例：1 到 10 求和（课件原话）

```bash
#!/bin/bash
i=1
s=0
until [ $i -gt 10 ]
do
     s=$(($s+$i))
     i=$(($i+1))
done
echo "Sum is: $s"
```

**检验**：`bash until_sum.sh`，期望输出 `Sum is: 55`。

> **注意条件写的是 `-gt 10`（大于 10）而不是 `-le 10`** —— 因为 until 是"条件**成立**就停"。
> 如果照抄 while 的 `[ $i -le 10 ]` 写进 until，循环**一次都不会执行**（因为 `1 <= 10` 一开始就成立，直接停）。**这是 until 最容易出的错。**

#### 企业小案例：等待 nginx 启动（课件原话，★ 很实用）

```bash
#!/bin/bash
until ss -ant |grep -q ":80";do
    echo "等待nginx启动..."
    sleep 2
done
echo "nginx 已启动"
```

**逐句拆解**：

| 代码 | 含义 |
| --- | --- |
| `ss -ant 管道符 grep -q ":80"` | 看有没有 80 端口的连接；**`-q` = quiet，只返回退出码不打印** |
| `until ... ; do` | **条件不成立就一直循环** —— 也就是"80 端口还没起来就一直等" |
| `sleep 2` | 每 2 秒检查一次 |

**为什么这个用法很有价值（★ 这是运维脚本的刚需）**：

```text
场景：先起数据库，再起依赖数据库的应用。
问题：数据库"启动了"不等于"能连了"（还要初始化，几十秒）。

笨办法：sleep 60        ← 猜一个时间，不是等太久就是等不够
好办法：until 探测到"真的能连了"才继续   ← 准确，且总耗时最短

所以：
  until ss -ant | grep -q ":3306"; do sleep 2; done     # 等 MySQL 端口
  until mysqladmin ping -h127.0.0.1 -uroot -p123456 --silent; do sleep 2; done  # 更准：真能连
```

> **★ 生产上必须加"最大等待时间"，否则会无限等下去**：
> ```bash
> #!/bin/bash
> MAX=30          # 最多等 30 次（60 秒）
> n=0
> until ss -ant | grep -q ":80"; do
>     n=$((n+1))
>     if [ "$n" -ge "$MAX" ]; then
>         echo "等待超时：nginx 60 秒内没有启动，请检查！"
>         exit 1
>     fi
>     echo "等待nginx启动... ($n/$MAX)"
>     sleep 2
> done
> echo "nginx 已启动（等待了 $((n*2)) 秒）"
> ```
> **"无限等待"是运维脚本的大忌** —— 一旦目标永远不会就绪，脚本就永远挂在那里，比报错还难排查。

### 1.4 特殊流程控制：break / continue / exit（课件 1.4）

> **课件原文的对比表（★ 背下来）**：

| 特殊流程语句 | 作用范围 | 效果（课件原话） |
| --- | --- | --- |
| **break** | **整个循环** | 跳出当前所在的循环 |
| **continue** | **本次迭代** | 跳过本次操作，继续下一次循环 |
| **exit** | **整个脚本** | 直接结束脚本，不再执行 exit 后面的语句或命令 |

**用一张图记住三者的区别**：

```text
for i in 1..10
   │
   ├─ i=1 → 正常执行
   ├─ i=2 → 正常执行
   ├─ i=3 → ★ 遇到 break    → 【整个循环结束】，后面的 i=4..10 再也不执行
   ...

   ├─ i=4 → ★ 遇到 continue → 【只跳过这一轮】，i=5 继续执行
   ...

   └─ i=7 → ★ 遇到 exit     → 【整个脚本结束】，循环后面的代码也不执行了
```

> **一句话记忆**：
> **「break 是『我不玩了』，continue 是『这个跳过，下一个』，exit 是『整个脚本到此为止』。」**

#### break 案例（课件原话）

```bash
#!/bin/bash
for ((i=1;i<=10;i=i+1))
do
      if [ $i -eq 4  ]
      then
         break
      fi
      echo $i
done
echo ----555---------
```

**预期输出（★ 仔细看）**：

```text
1
2
3
----555---------
```

> **为什么只到 3**：`i=4` 时进入 if，执行 `break` **直接跳出 for 循环**，所以 4 没有 `echo`，5~10 也没跑。
> **注意 `echo ----555---------` 还是执行了** —— 因为 break 只跳出**循环**，循环后面的代码继续跑。
> **这就是 break 和 exit 的核心区别。**

#### continue 案例（课件原话）

```bash
#!/bin/bash
for ((i=1;i<=10;i++))
do
  if [ $i -eq 4 ];then
      continue
  fi
  echo $i
done
```

**预期输出**：

```text
1
2
3
5
6
7
8
9
10
```

> **对比 break 的输出**：break 停在 3 就结束；continue 只是**少了 4**，其余照常。
> **`continue` 的实际用途**：在批量处理时"跳过不需要处理的那些"。
> ```bash
> # 例子：只压缩 .log 文件，跳过 .gz
> for f in /var/log/*; do
>     [[ "$f" == *.gz ]] && continue      # 已压缩的跳过
>     gzip "$f"
> done
> ```

#### exit 案例（课件原话，★ 含嵌套循环）

```bash
#!/bin/bash
for ((i=1;i<=10;i++))
do
  echo "第一层循环i值= $i"
  for ((j=1;j<=10;j++))
  do
    if [ $j -gt 3 ];then
      exit 123
      echo "1111"
    fi
   echo "第二层循环j值== $j"
  done
done
```

**预期输出（★ 仔细看它停在哪）**：

```text
第一层循环i值= 1
第二层循环j值== 1
第二层循环j值== 2
第二层循环j值== 3
```

然后脚本**整体结束**。

**三个关键点（★ 这题信息量很大）**：

| 观察 | 说明 |
| --- | --- |
| **`exit 123` 直接结束整个脚本** | 连**外层** for 循环也一起结束了 —— 这就是 exit 和 break 最本质的区别 |
| **`echo "1111"` 永远不执行** | 它在 `exit` 后面，脚本已经结束了（课件的对比表原话：**不再执行 exit 后面的语句或命令**） |
| **`exit 123` 的 123 是退出码** | 脚本结束时 **`$?` 就是 123**，可以用来告诉调用者"为什么结束" |

**验证退出码（★ 很重要的一步）**：

```bash
bash exit_demo.sh
echo "退出码是: $?"        # 期望输出：退出码是: 123
```

> **★ `exit [返回值]` 在生产里的用处**：
> ```bash
> exit 0    # 成功
> exit 1    # 一般错误
> exit 2    # 用法错误（参数不对）
>
> # 这样调用方就能判断：
> if bash my_script.sh; then
>     echo "脚本成功"
> else
>     echo "脚本失败，退出码 $?"
> fi
> ```
> **这也是"脚本能被别的脚本/CI 系统调用"的基础** —— 定时任务、Jenkins、Ansible 都靠退出码判断成败。

**break 只能跳一层（★ 常被问）**：

```bash
# 嵌套循环里，break 只跳出【最内层】循环
for i in 1 2 3; do
    for j in 1 2 3; do
        if [ "$j" -eq 2 ]; then
            break            # 只跳出内层 j 循环，外层 i 循环继续
        fi
        echo "i=$i j=$j"
    done
done
# 输出只有 i=1 j=1、i=2 j=1、i=3 j=1

# 想跳出多层怎么办？用 break 2（数字表示跳几层）
for i in 1 2 3; do
    for j in 1 2 3; do
        break 2              # 一次跳出两层
    done
done
```

> **对比 exit**：exit 不管嵌套多少层，**直接结束整个脚本**。所以"要跳出多层循环"时，exit 更省事，但它也把后面的代码一起砍了 —— **看你到底要不要执行后面的代码**。

---

## 二、Shell 函数（课件第二章）

### 2.0 为什么要有函数（课件原话）

> **课件原文**：我们常常要完成既定任务，有些重复的一些命令或语句，这个时候，我们可以把这些命令或语句**打包（封装）到一个固定的结构中，这个结构就是函数**。

**把课件的说法讲透（★ 函数解决三个问题）**：

```text
① 复用：同一段逻辑写一次，多处调用（不用复制粘贴）
② 可读：把 20 行细节收进一个名字里，主流程一眼看懂
③ 可维护：逻辑要改，只改函数里那一处

对比一下就明白：
  没有函数                       有函数
  ────────────────────────       ────────────────────────
  ping -c2 192.168.1.1 ...        ping_test 192.168.1.1
  if [ $? -eq 0 ]; then ...       ping_test 192.168.1.2
  ping -c2 192.168.1.2 ...        ping_test 192.168.1.3
  if [ $? -eq 0 ]; then ...
  （重复 N 遍）                    （主流程只有 3 行）
```

### 2.1 语法（课件给了三种写法，★ 都对）

```bash
# 写法一：function 关键字 + 圆括号（最规范）
function 函数名() {
   程序代码
}

# 写法二：只有圆括号（最常用）
函数名() {
   程序代码
}

# 写法三：只有 function 关键字（不用圆括号）
function 函数名 {
   程序代码
}
```

### 2.2 调用与参数（课件原话）

> **课件原文**：`函数名  参数...{参数1 参数2}`

```bash
函数名 参数1 参数2          # 调用时，参数跟在函数名后面，用空格分隔

# 函数内部用位置参数接收：
$1 $2 $3 ...                # 第 1、2、3 个参数
$#                          # 参数个数
$@                          # 所有参数
```

> **★ 三个必须知道的点**：
> ① **函数必须在"调用之前"定义**（bash 是从上往下读的，先定义后调用）；
> ② **函数里的 `$1` 是函数的参数，不是脚本的参数** —— 这两个容易混，见下面的坑；
> ③ **函数用 `return` 返回"退出码"（0~255），用 `echo` 返回"数据"** —— 这是 Shell 函数和别的语言最不一样的地方。
>
> | 想返回什么 | 用什么 | 怎么取 |
> | --- | --- | --- |
> | 数字/文本/结果 | `echo` 打出来 | `result=$(函数名)` |
> | 成功/失败状态 | `return 0` / `return 1` | 直接 `if 函数名; then` |

### 2.3 案例 1：求和函数（课件原话）

```bash
#!/bin/bash
sum() {
  s=0
  for ((i=0;i<=$num;i++))
  do
     s=$(($s+$i))
  done
  echo "Sum is: $s"
}
num=10
sum $num
```

**运行结果（课件原话）**：

```text
[root@r810-200 ~]# bash shell_fuction.sh
Sum is: 55
```

> **★ 但这一段写法有个"教学陷阱"，必须指出来**：
> 函数里用的是**全局变量 `$num`**（在函数外面赋的 `num=10`），**而不是通过参数 `$1` 接收**。
> 调用写的是 `sum $num`，看起来像"把 10 传进去"，**实际上函数根本没读 `$1`**，它读的是外面那个全局的 `num`。
> **后果**：如果外面把 `num` 改成 100，函数的行为就悄悄变了；这种"隐式依赖全局变量"的写法在复杂脚本里是 bug 温床。

**推荐写法（真正的"参数化函数"）**：

```bash
#!/bin/bash
sum() {
  local n=$1          # ★ 用 local 声明局部变量，值来自第 1 个参数
  local s=0
  local i
  for ((i=0;i<=n;i++))
  do
      s=$((s+i))
  done
  echo "$s"           # ★ 用 echo 把"结果"输出，让调用方用 $( ) 接住
}

result=$(sum 10)
echo "Sum is: $result"      # 期望：Sum is: 55

echo "1+..+100 = $(sum 100)"   # 期望：1+..+100 = 5050   ← 同一个函数，换个参数就能用
```

> **★ 三个改进点（这就是"从课件到生产"的距离）**：
> | 改进 | 为什么 |
> | --- | --- |
> | `local n=$1` 用参数而不是全局变量 | 函数不依赖外部状态，**换个地方调用也不会出错** |
> | `local s=0` `local i` | 局部变量不会污染外面的同名变量（bash 默认所有变量都是全局的！） |
> | `echo "$s"` 而不是 `echo "Sum is: $s"` | **函数只返回"数据"，不要夹带文字** —— 否则 `$( )` 接到的就是 `Sum is: 55`，没法参与后续计算 |

### 2.4 案例 2：批量 ping 检测函数（课件原话，★ 和第二期的 ping 案例合起来看）

```bash
#!/bin/bash
ping_test() {
   local ip=$1
   ping -c2 $ip &> /dev/null
   if [ $? -eq 0 ];then
      echo "$ip is alive!"
   fi
}

for ip in 172.22.4.{200..210};do
   ping_test $ip
done
```

**这段写法的好处（★ 对照第一期的 ping_net.sh）**：

```text
第一期的写法：循环里塞满了"ping + 判断 + 输出"的细节
本案例的写法：把细节封进 ping_test 函数，主流程只剩
              "对每个 ip，调用 ping_test" 一句话

★ 这就是"函数让主流程变清晰"的价值。
  以后要改判断逻辑（比如加"不通就告警"），只改函数那一处。
```

**完整可跑 + 检验（补上"不通"的提示）**：

```bash
cat > ping_func.sh <<'EOF'
#!/bin/bash
ping_test() {
    local ip=$1
    if ping -c2 -W1 "$ip" &> /dev/null; then
        echo "$ip is alive!"
    else
        echo "$ip is down"
    fi
}

for ip in 172.22.4.{200..210};do
    ping_test "$ip"
done
EOF

bash ping_func.sh
# 期望：11 行，每行 alive 或 down
```

> **★ 注意 `local ip=$1` 的 `local`**：
> **bash 里函数内不写 `local` 的变量默认是全局的**。如果这个函数被循环调用，用全局变量很容易出现"值被上一轮污染"的诡异 bug。
> **规则**：**函数内部一律用 `local`，除非你确实想改外部变量。**

**用 `return` 返回状态（★ 生产里更常用）**：

```bash
#!/bin/bash
ping_test() {
    local ip=$1
    ping -c2 -W1 "$ip" &> /dev/null
    return $?              # ★ 把 ping 的结果作为函数的退出码返回
}

ok=0; fail=0
for ip in 172.22.4.{200..210}; do
    if ping_test "$ip"; then
        echo "$ip OK";   ok=$((ok+1))
    else
        echo "$ip DOWN"; fail=$((fail+1))
    fi
done
echo "存活 $ok 台，不通 $fail 台"
```

> **这种写法的好处**：函数只负责"判断"，**决定怎么处理交给调用方** —— 这样同一个函数既能用来"打印结果"，也能用来"统计数量"，还能用来"不通就告警"。**这就是好函数的标志：职责单一。**

---

## 三、本篇闭环自检清单

| 检查项 | 命令或位置 | 通过标准 |
| --- | --- | --- |
| 三种循环会选型 | — | 能说出 for/while/until 各适合什么 |
| for 的 5 种清单都会写 | `{1..5}`、`{a,b,c}`、`$(ls)`、`*.sh` | 都跑过 |
| C 风格 for 会用 | `for ((i=1;i<=10;i++))` | 求和输出 55 |
| 批量 ping 能跑 | `ping_net.sh` | 11 行输出，且加了 `-W` |
| while 三种条件写法都懂 | `test` / `[ ]` / `[[ ]]` | 能说出 `[[ ]]` 的三个优势 |
| `[ ]` 和 `[[ ]]` 的差别 | `name=abc123` 那个实验 | 只有 `[[ ]]` 能模式匹配/正则 |
| while 不会写成死循环 | 求和脚本 | 有 `i=$(($i+1))` |
| until 的条件方向搞对了 | `until [ $i -gt 10 ]` | 求和输出 55 |
| 会用 until 等就绪 | `until ss -ant 管道符 grep -q ":80"` | 能等到 nginx 起来 |
| until 加了超时保护 | 带 MAX 的版本 | 超时会退出并报错 |
| break / continue / exit 分得清 | 三个案例 | 输出分别是 1 2 3 / 少一个 4 / 直接结束 |
| 知道 `exit 123` 的 123 是退出码 | `bash x.sh; echo $?` | 输出 123 |
| 知道 break 只跳一层 | `break 2` | 能说出跳多层怎么写 |
| 函数三种定义写法都会 | `sum()` / `function sum` | 都能跑 |
| 函数参数会用 `$1` | `sum 10` | 输出 55 |
| 会写"返回数据"的函数 | `result=$(sum 10)` | 能接到纯数字 |
| 会写"返回状态"的函数 | `return $?` + `if 函数; then` | 能统计 alive/down 数量 |
| 函数内部用 local | `local ip=$1` | 能说出为什么 |
| 循环菜单能退出 | 选项 4 加 `break` | 按 4 真的退出 |

**本篇要能独立写出来的 6 个脚本**：

```text
1. sum.sh              C 风格 for 求和
2. ping_net.sh         批量 ping（带 -W、带 alive/down）
3. create_users.sh     批量建用户（带验证和清理）
4. guard_nginx.sh      while true 守护进程（带 sleep）
5. until_wait.sh       until 等端口就绪（带超时保护）
6. ping_func.sh        函数版批量 ping（带 local 和 return）
```

---

## 四、常见坑速查

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `syntax error near unexpected token 'do'` | `do` 和 `for` 写同一行但**忘了分号** | `done` 另起一行，或写 `for x in y ;do` |
| `syntax error: unexpected end of file` | 忘了 `done` | 补上 `done`（嵌套循环要补够个数） |
| 脚本卡住不结束 | `ping` 没加 `-c`；或循环忘了改变量（死循环） | ping 加 `-c2 -W1`；循环里加 `i=$(($i+1))` |
| 批量 ping 特别慢 | 没加 `-W`，不通的地址要等默认超时 | 加 `-W 1` |
| CPU 100% | while 死循环没 `sleep`，检查太频繁 | 加 `sleep 3` 之类 |
| 计划任务跑了但服务还是没被拉起 | cron 最小粒度 1 分钟 | 改用 while 守护进程，或用 systemd `Restart=always` |
| `[[` 报 `command not found` | 脚本用的是 `sh`（dash） | 首行改成 `#!/bin/bash` |
| `until` 一次都没执行 | 条件方向写反了（照抄了 while 的 `-le`） | until 要用"成立就停"的条件，如 `-gt 10` |
| 菜单选 4 退不出去 | 外层 `while true` 但 `4)` 分支没 `break` | 加 `break` |
| `passwd --stdin` 报错 | 系统是 Ubuntu/Debian | 改用 `echo "user:pass" 管道符 chpasswd` |
| 函数里改了变量，外面也变了 | **bash 函数内不写 local 就是全局变量** | 函数内用 `local` 声明 |
| `$(sum 10)` 接到的是文字 | 函数里 `echo "Sum is: $s"` 夹带了文字 | 函数只 `echo` 纯数据，文字留给调用方 |
| 函数调用了却提示 command not found | **函数定义写在调用之后** | 先定义、后调用 |
| 函数里的 `$1` 拿到的是脚本参数 | 混淆了"脚本的 `$1`"和"函数的 `$1`" | 函数内的 `$1` 是**函数自己的**参数；脚本参数要提前存到变量里 |

---

## 五、自测问题（闭卷）

- [ ] `for` / `while` / `until` 分别适合什么场景？选型口诀是什么？
- [ ] `for` 循环的清单有哪 5 种写法？生产里最常用哪两种？
- [ ] C 风格 for 的三个部分是什么？中间用什么分隔？
- [ ] 批量 ping 为什么必须加 `-c` 和 `-W`？
- [ ] `test`、`[ ]`、`[[ ]]` 三者的区别是什么？
- [ ] `[ "$name" == a* ]` 和 `[[ $name == a* ]]` 结果为什么不一样？
- [ ] 写 while 循环最容易犯的两个错是什么？
- [ ] `until` 和 `while` 的关系是什么？条件方向要注意什么？
- [ ] `until` 等待服务就绪的写法是什么？为什么要加超时保护？
- [ ] break / continue / exit 三者的作用范围有什么区别？
- [ ] `break` 在嵌套循环里跳几层？想跳两层怎么写？
- [ ] `exit 123` 里的 123 是什么？怎么验证？
- [ ] 函数的三种定义写法是什么？函数必须先定义还是可以先调用？
- [ ] 函数怎么接收参数？函数里的 `$1` 和脚本的 `$1` 有什么区别？
- [ ] 函数怎么"返回数据"？怎么"返回状态"？分别用什么关键字？
- [ ] 为什么函数内部要用 `local`？不写会怎样？
- [ ] 为什么函数里不应该 `echo "结果说明: $值"`？
- [ ] `case + while true + read` 这个组合能做出什么？

---

## 六、与第四期的衔接

```text
第三期（本篇）学到的           第四期会怎么用
──────────────────────────────────────────────────
for i in {200..210}      →    改成 for ip in "${IPS[@]}"，用【数组】管理列表
while read line          →    读配置文件、读命令输出，逐行处理
函数封装逻辑              →    把巡检项写成函数，主流程只调用函数
循环 + case 菜单          →    综合巡检脚本（linux_health_check.sh）
循环里的管道统计          →    grep / awk / sort / uniq / wc 组合做日志分析
```

> **下一步**：先把本篇的 6 个脚本亲手敲一遍跑通（**尤其是 `guard_nginx.sh` 的"停掉 nginx 看它自动拉起"那个检验**，最能体现 while 的价值），再进第四期学数组和实战巡检。
