## :memo: Overview

PostgesSQL と MySQL の日時の演算について、どっちの関数がどっちのデータベースの関数なのかすぐ忘れるのでまとめておく。

## :floppy_disk: Database

PostgreSQL / MySQL

## :bookmark: Tag

`interval`, `extract(epoch)`, `dateadd`, `datediff`, `timediff`, `timestampdiff`, `age`

## :pencil2: Example

まずは PostgreSQL からまとめておく。`\x`で拡張表示を起動しておく。

```sql
\x
Expanded display is on.
```

日付の加算減算を行う場合は、`interval`を利用する。

```sql
with tmp as (
select now() as now
, now() + interval '1 year' as plus1year
, now() - interval '1 year' as minus1year
, now() + interval '1 month' as plus1month
, now() - interval '1 month' as minus1month
, now() + interval '1 day' as plus1day
, now() - interval '1 day' as minus1day
, now() + interval '1 hour' as plus1hour
, now() - interval '1 hour' as minus1hour
, now() + interval '1 minute' as plus1minute
, now() - interval '1 minute' as minus1minute
, now() + interval '1 second' as plus1second
, now() - interval '1 second' as minus1second
)
select * from tmp;

-[ RECORD 1 ]+------------------------------
now          | 2022-07-28 15:04:39.214876+09
plus1year    | 2023-07-28 15:04:39.214876+09
minus1year   | 2021-07-28 15:04:39.214876+09
plus1month   | 2022-08-28 15:04:39.214876+09
minus1month  | 2022-06-28 15:04:39.214876+09
plus1day     | 2022-07-29 15:04:39.214876+09
minus1day    | 2022-07-27 15:04:39.214876+09
plus1hour    | 2022-07-28 16:04:39.214876+09
minus1hour   | 2022-07-28 14:04:39.214876+09
plus1minute  | 2022-07-28 15:05:39.214876+09
minus1minute | 2022-07-28 15:03:39.214876+09
plus1second  | 2022-07-28 15:04:40.214876+09
minus1second | 2022-07-28 15:04:38.214876+09
```

日付の差はそのまま引き算すればよいが、`interval`で返ってくるので、整数(秒数)で返したい場合は、`extract(epoch)`を利用する。

```sql
with tmp as (
select now() as now
, now() + interval '1 year' as plus1year
, now() - interval '1 year' as minus1year
, now() + interval '1 month' as plus1month
, now() - interval '1 month' as minus1month
, now() + interval '1 day' as plus1day
, now() - interval '1 day' as minus1day
, now() + interval '1 hour' as plus1hour
, now() - interval '1 hour' as minus1hour
, now() + interval '1 minute' as plus1minute
, now() - interval '1 minute' as minus1minute
, now() + interval '1 second' as plus1second
, now() - interval '1 second' as minus1second
)
select
	plus1year - now as diff_year,
	plus1month - now as diff_month,
	plus1day - now as diff_day,
	plus1hour - now as diff_hour,
	plus1minute - now as diff_minute,
	plus1second - now as diff_second,
	extract(epoch from plus1hour - now) as timestampdiff_hour,
	extract(epoch from plus1minute - now) as timestampdiff_minute,
	extract(epoch from plus1second - now) as timestampdiff_second
from
	tmp

-[ RECORD 1 ]--------+------------
diff_year            | 365 days
diff_month           | 31 days
diff_day             | 1 day
diff_hour            | 01:00:00
diff_minute          | 00:01:00
diff_second          | 00:00:01
timestampdiff_hour   | 3600.000000
timestampdiff_minute | 60.000000
timestampdiff_second | 1.000000
;

```

2 つの日付の月数や年数の差を求めたい場合は、`extract(year)`,`extract(ymonth)`を利用して計算する方法もあるが、後で紹介する`age`関数の方法のほうが厳密な計算ができるので、あくまで参考としてまとめておく。

```sql
with tmp as (
select
	'2022-01-01 00:00:00'::timestamp with time zone as p1,
	'2023-07-01 00:00:00'::timestamp with time zone as p2
)
select
	(extract(year from p2) - extract(year from p1)) as diff_year,
	(extract(year from p2) - extract(year from p1))*12 as diff_year_scale_month,
	extract(month from p2) - extract(month from p1) as diff_month,
	(extract(year from p2) - extract(year from p1))*12 +
		extract(month from p2) - extract(month from p1) as diff_yearmonth
from
	tmp
;

-[ RECORD 1 ]---------+---
diff_year             | 1
diff_year_scale_month | 12
diff_month            | 6
diff_yearmonth        | 18
```

PostgreSQL では`age`関数を使うと返り値が`interval`型となり何年何ヶ月何日何時間何分何秒経過したかがわかる。

```sql
with tmp as (
select
	'2022-01-01 12:13:14'::timestamp with time zone as p1,
	'2023-07-03 15:10:01'::timestamp with time zone as p2
)
select
	p1, p2,
	age(p2, p1) as age,
	extract(year from age(p2, p1))*12 as diff_year,
	extract(year from age(p2, p1))*12 + extract(month from age(p2, p1)) as diff_yearmonth
from
	tmp
;
           p1           |           p2           |              age              | diff_year | diff_yearmonth
------------------------+------------------------+-------------------------------+-----------+----------------
 2022-01-01 12:13:14+09 | 2023-07-03 15:10:01+09 | 1 year 6 mons 2 days 02:56:47 |        12 |             18
```

`age`の返り値は`interval`なので型変換できないが、その場合は`date_part`を利用すれば OK。

```sql
select age('2022-09-11 00:00:00'::timestamp, '2022-09-10 00:00:00'::timestamp); -- interval
  age
-------
 1 day
(1 row)

select (age('2022-09-11 00:00:00'::timestamp, '2022-09-10 00:00:00'::timestamp))::integer; -- interval
ERROR:  cannot cast type interval to integer

select cast(age('2022-09-11 00:00:00'::timestamp, '2022-09-10 00:00:00'::timestamp) as integer) ; -- interval
ERROR:  cannot cast type interval to integer

select date_part('day', age('2022-09-11 00:00:00'::timestamp, '2022-09-10 00:00:00'::timestamp));
 date_part
-----------
         1
(1 row)

select date_part('day', age('2022-09-11 00:00:00'::timestamp, '2022-09-10 00:00:00'::timestamp))::real;
 date_part
-----------
         1
(1 row)
```

MySQL はここから。`interval`ではなく、`dateadd`でもよい。`\G`で表示を見やすいようにしておく。

```sql
with tmp as (
select now() as now
, now() + interval 1 year as plus1year
, now() - interval 1 year as minus1year
, now() + interval 1 month as plus1month
, now() - interval 1 month as minus1month
, now() + interval 1 day as plus1day
, now() - interval 1 day as minus1day
, now() + interval 1 hour as plus1hour
, now() - interval 1 hour as minus1hour
, now() + interval 1 minute as plus1minute
, now() - interval 1 minute as minus1minute
, now() + interval 1 second as plus1second
, now() - interval 1 second as minus1second
)
select * from tmp\G

*************************** 1. row ***************************
         now: 2022-07-28 15:14:17
   plus1year: 2023-07-28 15:14:17
  minus1year: 2021-07-28 15:14:17
  plus1month: 2022-08-28 15:14:17
 minus1month: 2022-06-28 15:14:17
    plus1day: 2022-07-29 15:14:17
   minus1day: 2022-07-27 15:14:17
   plus1hour: 2022-07-28 16:14:17
  minus1hour: 2022-07-28 14:14:17
 plus1minute: 2022-07-28 15:15:17
minus1minute: 2022-07-28 15:13:17
 plus1second: 2022-07-28 15:14:18
minus1second: 2022-07-28 15:14:16
```

`dateadd`の場合は下記の通り。

```sql
with tmp as (
select now() as now
, date_add(now(), interval +1 year) as plus1year
, date_add(now(), interval -1 year) as minus1year
, date_add(now(), interval +1 month) as plus1month
, date_add(now(), interval -1 month) as minus1month
, date_add(now(), interval +1 hour) as plus1hour
, date_add(now(), interval -1 hour) as minus1hour
, date_add(now(), interval +1 minute) as plus1minute
, date_add(now(), interval -1 minute) as minus1minute
, date_add(now(), interval +1 second) as plus1second
, date_add(now(), interval -1 second) as minussecond
)
select * from tmp\G

*************************** 1. row ***************************
         now: 2022-07-28 15:22:29
   plus1year: 2023-07-28 15:22:29
  minus1year: 2021-07-28 15:22:29
  plus1month: 2022-08-28 15:22:29
 minus1month: 2022-06-28 15:22:29
   plus1hour: 2022-07-28 16:22:29
  minus1hour: 2022-07-28 14:22:29
 plus1minute: 2022-07-28 15:23:29
minus1minute: 2022-07-28 15:21:29
 plus1second: 2022-07-28 15:22:30
 minussecond: 2022-07-28 15:22:28
```

MySQL では、`datediff`を利用する。整数(秒数)で返したい場合は、`timestampdiff`を利用する。

```sql
with tmp as (
select now() as now
, now() + interval 1 year as plus1year
, now() - interval 1 year as minus1year
, now() + interval 1 month as plus1month
, now() - interval 1 month as minus1month
, now() + interval 1 day as plus1day
, now() - interval 1 day as minus1day
, now() + interval 1 hour as plus1hour
, now() - interval 1 hour as minus1hour
, now() + interval 1 minute as plus1minute
, now() - interval 1 minute as minus1minute
, now() + interval 1 second as plus1second
, now() - interval 1 second as minus1second
)
select
	datediff(plus1year, now) as datediff_year,
	datediff(plus1month, now) as datediff_month,
	datediff(plus1day, now) as datediff_day,
	timediff(plus1hour, now) as timediff_hour,
	timediff(plus1minute, now) as timediff_minute,
	timediff(plus1second, now) as timediff_second,
	timestampdiff(second, now, plus1hour) as timestampdiff_hour,
	timestampdiff(second, now, plus1minute) as timestampdiff_minute,
	timestampdiff(second, now, plus1second) as timestampdiff_second
from
	tmp
\G

*************************** 1. row ***************************
       datediff_year: 365
      datediff_month: 31
        datediff_day: 1
       timediff_hour: 01:00:00
     timediff_minute: 00:01:00
     timediff_second: 00:00:01
  timestampdiff_hour: 3600
timestampdiff_minute: 60
timestampdiff_second: 1
```

2 つの日付の月数や年数の差を求めたい場合は、`year`,`month`を利用して計算する。

```sql
with tmp as (
select
	cast('2022-01-01 00:00:00' as datetime) as p1,
	cast('2023-07-01 00:00:00' as datetime) as p2
)
select
	(year(p2) - year(p1)) as diff_year,
	(year(p2) - year(p1))*12 as diff_year_scale_month,
	month(p2) - month(p1) as diff_month,
	(year(p2) - year(p1))*12 + month(p2) - month(p1) as diff_yearmonth
from
	tmp
\G

*************************** 1. row ***************************
            diff_year: 1
diff_year_scale_month: 12
           diff_month: 6
       diff_yearmonth: 18
```

PostgreSQL では`age`関数で年齢を簡単に計算できるが、MySQL では便利な関数がないので下記の方法で年齢を計算できる。

```sql
create table age(dt date);
insert into age
	(dt)
values
	('1988-02-28'),
	('1988-02-29'),
	('1988-03-01'),
	('1988-03-02'),
	('1988-03-03')
;

with tmp as (
select
	dt,
	20240302 as referdt
from
	age
)
select
	dt,
	referdt,
	cast(replace(dt, '-', '') as unsigned) as intdt,
	referdt - cast(replace(dt, '-', '') as unsigned) as dtdiff,
	(referdt - cast(replace(dt, '-', '') as unsigned))/10000 as age1,
	floor((referdt - cast(replace(dt, '-', '') as unsigned))/10000) as age2
from
	tmp
;

+------------+----------+----------+--------+---------+------+
| dt         | referdt  | intdt    | dtdiff | age1    | age2 |
+------------+----------+----------+--------+---------+------+
| 1988-02-28 | 20240228 | 19880228 | 360000 | 36.0000 |   36 |
| 1988-02-29 | 20240228 | 19880229 | 359999 | 35.9999 |   35 |
| 1988-03-01 | 20240228 | 19880301 | 359927 | 35.9927 |   35 |
| 1988-03-02 | 20240228 | 19880302 | 359926 | 35.9926 |   35 |
| 1988-03-03 | 20240228 | 19880303 | 359925 | 35.9925 |   35 |
+------------+----------+----------+--------+---------+------+
+------------+----------+----------+--------+---------+------+
| dt         | referdt  | intdt    | dtdiff | age1    | age2 |
+------------+----------+----------+--------+---------+------+
| 1988-02-28 | 20240229 | 19880228 | 360001 | 36.0001 |   36 |
| 1988-02-29 | 20240229 | 19880229 | 360000 | 36.0000 |   36 |
| 1988-03-01 | 20240229 | 19880301 | 359928 | 35.9928 |   35 |
| 1988-03-02 | 20240229 | 19880302 | 359927 | 35.9927 |   35 |
| 1988-03-03 | 20240229 | 19880303 | 359926 | 35.9926 |   35 |
+------------+----------+----------+--------+---------+------+
+------------+----------+----------+--------+---------+------+
| dt         | referdt  | intdt    | dtdiff | age1    | age2 |
+------------+----------+----------+--------+---------+------+
| 1988-02-28 | 20240301 | 19880228 | 360073 | 36.0073 |   36 |
| 1988-02-29 | 20240301 | 19880229 | 360072 | 36.0072 |   36 |
| 1988-03-01 | 20240301 | 19880301 | 360000 | 36.0000 |   36 |
| 1988-03-02 | 20240301 | 19880302 | 359999 | 35.9999 |   35 |
| 1988-03-03 | 20240301 | 19880303 | 359998 | 35.9998 |   35 |
+------------+----------+----------+--------+---------+------+
+------------+----------+----------+--------+---------+------+
| dt         | referdt  | intdt    | dtdiff | age1    | age2 |
+------------+----------+----------+--------+---------+------+
| 1988-02-28 | 20240302 | 19880228 | 360074 | 36.0074 |   36 |
| 1988-02-29 | 20240302 | 19880229 | 360073 | 36.0073 |   36 |
| 1988-03-01 | 20240302 | 19880301 | 360001 | 36.0001 |   36 |
| 1988-03-02 | 20240302 | 19880302 | 360000 | 36.0000 |   36 |
| 1988-03-03 | 20240302 | 19880303 | 359999 | 35.9999 |   35 |
+------------+----------+----------+--------+---------+------+
```

## :closed_book: Reference

- [9.9. 日付/時刻関数と演算子](https://www.postgresql.jp/document/13/html/functions-datetime.html)
- [12.7 日付および時間関数](https://dev.mysql.com/doc/refman/8.0/ja/date-and-time-functions.html#function_date-format)
