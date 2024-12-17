---
标题: 摄取数据并运行首个查询
摘要: 将一些来自 CSV 文件的数据摄取到你的数据库中。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [连续聚合，超级函数，分析]
---

# 导入数据并运行您的第一个查询

主要数据集由[Kaggle提供，包含多个CSV文件][kaggle-download]。此外，我们还收集了[有关体育场和每场比赛结果的其他信息][extra-download]，为您提供了额外的数据分析。

数据以多个CSV文件提供，每个文件对应数据库中的一个表，包含以下信息：

*   **game**
    *   每场比赛的信息（主队、客队、比赛周等）
    *   `game_id`是主键

*   **player**
    *   球员信息（display_name, college, position等）
    *   `player_id`是主键。

*   **play**
    *   比赛信息（比赛、进攻、季度、进攻次数、传球结果等）。有很多好的整体比赛信息可以分析。
    *   要查询特定的进攻，您需要一起使用`gameid`和`playid`，因为有些`playid`在不同比赛中会被重复使用。

*   **tracking**
    *   每秒多次采样的每次进攻的球员追踪信息。
    *   字段包括加速度、场上的X/Y坐标等。
    *   `x`和`y`使用Kaggle网站上数据描述中概述的坐标表示球员在场上的物理位置。
    *   这是数据库中最大的表（超过1800万行）。

*   **scores**
    *   每场比赛的最终结果。
    *   这个表可以使用`home_team_abb`和`visitor_team_abb`字段与追踪表连接。

*   **stadium_info**
    *   每个主队的体育场和额外信息，如`surface`、`roof_type`、`location`。

使用以下SQL创建表：

```sql
CREATE TABLE game (
    game_id INT PRIMARY KEY,
    game_date DATE,
    gametime_et TIME,
    home_team TEXT,
    visitor_team TEXT,
    week INT
);

CREATE TABLE player (
    player_id INT PRIMARY KEY,
    height TEXT,
    weight INT,
    birthDate DATE,
    collegeName TEXT,
    position TEXT,
    displayName TEXT
);

CREATE TABLE play (
    gameId INT,
    playId INT,
    playDescription TEXT,
    quarter INT,
    down INT,
    yardsToGo INT,
    possessionTeam TEXT,
    playType TEXT,
    yardlineSide TEXT,
    yardlineNumber INT,
    offenseFormation TEXT,
    personnelO TEXT,
    defendersInTheBox INT,
    numberOfPassRushers INT,
    personnelD TEXT,
    typeDropback TEXT,
    preSnapVisitorScore INT,
    preSnapHomeScore INT,
    gameClock TIME,
    absoluteYardlineNumber INT,
    penaltyCodes TEXT,
    penaltyJerseyNumber TEXT,
    passResult TEXT,
    offensePlayResult INT,
    playResult INT,
    epa DOUBLE PRECISION,
    isDefensivePI BOOLEAN
);

CREATE TABLE tracking (
    time TIMESTAMP,
    x DOUBLE PRECISION,
    y DOUBLE PRECISION,
    s DOUBLE PRECISION,
    a DOUBLE PRECISION,
    dis DOUBLE PRECISION,
    o DOUBLE PRECISION,
    dir DOUBLE PRECISION,
    event TEXT,
    player_id INT,
    displayName TEXT,
    jerseyNumber INT,
    position TEXT,
    frameId INT,
    team TEXT,
    gameid INT,
    playid INT,
    playDirection TEXT,
    route TEXT
);

CREATE TABLE scores (
    scoreid INT PRIMARY KEY,
    date DATE,
    visitor_team TEXT,
    visitor_team_abb TEXT,
    visitor_score INT,
    home_team TEXT,
    home_team_abb TEXT,
    home_score INT
);

CREATE TABLE stadium_info (
    stadiumid INT PRIMARY KEY,
    stadium_name TEXT,
    location TEXT,
    surface TEXT,
    roof_type TEXT,
    team_name TEXT,
    team_abbreviation TEXT,
    time_zone TEXT
);
```

为`tracking`表添加索引以提高查询性能：

```sql
CREATE INDEX idx_gameid ON tracking (gameid);
CREATE INDEX idx_playerid ON tracking (player_id);
CREATE INDEX idx_playid ON tracking (playid);
```

**从`tracking`表创建超表**

```sql
/*
tracking: 表名
time: 时间戳列名
*/
SELECT create_hypertable('tracking', 'time');
```

## 从CSV文件导入数据

game、player和play表有三个单独的CSV文件。对于tracking表，您需要从17个CSV文件中导入数据（每个赛季的一周一个文件）。

您可以使用Python脚本，该脚本使用psycopg2的`copy_expert`函数来导入数据：

```python
import config
import psycopg2

# 连接到数据库
conn = psycopg2.connect(database=config.DB_NAME,
                        host=config.HOST,
                        user=config.USER,
                        password=config.PASS,
                        port=config.PORT)

# 将CSV文件插入到指定表
def insert(csv_file, table_name):
    cur = conn.cursor()
    copy_sql = """
            COPY {table} FROM stdin WITH CSV HEADER
            DELIMITER as ','
            """.format(table=table_name)
    with open(csv_file, 'r') as f:
        cur.copy_expert(sql=copy_sql, file=f)
        conn.commit()
        cur.close()

print("Inserting games.csv")
insert("data/games.csv", "game")

print("Inserting plays.csv")
insert("data/plays.csv", "play")

print("Inserting players.csv")
insert("data/players.csv", "player")

print("Inserting stadium_info.csv")
insert("data/stadium_info.csv", "stadium_info")

print("Inserting scores.csv")
insert("data/scores.csv", "scores")

# 遍历每周的CSV文件并插入它们
for i in range(1, 18):
    print(f'Inserting week{i}'.format(i = str(i)))
    insert(f'data/week{i}.csv'.format(i=i), "tracking")

conn.close()
```

## 运行您的第一个查询

现在您已经导入了所有数据，可以运行第一个聚合查询并检查结果。在本教程的大多数示例查询中，您需要从`tracking`表中聚合数据，该表每个进攻包含每个球员的多行数据（因为数据在每次进攻中每秒多次采样）。

<Highlight type="important">
这些查询是超函数的示例。要访问超函数，您需要在开始之前安装[Timescale工具包](https://docs.timescale.com/use-timescale/latest/install-timescaledb-toolkit/)。
</Highlight>

### 传球比赛中，按球员和比赛计算比赛中跑动的码数

此查询为每场比赛中的每个球员汇总所有码数。然后，您可以将其与`player`表连接以获取球员详细信息：

```sql
SELECT t.player_id, p.displayname, SUM(dis) AS yards, t.gameid
FROM tracking t
LEFT JOIN player p ON t.player_id = p.player_id
GROUP BY t.player_id, p.displayname, t.gameid
ORDER BY t.gameid ASC, yards DESC;
```

您的数据应该如下所示：

|player_id |        displayname         |       yards        |   gameid   |
|-----------|-----------------------------|--------------------|------------|
|   2495454 | Julio Jones                 | 1030.6399999999971 | 2018090600|
|   2507763 | Mike Wallace                |  940.0099999999989 | 2018090600|
|   2552600 | Nelson Agholor              |  908.0299999999983 | 2018090600|
|   2540158 | Zach Ertz                   |  882.0899999999953 | 2018090600|
|   2539653 | Robert Alford               |  881.7599999999983 | 2018090600|
|   2555383 | Jalen Mills                 |  856.1199999999916 | 2018090600|

您可能已经注意到，由于我们需要聚合`tracking`表中的所有行以获取每个球员在每场比赛中的总码数，这个查询需要很长时间才能完成，PostgreSQL需要扫描2000万行数据。在我们的小型测试机上，这个查询通常需要25-30秒才能运行。

## 使用连续聚合进行更快的查询

我们感兴趣的大部分数据都是基于`tracking`数据的这种聚合。我们想知道每个球员在每次进攻或整个比赛中跑了多少距离。与其每次都让TimescaleDB查询和聚合原始数据，我们创建了一个[连续聚合][cagg]来显著提高查询和分析的速度。

### 创建每场比赛球员码数的连续聚合

```sql
CREATE MATERIALIZED VIEW player_yards_by_game
WITH (timescaledb.continuous) AS
SELECT t.player_id, t.gameid, t.position, t.team,
 time_bucket(INTERVAL '1 day', t."time") AS bucket,
 SUM(t.dis) AS yards,
  AVG(t.a) AS acc
FROM tracking t
GROUP BY t.player_id, t.gameid, t.position, t.team, bucket;
```

创建连续聚合后，修改查询以使用物化数据，您会注意到响应时间现在显著加快，在我们的测试机上不到一秒。

```sql
SELECT pyg.player_id, p.displayname, SUM(yards) AS yards, pyg.gameid
FROM player_yards_by_game pyg
LEFT JOIN player p ON pyg.player_id = p.player_id
GROUP BY pyg.player_id, p.displayname, pyg.gameid
ORDER BY pyg.gameid ASC, yards DESC;
```

在下一部分的大多数查询中，我们将使用这个连续聚合。您也可以尝试其他变体的物化数据，以回答TimescaleDB的更多问题。

[cagg]: /use-timescale/:currentVersion:/continuous-aggregates/
[extra-download]: https://assets.timescale.com/docs/downloads/nfl_2018.zip 
[kaggle-download]: https://www.kaggle.com/c/nfl-big-data-bowl-2021/data

