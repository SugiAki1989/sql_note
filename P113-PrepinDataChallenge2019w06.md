## :memo: Overview

Preppin' Data challenge の「2019: Week ６」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/03/2019-week-6.html)
- [Answer](https://preppindata.blogspot.com/2019/03/2019-week-6-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。

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

```sql

```

## :closed_book: Reference

None
