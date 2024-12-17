---
标题: 可视化开球前位置和球员移动情况
摘要: 使用 Matplotlib 创建足球比赛场景的可视化展示。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [连续聚合，超级函数，分析，Pandas，Matplotlib]
---

# 可视化开球前的位置和球员移动

有趣的是，NFL数据集包括了每个橄榄球比赛中球员移动的数据。将时间序列数据的变化可视化往往能提供更多的洞见。在本节中，我们使用`pandas`和`matplotlib`来可视化赛季中的一场比赛。

## 安装pandas和matplotlib

```bash
pip install pandas matplotlib
```

## 绘制足球场

```python
def generate_field():
    ""“生成一个带有线条编号和散点标记的真实美式足球场。

    返回：
        [元组]：(图形，轴)
    “""
    rect = patches.Rectangle((0, 0), 120, 53.3, 线宽=2,
                             边缘颜色='黑色', 填充颜色='绿色', zorder=0)

    fig, ax = plt.subplots(1, 图表大小=(12, 6.33))
    ax.add_patch(rect)

    # 线条编号
    plt.plot([10, 10, 20, 20, 30, 30, 40, 40, 50, 50, 60, 60, 70, 70, 80,
              80, 90, 90, 100, 100, 110, 110, 120, 0, 0, 120, 120],
             [0, 53.3, 53.3, 0, 0, 53.3, 53.3, 0, 0, 53.3, 53.3, 0, 0, 53.3,
              53.3, 0, 0, 53.3, 53.3, 0, 0, 53.3, 53.3, 53.3, 0, 0, 53.3], 
            颜色='白色')
    对于x在范围(20, 110, 10)中的每个值：
        numb = x
        如果x > 50：
            numb = 120-x
        plt.text(x, 5, str(numb - 10), 水平对齐='中心', 字体大小=20, 颜色='白色')
        plt.text(x-0.95, 53.3-5, str(numb-10), 
                 水平对齐='中心', 字体大小=20, 颜色='白色', 旋转=180)

    # 散点标记
    对于x在范围(11, 110)中的每个值：
        ax.plot([x, x], [0.4, 0.7], 颜色='白色')
        ax.plot([x, x], [53.0, 52.5], 颜色='白色')
        ax.plot([x, x], [22.91, 23.57], 颜色='白色')
        ax.plot([x, x], [29.73, 30.39], 颜色='白色')

    # 设置限制并隐藏轴
    plt.xlim(0, 120)
    plt.ylim(-5, 58.3)
    plt.axis('off')

    返回fig, ax
```

## 根据`game_id`和`play_id`绘制球员移动

```python
conn = psycopg2.connect(database="db",
                        host="host",
                        user="user",
                        password="pass",
                        port="111")

def draw_play(game_id, play_id, home_label='position', away_label='position', movements=False):
    ""“生成图表以可视化给定比赛中球员的开球前位置和移动。

    参数：
        game_id (int)
        play_id (int)
        home_label (str, 可选): 默认是'position'，但可以是'displayname'
          或其他表中可用的列名。
        away_label (str, 可选): 默认是'position'，但可以是'displayname'
          或其他表中可用的列名。
        movements (bool, 可选): 如果为False，仅绘制开球前的位置。
          如果为True，也绘制移动。
    “""
    # 查询给定比赛的所有追踪数据
    sql = "SELECT * FROM tracking WHERE gameid={game} AND playid={play} AND team='home'"\
    .format(game=game_id, play=play_id)
    home_team = pd.read_sql(sql, conn)

    sql = "SELECT * FROM tracking WHERE gameid={game} AND playid={play} AND team='away'"\
    .format(game=game_id, play=play_id)
    away_team = pd.read_sql(sql, conn)

    # 生成足球场
    fig, ax = generate_field()

    # 查询开球前球员位置
    home_pre_snap = home_team.query('event == "ball_snap"')
    away_pre_snap = away_team.query('event == "ball_snap"')

    # 使用散点图可视化开球前位置
    home_pre_snap.plot.scatter(x='x', y='y', ax=ax, 颜色='黄色', s=35, zorder=3)
    away_pre_snap.plot.scatter(x='x', y='y', ax=ax, 颜色='蓝色', s=35, zorder=3)

    # 使用*label*参数的值注释球员的位置或名称
    home_positions = home_pre_snap[home_label].tolist()
    away_positions = away_pre_snap[away_label].tolist()
    对于i, pos在enumerate(home_positions)中的每个值：
        ax.annotate(pos, (home_pre_snap['x'].tolist()[i], home_pre_snap['y'].tolist()[i]))
    对于i, pos在enumerate(away_positions)中的每个值：
        ax.annotate(pos, (away_pre_snap['x'].tolist()[i], away_pre_snap['y'].tolist()[i]))

    如果movements：
        # 可视化主队球员移动
        home_players = home_team['player_id'].unique().tolist()
        对于player_id在home_players中的每个值：
            df = home_team.query('player_id == {id}'.format(id=player_id))
            df.plot(x='x', y='y', ax=ax, 线宽=4, 图例=False)

        # 可视化客队球员移动
        away_players = away_team['player_id'].unique().tolist()
        对于player_id在away_players中的每个值：
            df = away_team.query('player_id == {id}'.format(id=player_id))
            df.plot(x='x', y='y', ax=ax, 线宽=4, 图例=False)

    # 查询比赛描述和控球队伍并在标题中添加它们
    sql = """SELECT gameid, playid, playdescription, possessionteam FROM play
             WHERE gameid = {game} AND playid = {play}""".format(game=game_id, play=play_id)
    play_info = pd.read_sql(sql, conn).to_dict('records')
    plt.title('控球队伍: {team}\n比赛: {play}'.format(team=play_info[0]['possessionteam'],
    play=play_info[0]['playdescription']))
    # 显示图表
    plt.show()
```

然后，您可以像这样运行`draw_play`函数来可视化开球前的球员位置：

```python
draw_play(game_id=2018112900,
          play_id=2826,
          movements=False)
```

![开球前球员图](https://assets.timescale.com/docs/images/tutorials/nfl_tutorial/player_movement_pre_snap.png) 

您还可以通过将`movements`设置为`True`来可视化比赛中球员的移动：

```python
draw_play(game_id=2018112900,
          play_id=2826,
          home_label='position',
          away_label='displayname',
          movements=True)
```

![球员移动图](https://assets.timescale.com/docs/images/tutorials/nfl_tutorial/player_movement.png) 

## 结论

我们希望本教程能帮助您看到，最初看似非时间序列的数据实际上在经过处理后可以成为时间序列数据。使用TimescaleDB，当您使用[超函数][api-hyperfunctions]和[连续聚合][api-caggs]时，分析时间序列数据可以变得简单（且有趣！）。我们鼓励您在自己的数据库中尝试这些函数，并尝试不同类型的分析。

