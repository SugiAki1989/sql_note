## :memo: Overview

アクション間隔分析を SQL で行う。アクション間隔分析は、ユーザーの何らかのアクションの実行間隔を分析するもの。例えば商品の購入間隔、登録から初回購入、試合日とチケット購入日の日数差、受注から配送までのリードタイムの間隔など、何らかのアクションの間隔を計算して、分析する際に利用する。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

RFM 分析の計算方法は下記の通り。

1. 特定のアクションを決定する
2. 各時点からのアクションの経過日数を会員ごとに計算する
3. 経過日数ごとに、会員数を集計する

ここでは、1 回目と 2 回目の購入間隔の分析を行うことにする。

```sql
with tmp as (
select
    customerid as cuid,
    invoicedate::date as dt
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid,
    invoicedate::date
)
select
    cuid,
    dt,
    lag(dt) over (partition by cuid order by dt asc) as lagdt,
    dt - lag(dt) over (partition by cuid order by dt asc) as diff_dt,
    row_number() over (partition by cuid order by dt asc) as seq
from
    tmp
;

 cuid  |     dt     |   lagdt    | diff_dt | seq
-------+------------+------------+---------+-----
 12346 | 2011-01-18 |            |         |   1
 12347 | 2010-12-07 |            |         |   1
 12347 | 2011-01-26 | 2010-12-07 |      50 |   2
 12347 | 2011-04-07 | 2011-01-26 |      71 |   3
 12347 | 2011-06-09 | 2011-04-07 |      63 |   4
 12347 | 2011-08-02 | 2011-06-09 |      54 |   5
 12347 | 2011-10-31 | 2011-08-02 |      90 |   6
 12347 | 2011-12-07 | 2011-10-31 |      37 |   7
 12348 | 2010-12-16 |            |         |   1
 12348 | 2011-01-25 | 2010-12-16 |      40 |   2
 12348 | 2011-04-05 | 2011-01-25 |      70 |   3
 12348 | 2011-09-25 | 2011-04-05 |     173 |   4
 12349 | 2011-11-21 |            |         |   1
 12350 | 2011-02-02 |            |         |   1
 12352 | 2011-02-16 |            |         |   1
 12352 | 2011-03-01 | 2011-02-16 |      13 |   2
 12352 | 2011-03-17 | 2011-03-01 |      16 |   3
 12352 | 2011-03-22 | 2011-03-17 |       5 |   4
 12352 | 2011-09-20 | 2011-03-22 |     182 |   5
 12352 | 2011-09-28 | 2011-09-20 |       8 |   6
 12352 | 2011-11-03 | 2011-09-28 |      36 |   7
 12353 | 2011-05-19 |            |         |   1
 12354 | 2011-04-21 |            |         |   1
 12355 | 2011-05-09 |            |         |   1
 12356 | 2011-01-18 |            |         |   1
 12356 | 2011-04-08 | 2011-01-18 |      80 |   2
 12356 | 2011-11-17 | 2011-04-08 |     223 |   3
```

このテーブルから`seq=2`を取って集計すれば、1~2 回目の日数差ごと会員数が計算できる。`seq=n`を変更すれば、2~3 回目などを自由に計算可能。

```sql
with tmp as (
select
    customerid as cuid,
    invoicedate::date as dt
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid,
    invoicedate::date
), tmp2 as (
select
    cuid,
    dt,
    lag(dt) over (partition by cuid order by dt asc) as lagdt,
    dt - lag(dt) over (partition by cuid order by dt asc) as diff_dt,
    row_number() over (partition by cuid order by dt asc) as seq
from
    tmp
)
select
    diff_dt,
    count(cuid) as cnt
from
    tmp2
where
    seq = 2
group by
    diff_dt
order by
    diff_dt asc
;

 diff_dt | cnt
---------+-----
       1 |  74
       2 |  65
       3 |  55
       4 |  75
       5 |  69
       6 |  87
       7 | 107
       8 |  81
       9 |  34
      10 |  49
      11 |  50
      12 |  25
      13 |  42
      14 |  52
      15 |  24
      16 |  21
      17 |  33
      18 |  17
      19 |  28
      20 |  23
      21 |  46
      22 |  17
      23 |  20
      24 |  12
      25 |  21
      26 |  20
      27 |  29
      28 |  45
      29 |  29
      30 |  23
      31 |  24
```

## :closed_book: Reference

None
