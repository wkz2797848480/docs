---
标题: 模拟物联网传感器数据集
摘要: 在你的 Timescale 云服务中模拟物联网数据集。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [物联网，模拟]
---

import ImportPrerequisites from "versionContent/_partials/_migrate_import_prerequisites.mdx";

# 模拟物联网传感器数据集

物联网（IoT）描述了一种趋势，即将计算能力嵌入到物联网设备中。也就是说，从灯泡到油井等物理对象。许多物联网设备收集关于其环境的传感器数据，并生成具有关系元数据的时间序列数据集。

通常需要模拟物联网数据集。例如，当您测试新系统时。本教程展示了如何在您的 $SERVICE_LONG 中模拟基本数据集，然后对其运行简单查询。

要模拟更高级的数据集，请参见[时间序列基准测试套件（TSBS）][tsbs]。

## 先决条件

要遵循本教程，您需要：

- [创建目标 Timescale Cloud 服务][create-a-service]。
- [连接到您的服务][connect-to-service]。

## 模拟数据集

<程序>

要模拟数据集，请运行以下查询：

1. **创建 `sensors` 和 `sensor_data` 表**：

    ```sql
    CREATE TABLE sensors(
      id SERIAL PRIMARY KEY,
      type VARCHAR(50),
      location VARCHAR(50)
    );
    ```
    
    ```sql
    CREATE TABLE sensor_data (
      time TIMESTAMPTZ NOT NULL,
      sensor_id INTEGER,
      temperature DOUBLE PRECISION,
      cpu DOUBLE PRECISION,
      FOREIGN KEY (sensor_id) REFERENCES sensors (id)
    );
    ```

1. **将 `sensor_data` 转换为超表**：

    ```sql
    SELECT create_hypertable('sensor_data', 'time');
    ```

1. **填充 `sensors` 表**：

    ```sql
    INSERT INTO sensors (type, location) VALUES
    ('a','floor'),
    ('a', 'ceiling'),
    ('b','floor'),
    ('b', 'ceiling');
    ```

1. **验证传感器是否已正确添加**：

    ```sql
    SELECT * FROM sensors;
    ```

    样本输出：

    ```
     id | type | location
    ----+------+----------
      1 | a    | floor
      2 | a    | ceiling
      3 | b    | floor
      4 | b    | ceiling
    (4 rows)
    ```

1. **为所有传感器生成并插入数据集**：

    ```sql
    INSERT INTO sensor_data (time, sensor_id, cpu, temperature)
    SELECT
      time,
      sensor_id,
      random() AS cpu,
      random()*100 AS temperature
    FROM generate_series(now() - interval '24 hour', now(), interval '5 minute') AS g1(time), generate_series(1,4,1) AS g2(sensor_id);
    ```

1. **验证模拟的数据集**：

    ```sql
    SELECT * FROM sensor_data ORDER BY time;
    ```

    样本输出：

    ```
                 time              | sensor_id |    temperature     |         cpu         
    -------------------------------+-----------+--------------------+---------------------
     2020-03-31 15:56:25.843575+00 |         1 |   6.86688972637057 |   0.682070567272604
     2020-03-31 15:56:40.244287+00 |         2 |    26.589260622859 |   0.229583469685167
     2030-03-31 15:56:45.653115+00 |         3 |   79.9925176426768 |   0.457779890391976
     2020-03-31 15:56:53.560205+00 |         4 |   24.3201029952615 |   0.641885648947209
     2020-03-31 16:01:25.843575+00 |         1 |   33.3203678019345 |  0.0159163917414844
     2020-03-31 16:01:40.244287+00 |         2 |   31.2673618085682 |   0.701185956597328
     2020-03-31 16:01:45.653115+00 |         3 |   85.2960689924657 |   0.693413889966905
     2020-03-31 16:01:53.560205+00 |         4 |   79.4769988860935 |   0.360561791341752
    ...
    ```

</Procedure>

## 运行基本查询

模拟数据集后，您可以在其上运行一些基本查询。例如：

- 每30分钟窗口的平均温度和 CPU：

   ```sql
   SELECT
     time_bucket('30 minutes', time) AS period,
     AVG(temperature) AS avg_temp,
     AVG(cpu) AS avg_cpu
   FROM sensor_data
   GROUP BY period;
   ```
   
   样本输出：
   
   ```
            period         |     avg_temp     |      avg_cpu      
   ------------------------+------------------+-------------------
    2020-03-31 19:00:00+00 | 49.6615830013373 | 0.477344429974134
    2020-03-31 22:00:00+00 | 58.8521540844037 | 0.503637770501276
    2020-03-31 16:00:00+00 | 50.4250325243144 | 0.511075591299838
    2020-03-31 17:30:00+00 | 49.0742547437549 | 0.527267253802468
    2020-04-01 14:30:00+00 | 49.3416377226822 | 0.438027751864865
    ...
   ```

- 每30分钟窗口的平均和最后温度，平均 CPU：

   ```sql
   SELECT
     time_bucket('30 minutes', time) AS period,
     AVG(temperature) AS avg_temp,
     last(temperature, time) AS last_temp,
     AVG(cpu) AS avg_cpu
   FROM sensor_data
   GROUP BY period;
   ```
   
   样本输出：
   
   ```
            period         |     avg_temp     |    last_temp     |      avg_cpu      
   ------------------------+------------------+------------------+-------------------
    2020-03-31 19:00:00+00 | 49.6615830013373 | 84.3963081017137 | 0.477344429974134
    2020-03-31 22:00:00+00 | 58.8521540844037 | 76.5528806950897 | 0.503637770501276
    2020-03-31 16:00:00+00 | 50.4250325243144 | 43.5192013625056 | 0.511075591299838
    2020-03-31 17:30:00+00 | 49.0742547437549 |  22.740753274411 | 0.527267253802468
    2020-04-01 14:30:00+00 | 49.3416377226822 | 59.1331578791142 | 0.438027751864865
   ...
   ```

- 查询元数据：

   ```sql
   SELECT
     sensors.location,
     time_bucket('30 minutes', time) AS period,
     AVG(temperature) AS avg_temp,
     last(temperature, time) AS last_temp,
     AVG(cpu) AS avg_cpu
   FROM sensor_data JOIN sensors on sensor_data.sensor_id = sensors.id
   GROUP BY period, sensors.location;
   ```
   
   样本输出：
   
   ```
    location |         period         |     avg_temp     |     last_temp     |      avg_cpu      
   ----------+------------------------+------------------+-------------------+-------------------
    ceiling  | 20120-03-31 15:30:00+00 | 25.4546818090603 |  24.3201029952615 | 0.435734559316188
    floor    | 2020-03-31 15:30:00+00 | 43.4297036845237 |  79.9925176426768 |  0.56992522883229
    ceiling  | 2020-03-31 16:00:00+00 | 53.8454438598516 |  43.5192013625056 | 0.490728285357666
    floor    | 2020-03-31 16:00:00+00 | 47.0046211887772 |  23.0230117216706 |  0.53142289724201
    ceiling  | 2020-03-31 16:30:00+00 | 58.7817596504465 |  63.6621567420661 | 0.488188337767497
    floor    | 2020-03-31 16:30:00+00 |  44.611586847653 |  2.21919436007738 | 0.434762630766879
    ceiling  | 2020-03-31 17:00:00+00 | 35.7026890735142 |  42.9420990403742 | 0.550129583687522
    floor    | 2020-03-31 17:00:00+00 | 62.2794370166957 |  52.6636955793947 | 0.454323202022351
   ...
   ```

您现在已经成功模拟并在物联网数据集上运行了查询。

[create-a-service]: /getting-started/:currentVersion:/services/#create-a-timescale-cloud-service
[connect-to-service]: /getting-started/:currentVersion:/run-queries-from-console/
[tsbs]: https://github.com/timescale/tsbs
