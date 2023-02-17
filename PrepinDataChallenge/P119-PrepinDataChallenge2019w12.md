## :memo: Overview

Preppin' Data challenge の「2019: Week 12」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/05/2019-week-12.html)
- [Answer](https://preppindata.blogspot.com/2019/05/2019-week-12-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。エラーログのデータで各システムのダウンタイムの比率を調べるというもの。

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

マニュアルのエラーリストと自動のエラーリストの 2 つがあるので、自動エラーと手動エラーの日付の時間の関係を元に、重複したエラーを除外していく。細かい詳細はソリューションを見ればわかる。まずはスタートチェック。

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
), tmp1 as (
select
    m.system, m.error, m.start_datetime, m.end_datetime,
    a.system, a.error, a.start_datetime, a.end_datetime
from
    mcel as m
left join
    ael as a
on
    m.start_datetime >= a.start_datetime and
    m.start_datetime <= a.end_datetime
)
select * from tmp1;

select * from tmp1;
 system |         error          |   start_datetime    |    end_datetime     | system |         error         |     start_datetime     |      end_datetime
--------+------------------------+---------------------+---------------------+--------+-----------------------+------------------------+------------------------
 Stock  | Planed Outage          | 2018-04-13 09:00:00 | 2018-04-15 09:00:00 | (null) | (null)                | (null)                 | (null)
 Stock  | Planed Outage          | 2018-05-13 09:00:00 | 2018-05-15 09:00:00 | (null) | (null)                | (null)                 | (null)
 Stock  | Planed Outage          | 2018-06-13 09:00:00 | 2018-06-15 09:00:00 | (null) | (null)                | (null)                 | (null)
 Stock  | Planed Outage          | 2018-07-13 09:00:00 | 2018-07-15 09:00:00 | (null) | (null)                | (null)                 | (null)
 Stock  | Planed Outage          | 2018-08-13 09:00:00 | 2018-08-15 09:00:00 | (null) | (null)                | (null)                 | (null)
 Sales  | Sue tripped on a cable | 2018-09-19 15:00:00 | 2018-09-19 15:15:00 | Sales  | Unknown disc location | 2018-09-19 14:57:00+09 | 2018-09-19 15:02:20+09
(6 rows)
```

次はエンドチェック。

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
), tmp1 as (
select
    m.system, m.error, m.start_datetime, m.end_datetime
from
    mcel as m
left join
    ael as a
on
    m.start_datetime >= a.start_datetime and
    m.start_datetime <= a.end_datetime
where
    a.system is null
)
select * from tmp1;

 system |     error     |   start_datetime    |    end_datetime     |
--------+---------------+---------------------+---------------------+
 Stock  | Planed Outage | 2018-04-13 09:00:00 | 2018-04-15 09:00:00 |
 Stock  | Planed Outage | 2018-05-13 09:00:00 | 2018-05-15 09:00:00 |
 Stock  | Planed Outage | 2018-06-13 09:00:00 | 2018-06-15 09:00:00 |
 Stock  | Planed Outage | 2018-07-13 09:00:00 | 2018-07-15 09:00:00 |
 Stock  | Planed Outage | 2018-08-13 09:00:00 | 2018-08-15 09:00:00 |
(5 rows)
```

最後は中間チェック。

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
), tmp1 as (
select
    m.system, m.error, m.start_datetime, m.end_datetime
from
    mcel as m
left join
    ael as a
on
    m.start_datetime >= a.start_datetime and
    m.start_datetime <= a.end_datetime
where
    a.system is null
), tmp2 as (
select
    t.system, t.error, t.start_datetime, t.end_datetime
from
    tmp1 as t
left join
    ael as a
on
    t.end_datetime >= a.start_datetime and
    t.end_datetime <= a.end_datetime
where
    a.system is null
)
select * from tmp2;

 system |     error     |   start_datetime    |    end_datetime
--------+---------------+---------------------+---------------------
 Stock  | Planed Outage | 2018-07-13 09:00:00 | 2018-07-15 09:00:00
 Stock  | Planed Outage | 2018-08-13 09:00:00 | 2018-08-15 09:00:00
(2 rows)

```

自動エラーと手動エラーをユニオンして、各システムごとのダウンタイムを計算する。

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
), tmp1 as (
select
    m.system, m.error, m.start_datetime, m.end_datetime
from
    mcel as m
left join
    ael as a
on
    m.start_datetime >= a.start_datetime and
    m.start_datetime <= a.end_datetime
where
    a.system is null
), tmp2 as (
select
    t.system, t.error, t.start_datetime, t.end_datetime
from
    tmp1 as t
left join
    ael as a
on
    t.end_datetime >= a.start_datetime and
    t.end_datetime <= a.end_datetime
where
    a.system is null
), tmp3 as (
select
    t.system, t.error, t.start_datetime, t.end_datetime,
    'ManualCaptureErrorList' as error_source
from
    tmp2 as t
left join
    ael as a
on
    t.start_datetime <= a.start_datetime and
    t.end_datetime >= a.end_datetime
where
    a.system is null
), tmp4 as(
select * from tmp3
union all
select *, 'AutomaticErrorLog' as error_source from ael
)
select
    *,
    round(extract(epoch from end_datetime - start_datetime)/(60*60),1) as DowntimeInHours
from
    tmp4
;

 system |         error         |     start_datetime     |      end_datetime      |      error_source      | downtimeinhours
--------+-----------------------+------------------------+------------------------+------------------------+-----------------
 Stock  | Planed Outage         | 2018-07-13 09:00:00+09 | 2018-07-15 09:00:00+09 | ManualCaptureErrorList |            48.0
 Stock  | Planed Outage         | 2018-08-13 09:00:00+09 | 2018-08-15 09:00:00+09 | ManualCaptureErrorList |            48.0
 Sales  | Disc full             | 2019-04-13 07:55:23+09 | 2019-04-13 08:23:12+09 | AutomaticErrorLog      |             0.5
 Stock  | Planned Outage        | 2018-04-13 09:03:22+09 | 2018-04-15 09:03:21+09 | AutomaticErrorLog      |            48.0
 Stock  | Planned Outage        | 2018-05-13 09:03:22+09 | 2018-05-16 09:03:21+09 | AutomaticErrorLog      |            72.0
 Stock  | Planned Outage        | 2018-06-13 09:03:22+09 | 2018-06-15 09:03:21+09 | AutomaticErrorLog      |            48.0
 Stock  | Planned Outage        | 2018-07-12 23:03:22+09 | 2018-07-13 07:00:03+09 | AutomaticErrorLog      |             7.9
 Stock  | Planned Outage        | 2018-08-12 23:03:22+09 | 2018-08-13 07:00:03+09 | AutomaticErrorLog      |             7.9
 Sales  | Unknown disc location | 2018-08-19 07:55:20+09 | 2018-08-19 08:31:55+09 | AutomaticErrorLog      |             0.6
 Stock  | Planned Outage        | 2018-09-12 23:03:22+09 | 2018-09-13 07:31:02+09 | AutomaticErrorLog      |             8.5
 Sales  | Unknown disc location | 2018-09-19 14:57:00+09 | 2018-09-19 15:02:20+09 | AutomaticErrorLog      |             0.1
(11 rows)
```

最後に集計してから紐付けて計算する。この部分は集計から JOIN をしなくても SQL であればウインドウ関数でパーティションを使えば、この処理は不要になる。おそらく Prep では LOD 計算が実装されるまでのお題のため、解答ではそうしていたものと思われる。

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
), tmp1 as (
select
    m.system, m.error, m.start_datetime, m.end_datetime
from
    mcel as m
left join
    ael as a
on
    m.start_datetime >= a.start_datetime and
    m.start_datetime <= a.end_datetime
where
    a.system is null
), tmp2 as (
select
    t.system, t.error, t.start_datetime, t.end_datetime
from
    tmp1 as t
left join
    ael as a
on
    t.end_datetime >= a.start_datetime and
    t.end_datetime <= a.end_datetime
where
    a.system is null
), tmp3 as (
select
    t.system, t.error, t.start_datetime, t.end_datetime,
    'ManualCaptureErrorList' as error_source
from
    tmp2 as t
left join
    ael as a
on
    t.start_datetime <= a.start_datetime and
    t.end_datetime >= a.end_datetime
where
    a.system is null
), tmp4 as(
select * from tmp3
union all
select *, 'AutomaticErrorLog' as error_source from ael
), tmp5 as (
select
    *,
    round(extract(epoch from end_datetime - start_datetime)/(60*60),1) as DowntimeInHours
from
    tmp4
), tmp6 as (
select
    system,
    sum(DowntimeInHours) as sum_DowntimeInHours
from
    tmp5
group by
    system
)
select
    *,
    round(t5.DowntimeInHours/t6.sum_DowntimeInHours, 2) as PercentOfSystemDowntime
from
    tmp5 as t5
left join
    tmp6 as t6
on
    t5.system = t6.system
;

 system |         error         |     start_datetime     |      end_datetime      |      error_source      | downtimeinhours | sum_downtimeinhours | percentofsystemdowntime
--------+-----------------------+------------------------+------------------------+------------------------+-----------------+---------------------+-------------------------
 Stock  | Planed Outage         | 2018-07-13 09:00:00+09 | 2018-07-15 09:00:00+09 | ManualCaptureErrorList |            48.0 |               288.3 |                    0.17
 Stock  | Planed Outage         | 2018-08-13 09:00:00+09 | 2018-08-15 09:00:00+09 | ManualCaptureErrorList |            48.0 |               288.3 |                    0.17
 Sales  | Disc full             | 2019-04-13 07:55:23+09 | 2019-04-13 08:23:12+09 | AutomaticErrorLog      |             0.5 |                 1.2 |                    0.42
 Stock  | Planned Outage        | 2018-04-13 09:03:22+09 | 2018-04-15 09:03:21+09 | AutomaticErrorLog      |            48.0 |               288.3 |                    0.17
 Stock  | Planned Outage        | 2018-05-13 09:03:22+09 | 2018-05-16 09:03:21+09 | AutomaticErrorLog      |            72.0 |               288.3 |                    0.25
 Stock  | Planned Outage        | 2018-06-13 09:03:22+09 | 2018-06-15 09:03:21+09 | AutomaticErrorLog      |            48.0 |               288.3 |                    0.17
 Stock  | Planned Outage        | 2018-07-12 23:03:22+09 | 2018-07-13 07:00:03+09 | AutomaticErrorLog      |             7.9 |               288.3 |                    0.03
 Stock  | Planned Outage        | 2018-08-12 23:03:22+09 | 2018-08-13 07:00:03+09 | AutomaticErrorLog      |             7.9 |               288.3 |                    0.03
 Sales  | Unknown disc location | 2018-08-19 07:55:20+09 | 2018-08-19 08:31:55+09 | AutomaticErrorLog      |             0.6 |                 1.2 |                    0.50
 Stock  | Planned Outage        | 2018-09-12 23:03:22+09 | 2018-09-13 07:31:02+09 | AutomaticErrorLog      |             8.5 |               288.3 |                    0.03
 Sales  | Unknown disc location | 2018-09-19 14:57:00+09 | 2018-09-19 15:02:20+09 | AutomaticErrorLog      |             0.1 |                 1.2 |                    0.08
(11 rows)
```

## :closed_book: Reference

None
