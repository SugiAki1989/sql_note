## :memo: Overview

`filter`句という便利なものが存在しているようなので、使い方を簡単にメモしておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`filter`, `case`

## :pencil2: Example

`filter`句を利用すると、条件に合うデータのみを対象に集計できる。イメージを付けるために簡単な例を記載しておく。ウインドウ関数のように記述するが、`over`の代わりに`filter(where condition)`と記述する。

```sql
select
    count(*),
    count(*) filter (where ind < 5) as filtered
from
    generate_series(1, 10) as ind
;
 count | filtered
-------+----------
    10 |        4
(1 row)
```

列の特定の行のみを集計したい場合、`case`文で記述すれば対応できる。

```sql
select
    job,
    count(1) as cnt,
    count(case when job = 'SALESMAN' then job else null end) as salesman,
    count(case when job = 'ANALYST' then job else null end) as analyst,
    count(case when job = 'PRESIDENT' then job else null end) as president
from
    emp
group by
    job
;


    job    | cnt | salesman | analyst | president
-----------+-----+----------+---------+-----------
 SALESMAN  |   4 |        4 |       0 |         0
 MANAGER   |   3 |        0 |       0 |         0
 ANALYST   |   2 |        0 |       2 |         0
 CLERK     |   4 |        0 |       0 |         0
 PRESIDENT |   1 |        0 |       0 |         1
(5 rows)
```

この`case`文は`filter`で書き換えが可能。

```sql
select
    job,
    count(1) as cnt,
    count(job) filter (where job = 'SALESMAN') as salesman,
    count(job) filter (where job = 'ANALYST') as analyst,
    count(job) filter (where job = 'PRESIDENT') as president
from
    emp
group by
    job
;

    job    | cnt | salesman | analyst | president
-----------+-----+----------+---------+-----------
 SALESMAN  |   4 |        4 |       0 |         0
 MANAGER   |   3 |        0 |       0 |         0
 ANALYST   |   2 |        0 |       2 |         0
 CLERK     |   4 |        0 |       0 |         0
 PRESIDENT |   1 |        0 |       0 |         1
(5 rows)
```

本当は実行計画とか内部実装とかもろもろの観点から調査する必要があるんだろうけど、そこまで能力がないので簡単に速度比較だけしておく。速度的にはそこまで変わらないみたい。

```sql

select setseed(0.1989);
create materialized view alphabets as (
select (array['A', 'B', 'C', 'D', 'E'])[ceil(random() * 5)] as alphabet
from generate_series(1, 50000000)
);

\timing

select
    alphabet,
    count(case when alphabet = 'A' then alphabet else null end) as Acount,
    count(case when alphabet = 'B' then alphabet else null end) as Bcount,
    count(case when alphabet = 'C' then alphabet else null end) as Ccount,
    count(case when alphabet = 'D' then alphabet else null end) as Dcount,
    count(case when alphabet = 'E' then alphabet else null end) as Ecount
from alphabets
group by alphabet;

 alphabet |  acount  | bcount  |  ccount  | dcount  |  ecount
----------+----------+---------+----------+---------+----------
 A        | 10003534 |       0 |        0 |       0 |        0
 B        |        0 | 9998156 |        0 |       0 |        0
 C        |        0 |       0 | 10000577 |       0 |        0
 D        |        0 |       0 |        0 | 9997619 |        0
 E        |        0 |       0 |        0 |       0 | 10000114
(5 rows)

Time: 4990.373 ms (00:04.990)


select
    alphabet,
    count(alphabet) filter (where alphabet = 'A') as Acount,
    count(alphabet) filter (where alphabet = 'B') as Bcount,
    count(alphabet) filter (where alphabet = 'C') as Ccount,
    count(alphabet) filter (where alphabet = 'D') as Dcount,
    count(alphabet) filter (where alphabet = 'E') as Ecount
from alphabets
group by alphabet;

 alphabet |  acount  | bcount  |  ccount  | dcount  |  ecount
----------+----------+---------+----------+---------+----------
 A        | 10003534 |       0 |        0 |       0 |        0
 B        |        0 | 9998156 |        0 |       0 |        0
 C        |        0 |       0 | 10000577 |       0 |        0
 D        |        0 |       0 |        0 | 9997619 |        0
 E        |        0 |       0 |        0 |       0 | 10000114
(5 rows)

Time: 4729.087 ms (00:04.729)

```

## :closed_book: Reference

- [4.2.7. Aggregate Expressions](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-AGGREGATES:~:text=4.2.7.%20%E9%9B%86%E8%A8%88-,%E5%BC%8F,-%E9%9B%86%E8%A8%88%E5%BC%8F%E3%81%AF)
