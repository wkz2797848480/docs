---
标题: 从云镜像安装 TimescaleDB
摘要: 从预先构建的云镜像安装自托管的 TimescaleDB
产品: [自托管]
关键词: [安装，自托管]
标签: [云镜像]
---

import WhereTo from "versionContent/_partials/_where-to-next.mdx";
import Skip from "versionContent/_partials/_selfhosted_cta.mdx";


## 安装TimescaleDB的预构建云映像

您可以在云托管服务提供商上安装TimescaleDB，使用预构建的公开可用的机器映像。这些说明向您展示如何在Amazon Web Services (AWS)上使用预构建的Amazon机器映像（AMI）。

### 跳过
当前可用的预构建云映像是：
*   Ubuntu 20.04 Amazon EBS-backed AMI

TimescaleDB AMI使用弹性块存储（EBS）附加卷。这允许您存储映像快照、动态IOPS配置，并在EC2实例停止运行时为您提供一定程度的数据保护。选择一个针对EBS附加卷优化的EC2实例类型。有关如何选择正确的EBS优化EC2实例类型的信息，请参阅AWS[实例配置文档][aws-instance-config]。

**注意：**
此部分展示了如何在AWS EC2仪表板内使用AMI。但是，您也可以使用AMI通过Cloudformation、Terraform、AWS CLI或任何其他支持公共AMI的AWS部署工具来构建实例。

### 步骤

### 从预构建的云映像安装TimescaleDB

1.  确保您有一个[Amazon Web Services账户][aws-signup]，并且已登录到[您的EC2仪表板][aws-dashboard]。
2.  导航到`Images → AMIs`。
3.  在搜索栏中，将搜索更改为`Public images`并输入_Timescale_搜索词，以找到所有可用的TimescaleDB映像。
4.  选择您想要使用的映像，并点击`Launch instance from image`。
    ![在AWS EC2中启动AMI](https://assets.timescale.com/docs/images/aws_launch_ami.webp)

完成安装后，连接到您的实例并配置数据库。有关连接到实例的信息，请参阅AWS[访问实例文档][aws-connect]。配置数据库的最简单方法是运行`timescaledb-tune`脚本，该脚本包含在`timescaledb-tools`包中。有关更多信息，请参见[配置][config]部分。

**注意：**
运行`timescaledb-tune`脚本后，您需要重新启动PostgreSQL服务以使配置更改生效。要重启服务，请运行`sudo systemctl restart postgresql.service`。

## 设置TimescaleDB扩展

当您安装了PostgreSQL和TimescaleDB后，连接到您的实例并设置TimescaleDB扩展。

### 步骤

#### 设置TimescaleDB扩展

1.  在您的实例上，在命令提示符下，以`postgres`超级用户身份连接到PostgreSQL实例：

    ```bash
    sudo -u postgres psql
    ```

2.  在提示符下，创建一个空数据库。例如，创建一个名为`tsdb`的数据库：

    ```sql
    CREATE database tsdb;
    ```

3.  连接到您创建的数据库：

    ```sql
    \c tsdb
    ```

4.  添加TimescaleDB扩展：

    ```sql
    CREATE EXTENSION IF NOT EXISTS timescaledb;
    ```

您可以通过在命令提示符下使用`\dx`命令来检查是否已安装TimescaleDB扩展。它看起来像这样：

```sql
tsdb=# \dx

                                      List of installed extensions
    Name     | Version |   Schema   |                            Description
-------------+---------+------------+-------------------------------------------------------------------
 plpgsql     | 1.0     | pg_catalog | PL/pgSQL procedural language
 timescaledb | 2.1.1   | public     | Enables scalable inserts and complex queries for time-series data
(2 rows)

(END)
```

## 接下来做什么

[去哪里][WhereTo]

[aws-signup]: https://portal.aws.amazon.com/billing/signup  
[aws-dashboard]: https://console.aws.amazon.com/ec2/  
[aws-instance-config]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-optimized.html  
[aws-connect]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html  
[install-psql]: /use-timescale/:currentVersion:/connecting/psql/  
[config]: /self-hosted/:currentVersion:/configuration/

[aws-signup]: https://portal.aws.amazon.com/billing/signup
[aws-dashboard]: https://console.aws.amazon.com/ec2/
[aws-instance-config]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-optimized.html
[aws-connect]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html
[install-psql]: /use-timescale/:currentVersion:/connecting/psql/
[config]: /self-hosted/:currentVersion:/configuration/
