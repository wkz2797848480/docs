---
标题: TimescaleDB 调优工具
摘要: 使用 timescaledb-tune 工具自动配置您的 TimescaleDB 实例
产品: [自托管]
关键词: [配置，设置，timescaledb-tune]
标签: [调优]
---

以下是您提供的文章翻译成中文，并保留了超链接和格式，使用Markdown文档呈现：

# TimescaleDB调优工具

为了帮助简化TimescaleDB的配置过程，您可以使用[`timescaledb-tune`][tstune]工具。这个工具根据您的系统情况（包括内存、CPU和PostgreSQL版本）来设置最常见的参数到推荐值。`timescaledb-tune`随TimescaleDB二进制版本作为依赖项一起打包，所以如果您是从二进制版本（包括Docker）安装TimescaleDB的，您应该已经可以访问这个工具。或者，您可以使用`go install`命令来安装它：

```bash
go install github.com/timescale/timescaledb-tune/cmd/timescaledb-tune@latest
```

`timescaledb-tune`工具会读取您系统的`postgresql.conf`文件，并为您的设置提供交互式建议。以下是工具运行的示例：

```bash
使用的postgresql.conf文件位于此路径：
/usr/local/var/postgres/postgresql.conf

这是否正确？[(y)es/(n)o]: y
备份写入到：
/var/folders/cr/example/T/timescaledb_tune.backup202101071520

shared_preload_libraries需要更新
当前：
#shared_preload_libraries = 'timescaledb'
建议：
shared_preload_libraries = 'timescaledb'
这样可以吗？[(y)es/(n)o]: y
成功：shared_preload_libraries将被更新

调整内存/并行性/WAL和其他设置？[(y)es/(n)o]: y
基于8.00 GB可用内存和4个CPU为PostgreSQL 12提供的建议

内存设置建议
当前：
shared_buffers = 128MB
#effective_cache_size = 4GB
#maintenance_work_mem = 64MB
#work_mem = 4MB
建议：
shared_buffers = 2GB
effective_cache_size = 6GB
maintenance_work_mem = 1GB
work_mem = 26214kB
这样可以吗？[(y)es/(s)kip/(q)uit]:
```

当您回答了这些问题后，更改将被写入您的`postgresql.conf`文件，并在下次重启时生效。

如果您是在一个新的实例上开始，并且不想逐组批准更改，您可以在运行工具时使用一些额外的标志自动接受并将建议附加到您的`postgresql.conf`文件末尾：

```bash
timescaledb-tune --quiet --yes --dry-run >> /path/to/postgresql.conf
```

[tstune]: https://github.com/timescale/timescaledb-tune

