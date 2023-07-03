## :memo: Overview

FunChart 分析を SQL で行う。FunChart 分析は、ある基準となる時点を 100％とし、それ以降の数値を基準となる時点に対するパーセントで示すことで、特定時点からの成長率を分析する手法。グラフが扇を広げたような形をしているためファンチャートと呼ばれるらしい。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

FunChart 分析の計算方法は下記の通り。

1. 特定の基準時点を決める
2. 特定の基準時点ごとの値に対し、各時点の値の割合を計算

```sql
with tmp as (
select
    stockcode,
    date_trunc('month', InvoiceDate) as ym,
    sum(unitprice) as price_sum
from
    onlinestore
where
    -- 売上TOP5の商品に限定するサブクエリ
    stockcode in (
        select
            stockcode
        from (
            select
                stockcode,
                sum(unitprice)
            from
                onlinestore
            group by
                stockcode
            order by
                sum(unitprice) desc
            limit
                5
            ) as s1
    )
group by
    stockcode,
    date_trunc('month', InvoiceDate)
order by
    stockcode,
    date_trunc('month', InvoiceDate)
)
select
    stockcode,
    ym,
    price_sum,
    first_value(price_sum) over (partition by stockcode order by ym asc) as specific_point,
    round(100 * (price_sum / first_value(price_sum) over (partition by stockcode order by ym asc))::numeric, 2) as ratio
from
    tmp
where
    ym between '2010-12-01' and '2011-03-01'
;

 stockcode |         ym          | price_sum | specific_point | ratio
-----------+---------------------+-----------+----------------+--------
 22423     | 2010-12-01 00:00:00 | 2676.0398 |      2676.0398 | 100.00
 22423     | 2011-01-01 00:00:00 | 2111.4001 |      2676.0398 |  78.90
 22423     | 2011-02-01 00:00:00 | 2061.2104 |      2676.0398 |  77.02
 22423     | 2011-03-01 00:00:00 | 3079.3804 |      2676.0398 | 115.07
 AMAZONFEE | 2010-12-01 00:00:00 | 66325.734 |      66325.734 | 100.00
 AMAZONFEE | 2011-01-01 00:00:00 |  33341.73 |      66325.734 |  50.27
 AMAZONFEE | 2011-02-01 00:00:00 |  10834.05 |      66325.734 |  16.33
 AMAZONFEE | 2011-03-01 00:00:00 |   11357.6 |      66325.734 |  17.12
 DOT       | 2010-12-01 00:00:00 |  24671.19 |       24671.19 | 100.00
 DOT       | 2011-01-01 00:00:00 | 13925.109 |       24671.19 |  56.44
 DOT       | 2011-02-01 00:00:00 |  10060.57 |       24671.19 |  40.78
 DOT       | 2011-03-01 00:00:00 | 11829.711 |       24671.19 |  47.95
 M         | 2010-12-01 00:00:00 |   4897.11 |        4897.11 | 100.00
 M         | 2011-01-01 00:00:00 |   5240.14 |        4897.11 | 107.01
 M         | 2011-02-01 00:00:00 | 5268.0796 |        4897.11 | 107.58
 M         | 2011-03-01 00:00:00 | 18800.389 |        4897.11 | 383.91
 POST      | 2010-12-01 00:00:00 |      1341 |           1341 | 100.00
 POST      | 2011-01-01 00:00:00 |   1661.98 |           1341 | 123.94
 POST      | 2011-02-01 00:00:00 |   1387.06 |           1341 | 103.44
 POST      | 2011-03-01 00:00:00 | 2098.1301 |           1341 | 156.46
(20 rows)
```

サブクエリ使ってみたが、`with`で全部まとめて、DAG とかで情報を残すほうが個人的には解釈しやすい。

## :closed_book: Reference

None
