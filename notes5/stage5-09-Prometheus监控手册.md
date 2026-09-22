Prometheus优势
Prometheus 在 云原生 和 容器化场景里是事实标准
是 CNCF（云原生基金会）核心项目,未来趋势
用 Kubernetes（K8s）的公司，都是用 Prometheus

Prometheus作用:
1. Prometheus 是一套 时间序列数据库 + 数据收集器 + 报警系统
2. 最大特点：拉取（Pull）模式，自己去各个目标机器采集数据
3. 支持非常灵活的监控指标（Metrics），可以采集任何你想要的“数值变化”


Prometheus 的核心组件
Prometheus Server:	拉数据、存数据、处理规则
Exporter:			负责把机器或应用的指标暴露出来
Alertmanager:		报警管理器，出问题时发通知（邮件、企业微信等）
Grafana:				可视化面板


Prometheus 的工作流程
Exporter 在被监控机器上暴露指标
Prometheus Server 定时去访问这些 URL，把数据拉回来
拉回来的数据存到自己的时序数据库里
有规则时触发报警，通知 Alertmanager
通过Grafana 连接 Prometheus，把数据变成好看的图



Prometheus安装
1. 创建 Prometheus 用户
   sudo useradd --no-create-home --shell /bin/false prometheus

2. 创建目录
   sudo mkdir /etc/prometheus
   sudo mkdir /var/lib/prometheus

3. 下载 Prometheus
   去官网 [https://prometheus.io/download/]
找到合适的版本
   例如:
   wget  https://github.com/prometheus/prometheus/releases/download/v2.53.5/prometheus-2.53.5.linux-amd64.tar.gz

4. 解压并放将对应文件到合适位置
   tar -xvf prometheus-2.53.5.linux-amd64.tar.gz
   cd prometheus-2.53.5.linux-amd64

sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/

sudo cp -r consoles/console_libraries/ /etc/prometheus
sudo cp prometheus.yml /etc/prometheus/

5. 修改权限
   sudo chown prometheus:prometheus /usr/local/bin/prometheus
   sudo chown prometheus:prometheus /usr/local/bin/promtool
   sudo chown -R prometheus:prometheus /etc/prometheus
   sudo chown -R prometheus:prometheus /var/lib/prometheus

6. 配置 prometheus.yml
   常用参数
   global:
   scrape_interval: 15s  # 每15秒拉取一次

scrape_configs:
- job_name: 'prometheus'
  static_configs:
    - targets: ['localhost:9090']		#目标主机

7. 创建 systemd 启动文件
   sudo vim /etc/systemd/system/prometheus.service
   [Unit]
   Description=Prometheus
   Wants=network-online.target
   After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.path=/var/lib/prometheus/

[Install]
WantedBy=multi-user.target

8. 启动并设置开机自启
   systemctl daemon-reload
   systemctl enable prometheus

9. 浏览器访问验证
   打开浏览器访问：
   http://<你的服务器IP>:9090
   出现 Prometheus 的界面，就成功

systemctl start prometheus
