## :memo: Overview

SQL の評価順序についてまとめる。記述順序と評価順序については下記の通り。

| No  | Write                                             | Evaluation                     |
| :-: | ------------------------------------------------- | ------------------------------ |
|  1  | select(including aggregation/window fun/distinct) | from                           |
|  2  | from                                              | on                             |
|  3  | join                                              | join                           |
|  4  | on                                                | where                          |
|  5  | where                                             | groupby(including aggregation) |
|  6  | groupby                                           | having                         |
|  7  | having                                            | window function                |
|  8  | orderby                                           | select                         |
|  9  | limit                                             | distinct                       |
| 10  | union                                             | union                          |
| 10  |                                                   | orderby                        |
| 11  |                                                   | limit                          |

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`a`, `a`, `a`, `a`, `a`

## :pencil2: Example

まず `from` と `join(on)`が評価されて、処理に必要なテーブルが準備される。サブクエリがある場合は、サブクエリ内で評価が順番に行われる。

次に `where` を評価して、テーブルの絞り込みを行う。つまり、このあとの計算には `where` で除外されたものは対象外となる。

そして、`groupby` で集計が行われ、`groupby` の集計結果に対して `having` でさらにデータを除外する。`having`には集計関数をそのまま利用する。

```sql
select
    state,
	count(*) as aggfunc
from
    legislators_terms
group by
    state
having
	count(*) > 3000
;

 state | aggfunc
-------+---------
 NY    |    4159
 PA    |    3252
```

`as`でエイリアスをつけていても、`as`は`select`の評価タイミングまで評価されないので、`having`が評価されるタイミングでは存在しないため、エラーが返される。

```sql
select
    state,
	count(*) as aggfunc
from
    legislators_terms
group by
    state
having
	aggfunc > 3000
;

ERROR:  column "aggfunc" does not exist
LINE 9:  aggfunc > 3000
```

!!! ??? MySQL では先にエイリアスを評価してくれるので、エラーがでない。

```sql
SELECT
	deptno,
    count(*) as aggfunc
FROM
    sqlcookbook.EMP
group by
	deptno
having
	aggfunc > 4
;

+--------+---------+
| deptno | aggfunc |
+--------+---------+
|     20 |       5 |
|     30 |       6 |
+--------+---------+
```

このタイミングで、window 関数の評価が始まるので、window 関数には、この時点の前に評価が完了している `groupby` の集計結果を利用できる。下記の例では、`count(*)`が window 関数内で使用されている。

```sql
select
    state,
	count(*) as aggfunc,
	sum(count(*)) over () as windowfunc
from
    legislators_terms
group by
    state
limit
    5
;

 state | aggfunc | windowfunc
-------+---------+------------
 DK    |      16 |      44063
 ND    |     170 |      44063
 NV    |     177 |      44063
 OH    |    2239 |      44063
 GU    |      24 |      44063
```

下記の例では、`count(*)`が window 関数内の`order by`でも使用できる。

```sql
select
    state,
	count(*) as aggfunc,
	rank() over (order by count(*) desc) as windowfunc
from
    legislators_terms
group by
    state
limit
    5
;

 state | aggfunc | windowfunc
-------+---------+------------
 NY    |    4159 |          1
 PA    |    3252 |          2
 OH    |    2239 |          3
 CA    |    2121 |          4
 IL    |    2011 |          5
```

この後に `select` が評価されることになり、 この結果に対して `distinct` が機能して重複を除外する。あとは `union`、`orderby`、`limit` が順に評価されてクエリの結果が表示される。

## :closed_book: Reference
