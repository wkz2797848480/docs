---
title: Simulate an IoT sensor dataset
excerpt: Simulate an IOT dataset in your Timescale Cloud service
products: [cloud, mst, self_hosted]
keywords: [IoT, simulate]
---

import ImportPrerequisites from "versionContent/_partials/_migrate_import_prerequisites.mdx";

# Simulate an IoT sensor dataset

The Internet of Things (IoT) describes a trend where computing capabilities are embedded into IoT devices. That is, physical objects, ranging from light bulbs to oil wells. Many IoT devices collect sensor data about their environment and generate time-series datasets with relational metadata.

It is often necessary to simulate IoT datasets. For example, when you are 
testing a new system. This tutorial shows how to simulate a basic dataset in your $SERVICE_LONG, and then run simple queries on it.

To simulate a more advanced dataset, see [Time-series Benchmarking Suite (TSBS)][tsbs].

## Prerequisites

To follow this tutorial, you need to:

- [Create a target Timescale Cloud service][create-a-service].
- [Connect to your service][connect-to-service].

## Simulate a dataset

<Procedure>

To simulate a dataset, run the following queries:

1. **Create the `sensors` and `sensor_data` tables**:

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

1. **Convert `sensor_data` into a hypertable**:

    ```sql
    SELECT create_hypertable('sensor_data', 'time');
    ```

1. **Populate the `sensors` table**:

    ```sql
    INSERT INTO sensors (type, location) VALUES
    ('a','floor'),
    ('a', 'ceiling'),
    ('b','floor'),
    ('b', 'ceiling');
    ```

1. **Verify that the sensors have been added correctly**:

    ```sql
    SELECT * FROM sensors;
    ```

    Sample output:

    ```
     id | type | location
    ----+------+----------
      1 | a    | floor
      2 | a    | ceiling
      3 | b    | floor
      4 | b    | ceiling
    (4 rows)
    ```

1. **Generate and insert a dataset for all sensors:**

    ```sql
    INSERT INTO sensor_data (time, sensor_id, cpu, temperature)
    SELECT
      time,
      sensor_id,
      random() AS cpu,
      random()*100 AS temperature
    FROM generate_series(now() - interval '24 hour', now(), interval '5 minute') AS g1(time), generate_series(1,4,1) AS g2(sensor_id);
    ```

1. **Verify the simulated dataset**:

    ```sql
    SELECT * FROM sensor_data ORDER BY time;
    ```

    Sample output:

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

## Run basic queries 

After you simulate a dataset, you can run some basic queries on it. For example:

- Average temperature and CPU by 30-minute windows:

   ```sql
   SELECT
     time_bucket('30 minutes', time) AS period,
     AVG(temperature) AS avg_temp,
     AVG(cpu) AS avg_cpu
   FROM sensor_data
   GROUP BY period;
   ```
   
   Sample output:
   
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

- Average and last temperature, average CPU by 30-minute windows:

   ```sql
   SELECT
     time_bucket('30 minutes', time) AS period,
     AVG(temperature) AS avg_temp,
     last(temperature, time) AS last_temp,
     AVG(cpu) AS avg_cpu
   FROM sensor_data
   GROUP BY period;
   ```
   
   Sample output:
   
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

- Query the metadata:

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
   
   Sample output:
   
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

You have now successfully simulated and run queries on an IoI dataset. 

[create-a-service]: /getting-started/:currentVersion:/services/#create-a-timescale-cloud-service
[connect-to-service]: /getting-started/:currentVersion:/run-queries-from-console/
[tsbs]: https://github.com/timescale/tsbs