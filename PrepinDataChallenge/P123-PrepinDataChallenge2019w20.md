## :memo: Overview

Preppin' Data challenge の「2019: Week 17」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/06/2019-week-20.html)
- [Answer](https://preppindata.blogspot.com/2019/07/2019-week-20-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。入院に関するデータで、医療費を計算するというお題。

```sql
-- t1: Cost per Visit-表1.csv
-- t2: Frequency of Check-ups-表1.csv
-- t3: Patient-表1.csv
-- t4: Scaffold-表1.csv

select * from p2019w20t1 limit 10;
 length_of_stay | cost_per_day
----------------+--------------
 1-3            |          100
 4-7            |           80
 8-14           |           75
(3 rows)

select * from p2019w20t2 limit 10;
 check_up | months_after_leaving | length_of_stay
----------+----------------------+----------------
 First    |                    1 |              2
 Second   |                    3 |              1
 Third    |                   12 |              4

select * from p2019w20t3 limit 10;
  name   | first_visit | length_of_stay
---------+-------------+----------------
 Nathan  | 6/2/19      |              4
 Ruth    | 6/5/19      |              6
 Georgie | 6/5/19      |              1
 Michael | 6/7/19      |              2
 Ben     | 6/8/19      |              1
 Sarah   | 6/8/19      |              3
 Joe     | 6/11/19     |             12
 Jose    | 6/20/19     |              6
 Dennis  | 6/22/19     |             13

select * from p2019w20t4 limit 10;
 value
-------
     0
     1
     2
     3
     4
     5
     6
     7
     8
     9
```

まずは入院費用が日毎に変わるので、計算しやすいようにタリーテーブルを利用して、加工する。

```sql
with tmp as (
select
    name, to_date(first_visit, 'mm/dd/yy') as first_visit, length_of_stay,
    value
from
    p2019w20t3 as t3
left join
    p2019w20t4 as t4
on
    t3.length_of_stay > t4.value
)

select * from tmp;

  name   | first_visit | length_of_stay | value
---------+-------------+----------------+-------
 Nathan  | 2019-06-02  |              4 |     0
 Nathan  | 2019-06-02  |              4 |     1
 Nathan  | 2019-06-02  |              4 |     2
 Nathan  | 2019-06-02  |              4 |     3
 Ruth    | 2019-06-05  |              6 |     0
 Ruth    | 2019-06-05  |              6 |     1
 Ruth    | 2019-06-05  |              6 |     2
 Ruth    | 2019-06-05  |              6 |     3
 Ruth    | 2019-06-05  |              6 |     4
 Ruth    | 2019-06-05  |              6 |     5
 Georgie | 2019-06-05  |              1 |     0
 Michael | 2019-06-07  |              2 |     0
 Michael | 2019-06-07  |              2 |     1
 Ben     | 2019-06-08  |              1 |     0
 Sarah   | 2019-06-08  |              3 |     0
 Sarah   | 2019-06-08  |              3 |     1
 Sarah   | 2019-06-08  |              3 |     2
```

タリーテーブルの値を利用して入院期間を計算する。

```sql
with tmp as (
select
    name, to_date(first_visit, 'mm/dd/yy') as date_mod, length_of_stay,
    value
from
    p2019w20t3 as t3
left join
    p2019w20t4 as t4
on
    t3.length_of_stay > t4.value
), tmp2 as (
select
    name, date_mod, value,
    value + 1 as value_one,
    (date_mod + cast(value::text || 'days' as interval))::date as date_mod2
from
    tmp
)

select * from tmp2;

  name   |  date_mod  | value | value_one | date_mod2
---------+------------+-------+-----------+------------
 Nathan  | 2019-06-02 |     0 |         1 | 2019-06-02
 Nathan  | 2019-06-02 |     1 |         2 | 2019-06-03
 Nathan  | 2019-06-02 |     2 |         3 | 2019-06-04
 Nathan  | 2019-06-02 |     3 |         4 | 2019-06-05
 Ruth    | 2019-06-05 |     0 |         1 | 2019-06-05
 Ruth    | 2019-06-05 |     1 |         2 | 2019-06-06
 Ruth    | 2019-06-05 |     2 |         3 | 2019-06-07
 Ruth    | 2019-06-05 |     3 |         4 | 2019-06-08
 Ruth    | 2019-06-05 |     4 |         5 | 2019-06-09
 Ruth    | 2019-06-05 |     5 |         6 | 2019-06-10
 Georgie | 2019-06-05 |     0 |         1 | 2019-06-05
 Michael | 2019-06-07 |     0 |         1 | 2019-06-07
 Michael | 2019-06-07 |     1 |         2 | 2019-06-08
 Ben     | 2019-06-08 |     0 |         1 | 2019-06-08
 Sarah   | 2019-06-08 |     0 |         1 | 2019-06-08
 Sarah   | 2019-06-08 |     1 |         2 | 2019-06-09
 Sarah   | 2019-06-08 |     2 |         3 | 2019-06-10
```

このテーブルに対して、入院期間ごとに費用が変わる入院料金マスタを紐付けて、

```sql
with tmp as (
select
    name, to_date(first_visit, 'mm/dd/yy') as date_mod, length_of_stay,
    value
from
    p2019w20t3 as t3
left join
    p2019w20t4 as t4
on
    t3.length_of_stay > t4.value
), tmp2 as (
select
    name, date_mod, value,
    value + 1 as value_one,
    (date_mod + cast(value::text || 'days' as interval))::date as date_mod2
from
    tmp
), tmp3 as (
select
    split_part(length_of_stay, '-', 1)::int as min_len_stay,
    split_part(length_of_stay, '-', 2)::int as max_len_stay,
    cost_per_day
from
    p2019w20t1
), tmp4 as (
select
    name,
    date_mod2,
    cost_per_day
from
    tmp2 as t2
left join
    tmp3 as t3
on
    t2.value_one >= t3.min_len_stay and
    t2.value_one <= t3.max_len_stay
)
select * from tmp4;

  name   | date_mod2  | cost_per_day
---------+------------+--------------
 Nathan  | 2019-06-02 |          100
 Nathan  | 2019-06-03 |          100
 Nathan  | 2019-06-04 |          100
 Nathan  | 2019-06-05 |           80
 Ruth    | 2019-06-05 |          100
 Ruth    | 2019-06-06 |          100
 Ruth    | 2019-06-07 |          100
 Ruth    | 2019-06-08 |           80
 Ruth    | 2019-06-09 |           80
 Ruth    | 2019-06-10 |           80
 Georgie | 2019-06-05 |          100
 Michael | 2019-06-07 |          100
 Michael | 2019-06-08 |          100
 Ben     | 2019-06-08 |          100
 Sarah   | 2019-06-08 |          100
 Sarah   | 2019-06-09 |          100
 Sarah   | 2019-06-10 |          100
```

集計すればお終い。

```sql
with tmp as (
select
    name, to_date(first_visit, 'mm/dd/yy') as date_mod, length_of_stay,
    value
from
    p2019w20t3 as t3
left join
    p2019w20t4 as t4
on
    t3.length_of_stay > t4.value
), tmp2 as (
select
    name, date_mod, value,
    value + 1 as value_one,
    (date_mod + cast(value::text || 'days' as interval))::date as date_mod2
from
    tmp
), tmp3 as (
select
    split_part(length_of_stay, '-', 1)::int as min_len_stay,
    split_part(length_of_stay, '-', 2)::int as max_len_stay,
    cost_per_day
from
    p2019w20t1
), tmp4 as (
select
    name,
    date_mod2,
    cost_per_day
from
    tmp2 as t2
left join
    tmp3 as t3
on
    t2.value_one >= t3.min_len_stay and
    t2.value_one <= t3.max_len_stay
)
select
    name,
    round(avg(cost_per_day),2) as avg_cost_per_day_per_person,
    sum(cost_per_day) as cost
from tmp4
group by name;

  name   | avg_cost_per_day_per_person | cost
---------+-----------------------------+------
 Jose    |                       90.00 |  540
 Michael |                      100.00 |  200
 Ben     |                      100.00 |  100
 Nathan  |                       95.00 |  380
 Joe     |                       82.92 |  995
 Ruth    |                       90.00 |  540
 Dennis  |                       82.31 | 1070
 Georgie |                      100.00 |  100
 Sarah   |                      100.00 |  300
(9 rows)
```

## :closed_book: Reference

None
