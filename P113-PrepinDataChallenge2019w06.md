## :memo: Overview

Preppin' Data challenge の「2019: Week ６」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/03/2019-week-6.html)
- [Answer](https://preppindata.blogspot.com/2019/03/2019-week-6-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。日常品を販売する企業の売上実績を集計して、既存のフォーマットにインサートする、というお題。

```sql
select * from p2019w06t1;

 country |  category   |    city    | units_sold |    table_names     |          file_paths
---------+-------------+------------+------------+--------------------+------------------------------
 England | Bar Soap    | London     |        475 | England - Mar 2019 | Copy of PD - Week 6 (1).xlsx
 England | Liquid Soap | London     |        364 | England - Mar 2019 | Copy of PD - Week 6 (1).xlsx
 England | Bar Soap    | Manchester |        195 | England - Mar 2019 | Copy of PD - Week 6 (1).xlsx
 England | Liquid Soap | Manchester |        223 | England - Mar 2019 | Copy of PD - Week 6 (1).xlsx
 England | Bar Soap    | York       |        275 | England - Mar 2019 | Copy of PD - Week 6 (1).xlsx
 England | Liquid Soap | York       |        327 | England - Mar 2019 | Copy of PD - Week 6 (1).xlsx
(6 rows)

select * from p2019w06t2;

 type_of_soap | manufacturing_cost_per_unit | selling_price_per_unit
--------------+-----------------------------+------------------------
 Bar          |                         0.8 |                      3
 Liquid       |                         0.2 |                    1.2
(2 rows)

select * from p2019w06t3;

 profit |   month    | country  |  category
--------+------------+----------+-------------
   1475 | 2019/01/19 | Scotland | Bar Soap
   1350 | 2019/02/19 | Scotland | Bar Soap
   1550 | 2019/03/19 | Scotland | Bar Soap
    755 | 2019/01/19 | Scotland | Liquid Soap
    695 | 2019/02/19 | Scotland | Liquid soap
    645 | 2019/03/19 | Scotland | Liquid Soap
   1955 | 2019/01/19 | England  | Bar Soap
   2145 | 2019/02/19 | England  | Bar Soap
   1630 | 2019/01/19 | England  | Liquid Soap
   1755 | 2019/02/19 | England  | Liquid Soap
(10 rows)
```

解答はこちら。

```sql
with tmp as (
select
   t1.country,
   t1.category,
   to_date('19 ' || split_part(t1.table_names, '-', 2), 'DD Mon YYYY') as month_mod,
   floor(t1.units_sold * (t2.selling_price_per_unit - t2.manufacturing_cost_per_unit)) as profit
from
   p2019w06t1 as t1
left join
   p2019w06t2 as t2
on
   t1.category = (t2.type_of_soap || ' Soap')
), tmp2 as (
(select
   country,
   sum(profit)::int as profit,
   category,
   month_mod as month
from
   tmp
group by
   country,
   category,
   month_mod)
union all
(select
   country,
   profit,
   category,
   month::date
from
   p2019w06t3)
)
select
   country,
   month,
   category,
   profit
from
   tmp2
order by
   country asc,
   month asc,
   category asc
;

 country  |   month    |  category   | profit
----------+------------+-------------+--------
 England  | 2019-01-19 | Bar Soap    |   1955
 England  | 2019-01-19 | Liquid Soap |   1630
 England  | 2019-02-19 | Bar Soap    |   2145
 England  | 2019-02-19 | Liquid Soap |   1755
 England  | 2019-03-19 | Bar Soap    |   2079
 England  | 2019-03-19 | Liquid Soap |    914
 Scotland | 2019-01-19 | Bar Soap    |   1475
 Scotland | 2019-01-19 | Liquid Soap |    755
 Scotland | 2019-02-19 | Bar Soap    |   1350
 Scotland | 2019-02-19 | Liquid soap |    695
 Scotland | 2019-03-19 | Bar Soap    |   1550
 Scotland | 2019-03-19 | Liquid Soap |    645
(12 rows)
```

まずは日付を処理しておく。設計上、19 日としていいみたい。

```sql
select
   to_date('19 ' || split_part(table_names, '-', 2), 'DD Mon YYYY') as mod_date
from
   p2019w06t1
;

    sp
------------
 2019-03-19
 2019-03-19
 2019-03-19
 2019-03-19
 2019-03-19
 2019-03-19
(6 rows)
```

あとはテーブルを紐付けて、利益を計算する。その後にグループ化集計、ユニオンでおわり。

```sql
select
   t1.country,
   t1.category,
   to_date('19 ' || split_part(t1.table_names, '-', 2), 'DD Mon YYYY') as month_mod,
   floor(t1.units_sold * (t2.selling_price_per_unit - t2.manufacturing_cost_per_unit)) as profit
from
   p2019w06t1 as t1
left join
   p2019w06t2 as t2
on
   t1.category = (t2.type_of_soap || ' Soap')
;

 country |  category   | month_mod  | profit
---------+-------------+------------+--------
 England | Bar Soap    | 2019-03-19 |   1045
 England | Bar Soap    | 2019-03-19 |    429
 England | Bar Soap    | 2019-03-19 |    605
 England | Liquid Soap | 2019-03-19 |    364
 England | Liquid Soap | 2019-03-19 |    223
 England | Liquid Soap | 2019-03-19 |    327
(6 rows)
```

## :closed_book: Reference

None
