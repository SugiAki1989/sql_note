## :memo: Overview

Preppin' Data challenge の「2019: Week 21」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/07/2019-week-21.html)
- [Answer](https://preppindata.blogspot.com/2019/07/2019-week-21-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。前回のデータに対して、追加の患者が発生し、年単位で入院期間を持つ患者がいるため、その処理が必要になる。

```sql
select * from p2019w21t1 limit 10;

 patient_name | first_visit | length_of_stay
--------------+-------------+----------------
 James        | 2/7/2019    |              7
 Jenny        | 3/7/2019    |              6
 Bonner       | 5/7/2019    |              5
 Sara         | 7/7/2019    |              3
 Natalia      | 7/7/2019    |              6
 Kamilla      | 8/7/2019    |              5
 Hesham       | 11/7/2019   |              9
 Andy         | 20/7/2019   |             14
 Hafeez       | 22/7/2019   |              2
 Tom          | 23/7/2019   |              8
(10 rows)


select * from p2019w20t2 limit 10;
 check_up | months_after_leaving | length_of_stay
----------+----------------------+----------------
 First    |                    1 |              2
 Second   |                    3 |              1
 Third    |                   12 |              4
```

フォーマットが揃ってないので、きれいにしてユニオンする。

```sql
with tmp as (
select name, to_date(first_visit, 'mm/dd/yy') as date_mod, length_of_stay from p2019w20t3
union all
select patient_name as name, to_date(first_visit, 'dd/mm/yyyy') as date_mod, length_of_stay from p2019w21t1
), tmp2 as (
select
    t1.name,
    t1.date_mod as date,
    t2.check_up,
    t2.months_after_leaving,
    t2.length_of_stay
from
    tmp as t1
cross join
    p2019w20t2 as t2
)
select * from tmp2;

   name   |    date    | check_up | months_after_leaving | length_of_stay
----------+------------+----------+----------------------+----------------
 Nathan   | 2019-06-02 | First    |                    1 |              2
 Nathan   | 2019-06-02 | Second   |                    3 |              1
 Nathan   | 2019-06-02 | Third    |                   12 |              4
 Ruth     | 2019-06-05 | First    |                    1 |              2
 Ruth     | 2019-06-05 | Second   |                    3 |              1
 Ruth     | 2019-06-05 | Third    |                   12 |              4
```

月単位で加算する分、日単位で加算する分を調整して、過去分を作成。すべての患者は検査を受ける必要があるため、これらの検査日と期間を患者データのリストに追加する必要がある。ここで重要なのは、検査頻度は、患者が病院を訪れた最初の日からではなく、患者が最初の初経を終えた日から測定されることに注意すること.たとえば、Jose が 2019 年 6 月 20 日に病院を訪れ、6 日間滞在した場合、彼は 2019 年 6 月 27 日に退院し、最初の検査は 1 か月後の 2019 年 7 月 27 日に行われる、とのこと。

```sql
with tmp as (
select name, to_date(first_visit, 'mm/dd/yy') as date_mod, length_of_stay from p2019w20t3
union all
select patient_name as name, to_date(first_visit, 'dd/mm/yyyy') as date_mod, length_of_stay from p2019w21t1
), tmp2 as (
select
    t1.name,
    t1.date_mod as date,
    t2.check_up,
    t2.months_after_leaving,
    t2.length_of_stay,
    (((date_mod + cast(t2.months_after_leaving::text || 'months' as interval)))::date
    + cast(t2.length_of_stay::text || 'days' as interval))::date as date2
from
    tmp as t1
cross join
    p2019w20t2 as t2
)
select * from tmp2;

   name   |    date    | check_up | months_after_leaving | length_of_stay |   date2
----------+------------+----------+----------------------+----------------+------------
 Nathan   | 2019-06-02 | First    |                    1 |              2 | 2019-07-04
 Nathan   | 2019-06-02 | Second   |                    3 |              1 | 2019-09-03
 Nathan   | 2019-06-02 | Third    |                   12 |              4 | 2020-06-06
 Ruth     | 2019-06-05 | First    |                    1 |              2 | 2019-07-07
 Ruth     | 2019-06-05 | Second   |                    3 |              1 | 2019-09-06
 Ruth     | 2019-06-05 | Third    |                   12 |              4 | 2020-06-09
 Georgie  | 2019-06-05 | First    |                    1 |              2 | 2019-07-07
 Georgie  | 2019-06-05 | Second   |                    3 |              1 | 2019-09-06
 Georgie  | 2019-06-05 | Third    |                   12 |              4 | 2020-06-09
 Michael  | 2019-06-07 | First    |                    1 |              2 | 2019-07-09
 Michael  | 2019-06-07 | Second   |                    3 |              1 | 2019-09-08
 Michael  | 2019-06-07 | Third    |                   12 |              4 | 2020-06-11

```

これが過去分を追加したテーブルなので、前回と同じ処理をすればおしまい。

```sql
with tmp as (
select name, to_date(first_visit, 'mm/dd/yy') as date_mod, length_of_stay from p2019w20t3
union all
select patient_name as name, to_date(first_visit, 'dd/mm/yyyy') as date_mod, length_of_stay from p2019w21t1
), tmp2 as (
select
    t1.name,
    t1.date_mod as date,
    t2.check_up,
    t2.months_after_leaving,
    t2.length_of_stay,
    (((date_mod + cast(t2.months_after_leaving::text || 'months' as interval)))::date
    + cast(t2.length_of_stay::text || 'days' as interval))::date as date2
from
    tmp as t1
cross join
    p2019w20t2 as t2
), tmp3 as (
select name, date_mod as date, length_of_stay from tmp
union all
select name, date2 as date, length_of_stay from tmp2
)
select * from tmp3 order by name, date asc;

   name   |    date    | length_of_stay
----------+------------+----------------
 Andy     | 2019-07-20 |             14
 Andy     | 2019-08-22 |              2
 Andy     | 2019-10-21 |              1
 Andy     | 2020-07-24 |              4
 Ben      | 2019-06-08 |              1
 Ben      | 2019-07-10 |              2
 Ben      | 2019-09-09 |              1
 Ben      | 2020-06-12 |              4
 Bonner   | 2019-07-05 |              5
 Bonner   | 2019-08-07 |              2
 Bonner   | 2019-10-06 |              1
 Bonner   | 2020-07-09 |              4
 Camiila  | 2019-07-25 |              9
 Camiila  | 2019-08-27 |              2
 Camiila  | 2019-10-26 |              1
 Camiila  | 2020-07-29 |              4
 Dennis   | 2019-06-22 |             13
 Dennis   | 2019-07-24 |              2
 Dennis   | 2019-09-23 |              1
 Dennis   | 2020-06-26 |              4
 Georgie  | 2019-06-05 |              1
 Georgie  | 2019-07-07 |              2
 Georgie  | 2019-09-06 |              1
 Georgie  | 2020-06-09 |              4
 Hafeez   | 2019-07-22 |              2
 Hafeez   | 2019-08-24 |              2
 Hafeez   | 2019-10-23 |              1
 Hafeez   | 2020-07-26 |              4
 Hesham   | 2019-07-11 |              9
 Hesham   | 2019-08-13 |              2
 Hesham   | 2019-10-12 |              1
 Hesham   | 2020-07-15 |              4
 James    | 2019-07-02 |              7
 James    | 2019-08-04 |              2
 James    | 2019-10-03 |              1
 James    | 2020-07-06 |              4
 Jenny    | 2019-07-03 |              6
 Jenny    | 2019-08-05 |              2
 Jenny    | 2019-10-04 |              1
 Jenny    | 2020-07-07 |              4
 Joe      | 2019-06-11 |             12
 Joe      | 2019-07-13 |              2
 Joe      | 2019-09-12 |              1
 Joe      | 2020-06-15 |              4
 Jonathan | 2019-07-26 |              2
 Jonathan | 2019-08-28 |              2
 Jonathan | 2019-10-27 |              1
 Jonathan | 2020-07-30 |              4
 Jose     | 2019-06-20 |              6
 Jose     | 2019-07-22 |              2
 Jose     | 2019-09-21 |              1
 Jose     | 2020-06-24 |              4
 Kamilla  | 2019-07-08 |              5
 Kamilla  | 2019-08-10 |              2
 Kamilla  | 2019-10-09 |              1
 Kamilla  | 2020-07-12 |              4
 Michael  | 2019-06-07 |              2
 Michael  | 2019-07-09 |              2
 Michael  | 2019-09-08 |              1
 Michael  | 2020-06-11 |              4
 Natalia  | 2019-07-07 |              6
 Natalia  | 2019-08-09 |              2
 Natalia  | 2019-10-08 |              1
 Natalia  | 2020-07-11 |              4
 Nathan   | 2019-06-02 |              4
 Nathan   | 2019-07-04 |              2
 Nathan   | 2019-09-03 |              1
 Nathan   | 2020-06-06 |              4
 Ruth     | 2019-06-05 |              6
 Ruth     | 2019-07-07 |              2
 Ruth     | 2019-09-06 |              1
 Ruth     | 2020-06-09 |              4
 Sara     | 2019-07-07 |              3
 Sara     | 2019-08-09 |              2
 Sara     | 2019-10-08 |              1
 Sara     | 2020-07-11 |              4
 Sarah    | 2019-06-08 |              3
 Sarah    | 2019-07-10 |              2
 Sarah    | 2019-09-09 |              1
 Sarah    | 2020-06-12 |              4
 Tom      | 2019-07-23 |              8
 Tom      | 2019-08-25 |              2
 Tom      | 2019-10-24 |              1
 Tom      | 2020-07-27 |              4
(84 rows)

```

## :closed_book: Reference

None
