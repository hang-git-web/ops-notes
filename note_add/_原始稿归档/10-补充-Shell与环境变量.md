# 10 补充：export、source、子 shell（面经第 2 题）

> **面经原题**：export 是什么？source（`.`）与 bash（`./`）的启动区别？
> 你当时答得"基本全对"，但有一个常见错误需要修正：**`$$` 在 bash 的子 shell 里不一定变**（见 2.3）。
> **30 秒答法**：**"export 是把变量从 shell 的局部变量变成环境变量，让子进程能继承；source 是在当前 shell 进程里解释执行脚本，PID 不变、能影响当前环境；`./脚本` 是另起一个进程执行，要有执行权限，脚本里的 cd、export 都不会影响当前 shell；`exec` 则是不新建进程、直接替换当前 shell 的进程映像，所以跑完回不去原来的 shell。"**（另一份面经就考了 `exec` 这一种，见 2.6）

---

## 一、export：局部变量与环境变量

### 1.1 两种变量

| 类型 | 怎么来 | 子进程能看到吗 | 怎么看 |
| --- | --- | --- | --- |
| **shell 局部变量** | `NAME=value` | ❌ **看不到** | `set`（会列出很多）|
| **环境变量** | `export NAME=value` | ✅ 能看到 | `env` / `printenv` |

```bash
A=hello              # 局部变量：只有当前 shell 知道
bash -c 'echo $A'    # 空 —— 子进程读不到

export A=hello       # 变成环境变量
bash -c 'echo $A'    # hello
```

### 1.2 常用操作

```bash
export PATH=$PATH:/usr/local/mysql/bin   # ★ 最常用的场景：把目录加进 PATH
export -p                                 # 看当前已导出的变量
export -n A                               # 取消"导出"（变量还在，只是不进子进程）
unset A                                   # 彻底删除变量
env | grep PATH                           # 只看环境变量
```

> **经典追问**：**"我 export 了，为什么新开的终端里没有？"**
> 因为 export 只对**当前 shell 及其子进程**有效。要永久生效得写进 `~/.bashrc`、`~/.bash_profile` 或 `/etc/profile`，然后 `source` 或用新会话加载。

### 1.3 几个必须记住的变量

| 变量 | 含义 |
| --- | --- |
| `PATH` | 命令查找目录（**"command not found" 先查它**）|
| `HOME` / `USER` | 家目录 / 当前用户 |
| `PS1` | 命令行提示符 |
| `LANG` | 语言与编码（乱码问题常在这）|
| `$$` | 当前 shell 的 PID |
| `$?` | 上一条命令的返回值（0 = 成功）|
| `$0` `$1` `$#` `$@` | 脚本名 / 参数 / 参数个数 / 全部参数 |

---

## 二、source 与 bash（`./`）的区别（★ 本题核心）

### 2.1 对比表（背下来）

| 对比项 | `source a.sh`（等价 `. a.sh`） | `bash a.sh` | `./a.sh` |
| --- | --- | --- | --- |
| 在哪个进程执行 | **当前 shell 进程内** | 新起一个 bash 子进程 | 新起一个进程（按 shebang 解释器）|
| PID / `$$` | **不变** | **变了**（是新 bash 的 PID）| 变了 |
| 需要什么权限 | **只需读权限** | **只需读权限** | **必须有执行权限 `x`** |
| 脚本里的 `cd` / `export` | **会影响当前 shell**（这就是 source 的意义）| 不影响 | 不影响 |
| 用不用首行 `#!/bin/bash` | 不用（当前 shell 直接解释）| 不用（显式指定了 bash）| **用**（shebang 决定解释器）|
| 典型用途 | 加载配置（`source ~/.bashrc`）| 跑脚本、强制用 bash 跑 | 跑自己写好的、加过 `+x` 的脚本 |

### 2.2 一句话解释

```text
source  = 把脚本内容读进来，在当前 shell 里一行行执行  → 影响当前环境
bash a.sh / ./a.sh = 叫来另一个进程干活，干完就走      → 不影响当前环境
```

### 2.3 关于 `$$` 的坑（★ 你面经里写错了，这里修正）

```bash
echo $$                 # 假设是 1234

# 情况 A：另起一个进程执行 → $$ 变了
bash -c 'echo $$'       # 输出新的 PID（比如 5678）✔ 你的理解对

# 情况 B：bash 的"子 shell"（括号、管道、命令替换）→ $$ 不变！
( echo $$ )             # 还是 1234（bash 的已知行为：$$ 表示原始 shell 的 PID）
( echo $BASHPID )       # 这才是子 shell 的真实 PID

# 结论：判断"我是不是在子 shell 里"，用 $BASHPID，不要用 $$
```

> **面试怎么说**：**"另起一个 bash 进程执行脚本时，`$$` 是新进程的 PID；但 bash 的子 shell（括号、管道）里 `$$` 仍是外层 shell 的 PID，要看真实 PID 得用 `$BASHPID`。"** 能说出这个区别是加分项。

### 2.4 权限要求为什么不同（★ 追问点）

```bash
bash a.sh     # bash 把 a.sh 当成"输入文件"读 → 只要有读权限
./a.sh        # 内核要 execve() 直接执行这个文件 → 必须有执行权限 x
```

> 所以：**"脚本 chmod +x 是为了能 `./` 直接执行；用 `bash 脚本` 跑的话，不给 x 权限也行。"**

### 2.5 顺带说清 `sh` 和 `bash`

| | `sh a.sh` | `bash a.sh` |
| --- | --- | --- |
| Ubuntu/Debian 上 | `sh` 常指向 **dash**（更严格、功能更少）| 真正的 bash |
| 后果 | `[[ ]]`、数组等 bash 语法会报错 | 正常 |

> **运维经验**：脚本首行写 `#!/bin/bash`；要兼容就用 `#!/bin/sh` 但只写 POSIX 语法；调试用 `bash -x a.sh` 看每一步。

### 2.6 `exec`：把当前 shell"顶替"掉（★ 另一份面经原题）

> **面经原题**：脚本里用 export 改了 PATH 并执行了 cd，分别用 `bash mod.sh`、`source mod.sh`、`exec ./mod.sh` 运行，**父 shell 的 PATH 和当前目录会怎么变？为什么？**

**先记结论**：

| 运行方式 | 是否新建进程 | PID | 父 shell 的 PATH | 父 shell 的当前目录 | 跑完回到原 shell 吗 |
| --- | --- | --- | --- | --- | --- |
| `bash mod.sh` | ✅ fork 一个子进程 | 变 | ❌ **不变** | ❌ **不变** | ✅ 会 |
| `./mod.sh` | ✅ 新进程（按 shebang 解释器）| 变 | ❌ 不变 | ❌ 不变 | ✅ 会 |
| `source mod.sh` | ❌ 在**当前 shell 内**逐行执行 | **不变** | ✅ **变了** | ✅ **变了** | ✅ 会（它就是这个 shell）|
| `exec ./mod.sh` | ❌ **不新建进程，直接把当前 shell 的进程映像替换掉** | **不变** | 变了，但**这个进程已经不是原来的 shell 了** | 同理 | ❌ **不回来**：脚本一结束，终端会话就结束 |

**为什么**：

```text
· 环境（PATH、当前目录）是"进程自己的属性"，父进程的环境不会被子进程影响
  → bash / ./ 跑脚本：改动只发生在子进程里，子进程退出就一起消失
  → source：改动就发生在当前 shell 进程里，所以能留下来
· exec 是系统调用（execve）：它"用新程序的代码和数据覆盖当前进程"
  → 进程号不变（PID 一样），但里面跑的已经不是原来的 shell 了
  → 所以脚本跑完没有"原来的 shell"可以回去，你会直接掉出这个会话
```

**exec 的三个常见用法**（说得出就说明真用过）：

```bash
# ① 不带命令时，exec 只做重定向，不替换进程（整个脚本的输出都进日志）
exec >> /var/log/myscript.log 2>&1

# ② 容器入口脚本里把主程序"顶上去"，让它成为 PID 1
exec "$@"          # 应用才能正确接收 SIGTERM，docker stop 才走得顺

# ③ 省掉一层 shell，直接替换进程
exec /usr/local/bin/app --config /etc/app.conf
```

> **关联记忆**：容器里"PID 1 收不到信号"的坑（见 `11`）就是靠 `exec "$@"` 解决的 —— 如果 shell 不退位，应用就不是 PID 1，`docker stop` 的 SIGTERM 会打在 shell 上。

---

## 三、子 shell 与"命令为什么没生效"（★ 高频坑）

```bash
# 括号 = 开一个子 shell，里面的 cd 不影响外面
( cd /tmp && pwd )      # 输出 /tmp
pwd                     # 还是原来的目录

# 花括号 = 在当前 shell 里执行（注意空格和分号）
{ cd /tmp; pwd; }       # 会影响当前 shell
```

| 写法 | 是否新进程 |
| --- | --- |
| `( 命令 )` | ✅ 子 shell |
| `{ 命令; }` | ❌ 当前 shell |
| `命令1 \| 命令2` | ✅ **管道两侧各是一个子 shell** |
| `$(命令)` 命令替换 | ✅ 子 shell |
| `bash 脚本` / `./脚本` | ✅ 新进程 |

> **经典事故**：
> ```bash
> count=0
> cat file | while read line; do count=$((count+1)); done
> echo $count      # 输出 0 ！因为 while 在子 shell 里，count 的变化丢了
> ```
> 解决办法：`while ... done < file`（用输入重定向，不要用管道）。

---

## 四、配置文件什么时候加载（cron 的坑就在这）

| 场景 | 加载什么 |
| --- | --- |
| **登录 shell**（登录、`su -`） | `/etc/profile` → `/etc/profile.d/*.sh` → `~/.bash_profile` → `~/.bashrc` → `/etc/bashrc` |
| **非登录交互 shell**（新开一个终端） | `~/.bashrc` → `/etc/bashrc` |
| **非交互 shell**（**cron、ssh 执行远程命令**） | **基本什么都不读** |

> **所以两个经典问题**：
> 1. **cron 里跑脚本提示 command not found** → cron 的 PATH 很干净，也不读 `~/.bashrc`。**解决：脚本里用绝对路径，或开头显式 `source /etc/profile` / 自己设 PATH。**
> 2. **为什么 source 一下才生效** → 改配置文件只是改了文件，当前 shell 的环境还是旧的；`source` 是把文件重新读进当前 shell。

**别混淆 `su` 和 `su -`**：`su tom` 只是切用户、环境几乎不变；`su - tom` 是完整登录（会读 profile，`PATH`、`HOME` 都变）。

---

## 五、面试可能这么问

**Q：export 是什么？**
A：把 shell 的局部变量导出成环境变量。局部变量只有当前 shell 能用，导出后**子进程也能继承**。典型场景是把软件安装目录加进 PATH。

**Q：source 和 `./` 执行脚本有什么区别？**
A：`source` 在当前 shell 里解释执行，PID 不变、只要读权限，脚本里的 `cd` 和 `export` 会影响当前环境；`./` 是另起一个进程执行，要有执行权限，跑完就退出，对环境没影响。

**Q：为什么 `./a.sh` 报 Permission denied，但 `bash a.sh` 能跑？**
A：`./` 需要内核直接执行文件，必须有 `x` 权限；`bash a.sh` 只是把文件当输入读，有读权限就行。

**Q：怎么让环境变量永久生效？**
A：写进 `/etc/profile`（所有用户）或 `~/.bashrc` / `~/.bash_profile`（当前用户），再 `source` 一次。注意 cron 和 systemd 服务不读 `.bashrc`，它们有自己的环境。

**Q：管道里修改变量为什么没生效？**
A：管道两侧是子 shell，子 shell 里的变量不会回到父 shell。改成 `while read` + 输入重定向，或用 `{ ... }` 在当前 shell 执行。

---

## 六、3 分钟自测

- [ ] 能说清局部变量和环境变量的区别，以及 export 做了什么事
- [ ] 能说出 source / bash / `./` 三者在进程、权限、影响范围上的差异
- [ ] **能说清 `exec ./mod.sh` 与 `source`、`bash` 的区别**（PID 不变但进程被替换、跑完不会回到原 shell）
- [ ] 知道 `exec >> log 2>&1` 和容器里 `exec "$@"` 这两个用法
- [ ] 知道 `./a.sh` 需要 x 权限、`bash a.sh` 不需要
- [ ] 能说清脚本里 `$$` 在新进程和子 shell 中的不同表现
- [ ] 能说出"cron 里 command not found"的原因和两种解法
- [ ] 知道 `su` 和 `su -` 的区别
- [ ] 能说出管道里变量丢失的原因和解决办法
