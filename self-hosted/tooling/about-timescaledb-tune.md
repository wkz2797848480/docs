---
标题: 关于 timescaledb - tune
摘要: 自动调整您的 TimescaleDB 数据库，使其与您的系统资源和 PostgreSQL 版本相匹配
产品: [自托管]
关键词: [配置，timescaledb - tune]
标签: [调整，设置]
---

# 关于timescaledb-tune

通过调整TimescaleDB数据库以匹配您的系统资源和PostgreSQL版本，获得更好的性能。`timescaledb-tune`是一个开源命令行工具，用于分析和调整您的数据库设置。

## 安装timescaledb-tune

`timescaledb-tune`随TimescaleDB的二进制版本一起打包发布。如果您从任何二进制版本安装了TimescaleDB，包括Docker，您已经可以使用它了。更多安装说明，请参见[GitHub仓库][github-tstune]。

## 使用timescaledb-tune调整数据库

在命令行中运行`timescaledb-tune`。该工具分析您的`postgresql.conf`文件，提供内存、并行性、预写日志等设置的建议。这些更改将被写入您的`postgresql.conf`文件，并在下次重启时生效。

<Procedure>

1.  在命令行中运行`timescaledb-tune`。要自动接受所有建议，请包含`--yes`标志。

    ```bash
    timescaledb-tune
    ```

1.  如果您没有使用`--yes`标志，请对每个提示做出响应，接受或拒绝建议。
1.  更改将被写入您的`postgresql.conf`文件。

</Procedure>

<Highlight type="note">
有关详细说明和其他选项，请参见[Github仓库](https://github.com/timescale/timescaledb-tune)中的文档。
</Highlight>

[github-tstune]: https://github.com/timescale/timescaledb-tune

