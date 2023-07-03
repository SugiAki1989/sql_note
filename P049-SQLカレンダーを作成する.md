## :memo: Overview

SQL で頭の体操をするには良さそうだったので、メモしておく。SQL でカレンダーを作成し、予定がない日は日付を表示し、予定がある日は予定を表示するカレンダーを作成する。

```sql
$ env LC_TIME=ja_ja.utf8 cal
    August 2022
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31
```

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`calender`, `extract`, `coalesce`

## :pencil2: Example

まずは 8 月の予定を登録するテーブルを作成する。この予定がカレンダーの日と同じ場所に表示されるようにしていく。

```sql
create table schedule(dt date, content varchar(255));
insert into
schedule(dt, content)
values
    ('2022-08-06','meeting'),
    ('2022-08-10','holiday'),
    ('2022-08-22','meeting'),
    ('2022-08-25','salary')
;
```

カレンダーの形式を作成するために、まずその月の 1 日を示すテーブルを使って、1 日からの基準距離`extract(day from tmp.dt) - extract(dow from tmp.dt) -1 as wk`を計算する。この数値が何に使えるのかは言葉よりも画像のほうがわかりやすいので、イメージを参照。

![クイックノート P1](https://user-images.githubusercontent.com/65038325/185779118-66e43c00-8d05-4e1c-a6be-f23e32ac4f1a.png)

```
with tmp as (
select dt ::date
from generate_series('2022-07-15'::date, '2022-09-20'::date, '1 day') as dt
), tmp2 as (
    select '2022-8-01'::date as first
)
select
    tmp.dt,
    extract(day from tmp.dt),
    extract(dow from tmp.dt),
    extract(day from tmp.dt) - extract(dow from tmp.dt) as miswk,
    extract(day from tmp.dt) - extract(dow from tmp.dt) -1 as wk
from
    tmp
inner join
    tmp2
on
    extract(month from tmp.dt) = extract(month from tmp2.first)
;
     dt     | extract | extract | miswk | wk
------------+---------+---------+-------+----
 2022-08-01 |       1 |       1 |     0 | -1
 2022-08-02 |       2 |       2 |     0 | -1
 2022-08-03 |       3 |       3 |     0 | -1
 2022-08-04 |       4 |       4 |     0 | -1
 2022-08-05 |       5 |       5 |     0 | -1
 2022-08-06 |       6 |       6 |     0 | -1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2022-08-07 |       7 |       0 |     7 |  6
 2022-08-08 |       8 |       1 |     7 |  6
 2022-08-09 |       9 |       2 |     7 |  6
 2022-08-10 |      10 |       3 |     7 |  6
 2022-08-11 |      11 |       4 |     7 |  6
 2022-08-12 |      12 |       5 |     7 |  6
 2022-08-13 |      13 |       6 |     7 |  6
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2022-08-14 |      14 |       0 |    14 | 13
 2022-08-15 |      15 |       1 |    14 | 13
 2022-08-16 |      16 |       2 |    14 | 13
 2022-08-17 |      17 |       3 |    14 | 13
 2022-08-18 |      18 |       4 |    14 | 13
 2022-08-19 |      19 |       5 |    14 | 13
 2022-08-20 |      20 |       6 |    14 | 13
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2022-08-21 |      21 |       0 |    21 | 20
 2022-08-22 |      22 |       1 |    21 | 20
 2022-08-23 |      23 |       2 |    21 | 20
 2022-08-24 |      24 |       3 |    21 | 20
 2022-08-25 |      25 |       4 |    21 | 20
 2022-08-26 |      26 |       5 |    21 | 20
 2022-08-27 |      27 |       6 |    21 | 20
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2022-08-28 |      28 |       0 |    28 | 27
 2022-08-29 |      29 |       1 |    28 | 27
 2022-08-30 |      30 |       2 |    28 | 27
 2022-08-31 |      31 |       3 |    28 | 27
(31 rows)
```

`wk`を取得するために集計をして呼び出せるようにしておく。

```sql
with tmp as (
select dt ::date
from generate_series('2022-07-15'::date, '2022-09-20'::date, '1 day') as dt
), tmp2 as (
    select '2022-8-01'::date as first
), tmp3 as (
select
    tmp.dt,
    extract(day from tmp.dt),
    extract(dow from tmp.dt),
    extract(day from tmp.dt) - extract(dow from tmp.dt) - 1 as wk
from
    tmp
inner join
    tmp2
on
    extract(month from tmp.dt) = extract(month from tmp2.first)
)
select
    wk
from
    tmp3
group by
    wk
;
 wk
----
 -1
  6
 13
 20
 27
(5 rows)
```

8 月の 1 日をしめす`tmp2.first`とさきほどの`wk`を利用して、カレンダーを作成する。

```sql
with tmp as (
select dt ::date
from generate_series('2022-07-15'::date, '2022-09-20'::date, '1 day') as dt
), tmp2 as (
    select '2022-8-01'::date as first
), tmp3 as (
select
    tmp.dt,
    extract(day from tmp.dt),
    extract(dow from tmp.dt),
    extract(day from tmp.dt) - extract(dow from tmp.dt) - 1 as wk
from
    tmp
inner join
    tmp2
on
    extract(month from tmp.dt) = extract(month from tmp2.first)
), tmp4 as (
select
    wk
from
    tmp3
group by
    wk
)
select
    wk,
    (tmp2.first + ((wk + 0) * interval '1 day'))::date as sun,
    (tmp2.first + ((wk + 1) * interval '1 day'))::date as mon,
    (tmp2.first + ((wk + 2) * interval '1 day'))::date as tue,
    (tmp2.first + ((wk + 3) * interval '1 day'))::date as wed,
    (tmp2.first + ((wk + 4) * interval '1 day'))::date as thu,
    (tmp2.first + ((wk + 5) * interval '1 day'))::date as fri,
    (tmp2.first + ((wk + 6) * interval '1 day'))::date as sat
from
    tmp4
cross join
    tmp2
;
 wk |    sun     |    mon     |    tue     |    wed     |    thu     |    fri     |    sat
----+------------+------------+------------+------------+------------+------------+------------
 -1 | 2022-07-31 | 2022-08-01 | 2022-08-02 | 2022-08-03 | 2022-08-04 | 2022-08-05 | 2022-08-06
  6 | 2022-08-07 | 2022-08-08 | 2022-08-09 | 2022-08-10 | 2022-08-11 | 2022-08-12 | 2022-08-13
 13 | 2022-08-14 | 2022-08-15 | 2022-08-16 | 2022-08-17 | 2022-08-18 | 2022-08-19 | 2022-08-20
 20 | 2022-08-21 | 2022-08-22 | 2022-08-23 | 2022-08-24 | 2022-08-25 | 2022-08-26 | 2022-08-27
 27 | 2022-08-28 | 2022-08-29 | 2022-08-30 | 2022-08-31 | 2022-09-01 | 2022-09-02 | 2022-09-03
(5 rows)
```

そして、この状態から各曜日ごとに、相関サブクエリを使って、`schedule`テーブルの日付と各曜日の日付を比較して、予定があればその値を返して、なければ`null`が返るので、`coalesce`で空文字に置き換えて、予定と日付を取得している。

```sql
with tmp as (
select dt ::date
from generate_series('2022-07-15'::date, '2022-09-20'::date, '1 day') as dt
), tmp2 as (
    select '2022-8-01'::date as first
), tmp3 as (
select
    tmp.dt,
    extract(day from tmp.dt),
    extract(dow from tmp.dt),
    extract(day from tmp.dt) - extract(dow from tmp.dt) - 1 as wk
from
    tmp
inner join
    tmp2
on
    extract(month from tmp.dt) = extract(month from tmp2.first)
), tmp4 as (
select
    wk
from
    tmp3
group by
    wk
), tmp5 as(
select
    wk,
    (tmp2.first + ((wk + 0) * interval '1 day'))::date as sun,
    (tmp2.first + ((wk + 1) * interval '1 day'))::date as mon,
    (tmp2.first + ((wk + 2) * interval '1 day'))::date as tue,
    (tmp2.first + ((wk + 3) * interval '1 day'))::date as wed,
    (tmp2.first + ((wk + 4) * interval '1 day'))::date as thu,
    (tmp2.first + ((wk + 5) * interval '1 day'))::date as fri,
    (tmp2.first + ((wk + 6) * interval '1 day'))::date as sat
from
    tmp4
cross join
    tmp2
)
select
    sun::text || ' ' || coalesce((select content from schedule as s where s.dt = tmp5.sun),'') as sun,
    mon::text || ' ' || coalesce((select content from schedule as s where s.dt = tmp5.mon),'') as mon,
    tue::text || ' ' || coalesce((select content from schedule as s where s.dt = tmp5.tue),'') as tue,
    wed::text || ' ' || coalesce((select content from schedule as s where s.dt = tmp5.wed),'') as wed,
    thu::text || ' ' || coalesce((select content from schedule as s where s.dt = tmp5.thu),'') as thu,
    fri::text || ' ' || coalesce((select content from schedule as s where s.dt = tmp5.fri),'') as fri,
    sat::text || ' ' || coalesce((select content from schedule as s where s.dt = tmp5.sat),'') as sat
from
    tmp5
;


     sun     |        mon         |     tue     |        wed         |        thu        |     fri     |        sat
-------------+--------------------+-------------+--------------------+-------------------+-------------+--------------------
 2022-07-31  | 2022-08-01         | 2022-08-02  | 2022-08-03         | 2022-08-04        | 2022-08-05  | 2022-08-06 meeting
 2022-08-07  | 2022-08-08         | 2022-08-09  | 2022-08-10 holiday | 2022-08-11        | 2022-08-12  | 2022-08-13
 2022-08-14  | 2022-08-15         | 2022-08-16  | 2022-08-17         | 2022-08-18        | 2022-08-19  | 2022-08-20
 2022-08-21  | 2022-08-22 meeting | 2022-08-23  | 2022-08-24         | 2022-08-25 salary | 2022-08-26  | 2022-08-27
 2022-08-28  | 2022-08-29         | 2022-08-30  | 2022-08-31         | 2022-09-01        | 2022-09-02  | 2022-09-03
(5 rows)

【参照】
$ env LC_TIME=ja_ja.utf8 cal
    August 2022
Su Mo Tu We Th Fr Sa
    1  2  3  4  5  6
 7  8  9 10 11 12 13
14 15 16 17 18 19 20
21 22 23 24 25 26 27
28 29 30 31
```

## :closed_book: Reference

- [SQL Hacks](https://www.oreilly.co.jp/books/9784873113319/)
