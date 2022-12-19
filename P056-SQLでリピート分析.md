## :memo: Overview

リピート分析を SQL で行う。リピート分析は、ユーザーが何らかのアクションをリピートしたかを分析するもの。例えば EC では商品を初回購入したあとにリピートした会員は何人いるのかを分析する際に利用する。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

RFM 分析の計算方法は下記の通り。

1. 特定のアクションを決定する
2. 会員ごとの購入順序を計算する
3. 購入順序ごとに、会員数を集計する

ここでは、商品の購入に関するリピート率を計算する。まずは購入順序を計算する。

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
    row_number() over (partition by cuid order by dt asc) as seq
from
    tmp
;
 cuid  |     dt     | seq
-------+------------+-----
 12346 | 2011-01-18 |   1
 12347 | 2010-12-07 |   1
 12347 | 2011-01-26 |   2
 12347 | 2011-04-07 |   3
 12347 | 2011-06-09 |   4
 12347 | 2011-08-02 |   5
 12347 | 2011-10-31 |   6
 12347 | 2011-12-07 |   7
 12348 | 2010-12-16 |   1
 12348 | 2011-01-25 |   2
(10 rows)
```

このテーブルから`seq`ごとに会員数を計算する。

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
    row_number() over (partition by cuid order by dt asc) as seq
from
    tmp
)
select
    seq,
    count(cuid)
from
    tmp2
group by
    seq
order by
    seq asc
;

 seq | count
-----+-------
   1 |  4372
   2 |  2991
   3 |  2133
   4 |  1634
   5 |  1246
   6 |   969
   7 |   772
   8 |   623
   9 |   507
  10 |   437
【略】
```

1 回のみ購入が`n`人、2 回購入が`n`人と計算できているので、1→2 回にリピートした人数、2→3 回にリピートした人数などを計算してリピート率を計算する。また、合わせて残存率も計算しておく。リピート率の計算で値が 0 しか返ってこない場合は、データ型の問題なので、小数点が計算できるようにキャスする必要がある。

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
    row_number() over (partition by cuid order by dt asc) as seq
from
    tmp
), tmp3 as (
select
    seq,
    count(cuid)::numeric as cnt
from
    tmp2
group by
    seq
order by
    seq asc
)
select
    seq,
    cnt,
    lag(cnt) over (order by seq) as lagcnt,
    first_value(cnt) over (order by seq asc) as first,
    round(
    100*(cnt / lag(cnt) over (order by seq asc))
    , 2) as repeat_ratio,
    round(
    100*(cnt / first_value(cnt) over (order by seq asc))
    , 2) as zanzon_ration
from
    tmp3
;

 seq | cnt  | lagcnt | first | repeat_ratio | zanzon_ration
-----+------+--------+-------+--------------+---------------
   1 | 4372 |        |  4372 |              |        100.00
   2 | 2991 |   4372 |  4372 |        68.41 |         68.41
   3 | 2133 |   2991 |  4372 |        71.31 |         48.79
   4 | 1634 |   2133 |  4372 |        76.61 |         37.37
   5 | 1246 |   1634 |  4372 |        76.25 |         28.50
   6 |  969 |   1246 |  4372 |        77.77 |         22.16
   7 |  772 |    969 |  4372 |        79.67 |         17.66
   8 |  623 |    772 |  4372 |        80.70 |         14.25
   9 |  507 |    623 |  4372 |        81.38 |         11.60
  10 |  437 |    507 |  4372 |        86.19 |         10.00
  11 |  366 |    437 |  4372 |        83.75 |          8.37
  12 |  312 |    366 |  4372 |        85.25 |          7.14
  13 |  272 |    312 |  4372 |        87.18 |          6.22
  14 |  237 |    272 |  4372 |        87.13 |          5.42
  15 |  212 |    237 |  4372 |        89.45 |          4.85
  16 |  189 |    212 |  4372 |        89.15 |          4.32
  17 |  167 |    189 |  4372 |        88.36 |          3.82
  18 |  144 |    167 |  4372 |        86.23 |          3.29
  19 |  122 |    144 |  4372 |        84.72 |          2.79
  20 |  113 |    122 |  4372 |        92.62 |          2.58
  21 |  102 |    113 |  4372 |        90.27 |          2.33
  22 |   86 |    102 |  4372 |        84.31 |          1.97
  23 |   77 |     86 |  4372 |        89.53 |          1.76
  24 |   72 |     77 |  4372 |        93.51 |          1.65
  25 |   62 |     72 |  4372 |        86.11 |          1.42
  26 |   60 |     62 |  4372 |        96.77 |          1.37
  27 |   50 |     60 |  4372 |        83.33 |          1.14
  28 |   46 |     50 |  4372 |        92.00 |          1.05
  29 |   44 |     46 |  4372 |        95.65 |          1.01
  30 |   40 |     44 |  4372 |        90.91 |          0.91
(30 rows)
```

## :closed_book: Reference

None
