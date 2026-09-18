四大常用组件+两个拓展组件

Zabbix Server：核心组件，负责接收Agent发送的监控数据，处理数据，生成图表，发送报警等
一台Zabbix只能有一个server

Zabbix Agent：负责收集被监控主机上的数据，发送给Zabbix Server。分主动模式和被动模式，安装在被管理服务器上

数据库：支持MySQL，PostgreSQL，MogoDB，用于存储配置信息，采集到的数据，报警信息等

Web前端：交互界面，部署在Web服务器上，支持Nginx和Apache等服务器

拓展：
Zabbix Proxy：代理组件，用于分担Server的负载，部署在子网内，收集数据后发送给Server
Zabbix API


常见问题：
Agent必须装吗？不一定，也可以用SNMP、IPMI,JMX
Server可以有多个吗？一个监控域只能有一个server，但可以做主备和高可用集群
Proxy是必需的吗？小规模不需要，分布式跨地域推荐proxy
数据库的选择问题：Mysql最常用


