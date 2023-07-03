## :memo: Overview

デシル 分析を SQL で行う。デシル 分析は、ユーザーを売上を高い順に並べて 10 等分し、ユーザーの売上高構成比を把握する分析手法のこと。対売上高貢献度の高い優良顧客層を知ることができる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`, `ntile`

## :pencil2: Example

デシル 分析の計算方法は下記の通り。

1. 会員の売上金額を合計し、降順で並び替える
2. 上位から 10％づつデシルフラグを付与する
3. 各デシルごとに売上金額を合計する
4. 売上構成比や累計構成比率を計算する

まずは会員の売上合計を計算し、デシルを付与する。10 等分デシルは`ntile`で付与できる。

```sql
with tmp as (
select
    customerid,
    sum(unitprice) as price_sum
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid
order by
    sum(unitprice) desc
)
select
    customerid,
    price_sum,
    ntile(10) over (order by price_sum desc) as decile
from
    tmp
;

 customerid | price_sum  | decile
------------+------------+--------
 14096      |   41376.21 |      1
 15098      |    40278.9 |      1
 14911      |  31060.812 |      1
 12744      |   25108.89 |      1
 16029      |   24111.14 |      1
 17841      |   20333.25 |      1
 12748      |  15115.576 |      1
 ~~~~略
```

ここからデシルごとに再合計する。

```sql
with tmp as (
select
    customerid,
    sum(unitprice) as price_sum
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid
order by
    sum(unitprice) desc
), tmp2 as (
select
    customerid,
    price_sum,
    ntile(10) over (order by price_sum desc) as decile
from
    tmp
)
select
    decile,
    sum(price_sum) as price_sum_per_decile
from
    tmp2
group by
    decile
order by
    decile asc
;

 decile | price_sum_per_decile
--------+----------------------
      1 |            772375.44
      2 |            217056.92
      3 |            133563.95
      4 |             93589.63
      5 |             67370.37
      6 |             48424.58
      7 |            34500.055
      8 |            22922.545
      9 |            13398.059
     10 |            4618.7134
(10 rows)
```

あとは再合計した売上を利用して、構成比や累計構成比を計算すればデシル分析のための準備は完了。

```sql
with tmp as (
select
    customerid,
    sum(unitprice) as price_sum
from
    onlinestore
where
    customerid != 'NA' and
    invoicedate between '2010-12-01' and '2010-12-31'
group by
    customerid
order by
    sum(unitprice) desc
), tmp2 as (
select
    customerid,
    price_sum,
    ntile(10) over (order by price_sum desc) as decile
from
    tmp
)
select
    decile,
    count(distinct customerid) as cntid,
    sum(price_sum) as sum_price,
    round(
        avg(price_sum)::numeric
        , 2) as avg_price,
    sum(sum(price_sum)) over (order by decile asc rows between unbounded preceding and current row) as cum_price,
    round(
        (100 * sum(sum(price_sum)) over (order by decile asc rows between unbounded preceding and current row) / sum(sum(price_sum)) over ())::numeric
        , 2
    ) as cum_price_ratio
from
    tmp2
group by
    decile
order by
    decile asc
;

 decile | cntid | sum_price | avg_price | cum_price | cum_price_ratio
--------+-------+-----------+-----------+-----------+-----------------
      1 |    95 |  33945.11 |    357.32 |  33945.11 |           39.44
      2 |    95 | 15025.971 |    158.17 |  48971.08 |           56.91
      3 |    95 | 10624.131 |    111.83 |  59595.21 |           69.25
      4 |    95 |  8079.109 |     85.04 |  67674.32 |           78.64
      5 |    95 |  6304.261 |     66.36 |  73978.58 |           85.96
      6 |    95 | 4632.4707 |     48.76 |  78611.05 |           91.35
      7 |    95 | 3484.0405 |     36.67 | 82095.086 |           95.40
      8 |    95 | 2310.0496 |     24.32 |  84405.13 |           98.08
      9 |    94 | 1265.0096 |     13.46 |  85670.14 |           99.55
     10 |    94 | 387.03995 |      4.12 |  86057.18 |          100.00
(10 rows)
```

## :closed_book: Reference

None
