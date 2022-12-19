## :memo: Overview

前回に引き続き、そんな使い方あったのかシリーズ。ここでは`distinct on`について。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`distinct on`

## :pencil2: Example

ドキュメントによると、`distinct on`は、

> SELECT DISTINCT ON ( expression [, …] ) keeps only the first row of each set of rows where the given expressions evaluate to equal
> (拙訳)`SELECT DISTINCT ON` ( expression [, …] ) は、指定された式が等しいと評価される行の各セットの最初の行のみを保持します

と書かれている。例えば、`emp`テーブルの各`job`ごとで 1 番給料をもらっている人を 1 名づつ選出したいとする。

```sql

select empno, ename, job, sal
from emp
where (job, sal) in (
    select job, max(sal) as max_sal
    from emp
    group by job
)
order by job;

 empno | ename  |    job    | sal
-------+--------+-----------+------
  7788 | SCOTT  | ANALYST   | 3000
  7902 | FORD   | ANALYST   | 3000
  7934 | MILLER | CLERK     | 1300
  7566 | JONES  | MANAGER   | 2975
  7839 | KING   | PRESIDENT | 5000
  7499 | ALLEN  | SALESMAN  | 1600
(6 rows)
```

`ANALYST`は`sal`が同じなので、2 名選出されてしまう。ここではどちらもよいので、連番を降って条件付けることで対応することもできる。

```sql
select empno, ename, job, sal
from (
    select empno, ename, job, sal, row_number() over (partition by job order by empno asc) as rowind
    from emp
    where (job, sal) in (
        select job, max(sal) as max_sal
        from emp
        group by job
)) as s
where rowind = 1
;
 empno | ename  |    job    | sal
-------+--------+-----------+------
  7788 | SCOTT  | ANALYST   | 3000
  7934 | MILLER | CLERK     | 1300
  7566 | JONES  | MANAGER   | 2975
  7839 | KING   | PRESIDENT | 5000
  7499 | ALLEN  | SALESMAN  | 1600
(5 rows)
```

このようなケースで、`distinct on`を使えば、「指定された式が等しいと評価される行の各セットの最初の行のみを保持」してくれるので、目的の結果が得られる。

```sql
select distinct on (job) empno, ename, job, sal
from emp
order by job;

 empno | ename  |    job    | sal
-------+--------+-----------+------
  7788 | SCOTT  | ANALYST   | 3000
  7934 | MILLER | CLERK     | 1300
  7782 | CLARK  | MANAGER   | 2450
  7839 | KING   | PRESIDENT | 5000
  7844 | TURNER | SALESMAN  | 1500
(5 rows)
```

使う機会はないかもしれないが、この性質を利用すれば、`oeder`で並び替えれば、最大、最小のユニークな値が取得できる。

```sql
select distinct on (job) job, sal
from emp
order by job, sal asc;
    job    | sal
-----------+------
 ANALYST   | 3000
 CLERK     |  800
 MANAGER   | 2450
 PRESIDENT | 5000
 SALESMAN  | 1250
(5 rows)


select distinct on (job) job, sal
from emp
order by job, sal desc;
    job    | sal
-----------+------
 ANALYST   | 3000
 CLERK     | 1300
 MANAGER   | 2975
 PRESIDENT | 5000
 SALESMAN  | 1600
(5 rows)
```

## :closed_book: Reference

- [DISTINCT Clause](https://www.postgresql.org/docs/current/sql-select.html#SQL-DISTINCT:~:text=chosen%20query%20plan.-,DISTINCT%20Clause,-If%20SELECT%20DISTINCT)
