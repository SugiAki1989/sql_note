## :memo: Overview

Preppin' Data challenge の「2019: Week 22」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/07/2019-week-22.html)
- [Answer](https://preppindata.blogspot.com/2019/07/2019-week-22-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。移動平均を手計算するというお題。

```sql
select * from p2019w22t1 limit 10;
    date    | sales
------------+--------
 2019-01-01 |  20.25
 2019-01-02 |  50.69
 2019-01-03 |     21
 2019-01-04 |  27.15
 2019-01-05 | 174.49
 2019-01-06 | 424.03
 2019-01-07 | 401.03
 2019-01-08 | 263.59
 2019-01-09 |  68.63
 2019-01-10 | 110.34
(10 rows)
```

ウインドウ関数を利用すれば簡単に計算できるが、不等号 JOIN を利用して、データをうまく変形すれば移動平均は計算できる。ここでは 7 日移動平均を計算する。

```sql
with past as (
select date, date - interval '6 day' as pastdate, sales from p2019w22t1
), tmp2 as (
select
    t1.date as t1date,
    t1.sales as t1sales,
    t2.date as t2date,
    t2.pastdate,
    t2.sales as t2sales
from
    p2019w22t1 as t1
inner join
    past as t2
on
    t2.date >= t1.date and
    t2.pastdate <= t1.date
order by
    t2date asc
)
select * from tmp2 limit 10;

   t1date   | t1sales |   t2date   |      pastdate       | t2sales
------------+---------+------------+---------------------+---------
 2019-01-01 |   20.25 | 2019-01-01 | 2018-12-26 00:00:00 |   20.25
 2019-01-01 |   20.25 | 2019-01-02 | 2018-12-27 00:00:00 |   50.69
 2019-01-02 |   50.69 | 2019-01-02 | 2018-12-27 00:00:00 |   50.69
 2019-01-03 |      21 | 2019-01-03 | 2018-12-28 00:00:00 |      21
 2019-01-01 |   20.25 | 2019-01-03 | 2018-12-28 00:00:00 |      21
 2019-01-02 |   50.69 | 2019-01-03 | 2018-12-28 00:00:00 |      21
 2019-01-03 |      21 | 2019-01-04 | 2018-12-29 00:00:00 |   27.15
 2019-01-02 |   50.69 | 2019-01-04 | 2018-12-29 00:00:00 |   27.15
 2019-01-01 |   20.25 | 2019-01-04 | 2018-12-29 00:00:00 |   27.15
 2019-01-04 |   27.15 | 2019-01-04 | 2018-12-29 00:00:00 |   27.15
(10 rows)
```

あとはグループ化するカラムと、集計するカラムを気をつけながら、集計すればおしまい。

```sql
with past as (
select date, date - interval '6 day' as pastdate, sales from p2019w22t1
), tmp2 as (
select
    t1.date as t1date,
    t1.sales as t1sales,
    t2.date as t2date,
    t2.pastdate,
    t2.sales as t2sales
from
    p2019w22t1 as t1
inner join
    past as t2
on
    t2.date >= t1.date and
    t2.pastdate <= t1.date
order by
    t2date asc
)
select
    t2date,
    case when count(1) < 7 then null else avg(t1sales) end as moving7avg
from
    tmp2
group by
    t2date
limit 10;

   t2date   |     moving7avg
------------+--------------------
 2019-01-01 |             (null)
 2019-01-02 |             (null)
 2019-01-03 |             (null)
 2019-01-04 |             (null)
 2019-01-05 |             (null)
 2019-01-06 |             (null)
 2019-01-07 | 159.80571428571426
 2019-01-08 | 194.56857142857143
 2019-01-09 | 197.13142857142853
 2019-01-10 |  209.8942857142857
(10 rows)
```

## :closed_book: Reference

None
