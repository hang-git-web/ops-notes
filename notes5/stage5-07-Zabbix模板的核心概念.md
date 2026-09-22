# stage5-07 Zabbix 模板的核心概念

> 前置:第 04、05 节已经能创建监控项和触发器。

## 一、知识点

### 1. 模板(Template)是什么

模板是**一组可复用的配置集合**,包含监控项、触发器、图形、Web 场景、自动发现规则和宏。把模板链接到主机,主机就自动获得这一整套监控配置。

价值:**配置一次,批量应用**。100 台 Nginx 服务器只需要维护一个模板。

### 2. 模板与主机的三种关系

| 操作 | 含义 | 用途 |
| --- | --- | --- |
| Link | 链接模板,主机获得模板里的实体 | 批量套用监控 |
| Unlink | 断开链接,但保留已生成的实体(变成主机自己的) | 需要个性化改造 |
| Unlink and clear | 断开并删除实体及相关数据 | 彻底清理 |

**关键认知**:主机上从模板继承来的监控项**不能直接修改**,必须回到模板改,所有链接该模板的主机同步生效。这是模板最大的价值,也是新手最容易困惑的点。

### 3. 宏(Macro)

宏是模板参数化的手段,格式 `{$NAME}`,让同一个模板适配不同主机:

```text
触发器: min(/Template/system.cpu.load[all,avg1],5m)>{$CPU_LOAD_MAX}
模板宏: {$CPU_LOAD_MAX} = 5
```

优先级:**主机宏 > 模板宏 > 全局宏**,所以可以给某台主机单独覆盖阈值。

### 4. 标签(Tags)

Zabbix 6.0 以后,原来用于分类的"应用集(Applications)"已被**标签**取代。标签的用途是分类过滤(如 `component: cpu`),在 Latest data 和 Problems 页面按标签筛选。

### 5. 自动发现(LLD)与模板嵌套

- **自动发现(LLD)**:动态发现磁盘分区、网卡、端口等,自动为每个对象生成监控项
- **模板嵌套**:模板可以链接其他模板,例如 `Linux by Zabbix agent` 里就链接了 `ICMP ping`

## 二、实操

### 2.1 查看内置模板的组成

**操作**

```text
Data collection → Templates → 搜索 Linux by Zabbix agent → 打开
```

重点看四个页签:

| 页签 | 看什么 |
| --- | --- |
| Items | 有多少监控项,key 分别是什么 |
| Triggers | 触发器表达式怎么写 |
| Macros | 有哪些可调参数(如 `{$CPU.UTIL.CRIT}`) |
| Discovery | 自动发现规则,如文件系统自动发现 |

**检验**:能说出至少 3 个监控项的 key 和 1 个模板宏。

### 2.2 创建自定义模板

**操作**

```text
Data collection → Templates → Create template

Template name: Custom Linux Monitor
Visible name:  自定义 Linux 监控
Template groups: Templates/Linux  (没有就新建 Templates)
→ Add
```

**检验**:Templates 列表里能搜到 `Custom Linux Monitor`。

### 2.3 往模板里加监控项和触发器

**操作**

```text
进入 Custom Linux Monitor → Items → Create item

Name: Nginx 80 端口监听数
Type: Zabbix agent
Key:  nginx.port.status
Type of information: Numeric (unsigned)
Update interval: 30s
→ Add
```

```text
进入 Triggers → Create trigger

Name:     Nginx 80 端口不可用
Severity: Average
Expression: last(/Custom Linux Monitor/nginx.port.status)=0
→ Add
```

再加一个用宏的触发器:

```text
进入 Macros → Add
  {$CPU_LOAD_MAX} = 3

进入 Triggers → Create trigger
  Name: CPU 负载过高(模板宏)
  Expression: min(/Custom Linux Monitor/system.cpu.load[all,avg1],5m)>{$CPU_LOAD_MAX}
```

**检验**:模板的 Items、Triggers、Macros 三个页签里都能看到刚加的内容;触发器表达式里的 `{$CPU_LOAD_MAX}` 显示为蓝色,说明宏被正确引用。

### 2.4 把模板链接到主机

**操作**

```text
Data collection → Hosts → zabbix-agent → Templates
  → Link new templates → 搜索 Custom Linux Monitor → 选择 → Update
```

**检验**

```text
1. Hosts → zabbix-agent → Items
   应能看到来自模板的监控项(列表里标注 Template: Custom Linux Monitor)

2. Monitoring → Latest data → 选 zabbix-agent → Apply
   应能看到 "Nginx 80 端口监听数" 的最新值
```

> 如果 Latest data 里没有数据:先在 server 上执行 `zabbix_get -s 192.168.171.137 -k nginx.port.status` 验证 key 本身可用。

### 2.5 验证宏生效

**操作**

```text
Data collection → Hosts → zabbix-agent → Macros → Add
  {$CPU_LOAD_MAX} = 0.01
→ Update
```

**检验**

```text
Monitoring → Problems
```

因为主机宏优先级高于模板宏,阈值被改到极小,应在 1~2 分钟内出现 `CPU 负载过高(模板宏)` 的问题。把宏改回 `3`,问题随之恢复。

这一步同时验证了三件事:模板生效、宏生效、主机宏优先级更高。

### 2.6 模板导出与导入(备份与迁移)

**操作**

```text
Data collection → Templates → 勾选 Custom Linux Monitor → Export → 选择 YAML → 导出
```

**检验**

```bash
ls -l ~/Downloads/*.yaml 2>/dev/null || ls -l /tmp/*.yaml
head -20 <导出的文件>
```

期望:文件里能看到 `zabbix_export`、templates、items、triggers 等字段。把这个文件导入到另一套 Zabbix 环境,即可复用整套配置。

## 三、闭环自检清单

| 检查项 | 位置 | 通过标准 |
| --- | --- | --- |
| 模板创建成功 | Templates 列表 | 能搜到 Custom Linux Monitor |
| 模板含实体 | Items / Triggers / Macros | 各有内容 |
| 宏被正确引用 | 触发器表达式 | 宏显示为蓝色 |
| 模板已链接 | Host 的 Templates 页签 | 显示已链接 |
| 主机获得继承实体 | Host 的 Items | 标注 Template 来源 |
| 数据在采集 | Latest data | 有实时值 |
| 宏生效 | 改宏后 Problems | 行为随之变化 |
| 可导出 | Export YAML | 文件内容完整 |

## 四、常见问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 主机上改不了监控项 | 继承自模板 | 回模板修改 |
| 改模板后主机没变化 | 缓存延迟 | 等 1 分钟或 `systemctl restart zabbix-server` |
| 宏不生效 | 名字拼错 / 作用域不对 | 核对 `{$名称}`,注意主机宏优先级更高的规则 |
| 链接模板后无数据 | key 在 agent 上不支持 | 用 `zabbix_get` 先验证 key |
| Unlink 后实体还在 | 这是 Unlink 的正常行为 | 需要彻底删除用 Unlink and clear |
| 找不到"应用集" | 新版本已用标签取代 | 用 Tags 分类,在 Latest data 里按标签过滤 |