## :memo: Overview

Preppin' Data challenge の「2019: Week 2」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/02/2019-week-2.html)
- [Answer](https://preppindata.blogspot.com/2019/02/2019-week-2-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。天気予報データを指標ごとに人間が可読しやすいように整形するというお題。

```sql
select * from p2019w02t1 limit 10;
  city  |     metric      | measure | value |    date
--------+-----------------+---------+-------+------------
 London | Max Temperature | Celsius |    13 | 16/02/2019
 London | Min Temperature | Celsius |     6 | 16/02/2019
 Londen | Precipitation   | mm      |     2 | 16/02/2019
 London | Wind Speed      | mph     |     7 | 16/02/2019
 London | Max Temperature | Celsius |    14 | 17/02/2019
 London | Min Temperature | Celsius |     9 | 17/02/2019
 Lond0n | Precipitation   | mm      |     0 | 17/02/2019
 Londen | Wind Speed      | mph     |    12 | 17/02/2019
 London | Max Temperature | Celsius |    11 | 18/02/2019
 london | Min Temperature | Celsius |     4 | 18/02/2019
(10 rows)

select * from p2019w02t2 limit 10;
   city    |     metric      | measure | value |    date
-----------+-----------------+---------+-------+------------
 Edinburgh | Max Temperature | Celsius |    11 | 16/02/2019
 Edinburgh | Min Temperature | Celsius |     8 | 16/02/2019
 Edinburgh | Precipitation   | mm      |     4 | 16/02/2019
 Edinburgh | Wind Speed      | mph     |    13 | 16/02/2019
 Edinburgh | Max Temperature | Celsius |    12 | 17/02/2019
 Edinborgh | Min Temperature | Celsius |     6 | 17/02/2019
 edinburgh | Precipitation   | mm      |     9 | 17/02/2019
 Edinburgh | Wind Speed      | mph     |    17 | 17/02/2019
 Edinburgh | Max Temperature | Celsius |     9 | 18/02/2019
 Edinburgh | Min Temperature | Celsius |     5 | 18/02/2019
(10 rows)
```

本来は Excel 形式で提供されており TableauPrep のデータインタプリタ機能で必要なセル範囲だけを読み込む必要があるが、PostgreSQL ではそれはできないので、csv に変換したものを利用する。あとは表記ゆれがひどいので修正するための一時的なマスタを作成して、横展開すれば修了。

```sql
with tmp as (
select * from p2019w02t1
union all
select * from p2019w02t2
), tmp2 as (
select
    distinct city,
    case when left(city, 1) = 'L' or left(city, 1) = 'n' then 'London'
    else 'Edinburgh' end as city_mod
from
    tmp
order by
    city asc
), tmp3 as (
select
    t2.city_mod,
    t1.metric || '-' ||t1.measure as metric_measure,
    t1.value,
    t1.date
from
    tmp as t1
left join
    tmp2 as t2
on
    t1.city = t2.city
)
select
    city_mod,
    to_date(date, 'DD/MM/YYYY') as date_mod,
    max(case when metric_measure = 'Max Temperature-Celsius' then value else null end) as Max_Temperature_Celsius,
    max(case when metric_measure = 'Min Temperature-Celsius' then value else null end) as Min_Temperature_Celsius,
    max(case when metric_measure = 'Precipitation-mm' then value else null end) as Precipitation_mm,
    max(case when metric_measure = 'Wind Speed-mph' then value else null end) as Wind_Speed_mph
from
    tmp3
group by
    city_mod,
    date
order by
    city_mod asc,
    date asc
;

# \pset null (null) を使用

 city_mod  |  date_mod  | max_temperature_celsius | min_temperature_celsius | precipitation_mm | wind_speed_mph
-----------+------------+-------------------------+-------------------------+------------------+----------------
 Edinburgh | 2019-02-16 |                      11 |                       8 |                4 |             13
 Edinburgh | 2019-02-17 |                      12 |                       6 |                9 |             17
 Edinburgh | 2019-02-18 |                       9 |                       5 |               11 |             20
 Edinburgh | 2019-02-19 |                       9 |                       7 |               14 |             13
 Edinburgh | 2019-02-20 |                      11 |                       8 |               18 |             13
 Edinburgh | 2019-02-21 |                      13 |                       8 |                8 |             11
 Edinburgh | 2019-02-22 |                      13 |                       9 |                8 |             12
 London    | 2019-02-16 |                      13 |                       6 |                2 |              7
 London    | 2019-02-17 |                      14 |                       9 |                0 |             12
 London    | 2019-02-18 |                      11 |                  (null) |               12 |             10
 London    | 2019-02-19 |                      12 |                       8 |                4 |         (null)
 London    | 2019-02-20 |                      13 |                       9 |                2 |             13
 London    | 2019-02-21 |                      15 |                       8 |                2 |              8
 London    | 2019-02-22 |                      16 |                       9 |                0 |              7
(14 rows)
```

## :closed_book: Reference

None
