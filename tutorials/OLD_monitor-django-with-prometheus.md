---
标题: 如何使用普罗米修斯（Prometheus）监控 Django 应用程序
摘要: 设置普罗米修斯来监控 Django 应用程序。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [普罗米修斯，Django，监控]
---

# 如何使用Prometheus监控Django应用

## 引言

Prometheus是一个开源的系统监控和告警工具包，可以用来轻松且低成本地监控基础设施和应用。在本教程中，我们将展示如何使用Prometheus监控Django应用。（即使你没有Django应用，我们也包括了一个可选步骤来创建一个，以便每个人都能跟随。）

## 前提条件

一台安装了以下软件的机器：

*   Python
*   [pip][get-pip]
*   一个本地运行的[Prometheus][get-prometheus]实例

<Highlight type="tip">
由于机器通常安装了多个版本的Python，在本教程中我们使用`python -m pip [foo]`语法而不是`pip [foo]`语法来调用pip。这是为了确保pip为我们使用的Python版本安装新组件。
</Highlight>

## 第1步 - 设置一个基本的Django应用（可选）

*（如果你已经有了一个Django应用，请跳过这一步。）*

### 安装Django

```bash
python -m pip install Django
```

*[更多关于Django的信息。][get-django]*

### 创建一个模板项目

导航到你想要创建项目的目录，并运行：

```bash
django-admin startproject mysite
```

这将在你当前的目录中创建一个`mysite`目录，看起来像这样（[更多信息][django-first-app]）：

```
mysite/
    manage.py
    mysite/
        __init__.py
        settings.py
        urls.py
        asgi.py
        wsgi.py
```

### 验证Django是否工作正常

切换到外部的`mysite`目录，并运行：

```bash
python manage.py runserver
```

你应该看到类似这样的输出：

````
执行系统检查...

系统检查没有发现问题（0个被抑制）。

您有未应用的迁移；在应用它们之前，您的应用可能无法正常工作。
运行'python manage.py migrate'来应用它们。

2020年3月10日 - 15:50:53
Django版本3.0，使用设置'mysite.settings'
在http://127.0.0.1:8000/启动开发服务器
用CONTROL-C退出服务器。
````

如果一切正常，那么请访问`http://127.0.0.1:8000/`。

（如果上述步骤没有成功，请查看[Django文档][django-first-app]进行故障排除。）

## 第2步 - 从你的Django应用中导出Prometheus风格的监控指标

我们使用[django-prometheus][get-django-prometheus]包来从我们的Django应用中导出Prometheus风格的监控指标。

### 安装django-prometheus

```bash
python -m pip install django-prometheus
```

### 修改`settings.py`和`urls.py`

在`settings.py`中添加：

```
INSTALLED_APPS = [
   ...
   'django_prometheus',
   ...
]

MIDDLEWARE = [
    'django_prometheus.middleware.PrometheusBeforeMiddleware',
    # 你所有的其他中间件都放在这里，包括默认的
    # 中间件如SessionMiddleware, CommonMiddleware,
    # CsrfViewmiddleware, SecurityMiddleware等。
    'django_prometheus.middleware.PrometheusAfterMiddleware',
]
```

在`urls.py`中，确保你在头部有这个：

```python
from django.conf.urls import include, path
```

然后在urlpatterns下添加这个：

```python
urlpatterns = [
    ...
    path('', include('django_prometheus.urls')),
]
```

### 验证指标正在被导出

重启应用并curl `/metrics`端点：

```bash
python manage.py runserver
curl localhost:8000/metrics
```

（或者，一旦你重启了应用，从你的网页浏览器访问[http://localhost:8000/metrics][localhost-metrics]。）

你应该看到类似这样的输出：

```
# HELP python_gc_objects_collected_total 在gc期间收集的对象
# TYPE python_gc_objects_collected_total counter
python_gc_objects_collected_total{generation="0"} 11716.0
python_gc_objects_collected_total{generation="1"} 1699.0
python_gc_objects_collected_total{generation="2"} 616.0
# HELP python_gc_objects_uncollectable_total 在GC期间发现的无法收集的对象
# TYPE python_gc_objects_uncollectable_total counter
python_gc_objects_uncollectable_total{generation="0"} 0.0
python_gc_objects_uncollectable_total{generation="1"} 0.0
python_gc_objects_uncollectable_total{generation="2"} 0.0
# HELP python_gc_collections_total 该代被收集的次数
# TYPE python_gc_collections_total counter
python_gc_collections_total{generation="0"} 7020.0
python_gc_collections_total{generation="1"} 638.0
python_gc_collections_total{generation="2"} 34.0
# HELP python_info Python平台信息
# TYPE python_info gauge
python_info{implementation="CPython",major="3",minor="8",patchlevel="0",version="3.8.0"} 1.0
...
```

## 第3步 - 将Prometheus指向你的Django应用指标端点

**（注意：本节假设你有一个本地运行的Prometheus实例。）**

### 更新`prometheus.yml`

在`scrape_configs:`下添加：

```yaml
- job_name: django
  scrape_interval: 10s
  static_configs:
  - targets:
    - localhost:8000
```

<Highlight type="note">
将`job_name`，`django`替换为你在Prometheus中为Django应用指标的首选前缀。例如，你可以使用`webapp`。
</Highlight>

### 重启Prometheus

```bash
./prometheus --config.file=prometheus.yml
```

### 验证Prometheus是否从你的Django应用中抓取指标：

一旦你在本地运行Prometheus，从你的网页浏览器访问[Prometheus表达式浏览器（运行在本地）][localhost-prom-browser]。

例如，你可以访问以下页面，它绘制了你的Django应用在过去一小时内收到的HTTP请求总数：

[Django HTTP请求图，本地提供][localhost-prom-example]

它应该看起来像这样：

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-tutorial-django-prometheus/prom_expression_browser.png" alt="Prometheus图显示最后一小时的总HTTP请求"/>

如果你想做更多的测试，多次访问你的Django应用并重新加载Prometheus表达式浏览器以确认它正在工作。同时，不妨探索Prometheus正在收集的其他Django指标。

## 第4步 - 监控应用的其他方面（可选）

[Django-prometheus][get-django-prometheus]非常强大，允许你轻松监控应用的其他方面，包括：

*   你的数据库
*   你的模型（例如，监控你的模型的创建/删除/更新速率）
*   你的缓存
*   你代码中的自定义指标

如何做到所有这些的更多信息[在这里][get-django-prometheus-more]。

## 后续步骤 [](#next-steps)

恭喜，你现在正在使用Prometheus监控你的Django应用。

[django-first-app]: https://docs.djangoproject.com/en/3.0/intro/tutorial01/ 
[get-django-prometheus-more]: https://github.com/korfuri/django-prometheus#monitoring-your-databases 
[get-django-prometheus]: https://github.com/korfuri/django-prometheus 
[get-django]: https://docs.djangoproject.com/en/3.0/topics/install/ 
[get-pip]: https://pip.pypa.io/en/latest/installing/#installing-with-get-pip-py 
[get-prometheus]: https://prometheus.io/docs/prometheus/latest/installation/ 
[localhost-metrics]: http://localhost:8000/metrics
[localhost-prom-browser]: http://localhost:9090/graph
[localhost-prom-example]: http://localhost:9090/graph?g0.range_input=1h&g0.stacked=1&g0.expr=django_http_requests_total_by_method_total&g0.tab=0

