## :memo: Overview

SQL でテーブルの横縦(wide2long)変換を行う基本的な方法をまとめる。横縦変換というよりもピボット、アンピボットのほうが馴染みが深い人も多いかもしれない。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`string_agg`, `split_part`, `group_concat`, `substring_index`

## :pencil2: Example

まずは縦横変換(long2wide)から行う。縦横変換を SQL で行う場合は`case`で集計したい値を別の列に取り出してそれを集計する方法が一般的。ここでは、`emp`テーブルの`job`ごとに各`deptno`で何人の社員が在籍しているかを集計する。SQL を見ればわかるが、展開したい列の数分、`case`が必要になる。

```sql
with tmp as (
-- 重複を除外するのであればdistinctを利用
select
	job, deptno
from
	emp
order by
	job
)
select
	job,
	-- 空欄を埋めたい場合は、nullではなく0などを利用
	sum(case when deptno = 10 then 1 else null end) as deptno10,
	sum(case when deptno = 20 then 1 else null end) as deptno20,
	sum(case when deptno = 30 then 1 else null end) as deptno30
from
	tmp
group by
	job
;

    job    | deptno10 | deptno20 | deptno30
-----------+----------+----------+----------
 ANALYST   |          |        2 |
 CLERK     |        1 |        2 |        1
 MANAGER   |        1 |        1 |        1
 PRESIDENT |        1 |          |
 SALESMAN  |          |          |        4
(5 rows)
```

少し変わった縦横変換もメモしておく。`job`列ごとに`empno`を積み上げたい場合の変換。

```sql
with tmp as (
select
	job,
	empno,
	row_number() over (partition by job order by empno) as num
from
	emp
)
select
	num,
	max(case when job = 'CLERK' then empno else null end) as CLERK,
	max(case when job = 'PRESIDENT' then empno else null end) as PRESIDENT,
	max(case when job = 'MANAGER' then empno else null end) as MANAGER,
	max(case when job = 'SALESMAN' then empno else null end) as SALESMAN,
	max(case when job = 'ANALYST' then empno else null end) as ANALYST
from
	tmp
group by
	num
order by
	num asc
;

 num | clerk | president | manager | salesman | analyst
-----+-------+-----------+---------+----------+---------
   1 |  7369 |      7839 |    7566 |     7499 |    7788
   2 |  7876 |           |    7698 |     7521 |    7902
   3 |  7900 |           |    7782 |     7654 |
   4 |  7934 |           |         |     7844 |
(4 rows)
```

結果のテーブルに`null`がはいって凸凹なので直感的に変な感じがするが、下記のような前処理したテーブルを集計している。

```sql
with tmp as (
select
	job,
	empno,
	row_number() over (partition by job order by empno) as num
from
	emp
)
select
	num,
	case when job = 'CLERK' then empno else null end as CLERK,
	case when job = 'PRESIDENT' then empno else null end as PRESIDENT,
	case when job = 'MANAGER' then empno else null end as MANAGER,
	case when job = 'SALESMAN' then empno else null end as SALESMAN,
	case when job = 'ANALYST' then empno else null end as ANALYST
from
	tmp
order by
	num asc
;

 num | clerk | president | manager | salesman | analyst
-----+-------+-----------+---------+----------+---------
   1 |       |      7839 |         |          |
   1 |       |           |    7566 |          |
   1 |       |           |         |          |    7788
   1 |  7369 |           |         |          |
   1 |       |           |         |     7499 |
   2 |       |           |    7698 |          |
   2 |       |           |         |          |    7902
   2 |  7876 |           |         |          |
   2 |       |           |         |     7521 |
   3 |  7900 |           |         |          |
   3 |       |           |         |     7654 |
   3 |       |           |    7782 |          |
   4 |  7934 |           |         |          |
   4 |       |           |         |     7844 |
(14 rows)
```

次は横縦変換(wide2long)を行う。横長のテーブルが必要なので、先程のテーブルを利用する。勿論、集計前の値を集計済みテーブルから復元することは基本的にできない。

```sql
with tmp as (
-- 重複を除外するのであればdistinctを利用
select
	job, deptno
from
	emp
order by
	job
), tmp2 as (
select
	job,
	-- 空欄を埋めたい場合は、nullではなく0などを利用
	sum(case when deptno = 10 then 1 else null end) as deptno10,
	sum(case when deptno = 20 then 1 else null end) as deptno20,
	sum(case when deptno = 30 then 1 else null end) as deptno30
from
	tmp
group by
	job
)
select job, deptno10 as deptno from tmp2 where deptno10 is not null
union all
select job, deptno20 as deptno from tmp2 where deptno20 is not null
union all
select job, deptno30 as deptno from tmp2 where deptno30 is not null
order by job;

    job    | deptno
-----------+--------
 ANALYST   |      2
 CLERK     |      1
 CLERK     |      2
 CLERK     |      1
 MANAGER   |      1
 MANAGER   |      1
 MANAGER   |      1
 PRESIDENT |      1
 SALESMAN  |      4
(9 rows)
```

## :closed_book: Reference

- [SQL Cookbook](https://www.oreilly.com/library/view/sql-cookbook/0596009763/)
- [SQL クックブック 第 2 版](https://www.oreilly.co.jp/books/9784873119779/)
