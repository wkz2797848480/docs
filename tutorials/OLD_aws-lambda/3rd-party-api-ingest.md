---
标题: 从第三方 API 提取并摄取数据
摘要: 构建一个数据管道，用于将数据从第三方金融 API 提取到 TimescaleDB 中。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，AWS Lambda，psycopg2，Pandas，GitHub 操作，管道]
---

# 从第三方API拉取并摄入数据

本教程构建了一个数据管道，从第三方财务API拉取数据并将其加载到TimescaleDB中。

本教程需要多个库。这可能会使您的部署包大小超过Lambda的250MB限制。您可以使用Docker容器将包大小扩展到10GB，为您提供更多的库和依赖灵活性。有关AWS Lambda容器支持的更多信息，请参见[AWS文档][aws-lambda-docs]。

本教程中使用的库：

*   [`pandas`][pandas]
*   `requests`
*   [`psycopg2`][psycopg2]
*   [`pgcopy`][pgcopy]

## 创建ETL函数

提取、转换和加载（ETL）函数用于从一个数据库拉取数据并将其摄入到另一个数据库中。在本教程中，ETL函数从名为Alpha Vantage的财务API拉取数据，并将数据插入到TimescaleDB中。连接使用环境变量中的值进行。

这是本教程中使用的ETL函数：

```python
# function.py:
import csv
import pandas as pd
import psycopg2
from pgcopy import CopyManager
import os

config = {'DB_USER': os.environ['DB_USER'],
         'DB_PASS': os.environ['DB_PASS'],
         'DB_HOST': os.environ['DB_HOST'],
         'DB_PORT': os.environ['DB_PORT'],
         'DB_NAME': os.environ['DB_NAME'],
         'APIKEY': os.environ['APIKEY']}

conn = psycopg2.connect(database=config['DB_NAME'],
                           host=config['DB_HOST'],
                           user=config['DB_USER'],
                           password=config['DB_PASS'],
                           port=config['DB_PORT'])
columns = ('time', 'price_open', 'price_close',
          'price_low', 'price_high', 'trading_volume', 'symbol')

def get_symbols():
   """从csv文件中读取符号。

   返回：
       [字符串列表]：符号
   """
   with open('symbols.csv') as f:
       reader = csv.reader(f)
       return [row[0] for row in reader]

def fetch_stock_data(symbol, month):
   """获取一个股票符号的历史日内数据（1分钟间隔）

   参数：
       symbol (字符串)：股票符号
       month (整数)：月份值，整数1-24（例如month=4获取过去4个月的数据）

   返回：
       元组列表：日内（K线）股票数据
   """
   interval = '1min'
   slice = 'year1month' + str(month) if month <= 12 else 'year2month1' + str(month)
   apikey = config['APIKEY']
   CSV_URL = 'https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY_EXTENDED&&#39;  \
             'symbol={symbol}&interval={interval}&slice={slice}&apikey={apikey}' \
             .format(symbol=symbol, slice=slice, interval=interval,apikey=apikey)
   df = pd.read_csv(CSV_URL)
   df['symbol'] = symbol

   df['time'] = pd.to_datetime(df['time'], format='%Y-%m-%d %H:%M:%S')
   df = df.rename(columns={'time': 'time',
                           'open': 'price_open',
                           'close': 'price_close',
                           'high': 'price_high',
                           'low': 'price_low',
                           'volume': 'trading_volume'}
                           )
   return [row for row in df.itertuples(index=False, name=None)]

def handler(event, context):
   symbols = get_symbols()
   for symbol in symbols:
       print("Fetching data for: ", symbol)
       for month in range(1, 2):
           stock_data = fetch_stock_data(symbol, month)
           print('Inserting data...')
           mgr = CopyManager(conn, 'stocks_intraday', columns)
           mgr.copy(stock_data)
           conn.commit()
```

## 添加依赖文件

当您创建了ETL函数后，您需要包括您想要安装的库。您可以通过在项目中创建一个名为`requirements.txt`的文本文件来列出库。这是本教程中使用的`requirements.txt`文件：

```txt
pandas
requests
psycopg2-binary
pgcopy
```

<Highlight type="note">
此示例在`requirements.txt`文件中使用`psycopg2-binary`而不是`psycopg2`。库的二进制版本包含其所有依赖项，因此您不需要单独安装它们。
</Highlight>

## 创建Dockerfile

当您设置了依赖项后，您可以为项目创建Dockerfile。

<Procedure>

### 创建Dockerfile

1.  使用AWS Lambda基础镜像：

    ```docker
    FROM public.ecr.aws/lambda/python:3.8
    ```

1.  将所有项目文件复制到根目录：

    ```docker
    COPY function.py .
    COPY requirements.txt .
    ```

1.  使用依赖文件安装库：

    ```docker
    RUN pip install -r requirements.txt
    CMD ["function.handler"]
    ```

</Procedure>

## 将镜像上传到ECR

要将容器镜像连接到Lambda函数，您需要将其上传到AWS Elastic Container Registry (ECR)。

<Procedure>

### 将镜像上传到ECR

1.  登录到Docker命令行界面：

    ```bash
    aws ecr get-login-password --region us-east-1 \
    | docker login --username AWS \
    --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
    ```

1.  构建镜像：

    ```bash
    docker build -t lambda-image .
    ```

1.  在ECR中创建一个仓库。在这个示例中，仓库名为`lambda-image`：

    ```bash
    aws ecr create-repository --repository-name lambda-image
    ```

1.  使用与仓库相同的名称标记您的镜像：

    ```bash
    docker tag lambda-image:latest <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/lambda-image:latest
    ```

1.  使用Docker将镜像部署到Amazon ECR：

    ```bash
    docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/lambda-image:latest        
    ```

</Procedure>

## 从容器创建Lambda函数

要从您的容器创建Lambda函数，您可以使用Lambda `create-function`命令。您需要将`--package-type`参数定义为`image`，并使用`--code`标志添加ECR Image URI：

```bash
aws lambda create-function --region us-east-1 \
--function-name docker_function --package-type Image \
--code ImageUri=<ECR Image URI> --role <ARN_LAMBDA_ROLE>
```

## 定时触发Lambda函数

如果您希望根据计划运行Lambda函数，可以设置EventBridge触发器。这会使用[`cron`表达式][cron-examples]创建一个规则。

<Procedure>

### 定时触发Lambda函数

1.  创建计划。在这个示例中，函数每天上午9点运行：

    ```bash
    aws events put-rule --name schedule-lambda --schedule-expression 'cron(0 9 * * ? *)'
    ```

1.  授予Lambda函数必要的权限：

    ```bash
    aws lambda add-permission --function-name <FUNCTION_NAME> \
    --statement-id my-scheduled-event --action 'lambda:InvokeFunction' \
    --principal events.amazonaws.com
    ```

1.  通过创建一个包含易于记忆的唯一字符串和Lambda函数ARN的`targets.json`文件，将函数添加到EventBridge规则：

    ```json
    [
      {
        "Id": "docker_lambda_trigger",
        "Arn": "<ARN_LAMBDA_FUNCTION>"
      }
    ]
    ```

1.  将Lambda函数，在此命令中称为`target`，添加到规则：

    ```bash
    aws events put-targets --rule schedule-lambda --targets file://targets.json
    ```

</Procedure>

<Highlight type="important">
如果您收到错误消息`Parameter ScheduleExpression is not valid`，您可能在cron表达式中犯了一个错误。请检查[cron表达式示例](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html#eb-cron-expressions)文档。
</Highlight>

您可以在AWS控制台中检查规则是否正确连接到Lambda函数。导航到Amazon EventBridge → Events → Rules，然后点击您创建的规则。Lambda函数的名称列在`Target(s)`下：

<img class="main-content__illustration" src="https://assets.timescale.com/docs/images/tutorials/aws-lambda-tutorial/targets.png&#34;  alt="Lamdba function target in AWS Console"/>

[aws-lambda-docs]: https://docs.aws.amazon.com/lambda/latest/dg/images-create.html 
[cron-examples]: https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html#eb-cron-expressions 
[pandas]: https://pandas.pydata.org/ 
[pgcopy]: https://github.com/G-Node/pgcopy 
[psycopg2]: https://github.com/jkehler/awslambda-psycopg2

