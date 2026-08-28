# awk 文本处理命令

## 1. 基础语法

```text
awk 'pattern { action }' 文件名
```

| 部分 | 说明 |
| --- | --- |
| pattern（模式） | 可以是正则表达式，匹配满足条件的行；**省略则匹配所有行** |
| action（动作） | 对匹配到的行执行的处理命令；**省略则默认打印整行** |

## 2. 入门示例

```bash
awk '{print}' filename          # 打印所有行（等同于 print $0）
awk '/root/' /etc/passwd        # 匹配包含 root 的行并打印
```

## 3. 内置变量

| 变量 | 含义 |
| --- | --- |
| `$0` | 当前整行文本 |
| `$1, $2, …, $NF` | 当前行的各个字段（默认按空格/制表符分隔），`$NF` 表示最后一个字段 |
| `NR` | 当前已处理的行号 |
| `NF` | 当前行中字段的个数 |
| `FS` | 输入字段分隔符，默认空白字符，可用 `-F` 修改（如 `-F:`） |
| `OFS` | 输出字段分隔符，默认空格 |
| `ORS` | 输出记录分隔符，默认换行符 |

## 4. 常见实战

**用冒号分隔提取 root 用户的 shell 路径：**

```bash
awk -F: '/root/ { print $7 }' /etc/passwd
```

**限定行范围（第 11~19 行）：**

```bash
awk 'NR > 10 && NR < 20' messages
```

**同时显示行号：**

```bash
awk 'NR > 10 && NR < 20 {print NR, $0}' messages
```

## 5. BEGIN 与 END 块

| 块 | 执行时机 | 用途 |
| --- | --- | --- |
| `BEGIN` | 处理文件**之前** | 初始化，比如打印表头 |
| `END` | 处理完所有行**之后** | 统计汇总，比如打印行数 |

**示例：统计文件行数和总字段数：**

```bash
awk 'BEGIN { print "开始统计..." }
     { total_lines++; total_fields += NF }
     END { print "总行数:", total_lines, "总字段数:", total_fields }' messages
```

## 6. 解析 /etc/passwd 实战

```bash
awk -F: '{print $1, $3, $7}' /etc/passwd        # 按冒号切割，打印用户名、UID、Shell

awk -F: '{print "用户名:", $1, "UID:", $3}' /etc/passwd   # 带标签输出
```

## 7. 进阶用法：循环

### ① for 循环遍历字段（日志分析）

```bash
awk '{
    count++;
    print "第 " count " 行数据:"
    for (i = 1; i <= NF; i++) {
        print "  字段 " i ":", $i
    }
    print "-----------------"
}
END {
    print "总访问日志行数:", count
}' access.log
```

代码解读：

- `count++`：统计行数
- `for (i = 1; i <= NF; i++)`：遍历当前行的所有字段 `$1, $2, ..., $NF`
- `print "字段 " i ":", $i`：打印字段编号和内容
- `END { print "总访问日志行数:", count }`：处理完所有行后打印总行数

### ② for 循环求平均分（scores.txt）

数据文件：

```text
Tom 80 75 90
Jerry 60 70 55
Spike 85 90 88
```

```bash
awk '{
    total = 0
    for (i = 2; i <= NF; i++) {
        total += $i
    }
    avg = total / (NF-1)
    print $1, "总分:", total, "平均分:", avg
}' scores.txt
```

代码解读：

- `$1`：第一列（姓名）
- `NF`：当前行的字段数
- `for (i = 2; i <= NF; i++)`：从第 2 列循环到最后一列，累加成绩
- `total += $i`：把每个成绩加到 total
- `avg = total / (NF-1)`：总成绩 ÷ 科目数（减 1 是因为第 1 列是姓名）

### ③ while 循环（shell 包着 awk 的监控场景）

```bash
while true; do
    cat messages | awk '/ERROR/ { print "检测到ERROR，停止监控！"; exit }'
    sleep 1
done
```

> ⚠️ 说明：这里的 `while true; do ... done` 是 **shell 的 while 循环**，awk 在里面做过滤；检测到 ERROR 时 awk `exit` 退出（但 shell 循环会继续）。如果想真正用 awk 自己的 while，语法是 `awk '{ while(条件){ ... } }'`，用途不同——前者是"定时监控"，后者是"行内循环处理"。

## 8. 一句话总结

**awk = 按列处理文本**：`-F` 定分隔符，`$1/$NF` 取字段，`NR/NF` 看行号列数，`BEGIN/END` 做初始化与汇总，`for/while` 做行内循环，专治日志分析和表格统计。

