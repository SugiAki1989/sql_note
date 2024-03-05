## :memo: Overview

差集合を計算する`except`を利用することで、テーブルの比較が簡単に実行できる。2 つのテーブルがあった場合に、重複している行、片方にしか存在していないデータを確認することができる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`except`, `union all`

## :pencil2: Example

まずは`except`の動きを確認するために、簡単な検証テーブルを作成する。`X`は`A~E`が含まれており、`Y`は`A`が重複して含まれ、`E`が含まれていないテーブル。

```sql
CREATE TABLE X (id VARCHAR(5));
INSERT INTO X VALUES ('A');
INSERT INTO X VALUES ('B');
INSERT INTO X VALUES ('C');
INSERT INTO X VALUES ('D');
INSERT INTO X VALUES ('E');

CREATE TABLE Y (id VARCHAR(5));
INSERT INTO Y VALUES ('A');
INSERT INTO Y VALUES ('A');
INSERT INTO Y VALUES ('B');
INSERT INTO Y VALUES ('C');
INSERT INTO Y VALUES ('D');
```

下記の通り記述すると、`x`の値と`y`の値を比較して、`y`に含まれる値は除外してくれる。`y`の値を使って、`x`の値を取り除くようなイメージ。
`x`は`A~E`を持っているので、`y`の`A~D`で`x`の値を除外すると`x`の`E`が残る。

```sql
select id from x
except
select id from y;

 id
----
 E
```

テーブルを入れ替えると、`y`の値については、`x`の値を使って取り除くと、全てなくなるので、空が返される。

```sql
select id from y
except
select id from x;

 id
----
(0 rows)
```

もう少し複雑なテーブルでを使って挙動を確認する。

```sql
create table xx (
  id int not null
  , name varchar(50)
  , dt int
);

INSERT INTO xx VALUES (1, 'A', '2024');
INSERT INTO xx VALUES (1, 'A', '2025');
INSERT INTO xx VALUES (2, 'B', '2024');
INSERT INTO xx VALUES (2, 'B', '2025');
INSERT INTO xx VALUES (2, 'B', '2026');
INSERT INTO xx VALUES (3, 'C', '2024');
INSERT INTO xx VALUES (4, 'D', '2025');
INSERT INTO xx VALUES (5, 'E', '2026');
INSERT INTO xx VALUES (5, 'E', '2026');

create table yy (
  id int not null
  , name varchar(50)
  , dt int
);

INSERT INTO yy VALUES (1, 'A', '2024');
INSERT INTO yy VALUES (1, 'A', '2025');
INSERT INTO yy VALUES (2, 'B', '2024');
INSERT INTO yy VALUES (2, 'B', '2026');
INSERT INTO yy VALUES (3, 'C', '2024');

select * from xx;
 id | name |  dt
----+------+------
  1 | A    | 2024
  1 | A    | 2025
  2 | B    | 2024
  2 | B    | 2025
  2 | B    | 2026
  3 | C    | 2024
  4 | D    | 2025
  5 | E    | 2026
  5 | E    | 2026
(9 rows)

select * from yy;
 id | name |  dt
----+------+------
  1 | A    | 2024
  1 | A    | 2025
  2 | B    | 2024
  2 | B    | 2026
  3 | C    | 2024
(5 rows)
```

選択するカラムを2個にすると、2カラムを考慮して、重複するレコードを返す。

```sql
select id, name from xx
intersect
select id, name from yy
;

 id | name
----+------
  1 | A
  2 | B
  3 | C
(3 rows)

select id, name, dt from xx
intersect all
select id, name, dt from yy
;
 id | name |  dt
----+------+------
  1 | A    | 2025
  1 | A    | 2024
  2 | B    | 2024
  2 | B    | 2026
  3 | C    | 2024
(5 rows)
```

`intersect`と`intersect all`の挙動は下記を見ると分かりやすい。`intersect`の場合は2カラムで重複している`[1,2],[3,5],[6]`行目が対象となり、重複しているレコードは除外されて3行が返される。`intersect all`だと、`[1,2],[3,5],[6]`行目が対象となり、そのまま返される。

```
id | name |  dt
----+------+------ -> yy table
  1 | A    | 2024  -> | 1 | A | 2024 | 
  1 | A    | 2025  -> | 1 | A | 2025 | 
  2 | B    | 2024  -> | 2 | B | 2024 | 
  2 | B    | 2025  
  2 | B    | 2026  -> | 2 | B | 2026 | 
  3 | C    | 2024  -> | 3 | C | 2024 | 
  4 | D    | 2025
  5 | E    | 2026
  5 | E    | 2026
```

今回のデータでは、3カラムにしても重複を除けないので、`intersect`と`intersect all`の結果は同じである。

```sql
select id, name, dt from xx
intersect
select id, name, dt from yy
;

 id | name |  dt
----+------+------
  1 | A    | 2024
  1 | A    | 2025
  2 | B    | 2024
  2 | B    | 2026
  3 | C    | 2024
(5 rows)

select id, name, dt from xx
intersect all
select id, name, dt from yy
;
 id | name |  dt
----+------+------
  1 | A    | 2024
  1 | A    | 2025
  2 | B    | 2024
  2 | B    | 2026
  3 | C    | 2024
(5 rows)
```

`except`で選択するカラムを2個にすると、2カラムを考慮して、1つめのテーブルにだけ存在するデータを取得する。`except all`の結果には注意が必要。`id=2 | name=B`が1行返されている。これは、`except`が重複を削除する一方で、`except all`は重複を削除しない。1つ目のテーブルに`id=2 | name=B`が3行、2つ目のテーブルに`id=2 | name=B`が2行しかないので、1行返される。

```sql
select id, name from xx
except
select id, name from yy
;

 id | name
----+------
  4 | D
  5 | E
(2 rows)

select id, name from xx
except all
select id, name from yy
;
  id | name
----+------
  2 | B
  4 | D
  5 | E
  5 | E
(4 rows)
```

下記を見ると分かりやすい。

```
id | name |  dt
----+------+------ -> yy table
  1 | A    | 2024  -> | 1 | A | 2024 | 
  1 | A    | 2025  -> | 1 | A | 2025 | 
  2 | B    | 2024  -> | 2 | B | 2024 | 
  2 | B    | 2025  
  2 | B    | 2026  -> | 2 | B | 2026 | 
  3 | C    | 2024  -> | 3 | C | 2024 | 
  4 | D    | 2025
  5 | E    | 2026
  5 | E    | 2026
```

参考までに3カラム考慮した場合の結果を載せておく。

```sql
select id, name, dt from xx
except
select id, name, dt from yy
;
 id | name |  dt
----+------+------
  2 | B    | 2025
  4 | D    | 2025
  5 | E    | 2026
(3 rows)

select id, name, dt from xx
except all
select id, name, dt from yy
;
 id | name |  dt
----+------+------
  2 | B    | 2025
  4 | D    | 2025
  5 | E    | 2026
  5 | E    | 2026
(4 rows)
```

`except`と`group by`を組み合わせることで、重複している行、片方にしか存在していないデータを確認することができる。

```sql
(
select id, count(*) as cnt
from x
group by id
except
select id, count(*) as cnt
from y
group by id
)
union all
(
select id, count(*) as cnt
from y
group by id
except
select id, count(*) as cnt
from x
group by id
);

 id | cnt
----+-----
 A  |   1
 A  |   2
 E  |   1
(3 rows)
```

`union all`を基準に前後に分けて考えると、1 つ目のブロックでは、`x`を`y`の値で取り除くので`B`、`C`、`D`が除外されて、`A`、`E`が返される。`y`の`A`は`cnt`の数が異なるので、`x`の`A`が取り除かれることにはならない。

```sql
(
select id, count(*) as cnt
from x
group by id
--  id | cnt
-- ----+-----
--  A  |   1
--  B  |   1
--  C  |   1
--  D  |   1
--  E  |   1
except
select id, count(*) as cnt
from y
group by id
--  id | cnt
-- ----+-----
--  A  |   2
--  B  |   1
--  C  |   1
--  D  |   1
)
--  id | cnt
-- ----+-----
--  A  |   1
--  E  |   1
```

2 つ目のブロックでは、`y`を`x`の値で取り除くので`B`、`C`、`D`、が除外されて、`A`は`cnt＝2`が残る。

```sql
union all
(
select id, count(*) as cnt
from y
group by id
--  id | cnt
-- ----+-----
--  A  |   2
--  B  |   1
--  C  |   1
--  D  |   1
except
select id, count(*) as cnt
from x
group by id
--  id | cnt
-- ----+-----
--  B  |   1
--  C  |   1
--  D  |   1
--  E  |   1
--  A  |   1
);

```

この結果、テーブル`x`と`y`の違いは、`A`が 2 行重複しており、片方には`E`が含まれていないことがわかる。

```
 id | cnt
----+-----
 A  |   1
 A  |   2
 E  |   1
(3 rows)
```

[SQL クックブック 第 2 版](https://www.oreilly.co.jp/books/9784873119779/)に記載されている例では、より複雑な例が記載されている。

```sql
create view v
as
select * from emp where deptno != 10
union all
select * from emp where ename = 'WARD'
;

select * from v;
 empno | ename  |   job    | mgr  |  hiredate  | sal  | comm | deptno
-------+--------+----------+------+------------+------+------+--------
  7369 | SMITH  | CLERK    | 7902 | 1980-12-17 |  800 |      |     20
  7499 | ALLEN  | SALESMAN | 7698 | 1981-02-20 | 1600 |  300 |     30
  7521 | WARD   | SALESMAN | 7698 | 1981-02-22 | 1250 |  500 |     30
  7566 | JONES  | MANAGER  | 7839 | 1981-04-02 | 2975 |      |     20
  7654 | MARTIN | SALESMAN | 7698 | 1981-09-28 | 1250 | 1400 |     30
  7698 | BLAKE  | MANAGER  | 7839 | 1981-05-01 | 2850 |      |     30
  7788 | SCOTT  | ANALYST  | 7566 | 1982-12-09 | 3000 |      |     20
  7844 | TURNER | SALESMAN | 7698 | 1981-09-08 | 1500 |    0 |     30
  7876 | ADAMS  | CLERK    | 7788 | 1983-01-12 | 1100 |      |     20
  7900 | JAMES  | CLERK    | 7698 | 1981-12-03 |  950 |      |     30
  7902 | FORD   | ANALYST  | 7566 | 1981-12-03 | 3000 |      |     20
  7521 | WARD   | SALESMAN | 7698 | 1981-02-22 | 1250 |  500 |     30
(12 rows)

(
select
	empno, ename, job, mgr, hiredate, sal, comm, deptno,
	count(*) as cnt
from
	v
group by
	empno, ename, job, mgr, hiredate, sal, comm, deptno
except
select
	empno, ename, job, mgr, hiredate, sal, comm, deptno,
	count(*) as cnt
from
	emp
group by
	empno, ename, job, mgr, hiredate, sal, comm, deptno
)
union all
(
select
	empno, ename, job, mgr, hiredate, sal, comm, deptno,
	count(*) as cnt
from
	emp
group by
	empno, ename, job, mgr, hiredate, sal, comm, deptno
except
select
	empno, ename, job, mgr, hiredate, sal, comm, deptno,
	count(*) as cnt
from
	v
group by
	empno, ename, job, mgr, hiredate, sal, comm, deptno
)
;

 empno | ename  |    job    | mgr  |  hiredate  | sal  | comm | deptno | cnt
-------+--------+-----------+------+------------+------+------+--------+-----
  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250 |  500 |     30 |   1
  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250 |  500 |     30 |   2
  7934 | MILLER | CLERK     | 7782 | 1982-01-23 | 1300 |      |     10 |   1
  7839 | KING   | PRESIDENT |      | 1981-11-17 | 5000 |      |     10 |   1
  7782 | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450 |      |     10 |   1
(5 rows)
```

## :closed_book: Reference

- [SQL Cookbook](https://www.oreilly.com/library/view/sql-cookbook/0596009763/)
- [SQL クックブック 第 2 版](https://www.oreilly.co.jp/books/9784873119779/)
