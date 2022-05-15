# 安装Prometheus Server

Prometheus基于Golang编写，编译后的软件包，不依赖于任何的第三方依赖。用户只需要下载对应平台的二进制包，解压并且添加基本的配置即可正常启动Prometheus Server。

## 从二进制包安装

对于非Docker用户，可以从https://prometheus.io/download/找到最新版本的Prometheus Sevrer软件包：

```shell
export VERSION=2.35.0
curl -LO  https://github.com/prometheus/prometheus/releases/download/v$VERSION/prometheus-$VERSION.linux-amd64.tar.gz
```

解压，并将Prometheus相关的命令，添加到系统环境变量路径即可：

```shell
tar -xzf prometheus-${VERSION}.linux-amd64.tar.gz
mkdir /data/prometheus -p
mv prometheus-${VERSION}.linux-amd64/ /usr/local/prometheus
```

解压后当前目录会包含默认的Prometheus配置文件promethes.yml:

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```

Promtheus作为一个时间序列数据库，其采集的数据会以文件的形式存储在本地中，默认的存储路径为`data/`，因此我们需要先手动创建该目录：

```
mkdir -p data
```

用户也可以通过参数`--storage.tsdb.path="data/"`修改本地数据存储的路径。

启动prometheus服务，其会默认加载当前路径下的prometheus.yaml文件：

./prometheus

正常的情况下，你可以看到以下输出内容：

```shell
level=info ts=2018-10-23T14:55:14.499484Z caller=main.go:554 msg="Starting TSDB ..."

level=info ts=2018-10-23T14:55:14.499531Z caller=web.go:397 component=web msg="Start listening for connections" address=0.0.0.0:9090

level=info ts=2018-10-23T14:55:14.507999Z caller=main.go:564 msg="TSDB started"

level=info ts=2018-10-23T14:55:14.508068Z caller=main.go:624 msg="Loading configuration file" filename=prometheus.yml

level=info ts=2018-10-23T14:55:14.509509Z caller=main.go:650 msg="Completed loading of configuration file" filename=prometheus.yml

level=info ts=2018-10-23T14:55:14.509537Z caller=main.go:523 msg="Server is ready to receive web requests."
```

## systemd接管prometheus

```shell
cat > /usr/lib/systemd/system/prometheus.service  <<EOF
[Unit]
Description=https://prometheus.io

[Service]
Restart=on-failure
ExecStart=/usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --storage.tsdb.path=/data/prometheus --web.console.libraries=/usr/local/prometheus/console_libraries --web.console.templates=/usr/local/prometheus/consoles

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable prometheus --now
```



## 使用容器安装

对于Docker用户，直接使用Prometheus的镜像即可启动Prometheus Server：

```shell
docker run -p 9090:9090 -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

启动完成后，可以通过[http://localhost:9090](http://localhost:9090/)访问Prometheus的UI界面：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPWdZfpgg-IxWhxja2D%2F-LPMFp6m_uMCLYKqaoBZ%2Fprometheus-ui-graph.png?generation=1540310329044095&alt=media)

<center style="font-size:14px;color:#000;">Prometheus UI</center> 

# 使用Node Exporter采集主机数据

## 安装Node Exporter

在Prometheus的架构设计中，Prometheus Server并不直接服务监控特定的目标，其主要任务负责数据的收集，存储并且对外提供数据查询支持。因此为了能够能够监控到某些东西，如主机的CPU使用率，我们需要使用到Exporter。Prometheus周期性的从Exporter暴露的HTTP服务地址（通常是/metrics）拉取监控样本数据。

从上面的描述中可以看出Exporter可以是一个相对开放的概念，其可以是一个独立运行的程序独立于监控目标以外，也可以是直接内置在监控目标中。只要能够向Prometheus提供标准格式的监控样本数据即可。

这里为了能够采集到主机的运行指标如CPU, 内存，磁盘等信息。我们可以使用[Node Exporter](https://github.com/prometheus/node_exporter)。

Node Exporter同样采用Golang编写，并且不存在任何的第三方依赖，只需要下载，解压即可运行。可以从https://prometheus.io/download/获取最新的node exporter版本的二进制包。

```shell
curl -OL https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
tar -xzf node_exporter-1.3.1.linux-amd64.tar.gz
```

systemd接管node_exporter

```shell
mkdir /usr/local/node_exporter
mv node_exporter-1.3.1.linux-amd64/node_exporter /usr/local/node_exporter/

cat > /usr/lib/systemd/system/prometheus.service  <<EOF
[Unit]
Description=node_exporter:9100

[Service]
Restart=on-failure
ExecStart=/usr/local/node_exporter/node_exporter 

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable node_exporter --now
```

启动成功后，可以看到以下输出：

```shell
INFO[0000] Listening on :9100                            source="node_exporter.go:76"
```

访问http://localhost:9100/可以看到以下页面：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp6oc_SZOU4__NeX%2Fnode_exporter_home_page.png?generation=1540136067584405&alt=media)

<center style="font-size:14px;color:#000;">Node Export</center> 

## 初始Node Exporter监控指标

访问http://localhost:9100/metrics，可以看到当前node exporter获取到的当前主机的所有监控数据，如下所示：

![image-20220514210459521](https://s2.loli.net/2022/05/15/4Cm8jzrnlcXeuaW.png)

<center style="font-size:14px;color:#000;">主机监控指标</center> 

每一个监控指标之前都会有一段类似于如下形式的信息：

```shell
# HELP node_cpu Seconds the cpus spent in each mode.
# TYPE node_cpu counter
node_cpu{cpu="cpu0",mode="idle"} 362812.7890625
# HELP node_load1 1m load average.
# TYPE node_load1 gauge
node_load1 3.0703125
```

其中HELP用于解释当前指标的含义，TYPE则说明当前指标的数据类型。在上面的例子中node_cpu的注释表明当前指标是cpu0上idle进程占用CPU的总时间，CPU占用时间是一个只增不减的度量指标，从类型中也可以看出node_cpu的数据类型是计数器(counter)，与该指标的实际含义一致。又例如node_load1该指标反映了当前主机在最近一分钟以内的负载情况，系统的负载情况会随系统资源的使用而变化，因此node_load1反映的是当前状态，数据可能增加也可能减少，从注释中可以看出当前指标类型为仪表盘(gauge)，与指标反映的实际含义一致。

除了这些以外，在当前页面中根据物理主机系统的不同，你还可能看到如下监控指标：

- node_boot_time：系统启动时间
- node_cpu：系统CPU使用量
- node*disk**：磁盘IO
- node*filesystem**：文件系统用量
- node_load1：系统负载
- node*memeory**：内存使用量
- node*network**：网络带宽
- node_time：当前系统时间
- go_*：node exporter中go相关指标
- process_*：node exporter自身进程相关运行指标

## 从Node Exporter收集监控数据

为了能够让Prometheus Server能够从当前node exporter获取到监控数据，这里需要修改Prometheus配置文件。编辑prometheus.yml并在scrape_configs节点下添加以下内容:

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  # 采集node exporter监控数据
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

重新启动Prometheus Server

访问[http://localhost:9090](http://localhost:9090/)，进入到Prometheus Server。如果输入“up”并且点击执行按钮以后，可以看到如下结果：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp6sd1HomUq2AhEt%2Fprometheus_ui_up_query.png?generation=1540136068422883&alt=media)

<center style="font-size:12px;color:#000;">Expression Browser</center> 

如果Prometheus能够正常从node exporter获取数据，则会看到以下结果：

```shell
up{instance="localhost:9090",job="prometheus"}    1
up{instance="localhost:9100",job="node"}    1
```

其中“1”表示正常，反之“0”则为异常。

使用PromQL查询监控数据



Prometheus UI是Prometheus内置的一个可视化管理界面，通过Prometheus UI用户能够轻松的了解Prometheus当前的配置，监控任务运行状态等。 通过`Graph`面板，用户还能直接使用`PromQL`实时查询监控数据：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp6uLzmEmN2Fa-ZF%2Fprometheus_ui_graph_query.png?generation=1540136068434550&alt=media)

Graph Query

切换到`Graph`面板，用户可以使用PromQL表达式查询特定监控指标的监控数据。如下所示，查询主机负载变化情况，可以使用关键字`node_load1`可以查询出Prometheus采集到的主机负载的样本数据，这些样本数据按照时间先后顺序展示，形成了主机负载随时间变化的趋势图表：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp6w2a-tNlAZqMYf%2Fnode_node1_graph.png?generation=1540136068457052&alt=media)

主机负载情况

PromQL是Prometheus自定义的一套强大的数据查询语言，除了使用监控指标作为查询关键字以为，还内置了大量的函数，帮助用户进一步对时序数据进行处理。例如使用`rate()`函数，可以计算在单位时间内样本数据的变化情况即增长率，因此通过该函数我们可以近似的通过CPU使用时间计算CPU的利用率：

```sql
rate(node_cpu[2m])
```

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp6ykiXFktbE7AoS%2Fnode_cpu_usage_by_cpu_and_mode.png?generation=1540136066903518&alt=media)

系统进程的CPU使用率

这时如果要忽略是哪一个CPU的，只需要使用without表达式，将标签CPU去除后聚合数据即可：

```sql
avg without(cpu) (rate(node_cpu[2m]))
```

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp7-zSOBoQPoKmGt%2Fnode_cpu_usage_by_mode.png?generation=1540136067957814&alt=media)

系统各mode的CPU使用率

那如果需要计算系统CPU的总体使用率，通过排除系统闲置的CPU使用率即可获得:

```sql
1 - avg without(cpu) (rate(node_cpu{mode="idle"}[2m]))
```

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp71s5xiaeewHj_D%2Fnode_cpu_usage_total.png?generation=1540136065700869&alt=media)

系统CPU使用率

通过PromQL我们可以非常方便的对数据进行查询，过滤，以及聚合，计算等操作。通过这些丰富的表达书语句，监控指标不再是一个单独存在的个体，而是一个个能够表达出正式业务含义的语言。



## 监控数据可视化

Prometheus UI提供了快速验证PromQL以及临时可视化支持的能力，而在大多数场景下引入监控系统通常还需要构建可以长期使用的监控数据可视化面板（Dashboard）。这时用户可以考虑使用第三方的可视化工具如Grafana，Grafana是一个开源的可视化平台，并且提供了对Prometheus的完整支持。

```shell
docker run -d -p 3000:3000 grafana/grafana
```

访问[http://localhost:3000](http://localhost:3000/)就可以进入到Grafana的界面中，默认情况下使用账户admin/admin进行登录。在Grafana首页中显示默认的使用向导，包括：安装、添加数据源、创建Dashboard、邀请成员、以及安装应用和插件等主要流程:

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp733ZL0hekE5YPo%2Fget_start_with_grafana2.png?generation=1540136069592465&alt=media)

Grafana向导

这里将添加Prometheus作为默认的数据源，如下图所示，指定数据源类型为Prometheus并且设置Prometheus的访问地址即可，在配置正确的情况下点击“Add”按钮，会提示连接成功的信息：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp75TqWkguvVkIKH%2Fadd_default_prometheus_datasource.png?generation=1540136069024088&alt=media)

添加Prometheus作为数据源

在完成数据源的添加之后就可以在Grafana中创建我们可视化Dashboard了。Grafana提供了对PromQL的完整支持，如下所示，通过Grafana添加Dashboard并且为该Dashboard添加一个类型为“Graph”的面板。 并在该面板的“Metrics”选项下通过PromQL查询需要可视化的数据：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp770M-VxTvT4XU4%2Ffirst_grafana_dashboard.png?generation=1540136067470329&alt=media)

第一个可视化面板

点击界面中的保存选项，就创建了我们的第一个可视化Dashboard了。 当然作为开源软件，Grafana社区鼓励用户分享Dashboard通过https://grafana.com/dashboards网站，可以找到大量可直接使用的Dashboard：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp79QVtMfJXHS6kb%2Fgrafana_dashboards.png?generation=1540136069596798&alt=media)

用户共享的Dashboard

Grafana中所有的Dashboard通过JSON进行共享，下载并且导入这些JSON文件，就可以直接使用这些已经定义好的Dashboard：

![img](https://2584451478-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-LBdoxo9EmQ0bJP2BuUi%2F-LPMFlGDFIX7wuLhSHx9%2F-LPMFp7BMwAjVVS2GA8a%2Fnode_exporter_dashboard.png?generation=1540136069618026&alt=media)

Host Stats Dashboard