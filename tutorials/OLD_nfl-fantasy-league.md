---
标题: 借助 TimescaleDB 赢得你的美国国家橄榄球联盟（NFL）梦幻联赛
摘要: 使用 TimescaleDB 摄取并分析美式橄榄球数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [分析，psycopg2]
---

# 用TimescaleDB赢得你的NFL梦幻联赛

本教程是关于如何使用TimescaleDB摄入和分析美式足球数据的逐步指南。

我们使用的 数据集由国家橄榄球联盟（NFL）提供，包含2018年NFL赛季所有传球数据。我们将使用Python将这个数据集摄入到TimescaleDB中，并开始探索它，以发现可能帮助你在下一个梦幻赛季获胜的球员有趣信息。如果你不是NFL球迷，这个教程仍然可以帮助你开始使用TimescaleDB，并使用SQL和Python探索现实世界的数据集。

1.  [创建表格](#创建表格)
2.  [从CSV文件中摄入数据](#从CSV文件中摄入数据)
3.  [分析NFL数据](#分析nfl数据)
4.  [可视化开球前位置和球员移动](#可视化开球前位置和球员移动)

## 前提条件

*   Python 3
*   TimescaleDB（见[安装选项][install-timescale]）
*   [Psql][psql-install]或任何其他PostgreSQL客户端（例如，DBeaver）

## 下载数据集

*   [NFL数据集可在Kaggle上下载。][kaggle-download]
*   [额外的体育场和得分数据集（.zip）（来源：wikipedia.com）。][extra-download]

## 创建表格

你需要创建六个表格：

*   **game**

  每场比赛的信息，`game_id`是主键。

*   **player**

  球员信息，`player_id`是主键。

*   **play**

  比赛信息。要查询特定的比赛，你需要一起使用gameid和playid。

*   **tracking**

  每次比赛的球员追踪信息。这将是数据库中最大的表格（超过1800万行）。重要字段是`x`和`y`，因为它们指示了球员在场上的物理位置。

*   **scores**

  每场比赛的最终结果。这个表格可以通过`home_team_abb`和`visitor_team_abb`字段与追踪表格连接。

*   **stadium_info**

  每个主体育场和额外信息，如`surface`、`roof_type`、`location`。

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
    nflId INT,
    displayName TEXT,
    jerseyNumber INT,
    position TEXT,
    frameId INT,
    team TEXT,
    gameid INT,
    player_id INT,
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

CREATE TABLE stadium_info(
    stadiumid INT PRIMARY KEY,
    stadium_name TEXT,
    location TEXT,
    surface TEXT,
    roof_type TEXT,
    team_name TEXT,
    team_abbreviation TEXT,
    time_zone TEXT
)

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

## 从CSV文件中摄入数据

游戏、球员和比赛表有三个单独的CSV文件。对于追踪表，你需要从17个CSV文件中导入数据（每个赛季的一周一个文件）。

你可以使用一个Python脚本，该脚本使用psycopg2的`copy_expert`函数来摄入数据：

```python
import config
import psycopg2

# 连接到数据库
conn = psycopg2.connect(database=config.DB_NAME,
                        host=config.HOST,
                        user=config.USER,
                        password=config.PASS,
                        port=config.PORT)

# 将CSV文件插入到指定的表中
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

# 遍历每个周的CSV文件并插入它们
for i in range(1, 18):
    print("Inserting week{i}".format(str(i)))
    insert("data/week{i}.csv".format(i=i), "tracking")

conn.close()
```

## 分析NFL数据

现在你已经摄入了所有数据，让我们看看如何使用PostgreSQL和TimescaleDB分析数据，以帮助你完善你的梦幻选秀策略，赢得你的梦幻赛季。

这些分析包括可视化，帮助你看到这些数据的潜在用途。这些是使用Matplotlib Python模块创建的，这是许多优秀可视化工具之一。

为了优化分析，你需要创建一个连续聚合。连续聚合大大减少了查询运行时间，运行速度高达30倍。这个连续聚合汇总了一天内所有球员的运动码数，并按球员ID和比赛ID分组。

```sql
CREATE MATERIALIZED VIEW player_yards_by_game
WITH (timescaledb.continuous) AS
SELECT t.player_id, t.gameid,
 time_bucket(INTERVAL '1 day', t."time") AS bucket,
 SUM(t.dis) AS yards
FROM tracking t
GROUP BY t.player_id, t.gameid, bucket;
```

1.  [传球比赛中，按球员和比赛计算的码数](#传球比赛中-按球员和比赛计算的码数)
2.  [球员在比赛中的平均码数](#球员在比赛中的平均码数)
3.  [按球员类型计算每场比赛的平均和中位数码数（不计算个人的平均水平）](#按球员类型计算每场比赛的平均和中位数码数-不计算个人的平均水平)
4.  [球员在进攻时参与的快照比赛次数](#球员在进攻时参与的快照比赛次数)
5.  [比赛次数与得分对比](#比赛次数与得分对比)
6.  [每个位置前三名球员的平均比赛码数](#每个位置前三名球员的平均比赛码数)

### **传球比赛中，按球员和比赛计算的码数**

使用此查询从连续聚合中获取码数数据。然后你可以将该数据与球员表连接以获取球员详细信息。

```sql
SELECT a.player_id, display_name, SUM(yards) AS yards, gameid
FROM player_yards_by_game a
LEFT JOIN player p ON a.player_id = p.player_id
GROUP BY a.player_id, display_name, gameid
ORDER BY gameid ASC, display_name
```

你的数据应该像这样：

|player_id| display_name | yards | gameid  |
|-----| ------------- |:-------------:| -----:|
|2555415| Austin Hooper     | 765.52 | 2018090600 |
|2556445| Brian Poole    | 661.74     |   2018090600 |
|2560854| Calvin Ridley | 822.3     |    2018090600 |

这个查询可以作为许多其他分析问题的基础。本节将返回此查询以进行进一步分析。

### **球员在比赛中的平均码数**

这个查询使用TimescaleDB的一个百分位函数来找到单个球员每场比赛的平均码数。

```sql
WITH sum_yards AS (
  SELECT a.player_id, display_name, SUM(yards) AS yards, gameid
  FROM player_yards_by_game a
  LEFT JOIN player p ON a.player_id = p.player_id
  GROUP BY a.player_id, display_name, gameid
)
SELECT player_id, display_name, mean(percentile_agg(yards)) as yards
FROM sum_yards
GROUP BY player_id, display_name
ORDER BY yards DESC
```

当你运行这个查询时，可能会注意到第一行的`player_id`和`display_name`为空。这一行代表足球的平均码数据。

### **按球员类型计算每场比赛的平均和中位数码数（不计算个人的平均水平）**

对于这个查询，你使用TimescaleDB的另一个百分位函数`percentile_agg`。你将`percentile_agg`函数设置为找到第五十个百分位，它返回大约的中位数。

```sql
WITH sum_yards AS (
--Add position to the table to allow for grouping by it later
  SELECT a.player_id, display_name, SUM(yards) AS yards, p.position, gameid
  FROM player_yards_by_game a
  LEFT JOIN player p ON a.player_id = p.player_id
  GROUP BY a.player_id, display_name, p.position, gameid
)
--Find the mean and median for each position type
SELECT position, mean(percentile_agg(yards)) AS mean_yards, approx_percentile(0.5, percentile_agg(yards)) AS median_yards
FROM sum_yards
GROUP BY position
ORDER BY mean_yards DESC
```

如果你滚动到结果的底部，你应该看到：

|position| mean_yards        | median_yards  |
|-----| ------------- |:----------------:|
|HB| 275.04279069767404    | 250.88667462709043 |
|DE| 185.76162011173133   | 33.750683636185684 |
|FB| 100.37912844036691 | 67.0876116670915 |
|DT| 19.692499999999992  | 17.796475991050432 |

注意，防守端锋（DE）位置的均值和中位数之间有很大的差异。中位数数据意味着大多数DE球员在传球比赛中并没有跑很多。然而，均值数据意味着一些DE球员必须跑了很多。你可能想要找出这些表现出色的防守球员是谁。

### **球员在进攻时参与的快照比赛次数**

在这个查询中，你计算了球员在进攻时参与的传球事件次数。你可能注意到这个查询比使用连续聚合的上述查询慢得多。这里看到的速度与不使用连续聚合的其他查询相比是可比的。

```sql
WITH snap_events AS (
-- Create a table that filters the play events to show only snap plays
-- and display the players team information
 SELECT DISTINCT player_id, t.event, t.gameid, t.playid,
  CASE
   WHEN t.team = 'away' THEN g.visitor_team
   WHEN t.team = 'home' THEN g.home_team
   ELSE NULL
   END AS team_name
 FROM tracking t
 LEFT JOIN game g ON t.gameid = g.game_id
 WHERE t.event LIKE '%snap%'
)
-- Count these events and filter results to only display data when the player was
-- on the offensive
SELECT a.player_id, pl.display_name, COUNT(a.event) AS play_count, a.team_name
FROM snap_events a
LEFT JOIN play p ON a.gameid = p.gameid AND a.playid = p.playid
LEFT JOIN player pl ON a.player_id = pl.player_id
WHERE a.team_name = p.possessionteam
GROUP BY a.player_id, pl.display_name, a.team_name
ORDER BY play_count DESC
```

注意，两次传球比赛中最高的传球次数是本·罗斯里斯伯格和朱朱·史密斯-舒斯特，分别是匹兹堡钢人队的四分卫和外接手。这两个可能是你在选秀你的梦幻足球联赛时要考虑的两个好选择。

### **比赛次数与得分对比**

使用此查询获取2018赛季每场比赛的进攻次数和最终得分数据。这些数据按球队分开，以便我们可以比较进攻次数与球队的胜负。

```sql
WITH play_count AS (
-- Count distinct plays, join on the stadium and game tables for team names and game date
SELECT gameid, COUNT(playdescription) AS plays, p.possessionteam as team_name, g.game_date
FROM play p
LEFT JOIN game g ON p.gameid = g.game_id
GROUP BY gameid, team_name, game_date
), visiting_games AS (
-- Join on scores to grab only the visting team's data
SELECT gameid, plays, s.visitor_team as team_name, s.visitor_score AS team_score FROM play_count p
INNER JOIN scores s ON p.team_name = s.visitor_team_abb
AND p.game_date = s."date"
), home_games AS (
-- Join on scores to grab only the home team's data
SELECT gameid, plays, s.home_team AS team_name , s.home_score AS team_score FROM play_count p
INNER JOIN scores s ON p.team_name = s.home_team_abb
AND p.game_date = s."date"
)
-- union the two resulting tables together
SELECT * FROM visiting_games
UNION ALL
SELECT * FROM home_games
ORDER BY gameid ASC, team_score DESC
```

下面的图像是你可以利用这个查询收集的数据创建的可视化示例。散点图按组显示，显示获胜球队的比赛次数和得分为金色，失利球队的比赛次数和得分为棕色。

<img src="https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/nfl_tutorial/wins_vs_plays.png" alt="Wins vs Plays">

y轴，或一支球队在单场比赛中的进攻次数显示，更多的进攻次数并不总是意味着保证胜利。事实上，单场比赛进攻次数最多的前三支球队似乎都输了。你可以从这个查询中收集到许多有趣的事实，这个散点图只是其中一种可能性。

### **每个位置前三名球员的平均比赛码数**

你可以使用这个PostgreSQL查询提取个人球员在单场比赛中跑的平均码数。这个查询只包括每个位置类型中最高的三名球员的平均码数值。数据按所有球员的平均每场比赛码数排序。这在稍后变得重要。

注意：由于平均码数值较低，此查询排除了一些位置类型，被排除的位置有踢球手、弃踢手、鼻锋、长开球手和防守截锋

```sql
WITH total_yards AS (
-- This table sums the yards a player runs over each game
 SELECT t.player_id, SUM(yards) AS yards, t.gameid
 FROM player_yards_by_game t
 GROUP BY t.player_id, t.gameid
), avg_yards AS (
-- This table takes the average of the yards run by each player and calls out thier position
 SELECT p.player_id, p.display_name, AVG(yards) AS avg_yards, p."position"
 FROM total_yards t
 LEFT JOIN player p ON t.player_id = p.player_id
 GROUP BY p.player_id, p.display_name, p."position"
), ranked_vals AS (
-- This table ranks each player by the average yards they run per game
SELECT a.*, RANK() OVER (PARTITION BY a."position" ORDER BY avg_yards DESC)
FROM avg_yards AS a
), ranked_positions AS (
-- This table takes the average of the average yards run for each player so that we can order
-- the positions by this average of averages
SELECT v."position", AVG(v.avg_yards) AS avg_yards_positions
FROM ranked_vals v
GROUP BY v."position"
)
SELECT v.*, p.avg_yards_positions FROM ranked_vals v
LEFT JOIN ranked_positions p ON v.position = p.position
WHERE v.rank <= 3 AND v.position != 'null' AND v.position NOT IN ('K', 'P', 'NT', 'LS', 'DT')
ORDER BY p.avg_yards_positions DESC, v.rank ASC
```

这是你可以用这些数据创建的一种可能的可视化：

<img 
src="https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/nfl_tutorial/top_3_players.png" alt="Top Three Players by Position">
注意，自由安全卫球员的平均码数总体上高于外接手（这是因为我们如上所述对数据进行了排序）。然而，个别外接手平均每场比赛跑的码数更多。同时，注意凯尔·朱斯奇克比其他全卫球员平均跑的码数多得多。

## 可视化开球前位置和球员移动

**安装pandas和matplotlib：**

```bash
pip install pandas matplotlib
```

**绘制足球场**

```python
def generate_field():
    """生成一个逼真的美式足球场，带有线条编号和散列标记。

    Returns:
        [tuple]: (figure, axis)
    """
    rect = patches.Rectangle((0, 0), 120, 53.3, linewidth=2,
                             edgecolor='black', facecolor='green', zorder=0)

    fig, ax = plt.subplots(1, figsize=(12, 6.33))
    ax.add_patch(rect)

    # 线条编号
    plt.plot([10, 10, 20, 20, 30, 30, 40, 40, 50, 50, 60, 60, 70, 70, 80,
              80, 90, 90, 100, 100, 110, 110, 120, 0, 0, 120, 120],
             [0, 53.3, 53.3, 0, 0, 53.3, 53.3, 0, 0, 53.3, 53.3, 0, 0, 53.3,
              53.3, 0, 0, 53.3, 53.3, 0, 0, 53.3, 53.3, 53.3, 0, 0, 53.3],
             color='white')
    for x in range(20, 110, 10):
        numb = x
        if x > 50:
            numb = 120-x
        plt.text(x, 5, str(numb - 10), horizontalalignment='center', fontsize=20, color='white')
        plt.text(x-0.95, 53.3-5, str(numb-10),
                 horizontalalignment='center', fontsize=20, color='white',rotation=180)

    # 散列标记
    for x in range(11, 110):
        ax.plot([x, x], [0.4, 0.7], color='white')
        ax.plot([x, x], [53.0, 52.5], color='white')
        ax.plot([x, x], [22.91, 23.57], color='white')
        ax.plot([x, x], [29.73, 30.39], color='white')

    # 设置限制并隐藏轴
    plt.xlim(0, 120)
    plt.ylim(-5, 58.3)
    plt.axis('off')

    return fig, ax
```

**根据`game_id`和`play_id`绘制球员移动**

```python
conn = psycopg2.connect(database="db",
                        host="host",
                        user="user",
                        password="pass",
                        port="111")

def draw_play(game_id, play_id, home_label='position', away_label='position', movements=False):
    """生成图表以可视化球员开球前的位置和
      比赛中的运动。

    参数：
        game_id (int)
        play_id (int)
        home_label (str, 可选): 默认是'position'，但可以是'displayname'
          或其他表中可用的列名。
        away_label (str, 可选): 默认是'position'，但可以是'displayname'
          或其他表中可用的列名。
        movements (bool, 可选): 如果为False，只绘制开球前的位置。
          如果为True，也绘制运动轨迹。
    """
    # 查询给定比赛的所有追踪数据
    sql = "SELECT * FROM tracking WHERE gameid={game} AND playid={play} AND team='home'"\
    .format(game=game_id, play=play_id)
    home_team = pd.read_sql(sql, conn)

    sql = "SELECT * FROM tracking WHERE gameid={game} AND playid={play} AND team='away'"\
    .format(game=game_id, play=play_id)
    away_team = pd.read_sql(sql, conn)

    # 生成橄榄球场
    fig, ax = generate_field()

    # 查询开球前球员位置
    home_pre_snap = home_team.query('event == "ball_snap"')
    away_pre_snap = away_team.query('event == "ball_snap"')

    # 使用散点图可视化开球前位置
    home_pre_snap.plot.scatter(x='x', y='y', ax=ax, color='yellow', s=35, zorder=3)
    away_pre_snap.plot.scatter(x='x', y='y', ax=ax, color='blue', s=35, zorder=3)

    # 使用*label*参数的值注释图表上的球员位置或名称
    home_positions = home_pre_snap[home_label].tolist()
    away_positions = away_pre_snap[away_label].tolist()
    for i, pos in enumerate(home_positions):
        ax.annotate(pos, (home_pre_snap['x'].tolist()[i], home_pre_snap['y'].tolist()[i]))
    for i, pos in enumerate(away_positions):
        ax.annotate(pos, (away_pre_snap['x'].tolist()[i], away_pre_snap['y'].tolist()[i]))

    if movements:
        # 可视化主队球员运动
        home_players = home_team['player_id'].unique().tolist()
        for player_id in home_players:
            df = home_team.query('player_id == {id}'.format(id=player_id))
            df.plot(x='x', y='y', ax=ax, linewidth=4, legend=False)

        # 可视化客队球员运动
        away_players = away_team['player_id'].unique().tolist()
        for player_id in away_players:
            df = away_team.query('player_id == {id}'.format(id=player_id))
            df.plot(x='x', y='y', ax=ax, linewidth=4, legend=False)

    # 查询比赛描述和控球队伍并添加到标题中
    sql = """SELECT gameid, playid, playdescription, possessionteam FROM play
             WHERE gameid = {game} AND playid = {play}""".format(game=game_id, play=play_id)
    play_info = pd.read_sql(sql, conn).to_dict('records')
    plt.title('Possession team: {team}\nPlay: {play}'.format(team=play_info[0]['possessionteam'],
                                                             play=play_info[0]['playdescription']))
    # 显示图表
    plt.show()
```

然后，你可以像这样运行`draw_play`函数来可视化开球前的球员位置：

```python
draw_play(game_id=2018112900,
          play_id=2826,
          movements=False)
```

![pre snap players figure](https://assets.timescale.com/docs/images/tutorials/nfl_tutorial/player_movement_pre_snap.png) 

你也可以通过将`movements`设置为`True`来可视化比赛中球员的移动：

```python
draw_play(game_id=2018112900,
          play_id=2826,
          home_label='position',
          away_label='displayname',
          movements=True)
```

![player movement figure](https://assets.timescale.com/docs/images/tutorials/nfl_tutorial/player_movement.png) 

## 资源

*   [Kaggle上的NFL Big Data Bowl 2021](https://www.kaggle.com/c/nfl-big-data-bowl-2021) 

[extra-download]: https://assets.timescale.com/docs/downloads/nfl_2018.zip 
[install-timescale]: /getting-started/latest/
[kaggle-download]: https://www.kaggle.com/c/nfl-big-data-bowl-2021/data 
[psql-install]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql



