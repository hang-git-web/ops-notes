# tar 归档与备份

## 一、tar 是什么

**tar（Tape Archive，磁带归档）** 是一个归档工具，用于把多个文件和目录打包成单个归档文件。

> 关键认知：**tar 只负责打包，不负责压缩**。`-z` / `-j` / `-J` 是调用外部的 gzip / bzip2 / xz 程序来压缩。

### 为什么用 tar

- **保持文件属性**：权限、所有者、时间戳、符号链接等元数据都能保留
- **增量/全量备份**：全量是默认行为；增量需要配合 `-g` 快照文件实现（见下文）
- **结构完整**：目录层级、空目录、软链接都能打包

---

## 二、基本语法

### 1. 创建归档

```bash
tar -cvf archive.tar /path/to/directory
```

### 2. 解压归档

```bash
tar -xvf archive.tar
```

### 常用选项

| 选项 | 含义 |
| ---- | ---- |
| `c` | 创建归档 |
| `x` | 解压归档 |
| `t` | 列出归档内容（不实际解压） |
| `v` | 显示详细过程 |
| `f` | 指定归档文件名（**必须跟文件名**） |
| `r` | 追加文件到归档（仅未压缩归档） |
| `u` | 更新归档（仅当文件比归档中更新时追加） |
| `-C` | 切换目录（打包/解压时指定路径） |
| `p` | 保留权限（root 解压时默认保留） |
| `g` | 使用快照文件实现增量备份 |

> 注意：`c` / `x` / `t` 三种操作模式只能三选一。

查看归档内容：

```bash
tar -tvf archive.tar
```

---

## 三、压缩归档

| 压缩方式 | 选项 | 特点 |
| -------- | ---- | ---- |
| gzip | `tar -czvf` | 压缩率低，速度快 |
| bzip2 | `tar -cjvf` | 压缩率适中 |
| xz | `tar -cJvf` | 压缩率高，速度极慢 |

解压时通常可以省略压缩选项，tar 会自动识别格式：

```bash
tar -xvf backup.tar.gz
tar -xvf backup.tar.xz
```

---

## 四、排除某些文件或目录

`--exclude` 的模式要**匹配归档内的路径**：

```bash
# ❌ 错误：/test 匹配不到 /home 下的 test
tar -cvf backup.tar /home --exclude /test

# ✅ 正确：写归档内的完整路径，选项用 = 号连接
tar -cvf backup.tar /home --exclude=/home/test
```

实用组合（排除开发目录和日志）：

```bash
tar -czvf backup.tar.gz /home \
  --exclude='*/node_modules' \
  --exclude='*/__pycache__' \
  --exclude='*.log'

# 排除项太多时用列表文件
tar -czvf backup.tar.gz /home --exclude-from=exclude.txt
```

---

## 五、-C 参数（切换目录）

### 1. 解压到指定目录

```bash
tar -xvf archive.tar -C /target/dir
```

### 2. 打包时不带绝对路径层级

```bash
# 直接打包绝对路径 → 解压会多出 home/user 层级，且 tar 会警告 "Removing leading '/'"
tar -cvf backup.tar /home/user

# 先进入目录再打包 → 归档内是 ./xxx，结构干净
tar -C /home/user -cvf backup.tar .
```

---

## 六、归档后追加与更新

> 仅适用于**未压缩**的归档，`-z` / `-j` / `-J` 压缩过的归档无法追加。

```bash
# 追加之前排除的 test 目录
tar -rvf home.tar home/test

# 更新：仅当文件比归档中更新时才写入
tar -uvf home.tar home/etc/passwd
```

---

## 七、增量备份

tar 本身不做增量备份，需要通过 `-g`（`--listed-incremental`）配合快照文件实现：

```bash
# 第一次：全量备份，生成快照文件 snapshot.snar
tar -g snapshot.snar -czvf backup1.tar.gz /home

# 第二次：只打包上次之后新增/修改的文件（增量）
tar -g snapshot.snar -czvf backup2.tar.gz /home
```

---

## 八、分卷归档

### 分卷

```bash
# 通过管道把归档按 50MB 一份切开
tar -cvf - /data/ | split -b 50M - data.tar.part
```

### 合并还原

```bash
cat data.tar.part* > data.tar
tar -xvf data.tar
```

---

## 九、其他实用技巧

### 解压部分文件 / 通配符筛选

```bash
tar -xvf archive.tar home/test           # 只解压其中一个文件
tar -tvf archive.tar --wildcards '*.txt' # 用通配符筛选查看
```

### 去掉顶层目录

```bash
# 归档内是 home/user/test，去掉 2 层后解压出来直接是 test
tar -xvf archive.tar --strip-components=2
```

### 流式备份（不生成中间文件）

```bash
tar -czvf - /data | ssh user@server 'cat > /backup/data.tgz'
```

`-f -` 表示输出到标准输出，通过管道直接传输。

### 备份进度与校验

```bash
tar --totals -czvf big.tar.gz /data   # 结束后显示总大小
tar -df archive.tar /data             # 比较归档与源目录是否一致
```

---

## 十、常见错误速查

| 错误 | 说明 |
| ---- | ---- |
| `--exclude /test` | 模式写错，需写归档内完整路径（`--exclude=/home/test`） |
| 对压缩归档执行 `-r` / `-u` | 追加/更新只支持未压缩归档 |
| 直接打包绝对路径 | 解压层级混乱，用 `-C` 进入目录再打包 |
| 以为 tar 自带压缩 | tar 只打包，压缩靠 gzip/bzip2/xz |
| 以为 tar 自带增量备份 | 增量需 `-g` 快照文件配合 |

---

## 十一、速查表

| 操作 | 命令 |
| ---- | ---- |
| 打包 | `tar -cvf archive.tar /path` |
| 打包 + gzip | `tar -czvf archive.tar.gz /path` |
| 打包 + xz | `tar -cJvf archive.tar.xz /path` |
| 解压 | `tar -xvf archive.tar` |
| 解压到指定目录 | `tar -xvf archive.tar -C /target` |
| 查看内容 | `tar -tvf archive.tar` |
| 排除目录 | `tar -cvf a.tar /home --exclude=/home/test` |
| 追加 | `tar -rvf a.tar newfile` |
| 增量备份 | `tar -g snap.snar -czvf b2.tgz /home` |
| 分卷 | `tar -cvf - /data \| split -b 50M - part` |
| 合并分卷 | `cat part* > data.tar` |

---

## 附：磁盘与文件查看命令

```bash
du -sh /path      # 查看目录总大小（人类可读）
ls -lrth          # 长格式 + 人类可读 + 按时间倒序，看最新/最大文件
```