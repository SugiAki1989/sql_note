## :memo: Overview

Preppin' Data challenge の「2019: Week 4」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/03/2019-week-4.html)
- [Answer](https://preppindata.blogspot.com/2019/03/2019-week-4-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。NBA の試合サマリーデータがあって、ここから様々なお題に答える、というお題。

```sql
select * from p2019w04t1 limit 10;
    date    |    opponent     |    result    |  w_l  |  hi_points   | hi_rebounds | hi_assists
------------+-----------------+--------------+-------+--------------+-------------+------------
 火, 1月 1  | vsBoston        | W120-111     | 21-17 | Aldridge 32  | Aldridge 9  | DeRozan 10
 金, 1月 4  | vsToronto       | W125-107     | 22-17 | Aldridge 23  | DeRozan 14  | DeRozan 11
 日, 1月 6  | vsMemphis       | W108-88      | 23-17 | White 19     | DeRozan 9   | Aldridge 7
 火, 1月 8  | @Detroit        | W119-107     | 24-17 | DeRozan 26   | DeRozan 7   | DeRozan 9
 木, 1月 10 | @Memphis        | L96-86       | 24-18 | Forbes 14    | Gasol 12    | DeRozan 4
 金, 1月 11 | vsOklahoma City | W154-147 2OT | 25-18 | Aldridge 56  | Aldridge 9  | DeRozan 11
 日, 1月 13 | @Oklahoma City  | L122-112     | 25-19 | Belinelli 24 | Poeltl 10   | DeRozan 4
 火, 1月 15 | vsCharlotte     | L108-93      | 25-20 | Aldridge 28  | Aldridge 10 | White 7
 木, 1月 17 | @Dallas         | W105-101     | 26-20 | Belinelli 17 | Poeltl 7    | DeRozan 9
 土, 1月 19 | @Minnesota      | W116-113     | 27-20 | Aldridge 25  | Aldridge 9  | Mills 8
(10 rows)
```

提供は Excel で、私の環境では Numbers で開いて、CSV にエクスポートして利用しているため。とりあえず、日付をきれいにしておく。

```sql
select
    date,
    split_part(date, ',', 2) as d,
    replace(split_part(date, ',', 2), ' ', '') as dd,
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 1) as d1,
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 2) as d2,
    ('2019-' ||
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 1)::text || '-' ||
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 2) ::text)::date as date_mod
from
    p2019w04t1
limit 10;

    date    |    d    |  dd   | d1 | d2 |  date_mod
------------+---------+-------+----+----+------------
 火, 1月 1  |  1月 1  | 1月1  | 1  | 1  | 2019-01-01
 金, 1月 4  |  1月 4  | 1月4  | 1  | 4  | 2019-01-04
 日, 1月 6  |  1月 6  | 1月6  | 1  | 6  | 2019-01-06
 火, 1月 8  |  1月 8  | 1月8  | 1  | 8  | 2019-01-08
 木, 1月 10 |  1月 10 | 1月10 | 1  | 10 | 2019-01-10
 金, 1月 11 |  1月 11 | 1月11 | 1  | 11 | 2019-01-11
 日, 1月 13 |  1月 13 | 1月13 | 1  | 13 | 2019-01-13
 火, 1月 15 |  1月 15 | 1月15 | 1  | 15 | 2019-01-15
 木, 1月 17 |  1月 17 | 1月17 | 1  | 17 | 2019-01-17
 土, 1月 19 |  1月 19 | 1月19 | 1  | 19 | 2019-01-19
(10 rows)
```

あとはいくつかカラムを処理して、条件付けれるようにしておく。

```sql
with tmp as (
select
    *,
    ('2019-' ||
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 1)::text || '-' ||
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 2) ::text)::date as date_mod
from
    p2019w04t1
)
select
    date_mod,
    case when left(opponent, 1) = '@' then 'home'
    else 'away' end as home_away,
    case when left(result, 1) = 'W' then 'win'
    when left(result, 1) = 'L' then 'lose'
    else 'tie' end as win_lose,
    split_part(hi_points, ' ', 1) as hi_points_player,
    split_part(hi_points, ' ', 2)::int as hi_points_score
from
    tmp
;
  date_mod  | home_away | win_lose | hi_points_player | hi_points_score
------------+-----------+----------+------------------+-----------------
 2019-01-01 | away      | win      | Aldridge         |              32
 2019-01-04 | away      | win      | Aldridge         |              23
 2019-01-06 | away      | win      | White            |              19
 2019-01-08 | home      | win      | DeRozan          |              26
 2019-01-10 | home      | lose     | Forbes           |              14
 2019-01-11 | away      | win      | Aldridge         |              56
 2019-01-13 | home      | lose     | Belinelli        |              24
 2019-01-15 | away      | lose     | Aldridge         |              28
 2019-01-17 | home      | win      | Belinelli        |              17
 2019-01-19 | home      | win      | Aldridge         |              25

```

このチャレンジは Prep の画面からでも計算しなくても結果がわかるということを示したいようなので、ここではテキトウに集計しておく。
下記は勝利かつホームゲームで最も得点をとった選手。

```sql
with tmp as (
select
    *,
    ('2019-' ||
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 1)::text || '-' ||
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 2) ::text)::date as date_mod,
    case when left(opponent, 1) = '@' then 'home'
    else 'away' end as home_away,
    case when left(result, 1) = 'W' then 'win'
    when left(result, 1) = 'L' then 'lose'
    else 'tie' end as win_lose,
    split_part(hi_points, ' ', 1) as hi_points_player,
    split_part(hi_points, ' ', 2)::int as hi_points_score
from
    p2019w04t1
)
select
    date_mod,
    win_lose,
    home_away,
    hi_points_player,
    hi_points_score
from
    tmp
where
    win_lose = 'win' and home_away = 'home'
order by
    hi_points_score desc
limit 3
;

  date_mod  | win_lose | home_away | hi_points_player | hi_points_score
------------+----------+-----------+------------------+-----------------
 2019-12-30 | win      | home      | Aldridge         |              38
 2019-10-23 | win      | home      | Aldridge         |              37
 2019-11-24 | win      | home      | Aldridge         |              33
```

> Q6. 2018 年 10 月に最も頻繁にゲームで最も多くのポイントを獲得したプレーヤーは?
> 答え: デローザン

実際の Prep ファイルがないので確認できないが、これは Aldridge だと思われる。

```sql
with tmp as (
select
    *,
    ('2019-' ||
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 1)::text || '-' ||
    split_part(replace(split_part(date, ',', 2), ' ', ''), '月', 2) ::text)::date as date_mod,
    case when left(opponent, 1) = '@' then 'home'
    else 'away' end as home_away,
    case when left(result, 1) = 'W' then 'win'
    when left(result, 1) = 'L' then 'lose'
    else 'tie' end as win_lose,
    split_part(hi_points, ' ', 1) as hi_points_player,
    split_part(hi_points, ' ', 2)::int as hi_points_score
from
    p2019w04t1
)
select
    date_mod,
    win_lose,
    home_away,
    hi_points_player,
    hi_points_score
from
    tmp
where
    substr(date_mod::text, 1, 7) = '2019-10'
order by
    hi_points_score desc
limit 3
;

  date_mod  | win_lose | home_away | hi_points_player | hi_points_score
------------+----------+-----------+------------------+-----------------
 2019-10-23 | win      | home      | Aldridge         |              37
 2019-10-30 | win      | away      | DeRozan          |              34
 2019-10-28 | win      | away      | DeRozan          |              30
(3 rows)
```

## :closed_book: Reference

None
