## :memo: Overview

MySQL で中央値を計算する方法をまとめておく。

## :floppy_disk: Database

PostgreSQL / MySQL

## :bookmark: Tag

`percentile_cont`, `generate_series`, `a`, `a`, `a`

## :pencil2: Example

例えば、PostgreSQL で中央値を計算したい場合、パーセンタイルの 50 パーセンタイルを取得することで、中央値を取得することができる。PostgreSQL ではレコード数が偶数の場合、50 パーセンタイルが 2 つ存在することになるため、continuous percentile という 2 つの平均する計算が実行される。

この例だと`5`,`6`の平均として`5.5`が中央値として計算される。

```sql

with tmp as (
	select num from generate_series(1, 10, 1) as num
)
select
	num,
	(select percentile_cont(0.5) within group(order by num) from tmp)
from
	tmp
;

 num | percentile_cont
-----+-----------------
   1 |             5.5
   2 |             5.5
   3 |             5.5
   4 |             5.5
   5 |             5.5
   6 |             5.5
   7 |             5.5
   8 |             5.5
   9 |             5.5
  10 |             5.5
```

MySQL では、中央値の定義通り計算することで算出できる。

```sql
with tmp as (
select
	td.generate_series as num
from
	(
	select 0 generate_series from dual where (@num:=1-1)*0
	union all
	select @num:=@num+1 from `information_schema`.columns limit 10
	) as td
), tmp2 as (
select
	num,
	count(*) over () as nrow,
	row_number() over(order by num asc) as id
from
	tmp
)
select
	avg(num) as median
from
	tmp2
where
	-- n=10の時、偶数の場合は剰余が0なので、5,6番目をとって平均
	(nrow % 2 = 0 and id in (nrow/2, (nrow/2) + 1))
	or
	-- n=9の時、奇数の場合は剰余が1なので、5番目をとって平均
	(nrow % 2 = 1 and id = nrow/2)
;

+--------+
| median |
+--------+
| 5.5000 |
+--------+
```

他にも MySQL では`cume_dist`を利用すればで実現できる。`cume_dist`と`percent_rank`は似ている関数なので、違いを確認しておく。`cume_dist`と`percent_rank`を実行すると、なんか違うことをやっている関数であることがわかる。

`percent_rank`はランクをパーセントで返してくれるので、5 位といわれても何人中なのか？という疑問が出てくるもののパーセントでランクを返してくれと、相対的なランクを理解しやすい。

```sql
with tmp as (
select
	td.generate_series as num
from
	(
	select 0 generate_series from dual where (@num:=1-1)*0
	union all
	select @num:=@num+1 from `information_schema`.columns limit 5
	) as td
)
select
    num,
	round(cume_dist() over (order by num), 2) as cumedist,
	round(percent_rank() over (order by num), 2) as percentrank
from
	tmp
;

+------+----------+-------------+
| num  | cumedist | percentrank |
+------+----------+-------------+
|    1 |      0.2 |           0 |
|    2 |      0.4 |        0.25 |
|    3 |      0.6 |         0.5 |
|    4 |      0.8 |        0.75 |
|    5 |        1 |           1 |
+------+----------+-------------+
```

ドキュメントを読んでもあまり理解できないので、計算式の違いを眺めることにする。

```sql
with tmp as (
select
	td.generate_series as num
from
	(
	select 0 generate_series from dual where (@num:=1-1)*0
	union all
	select @num:=@num+1 from `information_schema`.columns limit 20
	) as td
union all select 13
)
select
    num,
	round(cume_dist() over (order by num), 2) as a
from
	tmp
where
	 num between 10 and 15
union all
select
    num,
	round(percent_rank() over (order by num), 2) as a
from
	tmp
where
	 num between 10 and 15
;


+------+------+
| num  | a    | 　
+------+------+ cume_dist
|   10 | 0.14 | = 1/7
|   11 | 0.29 | = 1/7
|   12 | 0.43 | = 3/7
|   13 | 0.71 | = 5/7 ※同値は同じ累積分布値に評価
|   13 | 0.71 | = 5/7 ※同値は同じ累積分布値に評価
|   14 | 0.86 | = 6/7
|   15 |    1 | = 7/7
~~~~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~percent_rank
|   10 |    0 | = (1-1)/7-1
|   11 | 0.17 | = (2-1)/7-1
|   12 | 0.33 | = (3-1)/7-1
|   13 |  0.5 | = (4-1)/7-1 ※タイなのでランク5が飛ぶ
|   13 |  0.5 | = (4-1)/7-1 ※タイなのでランク5が飛ぶ
|   14 | 0.83 | = (6-1)/7-1
|   15 |    1 | = (7-1)/7-1
+------+------+
```

ということで、最初のデータに対して、中央値を計算しておく。

```sql
with tmp as (
select
	td.generate_series as num
from
	(
	select 0 generate_series from dual where (@num:=1-1)*0
	union all
	select @num:=@num+1 from `information_schema`.columns limit 10
	) as td
), tmp2 as (
select
	num, cume_dist() over (order by num) as p50
from
	tmp
), tmp3 as (
select num, p50 from tmp2 where p50 <= 0.5
union
select num, p50 from tmp2 where p50 >= 0.5
)
select
    avg(num) as median
from
	tmp3
;

+--------+
| median |
+--------+
| 5.5000 |
+--------+
```

## :closed_book: Reference

- [9.21. Aggregate Functions](https://www.postgresql.org/docs/14/functions-aggregate.html#:~:text=Table-,9.59,-.%C2%A0Ordered%2DSet%20Aggregate)
- [12.21.1 Window 関数の説明](https://dev.mysql.com/doc/refman/8.0/ja/window-function-descriptions.html#function_cume-dist)
