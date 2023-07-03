## :memo: Overview

Preppin' Data challenge の「2019: Week 19」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/06/2019-week-19.html)
- [Answer](https://preppindata.blogspot.com/2019/06/2019-week-19-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。ツール・ド・フランスのレース結果のデータ。特定の条件に合致するチームを見つけるというお題。

```sql
select * from p2019w19t1 limit 10;

 rank |       rider       | rider_no_ |         team          |    times     |      gap
------+-------------------+-----------+-----------------------+--------------+----------------
    1 | GERAINT THOMAS    |         8 | TEAM SKY              | 83h 17' 13'' | -
    2 | TOM DUMOULIN      |        32 | TEAM SUNWEB           | 83h 19' 04'' | + 00h 01' 51''
    3 | CHRIS FROOME      |         1 | TEAM SKY              | 83h 19' 37'' | + 00h 02' 24''
    4 | PRIMOŽ ROGLIC     |       166 | TEAM LOTTO NL - JUMBO | 83h 20' 35'' | + 00h 03' 22''
    5 | STEVEN KRUIJSWIJK |       161 | TEAM LOTTO NL - JUMBO | 83h 23' 21'' | + 00h 06' 08''
    6 | ROMAIN BARDET     |        21 | AG2R LA MONDIALE      | 83h 24' 10'' | + 00h 06' 57''
    7 | MIKEL LANDA MEANA |        75 | MOVISTAR TEAM         | 83h 24' 50'' | + 00h 07' 37''
    8 | DANIEL MARTIN     |        91 | UAE TEAM EMIRATES     | 83h 26' 18'' | + 00h 09' 05''
    9 | ILNUR ZAKARIN     |       141 | TEAM KATUSHA ALPECIN  | 83h 29' 50'' | + 00h 12' 37''
   10 | NAIRO QUINTANA    |        71 | MOVISTAR TEAM         | 83h 31' 31'' | + 00h 14' 18''
(10 rows)
```

ギャップタイムを秒数に変換する。

```sql
select
    rank,
    rider,
    rider_no_,
    team,
    times,
    -- 'や''やh、半角スペースのいずれかの文字を削除
    regexp_replace(times, '[''h ]', '', 'g') as time2,
    left(regexp_replace(times, '[''h ]', '', 'g'), 2)::int * 60 * 60
    +
    substring(regexp_replace(times, '[''h ]', '', 'g'), 3, 2)::int * 60
    +
    right(regexp_replace(times, '[+''h ]', '', 'g'), 2)::int as time_sec,

    left(regexp_replace(replace(gap, '-', '0'), '[+''h ]', '', 'g'),2)::int * 60 * 60
    +
    case when substring(regexp_replace(replace(gap, '-', '0'), '[+''h ]', '', 'g'),3,2) = '' then 0
        else substring(regexp_replace(replace(gap, '-', '0'), '[+''h ]', '', 'g'),3,2)::int * 60 end
    +
    right(regexp_replace(replace(gap, '-', '0'), '[+''h ]', '', 'g'),2)::int as gap_sec
from
    p2019w19t1
limit 10;

 rank |       rider       | rider_no_ |         team          |    times     | time2  | time_sec | gap_sec
------+-------------------+-----------+-----------------------+--------------+--------+----------+---------
    1 | GERAINT THOMAS    |         8 | TEAM SKY              | 83h 17' 13'' | 831713 |   299833 |       0
    2 | TOM DUMOULIN      |        32 | TEAM SUNWEB           | 83h 19' 04'' | 831904 |   299944 |     111
    3 | CHRIS FROOME      |         1 | TEAM SKY              | 83h 19' 37'' | 831937 |   299977 |     144
    4 | PRIMOŽ ROGLIC     |       166 | TEAM LOTTO NL - JUMBO | 83h 20' 35'' | 832035 |   300035 |     202
    5 | STEVEN KRUIJSWIJK |       161 | TEAM LOTTO NL - JUMBO | 83h 23' 21'' | 832321 |   300201 |     368
    6 | ROMAIN BARDET     |        21 | AG2R LA MONDIALE      | 83h 24' 10'' | 832410 |   300250 |     417
    7 | MIKEL LANDA MEANA |        75 | MOVISTAR TEAM         | 83h 24' 50'' | 832450 |   300290 |     457
    8 | DANIEL MARTIN     |        91 | UAE TEAM EMIRATES     | 83h 26' 18'' | 832618 |   300378 |     545
    9 | ILNUR ZAKARIN     |       141 | TEAM KATUSHA ALPECIN  | 83h 29' 50'' | 832950 |   300590 |     757
   10 | NAIRO QUINTANA    |        71 | MOVISTAR TEAM         | 83h 31' 31'' | 833131 |   300691 |     858
(10 rows)
```

あとは特定の条件に合致するチームを取り出して、秒数を分数に変換する。

```sql
with tmp as (
select
    rank,
    rider,
    rider_no_,
    team,
    times,
    -- 'や''やh、半角スペースのいずれかの文字を削除
    regexp_replace(times, '[''h ]', '', 'g') as time2,
    left(regexp_replace(times, '[''h ]', '', 'g'), 2)::int * 60 * 60
    +
    substring(regexp_replace(times, '[''h ]', '', 'g'), 3, 2)::int * 60
    +
    right(regexp_replace(times, '[+''h ]', '', 'g'), 2)::int as time_sec,

    left(regexp_replace(replace(gap, '-', '0'), '[+''h ]', '', 'g'),2)::int * 60 * 60
    +
    case when substring(regexp_replace(replace(gap, '-', '0'), '[+''h ]', '', 'g'),3,2) = '' then 0
        else substring(regexp_replace(replace(gap, '-', '0'), '[+''h ]', '', 'g'),3,2)::int * 60 end
    +
    right(regexp_replace(replace(gap, '-', '0'), '[+''h ]', '', 'g'),2)::int as gap_sec
from
    p2019w19t1
), tmp2 as (
select
    team,
    avg(gap_sec) as team_avg_gap_in_sec,
    count(rider) as num_of_riders,
    trunc(avg(gap_sec)/ 60, 0) as team_avg_gap_in_min
from
    tmp
group by
    team
having
    count(rider) >= 7 and
    avg(gap_sec) <= 100 * 60
)
select * from tmp2;

     team      |  team_avg_gap_in_sec  | num_of_riders | team_avg_gap_in_min
---------------+-----------------------+---------------+---------------------
 TEAM SKY      | 5795.5714285714285714 |             7 |                  96
 MOVISTAR TEAM | 5836.8571428571428571 |             7 |                  97
(2 rows)
```

## :closed_book: Reference

None
