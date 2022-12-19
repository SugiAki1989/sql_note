## :memo: Overview

PostgesSQL と MySQL の日時の形式変換について、どっちの関数がどっちのデータベースの関数なのかすぐ忘れるのでまとめておく。

## :floppy_disk: Database

PostgreSQL / MySQL

## :bookmark: Tag

`date_trunc`, `extract`, `date_part`, `date_format`, `substr`

## :pencil2: Example

まずは PostgreSQL からまとめておく。

表示をみやすくするために`\x`で拡張モードを ON にしておく。

```sql
\x
Expanded display is on.
```

日付の形式を整えるには`date_trunc`を利用して引数に精度を記述する。`date_trunc`では`timestamp`型のデータに対しては、表示を丸めて`timestamp`型のまま返却してくれる。

```sql
select now()
, date_trunc('year', now()) as year
, date_trunc('month', now()) as month
, date_trunc('day', now()) as day
, date_trunc('hour', now()) as hour
, date_trunc('minute', now()) as minute
, date_trunc('second', now()) as second
;

-[ RECORD 1 ]------------------------
now    | 2022-07-26 15:44:57.15992+09
year   | 2022-01-01 00:00:00+09
month  | 2022-07-01 00:00:00+09
day    | 2022-07-26 00:00:00+09
hour   | 2022-07-26 15:00:00+09
minute | 2022-07-26 15:44:00+09
second | 2022-07-26 15:44:57+09
```

日時を省略したい場合は`substr`を利用する。

```sql
select now()
, substr(now()::text, 1, 4) as y
, substr(now()::text, 1, 7) as ym
, substr(now()::text, 1, 10) as ymd
, substr(now()::text, 1, 13) as ymdh
, substr(now()::text, 1, 16) as ymdhm
, substr(now()::text, 1, 19) as ymdhms
;

-[ RECORD 1 ]-------------------------
now    | 2022-08-19 13:45:42.114622+09
y      | 2022
ym     | 2022-08
ymd    | 2022-08-19
ymdh   | 2022-08-19 13
ymdhm  | 2022-08-19 13:45
ymdhms | 2022-08-19 13:45:42
```

`timestamp`型のデータではなく、構成要素を取り出したい場合は`extract`を利用する。

```sql
select now()
, extract(year from now()) as year
, extract(month from now()) as month
, extract(day from now()) as day
, extract(hour from now()) as hour
, extract(minute from now()) as minute
, extract(second from now()) as second
, extract(dow from now()) as dow -- 0が日曜~6が土曜
;

-[ RECORD 1 ]-------------------------
now    | 2022-07-26 15:48:04.065604+09
year   | 2022
month  | 7
day    | 26
hour   | 15
minute | 48
second | 4.065604
dow    | 2
```

`date_part`を利用しても同様の結果が得られる。

```sql
select now()
, date_part('year', now()) as year
, date_part('month', now()) as month
, date_part('day', now()) as day
, date_part('hour', now()) as hour
, date_part('minute', now()) as minute
, date_part('second', now()) as second
;

-[ RECORD 1 ]-------------------------
now    | 2022-07-26 15:48:36.688868+09
year   | 2022
month  | 7
day    | 26
hour   | 15
minute | 48
second | 36.688868
```

`to_number(to_char())`を利用しても同様の結果が得られる。

```sql
select now()
, to_number(to_char(now(), 'yyyy'),'9999') as y
, to_number(to_char(now(), 'mm'),'99') as m
, to_number(to_char(now(), 'dd'),'99') as d
, to_number(to_char(now(), 'hh24'),'99') as h
, to_number(to_char(now(), 'mi'),'99') as m
, to_number(to_char(now(), 'ss'),'99') as s
;

-[ RECORD 1 ]----------------------
now | 2022-07-26 18:05:08.534896+09
y   | 2022
m   | 7
d   | 26
h   | 18
m   | 5
s   | 8
```

ここからは MySQL。日付の形式を整えるには`date_format`を利用して引数に形式を記述する。表示オプションを変更するためにクエリ末尾に`\G`をつける。

```sql
select now()
, date_format(now(), '%Y-01-01 00:00:00') as y_full
, date_format(now(), '%Y-%m-01 00:00:00') as ym_full
, date_format(now(), '%Y-%m-%d 00:00:00') as ymd_full
, date_format(now(), '%Y-%m-%d %H:00:00') as ymdh_full
, date_format(now(), '%Y-%m-%d %H:%m:00') as ymdhm_full
, date_format(now(), '%Y-%m-%d %H:%m:%s') as ymdhms_full
\G

*************************** 1. row ***************************
      now(): 2022-07-26 15:52:09
     y_full: 2022-01-01 00:00:00
    ym_full: 2022-07-01 00:00:00
   ymd_full: 2022-07-26 00:00:00
  ymdh_full: 2022-07-26 15:00:00
 ymdhm_full: 2022-07-26 15:07:00
ymdhms_full: 2022-07-26 15:07:09
```

特定の部分までの構成要素を取り出したい場合は`date_format`を利用して、形式を調整する。

```sql
select now()
, date_format(now(), '%Y') as y
, date_format(now(), '%Y-%m') as ym
, date_format(now(), '%Y-%m-%d') as ymd
, date_format(now(), '%Y-%m-%d %H') as ymdh
, date_format(now(), '%Y-%m-%d %H:%m') as ymdhm
, date_format(now(), '%Y-%m-%d %H:%m:%s') as ymdhms
\G

*************************** 1. row ***************************
 now(): 2022-07-26 15:55:35
     y: 2022
    ym: 2022-07
   ymd: 2022-07-26
  ymdh: 2022-07-26 15
 ymdhm: 2022-07-26 15:07
ymdhms: 2022-07-26 15:07:35
```

単純に構成要素を取り出したい場合は下記の関数を利用する。

```sql
select now()
, year(now()) as year
, month(now()) as month
, day(now()) as day
, hour(now()) as hour
, minute(now()) as minute
, second(now()) as second
, dayofweek(now()) as dow -- 1が日曜~7が土曜
\G

*************************** 1. row ***************************
 now(): 2022-07-26 15:55:53
  year: 2022
 month: 7
   day: 26
  hour: 15
minute: 55
second: 53
   dow: 3
```

## :closed_book: Reference

- [9.9. 日付/時刻関数と演算子](https://www.postgresql.jp/document/13/html/functions-datetime.html)
- [12.7 日付および時間関数](https://dev.mysql.com/doc/refman/8.0/ja/date-and-time-functions.html#function_date-format)
