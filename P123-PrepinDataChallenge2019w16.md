## :memo: Overview

Preppin' Data challenge の「2019: Week 16」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/05/2019-week-16.html)
- [Answer](https://preppindata.blogspot.com/2019/06/2019-week-16-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。

```sql
-- t1: Sales_BarSoap.csv
-- t2: Sales_BudgetSoap.csv
-- t3: Sales_LiquidSoap.csv
-- t4: Sales_PlasmaSoap.csv
-- t5: Sales_SoapAccessories.csv

select * from p2019w16t1 limit 10;
           email           | order_total | order_date
---------------------------+-------------+------------
 tgeerebe@friendfeed.com   |         2.8 | 2018-09-01
 dbarthel84@youku.com      |        88.6 | 2018-10-01
 zwyard2n@cbsnews.com      |        44.6 | 2018-10-04
 dshewan3y@guardian.co.uk  |        99.3 | 2018-06-25
 mcarcas40@studiopress.com |        26.3 | 2019-02-18
 ienburygj@yale.edu        |         7.3 | 2018-08-24
 abenedetti8t@hud.gov      |        66.9 | 2019-01-02
 troylancefu@imgur.com     |        77.5 | 2018-07-03
 fohmsk1@cdbaby.com        |        25.1 | 2018-08-02
 tfireman5k@harvard.edu    |        46.7 | 2018-08-13
(10 rows)
```

```sql
with tmp as (
select * from p2019w16t1
union all
select * from p2019w16t2
union all
select * from p2019w16t3
union all
select * from p2019w16t4
union all
select * from p2019w16t5
), tmp2 as (
select

from
    tmp
)
select *
from tmp2
```

## :closed_book: Reference

None
