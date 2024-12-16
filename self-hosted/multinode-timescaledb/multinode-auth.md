---
标题: 多节点认证
摘要: 配置访问节点和数据节点之间的认证
产品: [自托管]
关键词: [多节点，认证]
标签: [管理]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 多节点认证

当您设置好实例后，需要配置它们以接受从访问节点到数据节点的连接。您为此选择的认证机制可以与外部客户端用于连接访问节点的机制不同。您如何设置多节点集群取决于您选择的认证机制。选项包括：

*   信任认证。这是最简单的方法，但也是最不安全的。如果您正在尝试多节点，这是一个很好的起点，但不推荐用于生产集群。
*   密码认证。每个用户角色需要一个内部密码来建立访问节点和数据节点之间的连接。这种方法比证书认证更容易设置，但只提供基本级别的保护。
*   证书认证。每个用户角色需要一个来自证书颁发机构的证书来建立访问节点和数据节点之间的连接。这种方法比密码认证更复杂，但更安全，更容易自动化。

<Highlight type="important">
超越简单的信任方法来创建一个安全的系统可能会很复杂，但对于您的环境来说，适当地保护数据库是非常重要的。我们不推荐任何一种安全模型，但鼓励您进行风险评估，并实施最适合您环境的安全模型。
</Highlight>

## 信任认证

信任所有传入连接是让您的多节点环境快速运行起来的最快方法，但这不是一种安全的操作方式。仅用于开发概念验证，不要将这种方法用于生产安装。

<Highlight type="warning">
信任认证方法允许对所有节点的不安全访问。不要在生产中使用这种方法。这不是一种安全的操作方式。
</Highlight>

<Procedure>

### 设置信任认证

1.  使用 `psql` 连接到访问节点，并找到 `pg_hba.conf` 文件：

    ```sql
    SHOW hba_file;
    ```

1.  在您喜欢的文本编辑器中打开 `pg_hba.conf` 文件，并添加这一行。在这个例子中，访问节点位于 IP `192.0.2.20`，掩码长度为 `32`。您可以添加以下两行中的任意一行：

    ```txt

    # 使用本地回环 TCP/IP 连接

    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    host    all             all             192.0.2.20/32            trust

    # 与前一行相同，但使用单独的 netmask 列

    # TYPE  DATABASE        USER            IP-ADDRESS      IP-MASK             METHOD
    host    all             all             192.0.2.20      255.255.255.255    trust

1.  在命令提示符下，重新加载服务器配置：

    ```bash
    pg_ctl reload
    ```

    在某些操作系统上，您可能需要使用 `pg_ctlcluster` 命令代替。

1.  如果您尚未这样做，请将数据节点添加到访问节点。有关说明，请参见[多节点设置][multi-node-setup]部分。
1.  在访问节点上，创建信任角色。在这个例子中，我们称角色为 `testrole`：

    ```sql
    CREATE ROLE testrole;
    ```

    **可选**：如果外部客户端需要以 `testrole` 身份连接到访问节点，创建角色时添加 `LOGIN` 选项。如果您希望要求外部客户端输入密码，还可以添加 `PASSWORD` 选项。
1.  允许信任角色访问数据节点的外部服务器对象。确保您包含了所有数据节点名称：

    ```sql
    GRANT USAGE ON FOREIGN SERVER <data node name>, <data node name>, ... TO testrole;
    ```

1.  在访问节点上，使用 [`distributed_exec`][distributed_exec] 命令将角色添加到所有数据节点：

    ```sql
    CALL distributed_exec($$ CREATE ROLE testrole LOGIN $$);
    ```

<Highlight type="important">
确保您在数据节点上为角色创建了具有 `LOGIN` 权限的角色，即使您在访问节点上不使用此权限。对于所有其他权限，请确保它们在访问节点和数据节点上相同。
</Highlight>

</Procedure>

## 密码认证

密码认证要求每个用户角色在建立访问节点和数据节点之间的连接之前都知道一个密码。这个内部密码仅由访问节点使用，不需要与客户端用于连接访问节点的密码相同。外部用户根本不需要共享内部密码，它可以由数据库管理员设置和管理。

访问节点存储内部密码，以便验证数据节点提供的密码是否正确。我们建议您将密码存储在访问节点上的本地密码文件中，本节将向您展示如何设置。然而，如果您的环境更适合使用用户映射来存储密码，您也可以这样做。这比本地密码文件稍微不安全一些，因为它需要为您的集群中的每个数据节点都有一个映射。

本节使用 SCRAM SHA-256 密码认证设置您的密码认证。对于其他密码认证方法，请参见[PostgreSQL 认证文档][auth-password]。

开始之前，请检查您是否可以使用 `postgres` 用户名登录到您的访问节点。

<Procedure>

### 设置密码认证

1.  在访问节点上，打开 `postgresql.conf` 配置文件，并添加或编辑这一行：

    ```txt
    password_encryption = 'scram-sha-256'  # md5 或 scram-sha-256
    ```

1.  对于每个数据节点重复此操作。
1.  在每个数据节点上，在 `psql` 提示符下，找到 `pg_hba.conf` 配置文件：

    ```sql
    SHOW hba_file
    ```

1.  在每个数据节点上，打开 `pg_hba.conf` 配置文件，并添加或编辑这一行以启用对访问节点的加密认证：

    ```txt
    # IPv4 本地连接：
    # TYPE  DATABASE  USER  ADDRESS      METHOD
    host    all       all   192.0.2.20   scram-sha-256 #其中 '192.0.2.20' 是访问节点 IP
    ```

1.  在访问节点上，打开或创建密码文件 `data/passfile`。此文件存储访问节点连接到数据节点上的每个角色的密码。如果您需要更改密码文件的位置，请调整 `postgresql.conf` 配置文件中的 `timescaledb.passfile` 设置。
1.  在访问节点上，打开 `passfile` 文件，并为每个用户添加类似这样的一行，从 `postgres` 用户开始：

    ```bash
    *:*:*:postgres:xyzzy #假设 'xyzzy' 是 'postgres' 用户的密码
    ```

1.  在访问节点上，在命令提示符下更改 `passfile` 文件的权限：

    ```bash
    chmod 0600 passfile
    ```

1.  在访问节点和每个数据节点上，重新加载服务器配置以应用更改：

    ```bash
    pg_ctl reload
    ```

1.  如果您尚未这样做，请将数据节点添加到访问节点。有关说明，请参见[多节点设置][multi-node-setup]部分。
1.  在访问节点上，在 `psql` 提示符下创建额外的角色，并授予它们访问数据节点的外部服务器对象的权限：

    ```sql
    CREATE ROLE testrole PASSWORD 'clientpass' LOGIN;
    GRANT USAGE ON FOREIGN SERVER <data node name>, <data node name>, ... TO testrole;
    ```

    `clientpass` 密码由外部客户端用于以 `testrole` 用户身份连接到访问节点。如果访问节点配置为接受其他认证方法，或者角色不是登录角色，则您可能不需要执行此步骤。
1.  在访问节点上，使用 [`distributed_exec`][distributed_exec] 将新角色添加到每个数据节点。确保您添加了 `PASSWORD` 参数以指定连接到数据节点时使用的不同密码：

    ```sql
    CALL distributed_exec($$ CREATE ROLE testrole PASSWORD 'internalpass' LOGIN $$);
    ```

1.  在访问节点上，将新角色添加到您之前创建的 `passfile` 中，添加这样的一行：

    ```bash
    *:*:*:testrole:internalpass #假设 'internalpass' 是用于连接到数据节点的密码
    ```

<Highlight type="important">
您在设置密码认证之前创建的任何用户密码需要重新创建，以便它们使用新的加密方法。
</Highlight>

</Procedure>

## 证书认证

这种方法比密码认证设置更复杂，但它更安全，更容易自动化，并且可以根据您的安全环境进行定制。

要使用证书，访问节点和每个数据节点需要三个文件：

*   根 CA 证书，称为 `root.crt`。此证书在系统中作为信任的根。它用于验证其他证书。
*   节点证书，称为 `server.crt`。此证书为节点在系统中提供可信任的身份。
*   节点证书密钥，称为 `server.key`。这提供了对节点证书所有权的证明。确保您在生成它的节点上保持此文件的私密性。

您可以从商业证书颁发机构（CA）购买证书，或生成您自己的自签名 CA。本节向您展示如何使用您的访问节点证书来创建和签署数据节点的新用户证书。

密钥和证书在数据节点和访问节点上有不同的用途。对于访问节点，签名证书用于验证访问的用户证书。对于数据节点，签名证书向访问节点认证节点。

<Procedure>

### 为访问节点生成自签名根证书

1.  在访问节点上，在命令提示符下生成一个名为 `auth.key` 的私钥：

    ```bash
    openssl genpkey -algorithm
 rsa -out auth.key
    ```

1.  为证书颁发机构（CA）生成一个自签名根证书，称为 `root.cert`：

    ```bash
    openssl req -new -key auth.key -days 3650 -out root.crt -x509
    ```

1.  完成脚本提出的问题以创建您的根证书。输入您的回答，按 `enter` 接受括号中显示的默认值，或输入 `.` 留空。例如：

    ```txt
    国家名称（两位代码）[AU]:US
    州或省名称（全名）[Some-State]:New York
    本地名称（例如，城市）[]:New York
    组织名称（例如，公司）[Internet Widgets Pty Ltd]:Example Company Pty Ltd
    组织单位名称（例如，部门）[]:
    通用名称（例如，服务器 FQDN 或您的名称）[]:http://cert.example.com/ 
    电子邮件地址[]:
    ```

</Procedure>

当您在访问节点上创建了根证书后，您可以为每个数据节点生成证书和密钥。为此，您需要为每个数据节点创建一个证书签名请求（CSR）。

默认的密钥名称为 `server.key`，证书为 `server.crt`。它们存储在一起，在数据节点实例的 `data` 目录中。

默认的 CSR 名称为 `server.csr`，您需要使用在访问节点上创建的根证书来签署它。

### 为数据节点生成密钥和证书

1.  在访问节点上，生成一个名为 `server.csr` 的证书签名请求（CSR），并创建一个名为 `server.key` 的新密钥：

    ```bash
    openssl req -out server.csr -new -newkey rsa:2048 -nodes \
    -keyout server.key
    ```

1.  使用您之前创建的根证书 CA，名为 `auth.key`，签署 CSR：

    ```bash
    openssl ca -extensions v3_intermediate_ca -days 3650 -notext \
    -md sha256 -in server.csr -out server.crt
    ```

1.  将 `server.crt` 和 `server.key` 文件从访问节点移动到每个数据节点的 `data` 目录中。根据您的网络设置，您可能需要使用便携式媒体。
1.  将根证书文件 `root.crt` 从访问节点复制到每个数据节点的 `data` 目录中。根据您的网络设置，您可能需要使用便携式媒体。

</Procedure>

当您创建了证书和密钥，并将所有文件移动到数据节点的正确位置后，您可以配置数据节点使用 SSL 认证。

<Procedure>

### 配置数据节点使用 SSL 认证

1.  在每个数据节点上，打开 `postgresql.conf` 配置文件，并添加或编辑 SSL 设置以启用证书认证：

    ```txt
    ssl = on
    ssl_ca_file = 'root.crt'
    ssl_cert_file = 'server.crt'
    ssl_key_file = 'server.key'
    ```

1.  []()<optional />如果您希望访问节点也使用证书认证进行登录，请在访问节点上也进行这些更改。

1.  在每个数据节点上，打开 `pg_hba.conf` 配置文件，并添加或编辑这一行以允许任何 SSL 用户使用客户端证书认证登录：

    ```txt
    # TYPE    DATABASE  USER        ADDRESS   METHOD  OPTIONS
    hostssl   all       all         all       cert    clientcert=1
    ```

<Highlight type="note">
如果您使用证书和密钥的默认名称，则无需显式设置它们。配置默认寻找 `server.crt` 和 `server.key`。如果您为证书和密钥使用不同的名称，请确保在 `postgresql.conf` 配置文件中指定正确的名称。
</Highlight>

</Procedure>

当您的数据节点配置为使用 SSL 证书认证时，您需要为您的访问节点创建一个签名的证书和密钥。这允许访问节点登录到数据节点。

<Procedure>

### 为访问节点创建证书和密钥

1.  在访问节点上，作为 `postgres` 用户，使用 [md5sum][] 计算证书文件的基础名称，生成主题标识符，并创建密钥和证书文件的名称：

    ```bash
    pguser=postgres
    base=`echo -n $pguser | md5sum | cut -c1-32`
    subj="/C=US/ST=New York/L=New York/O=Timescale/OU=Engineering/CN=$pguser"
    key_file="timescaledb/certs/$base.key"
    crt_file="timescaledb/certs/$base.crt"
    ```

1.  生成一个新的随机用户密钥：

    ```bash
    openssl genpkey -algorithm RSA -out "$key_file"
    ```

1.  生成一个证书签名请求（CSR）。此文件是临时的，存储在 `data` 目录中，稍后将被删除：

    ```bash
    openssl req -new -sha256 -key $key_file -out "$base.csr" -subj "$subj"
    ```

1.  使用访问节点密钥签署 CSR：

    ```bash
    openssl ca -batch -keyfile server.key -extensions v3_intermediate_ca \
      -days 3650 -notext -md sha256 -in "$base.csr" -out "$crt_file"
    rm $base.csr
    ```

1.  将节点证书追加到用户证书。这完成了证书验证链，并确保所有证书都在数据节点上可用，直到存储在 `root.crt` 中的受信任证书：

    ```bash
    cat >>$crt_file <server.crt
    ```

<Highlight type="note">
默认情况下，用户密钥文件和证书存储在访问节点的 `data` 目录下，位于 `timescaledb/certs`。您可以使用 `timescaledb.ssl_dir` 配置变量更改此位置。
</Highlight>

</Procedure>

您的数据节点现在已设置为接受证书认证，数据节点和访问节点都有密钥，`postgres` 用户有证书。如果您尚未这样做，请将数据节点添加到访问节点。有关说明，请参见[多节点设置][multi-node-setup]部分。最后一步是添加额外的用户角色。

<Procedure>

### 设置额外的用户角色

1.  在访问节点上，在 `psql` 提示符下创建新用户并授予权限：

    ```sql
    CREATE ROLE testrole;
    GRANT USAGE ON FOREIGN SERVER <data node name>, <data node name>, ... TO testrole;
    ```

    如果您需要外部客户端以 `testrole` 身份连接到访问节点，请确保您也添加了 `LOGIN` 选项。您还可以通过添加 `PASSWORD` 选项启用密码认证。

1.  在访问节点上，使用 [`distributed_exec`][distributed_exec] 命令将角色添加到所有数据节点：

    ```sql
    CALL distributed_exec($$ CREATE ROLE testrole LOGIN $$);
    ```

</Procedure>

[auth-password]: https://www.postgresql.org/docs/current/auth-password.html 
[distributed_exec]: /api/:currentVersion:/distributed-hypertables/distributed_exec
[md5sum]: https://www.tutorialspoint.com/unix_commands/md5sum.htm 
[multi-node-setup]: /self-hosted/:currentVersion:/multinode-timescaledb/multinode-setup/
[user-mapping]: https://www.postgresql.org/docs/current/sql-createusermapping.html

