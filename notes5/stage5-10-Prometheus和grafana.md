一 Grafana 是一款开源的可视化工具，专门用来把监控数据做成炫酷的图表、仪表盘

二 Grafana 的核心特点:
数据源（Data Source):告诉 Grafana 去哪里找数据
Dashboard（仪表盘）:各种指标拼在一起的图形页面
Panel（面板）: Dashboard 里的一小块图，比如 CPU 使用率、内存曲线
可分享: 做完了可以生成分享链接


三 Grafana 安装示例（CentOS7）
1.添加 Grafana 仓库
sudo cat >/etc/yum.repos.d/grafana.repo <<EOF
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
EOF

2. 安装 Grafana
   sudo yum install -y grafana

3. 启动 Grafana
   sudo systemctl start grafana-server
   sudo systemctl enable grafana-server

4. 打开 Grafana 页面
   默认端口是 `3000`
   浏览器访问：

http://<你的IP>:3000

5. 默认登录账号
* 用户名：`admin`
* 密码：`admin`
  第一次登录会要求改密码，改掉就行

四 Grafana 连接 Prometheus
1 登录 Grafana 后台
左侧菜单 →  Configuration → Data Sources
2 点击 `Add data source`
选择 **Prometheus**
3. 配置 Prometheus 地址
   在 URL 里填：
   http://<Prometheus IP>:9090

五 导入官方 Dashboard
进入 Dashboard → Import
输入官方模板 ID，比如 `1860`
点击 Load
选好 Data Source（选你的 Prometheus）
点击Import

五 部署 node_exporter（被监控机）
在要监控的 Linux 机器上安装node_export:
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz

tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz

cp node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/

创建 systemd 服务：

vim /etc/systemd/system/node_exporter.service
内容：

[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target

启动：
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
默认监听 `9100` 端口

六 把被监控机加到 Prometheus 配置
修改 `/etc/prometheus/prometheus.yml`：

scrape_configs:
- job_name: 'node'
  static_configs:
    - targets: ['目标机 IP:9100']
      然后重启 Prometheus：
      sudo systemctl restart prometheus
      再浏览器里去 Targets 页面查看