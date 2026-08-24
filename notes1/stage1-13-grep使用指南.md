# grep 文本搜索命令

## 1. 常用参数

| 参数 | 作用 | 示例 |
| --- | --- | --- |
| `-i` | 忽略大小写 | `grep -i "error" log.txt` |
| `-v` | 反选（排除匹配行） | `grep -v "test" log.txt` |
| `-n` | 显示行号 | `grep -n "error" log.txt` |
| `-r` | 递归搜索目录 | `grep -r "TODO" src/` |
| `-c` | 只统计匹配行数 | `grep -c "error" log.txt` |
| `-o` | 只输出匹配的部分 | `grep -o "[0-9]*" log.txt` |
| `-l` | 只列出包含匹配的文件名 | `grep -rl "error" .` |
| `-w` | 全字匹配 | `grep -w "cat" file`（不匹配 catalog） |
| `-E` | 扩展正则（等价 egrep） | `grep -E "a\|b" file` |
| `-P` | Perl 正则（PCRE） | `grep -P "\d{4}" file` |
| `-A n` | 同时显示匹配行后 n 行 | `grep -A 2 "error" log.txt` |
| `-B n` | 同时显示匹配行前 n 行 | `grep -B 1 "error" log.txt` |
| `-C n` | 前后各 n 行 | `grep -C 3 "error" log.txt` |
| `--color` | 关键词高亮 | `grep --color=always "error" log.txt` |
| `--include` | 只搜指定类型文件 | `grep -r --include="*.js" "console.log" .` |

## 2. 基本用法

```bash
grep "error" /var/log/messages          # 普通搜索
grep -i "error" test.log                # 忽略大小写
grep -v "test" config.ini               # 排除含 test 的行
grep -n "error" test.log                # 带行号
grep -r --include="*.js" "console.log" .  # 递归搜索当前目录下所有 .js 文件
```

> 注意：`.` 表示当前目录，`/` 表示整个根目录，`grep -r` 时注意范围，别一上来搜整个根目录。

## 3. 正则表达式

### 基础正则（默认）

| 表达式 | 含义 | 示例 |
| --- | --- | --- |
| `^abc` | 以 abc 开头 | `grep "^abc" file` |
| `111$` | 以 111 结尾 | `grep "111$" file` |
| `^$` | 空行（开头结尾都为空） | `grep "^$" file` |
| `.` | 任意一个字符 | `grep "a.c" file` 匹配 abc、a1c |
| `*` | 前一个字符重复 0 次或多次 | `grep "a.*c" file` 匹配 a 开头 c 结尾 |
| `[0-9]` | 数字集合 | `grep "[0-9][0-9]" file` |

```bash
grep "^abc" file.txt        # 行首是 abc
grep "111$" file.txt        # 行尾是 111
grep "^$" file.txt          # 找出所有空行
grep "a.*c" file.txt        # a 和 c 之间任意内容
```

### 锚点 `^` `$` 与整行匹配（易混点）

- `a.*c`：**不加锚点**，只要行里某处有"a + 任意内容 + c"就匹配，位置不限
- `^ab$`：`^` 钉住行首、`$` 钉住行尾，**整行必须正好是 ab**

| 行内容 | `grep "a.*c"` | `grep "^ab$"` |
| --- | --- | --- |
| `abc` | ✅（a…c） | ❌（多了 c） |
| `ab` | ❌（没有 c） | ✅ 整行正好 ab |
| `a123c` | ✅ | ❌ |
| `xabcx` | ✅（藏在中间） | ❌ |
| `abcd` | ✅（ab…c） | ❌ |

```bash
echo "abc"     | grep "a.*c"    # ✅ 输出
echo "abc"     | grep "^ab$"    # 无输出
echo "ab"      | grep "a.*c"    # 无输出
echo "ab"      | grep "^ab$"    # ✅ 输出
echo "xabcx"   | grep "a.*c"    # ✅ 输出
echo "xabcx"   | grep "^ab$"    # 无输出
```

> 记住：`a.*c` 管"行里有这个模式"，`^ab$` 管"整行恰好是这些字符"。

### 扩展正则（-E）与 Perl 正则（-P）

```bash
grep -E "error|warning" log.txt      # 匹配 error 或 warning
grep -E "^[0-9]{3}-[0-9]{4}" file    # 电话号码开头
grep -P "\d{4}" file.txt             # 连续 4 个数字（\d=数字）
```

## 4. 高频场景

### ① 管道搭配使用

```bash
ps aux | grep nginx                 # 找进程
tail -f /var/log/nginx/access.log | grep "error"   # 实时过滤日志
cat test.log | grep "error" | grep "2026"          # 多级过滤
find / -name "*.conf" 2>/dev/null | grep nginx     # 配合 find
```

### ② 显示上下文（-A / -B / -C）

```bash
grep -A 2 "error" log.txt      # 匹配行 + 后 2 行
grep -B 1 "error" log.txt      # 匹配行 + 前 1 行
grep -C 3 "error" log.txt      # 前后各 3 行（排错最常用）
```

### ③ 统计空行

```bash
grep -c "^$" config.ini        # 输出空行数量
```

### ④ 找 IP 地址

```bash
grep -P "\d{1,3}(\.\d{1,3}){3}" access.log          # 粗略匹配
grep -P "([0-9]{1,3}\.){3}[0-9]{1,3}" access.log   # 同样效果
grep -oP "([0-9]{1,3}\.){3}[0-9]{1,3}" access.log | sort | uniq -c   # 统计每个 IP 出现次数
```

### ⑤ 关键词高亮

```bash
grep --color=always "error" log.txt    # 匹配词红色高亮
grep --color=always -n "error" log.txt # 高亮 + 行号
```

> 想让 grep 默认带高亮，可加别名：`alias grep='grep --color=auto'`

## 5. 踩坑与技巧

### ① 特殊字符转义

`.`、`*`、`$` 等是正则元字符，想匹配**字面量**需要转义：

```bash
grep "1\.txt" file        # 匹配 1.txt（\. 匹配点号本身）
grep "a\*c" file          # 匹配 a*c
grep "cost\$" file        # 匹配 cost$
```

> 单引号 `''` 防止 shell 解释 `$` 等符号；正则内的转义用 `\`。

### ② 过滤注释行（注意 -v 是排除）

```bash
grep "#" /etc/nginx/nginx.conf            # 显示含 # 的行（注释）
grep -v "#" /etc/nginx/nginx.conf         # 排除含 # 的行（去掉全部注释）
grep -v "^#" /etc/nginx/nginx.conf        # 排除 # 开头的行（保留行内注释）
grep "^[^#]" /etc/nginx/nginx.conf        # 只保留非注释开头、非空行
```

### ③ 中文与编码

```bash
LANG=zh_CN.UTF-8 grep "中文关键词" test.sh      # 指定中文编码环境
grep -a "关键词" test.sh                          # -a 把二进制文件当文本处理
```

> `--binary-files` 的正确用法：`grep --binary-files=text` 等价于 `-a`，参数取值是 `text` / `binary` / `without-match`，不是填"中文"。中文匹配主要靠系统 locale（`LANG=zh_CN.UTF-8`）。

### ④ 二进制文件

```bash
grep -a "error" app.bin        # 忽略二进制标记，按文本搜
```

## 6. 一句话总结

**grep = 按行找内容**：参数管"怎么找"（-i 忽略大小写、-v 排除、-n 行号、-r 递归），正则管"找什么"（^ 开头、$ 结尾、. 任意、* 重复），管道管"配合用"（tail/grep、ps/grep、grep/grep）。