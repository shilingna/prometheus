## 下载

```shell
wget https://github.com/ncabatoff/process-exporter/releases/download/v0.7.10/process-exporter-0.7.10.linux-amd64.tar.gz
tar -zxvf https://github.com/ncabatoff/process-exporter/releases/download/v0.7.10/process-exporter-0.7.10.linux-amd64.tar.gz
mv process-exporter-0.7.10.linux-amd64/ /usr/local/process-exporter
cd !$
```



## 配置文件

```bash
cat > process_name.yaml << EOF

process_names:
#  - name: "{{.Comm}}"
#    cmdline:
#    - '.+'

  - name: "{{.Matches}}"
    cmdline:
    - 'nginx'

  - name: "{{.Matches}}"
    cmdline:
    - '/opt/atlassian/confluence/bin/tomcat-juli.jar'

  - name: "{{.Matches}}"
    cmdline:
    - 'vsftpd'

  - name: "{{.Matches}}"
    cmdline:
    - 'redis-server'
EOF
```

### 配置文件格式

cmdline: 所选进程的唯一标识，ps -ef 可以查询到。如果改进程不存在，则不会有该进程的数据采集到。

```bash
例如：> ps -ef | grep redis
redis 4287 4127 0 Oct31 ? 00:58:12 redis-server *:6379
```

| {{.Comm}}     | groupname=”redis-server”                 | exe或者sh文件名称          |
| :------------ | :--------------------------------------- | :------------------------- |
| {{.ExeBase}}  | groupname=”redis-server *:6379”          | /                          |
| {{.ExeFull}}  | groupname=”/usr/bin/redis-server *:6379” | ps中的进程完成信息         |
| {{.Username}} | groupname=”redis”                        | 使用进程所属的用户进行分组 |
| {{.Matches}}  | groupname=”map[:redis]”                  | 表示配置到关键字“redis”    |



## systemd接管进程

```bash
vim /usr/lib/systemd/system/process_exporter.service
 
[Unit]
Description=Prometheus exporter for processors metrics, written in Go with pluggable metric collectors.
Documentation=https://github.com/ncabatoff/process-exporter
After=network.target
 
[Service]
Type=simple
User=prometheus
WorkingDirectory=/usr/local/process-exporter
ExecStart=/usr/local/process-exporter/process-exporter -config.path=/usr/local/process-exporter/process_name.yaml
Restart=on-failure
 
[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl start process_exporter
systemctl enable process_exporter
```



## prometheus接管

```bash
- job_name: 'doog_dev_prometheus'
  scrape_interval: 10s
  honor_labels: true
  metrics_path: '/metrics'

  static_configs:
  - targets: ['192.168.10.73:9090','192.168.10.73:9100']
    labels: {cluster: 'dev',type: 'basic',env: 'dev',job: 'prometheus',export: 'prometheus'}
  - targets: ['192.168.10.73:9256']
    labels: {cluster: 'dev',type: 'process',env: 'dev',job: 'prometheus',export: 'process_exporter'}

```

