---
标题: 使用连续聚合和超级函数进行高级分析
摘要: 利用连续聚合和超级函数对美国国家橄榄球联盟（NFL）球员活动开展高级分析。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [连续聚合，超级函数，分析]
---

# 使用连续聚合和超函数进行高级分析

在本教程中，您已经摄入了数据并运行了聚合查询。然后，您使用连续聚合提高了分析的性能。

现在，让我们探讨一些使用PostgreSQL和TimescaleDB分析数据的想法，以帮助您更深入地了解NFL赛季期间球员活动的情况。

<Highlight type="tip">
部分分析包括可视化，以帮助您看到这些数据的潜在用途。这些是使用[Matplotlib](https://matplotlib.org/) Python模块创建的，这是许多优秀可视化工具之一。</Highlight>

### 球员每场比赛的平均跑码数

此查询使用百分位近似[超函数][api-hyperfunctions]来查找单个球员每场比赛的平均跑码数。

```sql
WITH sum_yards AS (
  SELECT a.player_id, displayname, SUM(yards) AS yards, gameid
  FROM player_yards_by_game a
  LEFT JOIN player p ON a.player_id = p.player_id
  GROUP BY a.player_id, displayname, gameid
)
SELECT player_id, displayname, mean(percentile_agg(yards)) as yards
FROM sum_yards
GROUP BY player_id, displayname
ORDER BY yards DESC
```

您的数据应该如下所示：

|player_id|displayname|yards|
|-|-|-|
|NULL|NULL|2872.5647430830086|
|2508061|Antonio Brown|1125.1706666666641|
|2556214|Tyreek Hill|1007.1073333333323|
|2543495|Davante Adams|971.6339999999967|
|2543498|Brandin Cooks|969.3762499999964|

运行此查询时，您可能会注意到第一行的`player_id`和`displayname`为null。这一行代表足球的平均码数数据。

### 按球员类型计算每场比赛的平均和中位数跑码数

对于这个查询，您使用TimescaleDB的另一个百分位函数`percentile_agg`。您可以使用`percentile_agg`函数找到第五十百分位，即近似中位数。

```sql
WITH sum_yards AS (
  SELECT a.player_id, displayname, SUM(yards) AS yards, p.position, gameid
  FROM player_yards_by_game a
  LEFT JOIN player p ON a.player_id = p.player_id
  GROUP BY a.player_id, displayname, p.position, gameid
)
SELECT position, mean(percentile_agg(yards)) AS mean_yards, approx_percentile(0.5, percentile_agg(yards)) AS median_yards
FROM sum_yards
GROUP BY position
ORDER BY mean_yards DESC
```

如果您滚动到结果的底部，您应该看到：

|position|mean_yards|median_yards|
|-|-|-|
|HB|275.04279069767404|250.88667462709043|
|DE|185.76162011173133|33.750683636185684|
|FB|100.37912844036691|67.0876116670915|
|DT|19.692499999999992|17.796475991050432|

注意到防守端锋（DE）位置的平均值和中位数值之间有很大的差距。中位数数据意味着大多数DE球员在传球比赛中并没有跑很多。然而，平均数据意味着一些DE球员必须跑了很多。

### 球员在进攻时的快照比赛次数

在这个查询中，您计算了球员参与传球事件的次数。您可能会注意到这个查询比使用连续聚合的上述查询运行得慢得多。这里看到的速度与不使用连续聚合的其他查询相当。

```sql
WITH snap_events AS (
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
SELECT a.player_id, pl.displayname, COUNT(a.event) AS play_count, a.team_name
FROM snap_events a
LEFT JOIN play p ON a.gameid = p.gameid AND a.playid = p.playid
LEFT JOIN player pl ON a.player_id = pl.player_id
WHERE a.team_name = p.possessionteam
GROUP BY a.player_id, pl.displayname, a.team_name
ORDER BY play_count DESC
```

注意到两次最高传球比赛是本·罗斯里斯伯格（Ben Roethlisberger）和朱朱·史密斯-舒斯特（JuJu Smith-Schuster）的，分别是匹兹堡钢人队的四分卫和外接手。

|player_id|displayname|play_count|team|
|-|-|-|-|
|2506109|Ben Roethlisberger|725|PIT|
|2558149|JuJu Smith-Schuster|691|PIT|
|2533031|Andrew Luck|683|IND|

### 比赛次数与得分对比

使用此查询获取2018赛季每场比赛的进攻次数和最终得分数据。这些数据按球队分开，以便我们可以比较球队的进攻次数与胜利或失败。

```sql
WITH play_count AS (
  SELECT gameid, COUNT(playdescription) AS plays, p.possessionteam as team_name, g.game_date
  FROM play p
  LEFT JOIN game g ON p.gameid = g.game_id
  GROUP BY gameid, team_name, game_date
), visiting_games AS (
  SELECT gameid, plays, s.visitor_team as team_name, s.visitor_score AS team_score FROM play_count p
  INNER JOIN scores s ON p.team_name = s.visitor_team_abb
  AND p.game_date = s."date"
), home_games AS (
  SELECT gameid, plays, s.home_team AS team_name , s.home_score AS team_score FROM play_count p
  INNER JOIN scores s ON p.team_name = s.home_team_abb
  AND p.game_date = s."date"
)
SELECT * FROM visiting_games
UNION ALL
SELECT * FROM home_games
ORDER BY gameid ASC, team_score DESC
```

这张图片是您可以使用此查询结果创建的可视化示例。散点图按组显示，获胜球队的进攻次数和得分为金色，失利球队的进攻次数和得分为棕色。

![胜利与进攻次数对比](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/nfl_tutorial/wins_vs_plays.png)

y轴，或一支球队在单场比赛中的进攻次数，显示更多的《传球》进攻并不总是意味着保证胜利。事实上，单场比赛进攻次数最多的前三支球队似乎都输了。在足球中，这在逻辑上是合理的，因为落后的球队在比赛后期往往会传球更多。您可以从这类查询中了解到许多有趣的事实，这个散点图只是其中一种可能性。

### 每个位置前三名球员每场比赛的平均码数

您可以使用此PostgreSQL查询提取个人球员在单场比赛中跑的平均码数。此查询仅包括每个位置类型中平均码数最高的前三名球员。数据按所有球员的平均码数排序。这在稍后变得重要。

<Highlight type="note">
由于平均码数过低，此查询排除了一些位置类型，被排除的位置有踢球手、弃踢手、鼻截锋、长开球手和防守截锋。</Highlight>

```sql
WITH total_yards AS (
  SELECT t.player_id, SUM(yards) AS yards, t.gameid
  FROM player_yards_by_game t
  GROUP BY t.player_id, t.gameid
), avg_yards AS (
  SELECT p.player_id, p.displayname, AVG(yards) AS avg_yards, p."position"
  FROM total_yards t
  LEFT JOIN player p ON t.player_id = p.player_id
  GROUP BY p.player_id, p.displayname, p."position"
), ranked_vals AS (
  SELECT a.*, RANK() OVER (PARTITION BY a."position" ORDER BY avg_yards DESC)
  FROM avg_yards AS a
), ranked_positions AS (
  SELECT v."position", AVG(v.avg_yards) AS avg_yards_positions
  FROM ranked_vals v
  GROUP BY v."position"
)
SELECT v.*, p.avg_yards_positions FROM ranked_vals v
LEFT JOIN ranked_positions p ON v.position = p.position
WHERE v.rank <= 3 AND v.position != 'null' AND v.position NOT IN ('K', 'P', 'NT', 'LS', 'DT')
ORDER BY p.avg_yards_positions DESC, v.rank ASC
```

这是您可以使用这些数据创建的一种可能的可视化：

![每个位置前三名球员](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/nfl_tutorial/top_3_players.png)

注意到自由安全卫球员的平均码数总体上高于外接手（这是因为我们如上所述对数据进行了排序）。然而，个别外接手平均每场比赛跑的码数更多。同时，注意到凯尔·朱希乔克（Kyle Juszczyk）平均每场比赛比其他全卫球员跑的码数多得多。

## 这只是半场休息！

这些示例查询只是您可以对任何时间序列数据进行的分析的开始示例，使用常规SQL和连续聚合等有用特性。考虑加入我们提供的运动场数据，看看球队是否倾向于在米尔高体育场得分或跑得更少。天然草皮或人造草皮是否一致影响任何球队？


