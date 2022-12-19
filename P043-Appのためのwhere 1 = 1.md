## :memo: Overview

`where 1 = 1`という記述を`where`句にたまに見かけるかもしれないが、この役割についてまとめておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`where`

## :pencil2: Example

`select *`をする結果を最初に示しておく。これはテーブル全部を抽出することになる。

```sql
select
    *
from
    emp
;

 empno | ename  |    job    | mgr  |  hiredate  | sal  | comm | deptno
-------+--------+-----------+------+------------+------+------+--------
  7369 | SMITH  | CLERK     | 7902 | 1980-12-17 |  800 |      |     20
  7499 | ALLEN  | SALESMAN  | 7698 | 1981-02-20 | 1600 |  300 |     30
  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250 |  500 |     30
  7566 | JONES  | MANAGER   | 7839 | 1981-04-02 | 2975 |      |     20
  7654 | MARTIN | SALESMAN  | 7698 | 1981-09-28 | 1250 | 1400 |     30
  7698 | BLAKE  | MANAGER   | 7839 | 1981-05-01 | 2850 |      |     30
  7782 | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450 |      |     10
  7788 | SCOTT  | ANALYST   | 7566 | 1982-12-09 | 3000 |      |     20
  7839 | KING   | PRESIDENT |      | 1981-11-17 | 5000 |      |     10
  7844 | TURNER | SALESMAN  | 7698 | 1981-09-08 | 1500 |    0 |     30
  7876 | ADAMS  | CLERK     | 7788 | 1983-01-12 | 1100 |      |     20
  7900 | JAMES  | CLERK     | 7698 | 1981-12-03 |  950 |      |     30
  7902 | FORD   | ANALYST   | 7566 | 1981-12-03 | 3000 |      |     20
  7934 | MILLER | CLERK     | 7782 | 1982-01-23 | 1300 |      |     10
(14 rows)
```

`where 1 = 1`をつけた結果と最初の結果は何も変わらない。`where 1 = 1`は全レコードで`true`を返すことになるので、なんの条件もついていない場合と何も変わらない。

```sql
select
    *
from
    emp
where
    1 = 1
;

 empno | ename  |    job    | mgr  |  hiredate  | sal  | comm | deptno
-------+--------+-----------+------+------------+------+------+--------
  7369 | SMITH  | CLERK     | 7902 | 1980-12-17 |  800 |      |     20
  7499 | ALLEN  | SALESMAN  | 7698 | 1981-02-20 | 1600 |  300 |     30
  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250 |  500 |     30
  7566 | JONES  | MANAGER   | 7839 | 1981-04-02 | 2975 |      |     20
  7654 | MARTIN | SALESMAN  | 7698 | 1981-09-28 | 1250 | 1400 |     30
  7698 | BLAKE  | MANAGER   | 7839 | 1981-05-01 | 2850 |      |     30
  7782 | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450 |      |     10
  7788 | SCOTT  | ANALYST   | 7566 | 1982-12-09 | 3000 |      |     20
  7839 | KING   | PRESIDENT |      | 1981-11-17 | 5000 |      |     10
  7844 | TURNER | SALESMAN  | 7698 | 1981-09-08 | 1500 |    0 |     30
  7876 | ADAMS  | CLERK     | 7788 | 1983-01-12 | 1100 |      |     20
  7900 | JAMES  | CLERK     | 7698 | 1981-12-03 |  950 |      |     30
  7902 | FORD   | ANALYST   | 7566 | 1981-12-03 | 3000 |      |     20
  7934 | MILLER | CLERK     | 7782 | 1982-01-23 | 1300 |      |     10
(14 rows)
```

これはデータベースを呼び出すアプリケーションのプログラム側の都合で、`where`句をつけた SQL の条件追加が容易にできるようにするため。なんの条件もついていないと、条件を付ける場合 SQL に`where`を追加する必要がある。

```sql
-- SQL
select col from table;
↓
where col = 'a'
↓
-- SQL
select col from table where col = 'a';
↓
and col = 'b'
↓
-- SQL
select col from table where col = 'a' and col = 'b';
```

このように追加するプログラムが条件を付ける場合、追加する場合で文字列がことなって面倒が増える。これを回避するために、`where 1 = 1`をつける。そうすることで、条件を付ける場合、条件を追加する場合で同じ文字列を追加すれば SQL を組み立てることができる。

```sql
-- SQL
select col from table where 1 = 1;
↓
and col = 'a'
↓
-- SQL
select col from table where 1 = 1 and col = 'a';
↓
and col = 'b'
↓
-- SQL
select col from table where 1 = 1 and col = 'a' and col = 'b';
```

分析用の SQL ではあまり必要ないかもしれないが、アドホックな分析で試行錯誤する際にはつけていれば少し楽できるかも。

## :closed_book: Reference

None
