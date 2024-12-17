---
标题: 为 TimescaleDB 创建数据应用程序编程接口（API）
摘要: 使用亚马逊网络服务（AWS）Lambda 创建一个应用程序编程接口（API），以便从你的 TimescaleDB 应用程序中获取数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，AWS Lambda，psycopg2，Pandas，GitHub 操作，管道]
---

# 创建TimescaleDB的数据API

本教程介绍如何创建一个API来从您的TimescaleDB实例中获取数据。它使用API网关触发一个Lambda函数，该函数随后从TimescaleDB获取请求的数据并以JSON格式返回。

## 从Lambda连接到TimescaleDB

要连接到TimescaleDB实例，您需要使用数据库连接器库。本教程使用[`psycopg2`][psycopg2]。

`psycopg2`数据库连接器不是标准Python库的一部分，也不包含在AWS Lambda中，因此您需要手动将该库包含在您的部署包中以使其可用。本教程使用[Lambda Layers][lambda-layers]来包含`psycopg2`。Lambda Layer是一个包含额外代码的存档，例如库或依赖项。Layers帮助您在函数代码中使用外部库，否则这些库将不可用。

此外，`psycopg2`需要与静态链接的库一起构建和编译，这是您无法直接在Lambda函数或层中完成的。解决这个问题的一个方法是下载[编译好的库版本][lambda-psycopg2]并将其用作Lambda Layer。

<Procedure>

### 将psycopg2库添加为Lambda层

1. 下载并解压编译好的`psycopg2`库：

    ```bash
    wget https://github.com/jkehler/awslambda-psycopg2/archive/refs/heads/master.zip 
    unzip master.zip
    ```

1.  在您下载库的目录中，将`psycopg2`文件复制到一个名为`/python/psycopg2/`的新目录中。确保您复制的目录与您的Python版本相匹配：

    ```bash
    cd awslambda-psycopg2-master/
    mkdir -p python/psycopg2/
    cp -r psycopg2-3.8/* python/psycopg2/
    ```

1.  压缩`python`目录并将压缩文件作为Lambda层上传：

  ```bash
  zip -r psycopg2_layer.zip python/
  aws lambda publish-layer-version --layer-name psycopg2 \
  --description "psycopg2 for Python3.8" --zip-file fileb://psycopg2_layer.zip \
  --compatible-runtimes python3.8
  ```

1.  在AWS Lambda控制台中检查您的`psycopg2`是否已作为Lambda层上传：
    ![aws layers](https://assets.timescale.com/docs/images/tutorials/aws-lambda-tutorial/layers.png) 

</Procedure>

## 创建函数以从数据库获取并返回数据

当层对您的Lambda函数可用时，您可以创建一个API来返回数据库中的数据。本节向您展示如何创建返回数据库数据的Python函数并将其上传到AWS Lambda。

<Procedure>

### 创建函数以从数据库获取并返回数据

1.  创建一个名为`timescaledb_api`的新目录，用于存储函数代码，并切换到新目录：

    ```bash
    mkdir timescaledb_api
    cd timescaledb_api
    ```

1.  在新目录中创建一个名为`function.py`的新函数，内容如下：

    ```python
    import json
    import psycopg2
    import psycopg2.extras
    import os

    def lambda_handler(event, context):

      db_name = os.environ['DB_NAME']
      db_user = os.environ['DB_USER']
      db_host = os.environ['DB_HOST']
      db_port = os.environ['DB_PORT']
      db_pass = os.environ['DB_PASS']

      conn = psycopg2.connect(user=db_user, database=db_name, host=db_host,
                           password=db_pass, port=db_port)

      sql = "SELECT * FROM stocks_intraday"

      cursor = conn.cursor(cursor_factory=psycopg2.extras.DictCursor)

      cursor.execute(sql)

      result = cursor.fetchall()
      list_of_dicts = []
      for row in result:
        list_of_dicts.append(dict(row))

      return {
       'statusCode': 200,
       'body': json.dumps(list_of_dicts, default=str),
       'headers': {
           "Content-Type": "application/json"
           }
      }
      ```

</Procedure>

## 将函数上传到AWS Lambda

当您创建了函数后，您可以将Python文件压缩并使用`create-function` AWS命令将其上传到Lambda。

<Procedure>

## 将函数上传到AWS Lambda

1.  在命令提示符下，压缩函数目录：

    ```bash
    zip function.zip function.py
    ```

1.  上传函数：

    ```bash
    aws lambda create-function --function-name simple_api_function \
    --runtime python3.8 --handler function.lambda_handler \
    --role <ARN_LAMBDA_ROLE> --zip-file fileb://function.zip \
    --layers <LAYER_ARN>
    ```

1.  您可以通过在AWS控制台使用此命令检查函数是否已正确上传：
    ![aws lambda uploaded](https://assets.timescale.com/docs/images/tutorials/aws-lambda-tutorial/lambda_function.png) 
1.  如果您对函数代码进行了更改，您需要再次压缩文件并使用`update-function-code`命令上传更改：

    ```bash
    zip function.zip function.py
    aws lambda update-function-code --function-name simple_api_function --zip-file fileb://function.zip
    ```

</Procedure>

## 将数据库配置添加到AWS Lambda

在您可以使用函数之前，您需要确保它可以连接到数据库。在上面的Python代码中，您指定了从环境变量中检索值，您还需要在Lambda环境中指定这些值。

<Procedure>

### 使用环境变量将数据库配置添加到AWS Lambda

1.  创建一个包含函数所需变量的JSON文件：

    ```json
    {
        "Variables": {"DB_NAME": "db",
          "DB_USER": "user",
          "DB_HOST": "host",
          "DB_PORT": "5432",
          "DB_PASS": "pass"}
    }
    ```

1.  上传您的连接详细信息。在这个示例中，包含变量的JSON文件保存在`file://env.json`：

    ``` bash
    aws lambda update-function-configuration \
    --function-name simple_api_function --environment file://env.json
    ```

1.  当配置上传到AWS Lambda后，您可以在函数中使用`os.environ`参数访问变量：

    ```python
    import os
    config = {'DB_USER': os.environ['DB_USER'],
          'DB_PASS': os.environ['DB_PASS'],
          'DB_HOST': os.environ['DB_HOST'],
          'DB_PORT': os.environ['DB_PORT'],
          'DB_NAME': os.environ['DB_NAME']}
    ```

</Procedure>

## 测试数据库连接

当您的函数代码与数据库连接详细信息一起上传后，您可以检查它是否检索到您期望的数据。

<Procedure>

### 测试数据库连接

1.  调用函数。确保包括函数的名称，并为输出文件提供一个名称。在这个示例中，输出文件称为`output.json`：

    ```bash
    aws lambda invoke --function-name simple_api_function output.json
    ```

1.  如果您的函数工作正常，您的输出文件看起来像这样：

    ```json
    {
      "statusCode": 200,
      "body": "[
                {
                  \"bucket_day\": \"2021-02-01 00:00:00\",
                  \"symbol\": \"AAPL\",
                  \"avg_price\": 135.32576933380264,
                  \"max_price\": 137.956910987,
                  \"min_price\": 131.131547781
                },
                {
                  \"bucket_day\": \"2021-01-18 00:00:00\",
                  \"symbol\": \"AAPL\",
                  \"avg_price\": 136.7006897398394,
                  \"max_price\": 144.628477898,
                  \"min_price\": 126.675666886
                },
                {
                  \"bucket_day\": \"2021-05-24 00:00:00\",
                  \"symbol\": \"AAPL\",
                  \"avg_price\": 125.4228325920157,
                  \"max_price\": 128.32,
                  \"min_price\": 123.21
                },
                ...
              ]",
      "headers": {
      "Content-Type": "application/json"
      }
    }
    ```

</Procedure>

## 创建一个新的API网关

现在您已经确认Lambda函数可以正常工作，您可以创建API网关。在AWS术语中，您正在设置一个[自定义Lambda集成][custom-lambda-integration]。

<Procedure>

### 创建一个新的API网关

1.  创建API。在这个例子中，新的API被称为`TestApiTimescale`。注意响应中的`id`字段，您稍后需要使用它来进行更改：

    ```bash
    aws apigateway create-rest-api --name 'TestApiTimescale' --region us-east-1
    {
      "id": "4v5u26yw85",
      "name": "TestApiTimescale2",
      "createdDate": "2021-08-23T13:21:13+02:00",
      "apiKeySource": "HEADER",
      "endpointConfiguration": {
          "types": [
              "EDGE"
          ]
      },
      "disableExecuteApiEndpoint": false
    }
    ```

1.  检索根资源的`id`，以添加一个新的GET端点：

    ```bash
    aws apigateway get-resources --rest-api-id <API_ID> --region us-east-1
    {
      "items": [
        {
            "id": "hs26aaaw56",
            "path": "/"
        },
        {
            "id": "r9cakv",
            "parentId": "hs26aaaw56",
            "pathPart": "ticker",
            "path": "/ticker",
            "resourceMethods": {
                "GET": {}
            }
        }
      ]
    }
    ```

1.  创建一个新的资源。在这个例子中，新资源被称为`ticker`：

    ```bash
    aws apigateway create-resource --rest-api-id <API_ID> \
    --region us-east-1 --parent-id <RESOURCE_ID> --path-part ticker
    {
      "id": "r9cakv",
      "parentId": "hs26aaaw56",
      "pathPart": "ticker",
      "path": "/ticker"
    }
    ```

1.  为根资源创建一个GET请求：

    ```bash
    aws apigateway put-method --rest-api-id <API_ID> \
    --region us-east-1 --resource-id <RESOURCE_ID> \
    --http-method GET --authorization-type "NONE" \
    --request-parameters method.request.querystring.symbol=false
    ```

1.  为`GET /ticker?symbol={symbol}`的方法请求设置一个`200 OK`响应：

    ```bash
    aws apigateway put-method-response --region us-east-1 \
    --rest-api-id <API_ID> --resource-id <RESOURCE_ID> \
    --http-method GET --status-code 200
    ```

1.  将API网关连接到Lambda函数：

    ```bash
    aws apigateway put-integration --region us-east-1 \
    --rest-api-id <API_ID> --resource-id <RESOURCE_ID> \
    --http-method GET --type AWS --integration-http-method POST \
    --uri <ARN_LAMBDA_FUNCTION> \
    --request-templates file://path/to/integration-request-template.json
    ```

1.  将Lambda函数的输出作为`200 OK`响应传递给客户端：

    ```bash
    aws apigateway put-integration-response --region us-east-1 \
    --rest-api-id <API_ID> / --resource-id <RESOURCE_ID> \
    --http-method GET --status-code 200 --selection-pattern ""
    ```

1.  部署API：

    ```bash
    aws apigateway create-deployment --rest-api-id <API_ID> --stage-name test
    ```

</Procedure>

## 测试API

您可以通过向端点发送GET请求来测试API是否正常工作，使用`curl`：

```bash
curl 'https://hlsu4rwrkl.execute-api.us-east-1.amazonaws.com/test/ticker?symbol=MSFT' 
[
   {
     "time": "2021-07-12 20:00:00",
     "price_open": 277.31,
     "price_close": 277.31,
     "price_low": 277.31,
     "price_high": 277.31,
     "trading_volume": 342,
     "symbol": "MSFT"
   }
]
```

如果一切正常，您将看到Lambda函数的输出。在这个例子中，它是MSFT（微软）的最新股票价格，以JSON格式显示。

[psycopg2]: https://www.psycopg.org/docs/ 
[lambda-layers]: https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html 
[lambda-psycopg2]: https://github.com/jkehler/awslambda-psycopg2 
[custom-lambda-integration]: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-custom-integrations.html 

## 创建一个Lambda函数以将数据插入数据库

当您为您的数据库创建了`GET` API后，您可以创建一个`POST` API。这允许您使用JSON有效载荷将数据插入数据库。

<Procedure>

### 创建一个Lambda函数以将数据插入数据库

1.  创建一个名为`insert_function.py`的新函数，内容如下：

    ```python
    import json
    import psycopg2
    import psycopg2.extras
    from psycopg2.extras import execute_values
    import os
    from typing import Dict

    def lambda_handler(event, context):

        db_name = os.environ['DB_NAME']
        db_user = os.environ['DB_USER']
        db_host = os.environ['DB_HOST']
        db_port = os.environ['DB_PORT']
        db_pass = os.environ['DB_PASS']

        conn = psycopg2.connect(user=db_user, database=db_name, host=db_host,
                              password=db_pass, port=db_port)

        cursor = conn.cursor()
        sql = "INSERT INTO stocks_intraday VALUES %s"

        records = json.loads(event["body"]).get("records")
        if  isinstance(records, Dict):
            values = [[value for value in records.values()], ]
        else:
            values = [[value for value in item.values()] for item in records]
        execute_values(cursor, sql, values)
        conn.commit()
        conn.close()

        return {
            'statusCode': 200,
            'body': json.dumps(event, default=str),
            'headers': {
                "Content-Type": "application/json"
                }
        }

      ```

1.  将函数上传到AWS Lambda：

    ```bash
    zip insert_function.zip insert_function.py
    aws lambda create-function --function-name insert_function \
    --runtime python3.8 --handler function.lambda_handler \
    --role <ARN_LAMBDA_ROLE> --zip-file fileb://insert_function.zip
    ```

1.  创建一个新的API网关，称为`InsertApi`：

    ```bash
    aws apigateway create-rest-api --name 'InsertApi' --region us-east-1
    ```

1.  检索根资源的`id`，并添加一个新的POST端点：

    ```bash
    aws apigateway get-resources --rest-api-id <API_ID> --region us-east-1
    {
      "items": [
        {
            "id": "hs26aaaw56",
            "path": "/"
        },
        {
            "id": "r9cakv",
            "parentId": "hs26aaaw56",
            "pathPart": "ticker",
            "path": "/ticker",
            "resourceMethods": {
                "GET": {}
            }
        }
      ]
    }
    ```

1.  创建一个新的资源。在这个例子中，新资源被称为`insert`：

    ```bash
    aws apigateway create-resource --rest-api-id <API_ID> --region us-east-1
    --parent-id <RESOURCE_ID> --path-part insert
    {
        "id": "arabc2",
        "parentId": "r5fc0ufn0h",
        "pathPart": "insert",
        "path": "/insert"
    }
    ```

1.  为`insert`资源创建一个POST请求：

    ```bash
    aws apigateway put-method --rest-api-id <API_ID> --region us-east-1 \
    --resource-id <RESOURCE_ID> --http-method POST --authorization-type "NONE"
    ```

1.  为`POST /insert`的方法请求设置一个`200 OK`响应：

    ```bash
    aws apigateway put-method-response --region us-east-1 \
    --rest-api-id <API_ID> --resource-id <RESOURCE_ID> \
    --http-method POST --status-code 200
    ```

1.  将API网关连接到Lambda函数：

    ```bash
    aws apigateway put-integration --region us-east-1 \
    --rest-api-id <API_ID> --resource-id <RESOURCE_ID> \
    --http-method POST --type AWS --integration-http-method POST \
    --uri <ARN_LAMBDA_FUNCTION> \
    --request
-templates '{ "application/json": "{\"statusCode\": 200}" }'
    ```

1.  将Lambda函数的输出作为`200 OK`响应传递给客户端：

    ```bash
    aws apigateway put-integration-response --region us-east-1 \
    --rest-api-id <API_ID> / --resource-id <RESOURCE_ID> \
    --http-method POST --status-code 200 --selection-pattern ""
    ```

1.  部署API：

    ```bash
    aws apigateway create-deployment --rest-api-id <API_ID> --stage-name test_post_api
    ```

</Procedure>

### 使用JSON有效载荷测试API

您可以通过发送带有JSON有效载荷的`POST`请求来测试API。

创建一个名为`post.json`的新有效载荷文件：

```json
{
    "records": [
        {
            "time": "2021-11-12 15:00:00",
            "symbol": "AAPL",
            "price_open": 149.8,
            "price_close": 149.81,
            "price_low": 149.73,
            "price_high": 149.73,
            "trading_volume": 17291
        },
        {
            "time": "2021-11-12 15:00:00",
            "symbol": "MSFT",
            "price_open": 337.15,
            "price_close": 337.15,
            "price_low": 337.15,
            "price_high": 337.15,
            "trading_volume": 562
        },
        {
            "time": "2021-11-12 15:00:00",
            "symbol": "FB",
            "price_open": 341.35,
            "price_close": 341.3,
            "price_low": 341.3,
            "price_high": 341.35,
            "trading_volume": 556
        }
    ]
}
```

使用`curl`发送请求：

```bash
curl -X POST -H "Content-Type: application/json" -d @./post.json
https://h45kwepq8g.execute-api.us-east-1.amazonaws.com/test_post_api/insert_function 
```

如果一切正常，您的JSON有效载荷文件的内容将被插入到数据库中。

|time|symbol|price_open|price_high|price_low|price_close|trading_volume|
|-|-|-|-|-|-|-|
|2021-11-12 21:00:00|AAPL|149.8|149.73|149.73|149.81|17291|
|2021-11-12 21:00:00|MSFT|337.15|337.15|337.15|337.15|562|
|2021-11-12 21:00:00|FB|341.35|341.35|341.3|341.3|556|

[custom-lambda-integration]: https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-custom-integrations.html 
[lambda-layers]: https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-concepts.html#gettingstarted-concepts-layer 
[lambda-psycopg2]: https://github.com/jkehler/awslambda-psycopg2/ 
[psycopg2]: https://pypi.org/project/psycopg2/
