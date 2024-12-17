---
标题: 结合使用 TimescaleDB 与亚马逊网络服务（AWS）Lambda
摘要: 学习如何一同使用 AWS Lambda 和 TimescaleDB。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，AWS Lambda，psycopg2，Pandas，GitHub 操作，管道]
---
# TimescaleDB与AWS Lambda

本节包含有关如何使用AWS Lambda和TimescaleDB的教程。

*   使用AWS Lambda和API Gateway为TimescaleDB创建数据API。
*   使用AWS Lambda和Docker从第三方API拉取数据并导入TimescaleDB。如果您有很多依赖项，这非常有用。
*   使用GitHub Actions持续部署您的Lambda函数。

## 前提条件

开始之前，请确保您已完成以下步骤：
日内股票数据分析教程。
本教程需要您在该教程中设置的表格和数据。

要完成本教程，您需要一个AWS账户。您还需要访问AWS命令行界面（CLI）。要检查是否已安装AWS CLI，请在命令提示符下使用此命令。如果已安装，该命令会显示版本号，如下所示：

```bash
aws --version
aws-cli/2.2.18 Python/3.8.8 Linux/5.10.0-1044-oem exe/x86_64.ubuntu.20 prompt/off
```

有关安装AWS CLI的更多信息，请参见[AWS安装说明][aws-install]。

<Highlight type="cloud" header="Timescale上的VPC" button="免费试用">
如果您在Timescale完成本教程，请确保您已在AWS和Timescale数据库上创建了VPC。有关设置VPC的更多信息，请参见[Timescale VPC部分][timescale-vpc]。
</Highlight>

## 编程语言

本教程中的代码示例使用Python，但您可以使用AWS Lambda支持的任何语言[lambda-supported-langs]。

## 资源

有关本教程中的主题的更多信息，请查看以下资源：

*   AWS CLI版本2参考
*   创建Lambda容器镜像
*   开始使用AWS Lambda
*   分析历史日内股票数据
*   分析加密货币市场数据

[3rd-party-ingest]: /tutorials/:currentVersion:/aws-lambda/3rd-party-api-ingest
[aws-cli2]: https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html 
[aws-install]: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html 
[create-data-api]: /tutorials/:currentVersion:/aws-lambda/create-data-api
[cryptocurrency-market-data]: /tutorials/:currentVersion:/analyze-cryptocurrency-data
[gh-actions]: /tutorials/:currentVersion:/aws-lambda/continuous-deployment
[intraday-stock-data]: /tutorials/:currentVersion:/
[lambda-container-images]: https://docs.aws.amazon.com/lambda/latest/dg/images-create.html 
[lambda-getting-started]: https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html 
[lambda-supported-langs]: https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html 

[timescale-vpc]: /use-timescale/latest/vpc/

