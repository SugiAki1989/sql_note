## :memo: Overview

PostgesSQLの日時の演算について、様々なケースをもとに理解を深めていく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`a`, `a`, `a`

## :pencil2: 今年が閏年かどうかを判定する

今年2024年は閏年である。与えれたデータにもよるかもしれないが、閏年かどうか判定したい。

```sql
with tmp as (
select generate_series('2000-01-01'::date, '2028-12-31'::date, '1 day')::date AS dt
)
select
	extract(year from dt) as y,
	extract(month from dt) as m,
	max(case when extract(day from dt) = 29 then 1 else 0 end) as checkleap
from
	tmp
where
	extract(month from dt) = 2
group by
	extract(year from dt),
	extract(month from dt)
;

  y   | m | checkleap
------+---+-----------
 2000 | 2 |         1
 2001 | 2 |         0
 2002 | 2 |         0
 2003 | 2 |         0
 2004 | 2 |         1
 2005 | 2 |         0
 2006 | 2 |         0
 2007 | 2 |         0
 2008 | 2 |         1
 2009 | 2 |         0
 2010 | 2 |         0
 2011 | 2 |         0
 2012 | 2 |         1
 2013 | 2 |         0
 2014 | 2 |         0
 2015 | 2 |         0
 2016 | 2 |         1
 2017 | 2 |         0
 2018 | 2 |         0
 2019 | 2 |         0
 2020 | 2 |         1
 2021 | 2 |         0
 2022 | 2 |         0
 2023 | 2 |         0
 2024 | 2 |         1
 2025 | 2 |         0
 2026 | 2 |         0
 2027 | 2 |         0
 2028 | 2 |         1
(29 rows)
```

下記のとおりSQLを書くことでも、SQLで判定できる。

```sql
select 
	max(to_char(tmp2.dy + x.id, 'DD')) as dy
from(
	select dy, to_char(dy, 'MM') as mth
		from (
			select cast(cast(date_trunc('year', current_date) as date) + interval '1 month' as date) as dy
	) as tmp1
) as tmp2, 
generate_series(0,29) x(id)
where 
	to_char(tmp2.dy + x.id, 'MM') = tmp2.mth
;

 dy
----
 29
(1 row)
```

まずは内側の結果から段階的に確認する。まずは`current_date`で実行日の日付を取得して、年単位に丸め、1ヶ月足すことで2月を作る。

```sql
select date_trunc('year', current_date) ;

       date_trunc
------------------------
 2024-01-01 00:00:00+09
(1 row)

select cast(cast(date_trunc('year', current_date) as date) + interval '1 month' as date) as dy;
     dy
------------
 2024-02-01
(1 row)
```

`to_char`関数で月を取得する。

```sql
select dy, to_char(dy, 'MM') as mth 
from (
	select cast(cast(date_trunc('year', current_date) as date) + interval '1 month' as date) as dy
  ) as tmp1;

     dy     | mth
------------+-----
 2024-02-01 | 02
(1 row)
```

あとは、`generate_series`とCROSS JOINでテーブルを拡張し、

```sql
select 
	*, to_char(tmp2.dy + x.id, 'DD') as dy2,
	tmp2.dy + x.id as c1,
	to_char(tmp2.dy + x.id, 'MM') as c2,
	case when to_char(tmp2.dy + x.id, 'MM') = tmp2.mth then 1 else null end as check
from(
	select dy, to_char(dy, 'MM') as mth
		from (
			select cast(cast(date_trunc('year', current_date) as date) + interval '1 month' as date) as dy
	) as tmp1
) as tmp2, 
generate_series(0,29) x(id);

     dy     | mth | id | dy2 |     c1     | c2 | check
------------+-----+----+-----+------------+----+-------
 2024-02-01 | 02  |  0 | 01  | 2024-02-01 | 02 |     1
 2024-02-01 | 02  |  1 | 02  | 2024-02-02 | 02 |     1
 2024-02-01 | 02  |  2 | 03  | 2024-02-03 | 02 |     1
 2024-02-01 | 02  |  3 | 04  | 2024-02-04 | 02 |     1
 2024-02-01 | 02  |  4 | 05  | 2024-02-05 | 02 |     1
(snip)
 2024-02-01 | 02  | 25 | 26  | 2024-02-26 | 02 |     1
 2024-02-01 | 02  | 26 | 27  | 2024-02-27 | 02 |     1
 2024-02-01 | 02  | 27 | 28  | 2024-02-28 | 02 |     1
 2024-02-01 | 02  | 28 | 29  | 2024-02-29 | 02 |     1
 2024-02-01 | 02  | 29 | 01  | 2024-03-01 | 03 |
(30 rows)
```

2月に一致している行から、`max`関数で最終日を取得すればおしまい。29日なので閏年と判断できる。


```sql
select 
	max(to_char(tmp2.dy + x.id, 'DD')) as dy
from(
	select dy, to_char(dy, 'MM') as mth
		from (
			select cast(cast(date_trunc('year', current_date) as date) + interval '1 month' as date) as dy
	) as tmp1
) as tmp2, 
generate_series(0,29) x(id)
where 
	to_char(tmp2.dy + x.id, 'MM') = tmp2.mth
;

dy
----
 29
(1 row)
```

## :pencil2: 月初月末を取得する方法

実行日の月初月末を計算したい場合のクエリ。まずは`date_trunc`関数で、実行日の月初を計算する。

```sql
select cast(date_trunc('month', current_date) as date) as first_day

 first_day
------------
 2024-03-01
(1 row)
```

次に1ヶ月足して、1日引くことで、実行日の月末を得ることができる。

```sql
select 
	first_day,
	cast(first_day + interval '1 month' - interval '1 day' as date) as lastday
from
	(
		select cast(date_trunc('month', current_date) as date) as first_day
	) as foo;

	 first_day  |  lastday
------------+------------
 2024-03-01 | 2024-03-31
(1 row)
```

普通に考えると、月の日数は、そのつき次第で28、29、30、31と変わるので、これに応じて場合が必要になるが、
このクエリで、次の月までに日数を飛ばして、戻ることで、どの日数の月末であれ、日付を計算できる。前から行くのではなく、回り込んで計算するイメージ。

## :pencil2: ほかにあれば


```sql

```

```sql

```

```sql

```

```sql

```



## :pencil2:　ほかにあれば


```sql

```

```sql

```

```sql

```

```sql

```


## :closed_book: Reference

- [9.9. 日付/時刻関数と演算子](https://www.postgresql.jp/document/13/html/functions-datetime.html)
