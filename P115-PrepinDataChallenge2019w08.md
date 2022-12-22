## :memo: Overview

Preppin' Data challenge の「2019: Week 8」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/04/2019-week-8.html)
- [Answer](https://preppindata.blogspot.com/2019/04/2019-week-8-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。店舗での盗難がいつ発生し、いつ在庫を調整して減額を反映したかを記録しているテーブルがあるので、ここから盗難と在庫調整の関係を集計するというお題。ただ、タイミングによっては調整されていないものある。

```sql
select * from p2019w08t1;

   type   |     action     |    date    | quantity | store_id | crime_ref_number
----------+----------------+------------+----------+----------+------------------
 Bar      | Theft          | 19/03/2019 |       10 | OX1      | S04P1
 Bar      | Stock Adjusted | 25/03/2019 |      -10 | OX1      | S04P1
 Bar      | Theft          | 22/03/2019 |        5 | OX1      | S04P2
 Bar      | Stock Adjusted | 25/03/2019 |       -4 | OX1      | S04P2
 Liquid   | Theft          | 02/03/2019 |      100 | WM1      | S04P3
 Liquid   | Stock Adjusted | 02/03/2019 |     -100 | WM1      | S04P3
 Bar      | Theft          | 03/04/2019 |        5 | WM2      | S04P4
 Luquid   | Theft          | 22/03/2019 |       14 | OX1      | S04P5
 Liquid   | Theft          | 02/02/2019 |        2 | WM2      | S04P6
 Liquid   | Stock Adjusted | 03/04/2019 |       -2 | WM2      | S04P6
 Soap Bar | Theft          | 28/02/2019 |       22 | WM1      | S04P7
 Soap Bar | Stock Adjusted | 01/03/2019 |      -20 | WM1      | S04P7
 Liquid   | Theft          | 27/02/2019 |      105 | OX1      | S04P8
 Liquid   | Stock Adjusted | 01/03/2019 |     -105 | OX1      | S04P8
(14 rows)

select * from p2019w08t2;

      branch_id
---------------------
 OX1 - Oxford Street
 WM1 - Wimbledon 1
 WM2 - Wimbledon 2
 ST1 - Stratford
(4 rows)
```

店舗データのキーがいつものごとくおかしいので処理してから紐付ける。`stock_variance`は合計して盗難数と在庫調整数を計算し、`stolen_volume`は最大を取れば盗まれた数がわかる。盗難前と盗難後の在庫レベル間の不均衡を示す値を生成する。つまり、盗まれた分すべてを調整できるわではない。

```sql
with p2019w08t2tmp as (
select
      split_part(branch_id, ' - ', 1) as  branch_id,
      split_part(branch_id, ' - ', 2) as  branch_name
from
      p2019w08t2
), p2019w08t1tmp as (
select
      case type
      when 'Soap Bar' then 'Bar'
      when 'Luquid' then 'Liquid'
      else type end as type_mod,
      action,
      date,
      quantity,
      store_id,
      crime_ref_number
from
      p2019w08t1
), tmp2 as (
select
      t1.type_mod as type,
      t1.action,
      t1.date,
      t1.quantity as stock_variance,
      t1.quantity as stolen_volume,
      case when t1.action = 'Stock Adjusted' then to_date(t1.date, 'DD/MM/YYYY') else null end as stock_adjusted,
      case when t1.action = 'Theft' then to_date(t1.date, 'DD/MM/YYYY') else null end as theft,
      t1.crime_ref_number,
      t1.store_id,
      t2.branch_name
from
      p2019w08t1tmp as t1
left join
      p2019w08t2tmp as t2
on
      t1.store_id = t2.branch_id
)
select * from tmp2
;

  type  |     action     |    date    | stock_variance | stolen_volume | stock_adjusted |   theft    | crime_ref_number | store_id |  branch_name
--------+----------------+------------+----------------+---------------+----------------+------------+------------------+----------+---------------
 Bar    | Theft          | 19/03/2019 |             10 |            10 | (null)         | 2019-03-19 | S04P1            | OX1      | Oxford Street
 Bar    | Stock Adjusted | 25/03/2019 |            -10 |           -10 | 2019-03-25     | (null)     | S04P1            | OX1      | Oxford Street
 Bar    | Theft          | 22/03/2019 |              5 |             5 | (null)         | 2019-03-22 | S04P2            | OX1      | Oxford Street
 Bar    | Stock Adjusted | 25/03/2019 |             -4 |            -4 | 2019-03-25     | (null)     | S04P2            | OX1      | Oxford Street
 Liquid | Theft          | 02/03/2019 |            100 |           100 | (null)         | 2019-03-02 | S04P3            | WM1      | Wimbledon 1
 Liquid | Stock Adjusted | 02/03/2019 |           -100 |          -100 | 2019-03-02     | (null)     | S04P3            | WM1      | Wimbledon 1
 Bar    | Theft          | 03/04/2019 |              5 |             5 | (null)         | 2019-04-03 | S04P4            | WM2      | Wimbledon 2
 Liquid | Theft          | 22/03/2019 |             14 |            14 | (null)         | 2019-03-22 | S04P5            | OX1      | Oxford Street
 Liquid | Stock Adjusted | 03/04/2019 |             -2 |            -2 | 2019-04-03     | (null)     | S04P6            | WM2      | Wimbledon 2
 Liquid | Theft          | 02/02/2019 |              2 |             2 | (null)         | 2019-02-02 | S04P6            | WM2      | Wimbledon 2
 Bar    | Theft          | 28/02/2019 |             22 |            22 | (null)         | 2019-02-28 | S04P7            | WM1      | Wimbledon 1
 Bar    | Stock Adjusted | 01/03/2019 |            -20 |           -20 | 2019-03-01     | (null)     | S04P7            | WM1      | Wimbledon 1
 Liquid | Theft          | 27/02/2019 |            105 |           105 | (null)         | 2019-02-27 | S04P8            | OX1      | Oxford Street
 Liquid | Stock Adjusted | 01/03/2019 |           -105 |          -105 | 2019-03-01     | (null)     | S04P8            | OX1      | Oxford Street
(14 rows)
```

最後に、盗難日と在庫調整日の日数さを計算して終わり。

```sql
with p2019w08t2tmp as (
select
      split_part(branch_id, ' - ', 1) as  branch_id,
      split_part(branch_id, ' - ', 2) as  branch_name
from
      p2019w08t2
), p2019w08t1tmp as (
select
      case type
      when 'Soap Bar' then 'Bar'
      when 'Luquid' then 'Liquid'
      else type end as type_mod,
      action,
      date,
      quantity,
      store_id,
      crime_ref_number
from
      p2019w08t1
), tmp2 as (
select
      t1.type_mod as type,
      t1.action,
      t1.date,
      t1.quantity as stock_variance,
      t1.quantity as stolen_volume,
      case when t1.action = 'Stock Adjusted' then to_date(t1.date, 'DD/MM/YYYY') else null end as stock_adjusted,
      case when t1.action = 'Theft' then to_date(t1.date, 'DD/MM/YYYY') else null end as theft,
      t1.crime_ref_number,
      t1.store_id,
      t2.branch_name
from
      p2019w08t1tmp as t1
left join
      p2019w08t2tmp as t2
on
      t1.store_id = t2.branch_id
), tmp3 as (
select
      branch_name,
      crime_ref_number,
      type,
      sum(stock_variance) as sum_stock_variance,
      max(stock_adjusted) as max_stock_adjusted,
      max(stolen_volume) as max_stolen_volume,
      max(theft) as max_theft
from
      tmp2
group by
      branch_name,
      crime_ref_number,
      type
)
select
      branch_name,
      sum_stock_variance,
      max_stock_adjusted - max_theft as days_to_complete_adjustment,
      max_stock_adjusted,
      crime_ref_number,
      type,
      max_stolen_volume,
      max_theft
from
      tmp3
;

  branch_name  | sum_stock_variance | days_to_complete_adjustment | max_stock_adjusted | crime_ref_number |  type  | max_stolen_volume | max_theft
---------------+--------------------+-----------------------------+--------------------+------------------+--------+-------------------+------------
 Wimbledon 2   |                  0 |                          60 | 2019-04-03         | S04P6            | Liquid |                 2 | 2019-02-02
 Wimbledon 2   |                  5 |                      (null) | (null)             | S04P4            | Bar    |                 5 | 2019-04-03
 Wimbledon 1   |                  0 |                           0 | 2019-03-02         | S04P3            | Liquid |               100 | 2019-03-02
 Wimbledon 1   |                  2 |                           1 | 2019-03-01         | S04P7            | Bar    |                22 | 2019-02-28
 Oxford Street |                 14 |                      (null) | (null)             | S04P5            | Liquid |                14 | 2019-03-22
 Oxford Street |                  0 |                           2 | 2019-03-01         | S04P8            | Liquid |               105 | 2019-02-27
 Oxford Street |                  0 |                           6 | 2019-03-25         | S04P1            | Bar    |                10 | 2019-03-19
 Oxford Street |                  1 |                           3 | 2019-03-25         | S04P2            | Bar    |                 5 | 2019-03-22
(8 rows)
```

## :closed_book: Reference

None
