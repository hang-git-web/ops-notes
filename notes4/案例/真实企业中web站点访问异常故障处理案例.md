# Web 站点故障排查实战:两个真实案例(端口不通 + 访问异常)

> 故障排查的核心是**分层定位**:先判断问题出在客户端、网络、系统还是应用,再深入分析。本文用两个真实案例,完整还原“从现象到根因”的排查过程,并附上可直接使用的自动化封禁脚本。

---

# 案例一:Web 站点发布失败 —— 端口 6666 无法访问

## 1. 故障背景

某 Web 站点需要对外发布。按照要求,端口不能使用 80 或 443,于是选用 6666。配置完成后,浏览器始终无法访问。

## 2. 排查思路:先定位故障层次

按下面的顺序逐层缩小范围:

```text
客户端请求发出去了吗? → 网络层能不能到达? → 服务层端口在监听吗? → 应用有没有响应?
```

抓包是最直接的定位手段,能够明确区分三种情况:

| 抓包现象 | 结论 |
| --- | --- |
| 完全抓不到包 | 请求根本没到达网卡:客户端没发、上游网络/运营商/云安全组拦截 |
| 有请求包,但没有响应 | 服务层问题:端口没监听、防火墙丢包、应用卡死 |
| 有请求也有响应,但页面不对 | 应用层问题:配置错误、代码异常 |

## 3. 使用 tcpdump 抓包

### 安装

```bash
yum install -y tcpdump
```

### 正确写法(注意:必须用过滤器,否则数据量巨大)

```bash
# 抓指定网卡的 6666 端口流量
tcpdump -i eth0 -nn port 6666

# 更精确:只抓 TCP、指定端口,并以数字显示地址和端口
tcpdump -i eth0 -nn 'tcp port 6666'
```

> 抓包属于“高危操作”:不加过滤器会抓到海量数据,可能让终端刷屏甚至卡死。生产环境建议加上 `-c`(抓够多少包就停)或写入文件:

```bash
# 只抓 100 个包
tcpdump -i eth0 -nn -c 100 port 6666

# 写入文件,事后分析(推荐)
tcpdump -i eth0 -nn -s0 -w /tmp/port6666.pcap port 6666
tcpdump -r /tmp/port6666.pcap -nn        # 读取分析
tcpdump -r /tmp/port6666.pcap -nn -A     # 以 ASCII 显示内容
```

### 本案例结果

客户端在浏览器里反复访问,`tcpdump` 却始终没有任何输出。

**结论:请求根本没有到达服务器网卡。** 说明问题不在服务端,而在请求发出或链路传输环节。

## 4. 分析根因

进一步排查发现:6666 属于**高危端口**,被运营商/云平台在网络层面拦截,请求根本无法送达服务器。

### 三类“不能用”的端口

| 分类 | 说明 | 典型端口 |
| --- | --- | --- |
| 运营商/云厂商限制的端口 | 需备案或单独申请,默认不放行 | 80、443、25 等 |
| 常见高危端口 | 历史漏洞多、木马和攻击常用,云平台默认封禁 | 135、137、138、139、445、1433、1521、3306、3389、4899、5800、5900、6666-6669 等 |
| 不常见高危端口 | 部分远控木马专用端口 | 12345、27374、31337、54320、54321 等 |

### 常见云平台的默认封禁端口(以阿里云 ECS 为例)

```text
25、135、137、138、139、445、593、1025、1434、
1521、3306、3389、4899、5800、5900、6666-6669
```

可以看到,**6666 正好落在被默认封禁的区间内**,这就是访问不通的直接原因。

## 5. 解决方案

| 方案 | 操作 |
| --- | --- |
| 换端口(推荐) | 改用不在封禁列表内的端口,如 8080、8088、8888、9090 等 |
| 申请放行 | 部分端口(如 80/443/25)可提交材料向云平台或运营商申请 |
| 排查云安全组 | 确认安全组入方向已放行该端口,否则同样访问不通 |
| 排查本机防火墙 | `firewall-cmd --list-all` 或 `iptables -nL` 确认没有拦截 |

换端口示例(源码与 Nginx 配置同时修改):

```nginx
server {
    listen 8080;
    server_name www.example.com;
}
```

```bash
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
ss -nltp | grep 8080
```

## 6. 案例一小结

- 抓包能快速判断“请求有没有到服务器”,是定位网络类故障的第一选择
- 抓不到包 ≠ 服务端有问题,更可能是**上游链路拦截**
- 发布站点前要确认端口是否在运营商/云平台的封禁范围内
- 生产环境优先使用 80、443 并完成备案;备用端口选 8080 一类常见安全端口

---

# 案例二:真实企业 Web 站点访问异常故障处理

## 1. 故障现象

企业 Web 站点访问缓慢甚至无法打开,需要快速定位并恢复。

## 2. 分层排查思路

```text
网络层 → 系统层 → 应用层
ping / telnet / iptables → top / free / df → systemctl / ss / 日志 / 配置
```

## 3. 网络层排查

```bash
# 1. 连通性
ping www.xxx.com

# 2. 业务端口是否可达(443 为 HTTPS 端口)
telnet www.xxx.com 443
# 推荐替代命令(更清晰,支持超时设置)
nc -vz -w 3 www.xxx.com 443
curl -I -m 5 https://www.xxx.com

# 3. 本机防火墙策略
iptables -nL
firewall-cmd --list-all
```

注意:

- `ping` 不通**不能**直接判定网络故障,很多生产服务器禁用了 ICMP
- 云主机还要检查**安全组**规则,它在系统防火墙之外
- 域名解析异常也会导致访问失败,可用 `nslookup` / `dig` 确认解析结果

## 4. 系统层排查

```bash
top            # CPU、负载、内存占用,找出占用最高的进程
free -h        # 内存使用情况
df -h          # 磁盘空间,注意日志把磁盘写满
vmstat 1       # CPU/内存/IO 全局视图
```

排查要点:

- `top` 中 `load average` 持续高于 CPU 核数,说明系统过载
- 关注 CPU 的 `%us`(用户态)、`%sy`(内核态)、`%wa`(IO 等待)占比,判断瓶颈类型
- 如果 `top` 已经定位到具体进程,可跳过后续的 `free`、`df`,直接查该进程

## 5. 应用层排查

```bash
# 服务状态
systemctl status nginx
systemctl status php-fpm

# 端口监听
ss -nltp

# 日志
tail -f /var/log/nginx/access.log
tail -n 100 /var/log/nginx/error.log

# 配置文件语法
nginx -t
```

## 6. 定位结果:恶意 IP 刷访问

查看访问日志时发现,某个 IP 在短时间内产生了大量请求:

```bash
tail -f /var/log/nginx/access.log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

输出示例:

```text
  58231 10.1.1.201
  42117 10.1.1.52
    356 正常业务IP
```

大量异常请求持续占用连接和 CPU,导致正常用户访问缓慢——典型的 **CC 攻击 / 恶意爬虫**行为。

## 7. 应急处理:封禁恶意 IP

```bash
# 封禁(注意是 -s,表示 source 源地址)
iptables -A INPUT -s 10.1.1.201 -j DROP
iptables -A INPUT -s 10.1.1.52  -j DROP

# 查看已添加的策略
iptables -nL
iptables -nL --line-numbers        # 带序号,便于删除

# 解封(按序号删除)
iptables -D INPUT 1
```

CentOS 7 若使用 firewalld,更推荐用富规则:

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.1.1.201" drop'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.1.1.52" drop'
firewall-cmd --reload
```

规则持久化(重启不丢失):

```bash
# iptables 方式
service iptables save
# 或
iptables-save > /etc/sysconfig/iptables
```

## 8. 进阶:自动化封禁恶意 IP

### 8.1 优化版脚本(含白名单、防重复封禁、日志记录)

```bash
#!/bin/bash
# 功能:统计 Nginx 访问日志,自动封禁访问量超阈值的 IP
# 建议路径:/usr/local/bin/ban_bad_ip.sh

NGINX_LOG="/var/log/nginx/access.log"
ACCESS_THRESHOLD=1000
OP_LOG="/var/log/nginx_ip_ban.log"
SCAN_LINES=20000

# 内网与本地地址白名单(避免误封)
is_whitelisted() {
    case "$1" in
        127.0.0.1|10.*|192.168.*|172.1[6-9].*|172.2[0-9].*|172.3[01].*) return 0 ;;
        *) return 1 ;;
    esac
}

# 只统计最近 20000 行,避免大日志拖慢脚本
tail -n "$SCAN_LINES" "$NGINX_LOG" \
| awk '{print $1}' \
| sort | uniq -c | sort -rn \
| while read -r count ip; do

    # 未超过阈值,跳过
    [ "$count" -le "$ACCESS_THRESHOLD" ] && continue

    # 白名单跳过
    if is_whitelisted "$ip"; then
        continue
    fi

    # 已封禁则不再重复添加
    if iptables -C INPUT -s "$ip" -j DROP 2>/dev/null; then
        echo "$(date '+%F %T') - IP $ip 已在封禁列表,访问次数 $count" >> "$OP_LOG"
        continue
    fi

    iptables -I INPUT 1 -s "$ip" -j DROP -m comment --comment "auto-ban"
    echo "$(date '+%F %T') - 封禁 IP: $ip,访问次数: $count" >> "$OP_LOG"
done

echo "执行完成,操作日志:$OP_LOG"
```

要点说明:

- 用 `tail -n 20000` 限定扫描范围,避免日志过大时脚本长时间占用 CPU
- 用 `iptables -C` 判断规则是否已存在,比 `iptables -nL | grep` 更准确、更快
- 用 `-I INPUT 1` 插入到最前面,规则优先级高
- 加入内网白名单,防止把负载均衡、监控、代理地址误封

### 8.2 大规模封禁:改用 ipset(性能更好)

恶意 IP 数量多时,几千条 iptables 规则会拖慢网络处理,推荐用 `ipset`:

```bash
# 初始化:创建集合(超时 86400 秒自动解封)
ipset create badip hash:ip timeout 86400
iptables -I INPUT -m set --match-set badip src -j DROP

# 封禁单个 IP
ipset add badip 10.1.1.201

# 查看集合内容
ipset list badip
```

脚本中把 `iptables` 那行替换为:

```bash
ipset add badip "$ip" -exist
```

### 8.3 定时执行

```bash
chmod +x /usr/local/bin/ban_bad_ip.sh
crontab -e
```

```cron
*/5 * * * * /usr/local/bin/ban_bad_ip.sh >> /var/log/ban_cron.log 2>&1
```

### 8.4 更成熟的方案:fail2ban

生产环境更推荐使用 **fail2ban**,它支持多种日志格式、自动解封、邮件告警和多种封禁动作,维护成本低于自研脚本。自研脚本适合应急和轻量场景。

## 9. 排查方法论总结

### 9.1 分层排查表

| 层次 | 排查命令 | 关注点 |
| --- | --- | --- |
| 网络层 | `ping`、`telnet`、`nc -vz`、`curl -I` | 链路是否通、端口是否可达 |
| 防火墙 | `iptables -nL`、`firewall-cmd --list-all` | 是否被本机规则拦截 |
| 云安全组 | 控制台查看 | 是否被平台层拦截 |
| 系统层 | `top`、`free -h`、`df -h`、`vmstat 1` | CPU、内存、磁盘、IO |
| 应用层 | `systemctl status`、`ss -nltp` | 服务是否运行、端口是否监听 |
| 日志层 | `access.log`、`error.log`、`catalina.out` | 真实报错与异常访问 |
| 配置层 | `nginx -t` | 配置语法与路径是否正确 |

### 9.2 排查原则

1. **先分层,再深入**:先确定问题在哪一层,避免一上来就改配置
2. **用数据说话**:抓包、日志、监控数据比猜测可靠得多
3. **一次只改一个变量**:改完立即验证,否则无法判断哪次改动生效
4. **先恢复业务,再追根因**:紧急故障先封禁、限流、回滚,保住业务

---

# 附录:命令勘误对照

排查笔记里几处命令写法有误,已在上文修正,对照如下:

| 原写法 | 问题 | 正确写法 |
| --- | --- | --- |
| `tcpdump -i eth0 and port 6666` | 语法错误,`and` 不能作为起始条件 | `tcpdump -i eth0 -nn port 6666` |
| `tcpdump -i eth3`(无过滤) | 无过滤条件,数据量过大可能卡死终端 | `tcpdump -i eth0 -nn -c 100 port 6666` |
| `telent` | 拼写错误 | `telnet`(或 `nc -vz`) |
| `freee` | 拼写错误 | `free -h` |
| `ss nltp` | 参数缺少 `-` | `ss -nltp` |
| `iptable` | 命令名错误 | `iptables` 或 `firewall-cmd` |
| `iptables -A INPUT -S IP -j DROP` | `-S` 是列出规则,大写的用法错误 | `iptables -A INPUT -s IP -j DROP` |
| `systemctl enabled mysql` | 语法错误;服务名在 CentOS 7 也是 `mysqld` | `systemctl enable mysqld` |
| 脚本结尾输出“蒸棒” | 错别字 | 建议删除,或改为正常的执行完成提示 |
| 脚本无白名单判断 | 有误封内网 IP 的风险 | 增加 `is_whitelisted` 判断 |
| 脚本用 `iptables -nL \| grep` 判重 | 效率低且可能误匹配 | 改用 `iptables -C INPUT -s "$ip" -j DROP` |