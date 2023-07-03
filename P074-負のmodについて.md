## :memo: Overview

ここでは負の`mod`についてまとめておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`mod`

## :pencil2: Example

これはテーブルの内容が見えているので、結果が事前にわかるかもしれないが、`mod`を使ってグループ分けしようとすると、カウントの数が合わない。目検で対応できるレベルなのですぐ理由がわかるが、大きなデータかつ面倒な分析 SQL の途中で使う場合、原因がすぐわからなかった。

```sql
with tmp as (
	select num from generate_series(1, 5, 1) as num
    union all
    select -5 as num
)
select
	mod(num, 5) as mod_group,
    count(1)
from
	tmp
group by
    mod(num, 5)
;

 mod_group | count
-----------+-------
         0 |     2
         1 |     1
         3 |     1
         4 |     1
         2 |     1
(5 rows)
```

種明かしはマイナスの数が混じっているためで、`mod(-5, 5)=0`になるためだった。

```sql
with tmp as (
	select num from generate_series(1, 5, 1) as num
    union all
    select -5 as num
)
select
	num,
    mod(num, 5)
from
	tmp
;

 num | mod
-----+-----
   1 |   1
   2 |   2
   3 |   3
   4 |   4
   5 |   0
  -5 |   0
(6 rows)
```

そもそも`mod`をマイナスの数に使おうとしたことないので、原因究明に少し時間がかかった。マイナスの`mod`の挙動をメモしておく。

```sql
with tmp as (
	select num from generate_series(-20, 10, 1) as num
    union all
)
select
    num,
    mod(num, 5)
from
	tmp
order by
    num asc
;

 num | mod
-----+-----
 -20 |   0
 -19 |  -4
 -18 |  -3
 -17 |  -2
 -16 |  -1
 -15 |   0
 -14 |  -4
 -13 |  -3
 -12 |  -2
 -11 |  -1
 -10 |   0
  -9 |  -4
  -8 |  -3
  -7 |  -2
  -6 |  -1
  -5 |   0
  -4 |  -4
  -3 |  -3
  -2 |  -2
  -1 |  -1
   0 |   0
   1 |   1
   2 |   2
   3 |   3
   4 |   4
   5 |   0
   6 |   1
   7 |   2
   8 |   3
   9 |   4
  10 |   0
(32 rows)
```

なるほど。

## :closed_book: Reference

None
