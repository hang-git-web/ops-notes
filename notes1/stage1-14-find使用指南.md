# find 文件查找命令

## 1. 基本用法

```text
find [路径] [表达式]
```

```bash
find /home -name "*.txt" -delete    # 在 /home 找 .txt 并删除
```

## 2. 按文件名搜索

| 选项 | 作用 | 示例 |
| --- | --- | --- |
| `-name` | 精确/模糊搜索文件名 | `find . -name "filename"` |
| `-iname` | 不区分大小写搜索 | `find . -iname "README"` |
| 通配符 `*` | 模糊匹配 | `find . -name "*.txt"` |
| 通配符 `?` | 匹配单个字符 | `find / -name "spell???.txt"` |

```bash
find . -name "filename"        # 精确搜索
find . -iname "readme"         # 忽略大小写
find . -name "*.txt"           # 模糊搜索所有 txt 文件
find / -name "spell*.txt"      # spell 开头的 txt
```

## 3. 按文件类型搜索（-type）

| -type 值 | 含义 |
| --- | --- |
| `d` | 目录 |
| `f` | 普通文件 |
| `l` | 软链接 |

```bash
find /var/ -type d -name "log"      # 查找名为 log 的目录
find / -type f -name "*.txt"        # 查找所有 txt 普通文件
find / -type l -name "*.so"         # 查找软链接
```

## 4. 排除特定类型文件（-not）

```bash
find / -name "*.txt" -not -name "example*"   # 找 txt，但排除 example 开头
```

## 5. 按文件大小搜索（-size）

```bash
find / -size +50M       # 大于 50M 的文件
find / -size -1M        # 小于 1M 的文件
find / -size 10M        # 正好 10M
```

## 6. 按时间搜索（-mtime / -atime / -ctime）

| 选项 | 含义 |
| --- | --- |
| `-mtime -1` | 最近 1 天**修改**过的文件 |
| `-atime +30` | 30 天没**打开（访问）**过 |
| `-ctime -1` | 最近 1 天**创建/状态改变**的文件 |

```bash
find /var/log -mtime -1     # 最近 1 天修改的
find / -atime +30           # 30 天没访问过的
find / -ctime -1            # 最近创建的
```

## 7. 多个条件组合（-o / -and）

| 运算符 | 含义 |
| --- | --- |
| `-o` | or（满足其一） |
| `-and`（默认） | and（全部满足） |

```bash
find / -name "*.tmp" -o -name "*.bak"          # tmp 或 bak
find / -type f -size +5M -mtime -2             # 且：5M 以上且 2 天内修改
```

## 8. 结合正则表达式（-regex）

```bash
find /etc -regex ".*\.conf$"    # 匹配 /etc 下所有 .conf 结尾的文件
```

> 注意：`-regex` 匹配的是**完整路径**，不是文件名，所以要用 `.*` 开头。

## 9. 找到后执行命令

### -exec 执行

```bash
find /var/log -name "*.log" -exec gzip {} \;
```

- `{}` 表示前面查找到的内容（占位符）
- `\;` 表示命令结束符（**反斜杠必须加**，否则 shell 会把它当成普通分号报错）

### -xargs 传参

```bash
find /data -name "*.log" | xargs rm -f    # 把找到的日志全部删除
```

### 删除空目录

```bash
find /data/ -type d -empty                # 先找空目录
find /data/ -type d -empty -exec rmdir {} \;   # 找到后删除
```

## 10. 避坑指南：权限不足

| 步骤 | 做法 |
| --- | --- |
| ① 检查用户权限 | `ls -l` 确认当前用户是否有该路径访问权 |
| ② 使用 sudo 提升权限 | `sudo find / -name "xxx"` |
| ③ 正确设置文件权限 | `chmod` / `chown` 调整属主权限 |

> 提示：`find /` 全盘搜索会产生大量权限报错，可加 `2>/dev/null` 屏蔽错误输出：
>
> ```bash
> find / -name "*.conf" 2>/dev/null
> ```

## 11. 一句话总结

**find = 按条件找文件**：`-name` 管名字、`-type` 管类型、`-size` 管大小、`-mtime` 管时间、`-o/-not` 管组合排除，`-exec` 管"找到后干活"。