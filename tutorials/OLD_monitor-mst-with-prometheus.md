---
标题: 如何为 TimescaleDB 托管服务设置普罗米修斯（Prometheus）端点
摘要: 使用普罗米修斯来监控你的 TimescaleDB 托管服务。
产品: [管理服务技术（MST）]
关键词: [普罗米修斯，监控，集成]
---

# 如何为托管的TimescaleDB数据库设置Prometheus端点

你可以通过使用[Prometheus][get-prometheus]来监控托管的TimescaleDB数据库，以获得更多关于其性能的洞察，Prometheus是一个流行的开源基于指标的系统监控解决方案。本教程将指导你为在[TimescaleDB托管服务][timescale-mst]上运行的数据库设置Prometheus端点。

这将公开来自[node_exporter][node-exporter-metrics]以及[pg_stats][pg-stats-metrics]的指标。

### 前提条件

为了进行本教程，你需要一个TimescaleDB数据库的托管服务。
要创建一个，请参考这些说明，了解如何[开始使用TimescaleDB的托管服务][timescale-mst-get-started]。

### 第1步：启用Prometheus服务集成

在导航栏中，选择“服务集成”。导航到服务集成，如下图所示。

<img class="main-content__illustration" src="https://s3.amazonaws.com/docs.iobeam.com/images/Prometheus_service_integration_0.png" alt="服务集成菜单选项"/>

这为你提供了添加Prometheus集成点的选项。
选择加号图标添加一个新的端点，并给它一个你选择的名称。
我们将我们的命名为`endpoint_dev`。

<img class="main-content__illustration" src="https://s3.amazonaws.com/docs.iobeam.com/images/Prometheus_service_integration_1.png" alt="在Timescale上创建Prometheus端点"/>

此外，请注意，为了访问服务，你将获得基本的认证信息和端口号。
这在设置你的Prometheus安装时使用，在`prometheus.yml`配置文件中。
这使你能够使这个托管的TimescaleDB端点成为Prometheus抓取的目标。

这是一个你可以在设置Prometheus安装时使用的示例配置文件，将目标端口、IP地址、用户名和密码替换为你的托管TimescaleDB实例的信息：

```yaml
# prometheus.yml用于监控Timescale实例
global:
  scrape_interval: 10s
  evaluation_interval: 10s
scrape_configs:
  - job_name: prometheus
    scheme: https
    static_configs:
      - targets: ['{TARGET_IP}:{TARGET_PORT}']
    tls_config:
      insecure_skip_verify: true
    basic_auth:
      username: {ENDPOINT_USERNAME}
      password: {ENDPOINT_PASSWORD}
remote_write:
  - url: "http://{ADAPTER_IP}:9201/write"
remote_read:
  - url: "http://{ADAPTER_IP}:9201/read"
```

### 第2步：将Prometheus端点与托管服务关联

接下来，我们希望将我们的Prometheus端点与我们的Timescale托管服务关联起来。
使用导航菜单，选择你想要监控的服务并点击“概览”标签。

导航到“服务集成”部分并点击“管理集成”按钮。

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-prometheus-endpoint-tutorial/Prometheus_service_integrations_4.png" alt="在托管服务上管理服务集成"/>

找到Prometheus集成选项并选择“使用Prometheus”。

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-prometheus-endpoint-tutorial/Prometheus_service_integration_2.png" alt="选择要集成的Prometheus集成"/>

接下来，选择第1步中创建的端点名称作为你想要与此服务一起使用的端点，然后点击“启用”按钮。
你可以使用相同的端点用于多个服务，或者为你想保持分开的服务使用不同的端点。

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-prometheus-endpoint-tutorial/Prometheus_service_integration_3.png" alt="选择要集成的Prometheus端点名称"/>

要检查这是否成功，请返回到托管服务的服务集成部分，并检查是否出现了“活跃”标志，以及你将服务与之关联的端点名称。

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-prometheus-endpoint-tutorial/Prometheus_service_integration_5.png" alt="成功！活跃的Prometheus端点名称"/>

恭喜你，你已成功在TimescaleDB的托管服务上设置了Prometheus端点！

[get-prometheus]: https://prometheus.io 
[node-exporter-metrics]: https://github.com/prometheus/node_exporter 
[pg-stats-metrics]: https://www.postgresql.org/docs/current/monitoring-stats.html 
[timescale-mst]: https://www.timescale.com/products 
[timescale-mst-get-started]: /mst/:currentVersion:/about-mst

