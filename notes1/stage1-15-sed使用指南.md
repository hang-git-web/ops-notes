# sed 文本流编辑器

## 1. 基础与特性

- **不用打开文件**就能批量处理文本，适合处理日志、代码、配置文件
- 基本结构：`sed '命令' 文件名`
- 默认**只输出到屏幕，不修改原文件**；要真正修改文件，加 `-i` 参数

```bash
sed 's/old/new/' file.txt      # 输出修改结果，原文件不变
sed -i 's/old/new/' file.txt   # 直接修改原文件
sed -i.bak 's/old/new/' file.txt  # 修改前先备份为 file.txt.bak
```

## 2. 替换（s）

### 基本替换

```bash
sed 's/2222/qwer/' test.sh     # 只替换每一行的第一个 2222
sed 's/2222/qwer/g' test.sh    # 全局替换（g = 匹配所有）
```

### 替换路径分隔符 #

默认用 `/` 作分隔符，替换内容里**本身带 `/`**（如路径）时，换成 `#` 免转义：

```bash
sed 's#/usr/local/bin#/opt/bin#g'
```

### 引用匹配项（& 和 \1）

`&` 表示"匹配到的整个内容"，`\1` 表示第 1 个括号捕获的内容。

**例 1：把所有数字加中括号**

```bash
echo "abc123def" | sed 's/[0-9]*/[&]/'
# 输出：abc[123]def
```

**例 2：日期格式 年月日 → 日月年**

```bash
echo "2026-08-24" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3-\2-\1/'
# 输出：24-08-2026
```

> 用了括号 `()` 就要加 `-E`（扩展正则），`\1 \2 \3` 依次对应三个括号。

## 3. 条件替换（地址）

### 行条件替换

```bash
sed '/error/ s/error/ERROR/g' log.txt   # 只对含 error 的行替换
sed 's/error/ERROR/g' log.txt           # 所有行都执行替换
```

> 两者在"筛选条件 = 替换目标"时结果一样；筛选条件不同（如只改注释行）才有区别：
>
> ```bash
> sed '/^#/ s/error/ERROR/g' config.txt   # 只替换 # 开头的行
> ```

### 行范围替换

```bash
sed '5,$ {/error/ s/error/ERROR/g}' log.txt   # 从第 5 行到末尾，对含 error 的行替换
sed '3,5 s/old/new/g' file.txt                # 只处理第 3~5 行
```

## 4. 删除（d）

### 基本删除

```bash
sed '/^$/d' file.txt          # 删除空行
sed '/关键词/d' 文件           # 删除所有含关键词的行
sed '3,5d' test.sh            # 删除第 3~5 行
sed '/111/,/god/d' test.sh    # 删除从含 111 的行到含 god 的行之间
```

### 反向删除（!d）

`!d` = "不匹配的才删"，即**只留下匹配的行**：

```bash
sed '/[0-9]/!d' test.sh       # 只保留含数字的行
sed '/[a-z]/!d' test.sh       # 只保留含字母的行
sed '/^#/!d' nginx.conf       # 只留下注释行
```

> 关键总结：**只有关键词（地址）才需要用 `/ /` 包裹**，行号、$、范围不需要。

## 5. 添加 / 插入内容（a / i）

### a 添加（在匹配行**后面**加）

```bash
sed '/yuanhang/a test-1' log.txt      # 在含 yuanhang 的每一行后面加 test-1
sed '$a 这里是追加的内容' log.txt      # 在最后一行后面追加
sed '3a 新内容' file.txt              # 在第 3 行后面添加
```

添加多行内容（用反斜杠续行）：

```bash
sed '/yuanhang/a test-1\
wqer\
/data/ta\
error' log.txt
```

GNU sed 也支持直接 `\n`：

```bash
sed '/yuanhang/a test-1\nwqer\n/data/ta\nerror' log.txt
```

### i 插入（在匹配行**前面**加）

```bash
sed '1i 这里是插入的内容' log.txt       # 在第一行前插入
sed '/yuanhang/i 新内容' log.txt        # 在含 yuanhang 的行前插入
```

> 区别记忆：**a = after（后面），i = insert（前面）**。

插入空行：

```bash
sed '/good/i\\' test.sh      # 在含 good 的行前插入一个空行
sed '/good/a\\' test.sh      # 在含 good 的行后插入一个空行
```

## 6. 易错点

| 问题 | 错误写法 | 正确写法 |
| --- | --- | --- |
| 单引号未闭合 | `'5,${s/stupid/clever} 1.txt` | `'5,${s/stupid/clever/}' 1.txt` |
| s 命令缺少结束符 | `s/stupid/clever` | `s/stupid/clever/` |
| 命令块未闭合 | `/error/ {s/a/b/g` | `/error/ { s/a/b/g }` |
| 文件名写进引号 | `sed 's/a/b/g file.txt'` | `sed 's/a/b/g' file.txt` |

## 7. 总结

1. 总体结构：`sed '内容' 文件名`
2. 删除（d）和添加（a/i）都要先**确定位置**：`定位/d`、`定位/a`、`定位/i`；替换不一定要定位，可以直接 `s`
3. 关键词用 `/ /` 包裹；路径替换用 `#` 分隔符（普通文本用 `/` 即可）
4. 定位单行：`1a`、`1i`、`1d`；但 s 替换多行范围比较特殊：`'1,${...}'`

## 8. 进阶：扩展正则（-E）

默认 sed 是基础正则，`+`、`?`、`()`、`{}` 等需要 `-E` 才生效：

```bash
sed -E 's/([0-9]+)/<\1>/g' file.txt      # 所有数字串加尖括号
sed -E 's/[0-9]{2,4}//g' file.txt       # 删除 2~4 位数字
sed -E 's/(^|,)([^,]+)(,|$)/\2/g' data  # 括号分组 + 引用
```

> 和 grep 一样：`-E` = 扩展正则（egrep），`-P` = Perl 正则（sed 无 -P，用 -E 即可）。