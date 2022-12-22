## :memo: Overview

Preppin' Data challenge の「2019: Week 12」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/05/2019-week-12.html)
- [Answer](https://preppindata.blogspot.com/2019/05/2019-week-12-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。エラーログのデータで

```sql
select * from p2019w12t1;
   start_date_time   |    end_date_time    | system |         error
---------------------+---------------------+--------+-----------------------
 13/04/2019 07:55:23 | 13/04/2019 08:23:12 | Sales  | Disc full
 13/04/2018 09:03:22 | 15/04/2018 09:03:21 | Stock  | Planned Outage
 13/05/2018 09:03:22 | 16/05/2018 09:03:21 | Stock  | Planned Outage
 13/06/2018 09:03:22 | 15/06/2018 09:03:21 | Stock  | Planned Outage
 12/07/2018 23:03:22 | 13/07/2018 07:00:03 | Stock  | Planned Outage
 12/08/2018 23:03:22 | 13/08/2018 07:00:03 | Stock  | Planned Outage
 19/08/2018 07:55:20 | 19/08/2018 08:31:55 | Sales  | Unknown disc location
 12/09/2018 23:03:22 | 13/09/2018 07:31:02 | Stock  | Planned Outage
 19/09/2018 14:57:00 | 19/09/2018 15:02:20 | Sales  | Unknown disc location
(9 rows)

select * from p2019w12t2;
 start_date | start_time |  end_date  | end_time | system |         error
------------+------------+------------+----------+--------+------------------------
 13/04/2018 | 09:00:00   | 15/04/2018 | 09:00:00 | Stock  | Planed Outage
 13/05/2018 | 09:00:00   | 15/05/2018 | 09:00:00 | Stock  | Planed Outage
 13/06/2018 | 09:00:00   | 15/06/2018 | 09:00:00 | Stock  | Planed Outage
 13/07/2018 | 09:00:00   | 15/07/2018 | 09:00:00 | Stock  | Planed Outage
 13/08/2018 | 09:00:00   | 15/08/2018 | 09:00:00 | Stock  | Planed Outage
 19/09/2018 | 15:00:00   | 19/09/2018 | 15:15:00 | Sales  | Sue tripped on a cable
(6 rows)
```

```sql
with mcel as (
select
    system,
    error,
    (to_date(start_date, 'DD/MM/YYYY')::text || ' ' || start_time)::timestamp as start_datetime,
    (to_date(end_date, 'DD/MM/YYYY')::text || ' ' || end_time)::timestamp as end_datetime
from
    p2019w12t2
), ael as (
select
    system,
    error,
    to_timestamp(start_date_time, 'DD/MM/YYYY HH24:MI:SS') as start_datetime,
    to_timestamp(end_date_time, 'DD/MM/YYYY HH24:MI:SS') as end_datetime
from
    p2019w12t1
)
select * from ael;

```

## :closed_book: Reference

None
