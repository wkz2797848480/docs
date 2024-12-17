---
标题: 使用 GitHub 操作实现 Lambda 持续部署
摘要: 在 GitHub 与亚马逊网络服务（AWS）的 Lambda 之间构建持续部署管道。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，AWS Lambda，psycopg2，Pandas，GitHub 操作，管道]
---

# 使用GitHub Actions进行Lambda持续部署

本教程构建了一个使用GitHub Actions在GitHub和AWS Lambda之间进行持续部署（CD）的管道。

将您的函数及其依赖项与AWS Lambda打包和部署有时可能是一个繁琐的工作。特别是如果您还希望在使用像GitHub这样的源代码管理平台开发代码后再将其推送到AWS Lambda时。

您可以使用GitHub Actions从GitHub仓库自动部署到AWS Lambda。您需要向仓库的`main`或`master`分支推送一个提交，然后让GitHub Actions创建部署包，并将您的代码部署到AWS Lambda。

## 前提条件

*   Git ([安装选项在这里](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)) 
*   GitHub CLI工具 ([安装选项在这里](https://github.com/cli/cli#installation)) 
*   AWS账户

<Procedure>

## 在AWS控制台中创建一个新的Lambda函数

1.  通过导航到AWS Lambda控制台并创建一个新的函数来创建一个名为`lambda-cd`的新Lambda函数：
    ![创建新函数](https://assets.timescale.com/docs/images/tutorials/aws-lambda-tutorial/create_new_function.png) 
1.  添加`lambda-cd`作为您的函数名称，选择您喜欢的运行时环境，然后点击`Create function`：
    ![从头开始创建Lambda](https://assets.timescale.com/docs/images/tutorials/aws-lambda-tutorial/from_scratch.png) 
    记下函数名称，您稍后需要它来配置部署文件。

</Procedure>

<Procedure>

## 创建一个新的GitHub仓库

现在您可以创建一个新的GitHub仓库，其中包含函数代码。

1.  创建一个新的GitHub [仓库](https://github.com/new). 
1.  在您的本地系统上，创建一个名为`lambda-cd`的新项目文件夹：

    ```bash
    mkdir lambda-cd
    cd lambda-cd
    ```

1.  创建您想要上传的Lambda函数。
    作为一个示例，这里有一个Python Lambda函数，它返回来自名为`stocks_intraday`的TimescaleDB表的数据。
    阅读教程在此处用Lambda构建TimescaleDB API

    ```bash
    touch function.py
    ```

    ```python
    # function.py:
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

        sql = "SELECT * FROM stocks_intraday ORDER BY time DESC LIMIT 10"
        cursor = conn.cursor(cursor_factory=psycopg2.extras.DictCursor)
        cursor.execute(sql)
        result = cursor.fetchall()

        return {
            'statusCode': 200,
            'body': json.dumps(list(result), default=str),
            'headers': {
                "Content-Type": "application/json"
            }
        }
    ```

1.  初始化一个git仓库并将项目推送到GitHub：

    ```bash
    git init
    git add function.py
    git commit -m "Initial commit: add Lambda function"
    git branch -M main
    git remote add origin <YOUR_GITHUB_PROJECT_URL.git>
    git push -u origin main
    ```

</Procedure>

此时，您拥有一个包含Lambda函数的GitHub仓库。现在您可以将此仓库连接到AWS Lambda函数。

## 连接GitHub和AWS Lambda

使用GitHub Actions将GitHub仓库连接到AWS Lambda。

<Procedure>

### 将AWS凭证添加到仓库

您需要将AWS凭证添加到仓库，以便它有权限连接到Lambda。
您可以通过使用GitHub命令行添加[GitHub secrets][github-secrets]来实现。

1.  使用GitHub进行身份验证：

    ```bash
    gh auth login
    ```

    选择您想要登录的账户。使用您的GitHub密码或认证令牌进行身份验证。
1.  将AWS凭证添加为GitHub secrets。
    通过使用GitHub secrets，您的凭证将被加密，并且无法公开查看。使用`gh secret set`命令逐个上传您的AWS凭证（系统会提示您粘贴每个凭证的值）：

    AWS_ACCESS_KEY_ID：

    ```bash
    gh secret set AWS_ACCESS_KEY_ID
    ```

    AWS_SECRET_ACCESS_KEY：

    ```bash
    gh secret set AWS_SECRET_ACCESS_KEY
    ```

    AWS_REGION：

    ```bash
    gh secret set AWS_REGION
    ```

1.  为了确保您的凭证已正确上传，您可以列出可用的GitHub secrets：

    ```bash
    gh secret list
    AWS_ACCESS_KEY_ID      Updated 2021-09-13
    AWS_SECRET_ACCESS_KEY  Updated 2021-09-13
    AWS_REGION             Updated 2021-09-13
    ```

</Procedure>

现在您知道您的AWS凭证可用于仓库。

<Procedure>

### 设置GitHub Actions

您现在可以设置一些基于["AWS Lambda Deploy" GitHub action](https://github.com/marketplace/actions/aws-lambda-deploy)的自动化操作，以自动部署到AWS Lambda。

1.  创建一个新的YAML文件，其中包含部署配置：

    ```bash
    touch .github/workflows/main.yml
    ```

1.  将此内容添加到文件中：

    ```yml
    name: deploy to lambda
    on:
      # Trigger the workflow on push or pull request,
      # but only for the main branch
      push:
        branches:
          - main
    jobs:

      deploy_source:
        name: deploy lambda from source
        runs-on: ubuntu-latest
        steps:
          - name: checkout source code
            uses: actions/checkout@v1
          - name: default deploy
            uses: appleboy/lambda-action@master
            with:
              aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
              aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
              aws_region: ${{ secrets.AWS_REGION }}
              function_name: lambda-cd
              source: function.py
    ```

    当主分支有新的推送时，此配置将代码部署到Lambda。

    如您在YAML文件中所见，AWS凭证是使用`${{ secrets.AWS_ACCESS_KEY_ID }}`语法访问的。
    确保在此配置文件中使用AWS控制台中显示的Lambda函数名称（在此示例中为"lambda-cd"）作为`function_name`属性的值。

</Procedure>

<Procedure>

### 测试管道

您可以通过将更改推送到GitHub来测试钩子是否工作。

1.  将更改推送到仓库：

    ```bash
    git add .github/workflows/main.yml
    git commit -m "Add deploy configuration file"
    git add function.py
    git commit -m "Update function code"
    git push
    ```

1.  导航到您的仓库的GitHub Actions页面，查看构建运行并成功：
    ![github action build](https://assets.timescale.com/docs/images/tutorials/aws-lambda-tutorial/github_action_lambda.png) 

</Procedure>

您现在已在GitHub和AWS Lambda之间设置了一个持续部署管道。

[create-data-api]: /tutorials/:currentVersion:/aws-lambda/create-data-api/
[github-secrets]: https://docs.github.com/en/actions/reference/encrypted-secrets

