## :memo: Overview

`inner join`の代わりに`where`でも同じように機能させることができる。ここでは`where`結合と呼ぶが、データベースのことを詳しく知らないので、歴史的な経緯や`where`結合が実装された理由などもわからないが、`where`結合を使うよりは`inner join`を使うほうが好ましいと個人的には思う。

分析計の SQL は長くなることが多いし、複数のテーブルを、多くのキーを使って結合することが多々ある。テーブルを結合する際に、`left join`、`inner join`、`anti join`、`full join`を組み合わせて利用することが多く、「テーブルを紐付けれる行為」はクエリの中で`join`句でまとめて管理して、可読性を上げたい。個人的には`with`句を多用するが、サブクエリを多用する場合、`from`句の中に長いクエリが書かれていて、その下に`where`句で`where`結合するのは読み難い(個人的なレベルに依存するが…)。要するに、「テーブルを紐付けれる行為」は`join`句でまとめて管理したい。

`where`結合が処理速度的に優れているのか検証してないので、わからないが、処理速度が犠牲になるかも(実際は直積を作るのでどうなんだろうか)。今の時代、データベースのスペックを上げることでなんとかなりやすいし、誤集計するならば遅い方がましだとと思う。アプリケーションのバックエンドのエンジニアからすれば、遅いクエリは困る、ということもあるだろうけど、こと分析ではこう考える。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`where`, `inner join`

## :pencil2: Example

`join`句がないのに`from`句にテーブルを複数記載して、`where`句で条件をつけることで`where`結合を行う。とりあえず、`from`句にテーブルを複数記述して、`where`句で条件はつけない場合、どのような集計結果が返るのか確認する。

```sql
select
    e.empno,
        e.ename,
        e.deptno as e_deptno,
        d.deptno as d_deptno,
        d.loc
from
    emp as e,
        dept as d
order by
    e.empno
;
 empno | ename  | e_deptno | d_deptno |   loc
-------+--------+----------+----------+----------
  7369 | SMITH  |       20 |       30 | CHICAGO
  7369 | SMITH  |       20 |       20 | DALLAS
  7369 | SMITH  |       20 |       40 | BOSTON
  7369 | SMITH  |       20 |       10 | NEW YORK
  7499 | ALLEN  |       30 |       40 | BOSTON
  7499 | ALLEN  |       30 |       30 | CHICAGO
  7499 | ALLEN  |       30 |       20 | DALLAS
  7499 | ALLEN  |       30 |       10 | NEW YORK
  7521 | WARD   |       30 |       30 | CHICAGO
  7521 | WARD   |       30 |       10 | NEW YORK
  7521 | WARD   |       30 |       20 | DALLAS
  7521 | WARD   |       30 |       40 | BOSTON
  7566 | JONES  |       20 |       10 | NEW YORK
  7566 | JONES  |       20 |       40 | BOSTON
  7566 | JONES  |       20 |       20 | DALLAS
  7566 | JONES  |       20 |       30 | CHICAGO
  7654 | MARTIN |       30 |       40 | BOSTON
  7654 | MARTIN |       30 |       20 | DALLAS
  7654 | MARTIN |       30 |       30 | CHICAGO
  7654 | MARTIN |       30 |       10 | NEW YORK
  7698 | BLAKE  |       30 |       30 | CHICAGO
  7698 | BLAKE  |       30 |       10 | NEW YORK
  7698 | BLAKE  |       30 |       40 | BOSTON
  7698 | BLAKE  |       30 |       20 | DALLAS
  7782 | CLARK  |       10 |       30 | CHICAGO
  7782 | CLARK  |       10 |       40 | BOSTON
  7782 | CLARK  |       10 |       20 | DALLAS
  7782 | CLARK  |       10 |       10 | NEW YORK
  7788 | SCOTT  |       20 |       30 | CHICAGO
  7788 | SCOTT  |       20 |       10 | NEW YORK
  7788 | SCOTT  |       20 |       20 | DALLAS
  7788 | SCOTT  |       20 |       40 | BOSTON
  7839 | KING   |       10 |       30 | CHICAGO
  7839 | KING   |       10 |       40 | BOSTON
  7839 | KING   |       10 |       10 | NEW YORK
  7839 | KING   |       10 |       20 | DALLAS
  7844 | TURNER |       30 |       10 | NEW YORK
  7844 | TURNER |       30 |       40 | BOSTON
  7844 | TURNER |       30 |       20 | DALLAS
  7844 | TURNER |       30 |       30 | CHICAGO
  7876 | ADAMS  |       20 |       30 | CHICAGO
  7876 | ADAMS  |       20 |       40 | BOSTON
  7876 | ADAMS  |       20 |       20 | DALLAS
  7876 | ADAMS  |       20 |       10 | NEW YORK
  7900 | JAMES  |       30 |       20 | DALLAS
  7900 | JAMES  |       30 |       40 | BOSTON
  7900 | JAMES  |       30 |       30 | CHICAGO
  7900 | JAMES  |       30 |       10 | NEW YORK
  7902 | FORD   |       20 |       20 | DALLAS
  7902 | FORD   |       20 |       30 | CHICAGO
  7902 | FORD   |       20 |       10 | NEW YORK
  7902 | FORD   |       20 |       40 | BOSTON
  7934 | MILLER |       10 |       10 | NEW YORK
  7934 | MILLER |       10 |       30 | CHICAGO
  7934 | MILLER |       10 |       20 | DALLAS
  7934 | MILLER |       10 |       40 | BOSTON
(56 rows)
```

結果を見ると、`from`句にテーブルを複数記述すると直積を計算していることがわかる。つまり、`emp`は 14 行、`dept`は 4 行なので、合計 56 行が返される。テーブルは 3 つ以上も記載できるけど、実データーベースの 3 テーブルの直積は、サイズが比較的小さいかもしれないマスタテーブルであっても億劫である。このテーブルに対して、`where`句で条件をつけたものを取り出すことになる。(直積を計算するので、サイズが大きいデータにはどうなんだろうか…)

まず、`deptno`で条件をつけてみると、`deptno`をキーにした`join`のような結果が返ってくる。

```sql
select
    e.empno,
    e.ename,
    e.deptno as e_deptno,
    d.deptno as d_deptno,
    d.loc
from
    emp as e,
        dept as d
where
    e.deptno = d.deptno
order by
    e.deptno
;
 empno | ename  | e_deptno | d_deptno |   loc
-------+--------+----------+----------+----------
  7934 | MILLER |       10 |       10 | NEW YORK
  7782 | CLARK  |       10 |       10 | NEW YORK
  7839 | KING   |       10 |       10 | NEW YORK
  7788 | SCOTT  |       20 |       20 | DALLAS
  7566 | JONES  |       20 |       20 | DALLAS
  7369 | SMITH  |       20 |       20 | DALLAS
  7876 | ADAMS  |       20 |       20 | DALLAS
  7902 | FORD   |       20 |       20 | DALLAS
  7521 | WARD   |       30 |       30 | CHICAGO
  7844 | TURNER |       30 |       30 | CHICAGO
  7499 | ALLEN  |       30 |       30 | CHICAGO
  7698 | BLAKE  |       30 |       30 | CHICAGO
  7654 | MARTIN |       30 |       30 | CHICAGO
  7900 | JAMES  |       30 |       30 | CHICAGO
(14 rows)
```

さらに`deptno`に特定の値をつけて条件をつけてみると、

```sql
select
    e.empno,
    e.ename,
    e.deptno as e_deptno,
    d.deptno as d_deptno,
    d.loc
from
    emp as e,
        dept10 as d
where
    e.deptno = d.deptno and
    e.deptno = 10
order by
    e.deptno
;
 empno | ename  | e_deptno | d_deptno |   loc
-------+--------+----------+----------+----------
  7782 | CLARK  |       10 |       10 | NEW YORK
  7839 | KING   |       10 |       10 | NEW YORK
  7934 | MILLER |       10 |       10 | NEW YORK
(3 rows)
```

同じ結果は`inner join`でも得られる。このテーブルであれば、`left join`でも同じ。

```sql
select
    e.empno,
	e.ename,
	e.deptno as e_deptno,
	d.deptno as d_deptno,
	d.loc
from
    emp as e
inner join
    dept as d
on
    e.deptno = d.deptno
where
    e.deptno = 10
order by
    e.deptno
;

 empno | ename  | e_deptno | d_deptno |   loc
-------+--------+----------+----------+----------
  7782 | CLARK  |       10 |       10 | NEW YORK
  7839 | KING   |       10 |       10 | NEW YORK
  7934 | MILLER |       10 |       10 | NEW YORK
(3 rows)
```

`where`結合でいくつか実験するために、`dept=10`しかないテーブル、`dept=10`が重複しているテーブルを用意する。

```sql
CREATE TABLE DEPT10 (DEPTNO integer, DNAME VARCHAR(14), LOC VARCHAR(13));
INSERT INTO DEPT10 VALUES (10, 'ACCOUNTING', 'NEW YORK');

CREATE TABLE DEPT_DUPLICATION (DEPTNO integer, JOB VARCHAR(14));
INSERT INTO DEPT_DUPLICATION VALUES (10, 'PRESIDENT');
INSERT INTO DEPT_DUPLICATION VALUES (10, 'MANAGER');
INSERT INTO DEPT_DUPLICATION VALUES (10, 'CLERK');

```

`where`結合が`inner join`と等価なのか確認するために、`dept=10`しかないテーブルを紐付ける。`emp`の`dept=10`以外のレコードは返されない。

```sql
select
    e.empno,
	e.ename,
	e.deptno as e_deptno,
	d.deptno as d_deptno,
	d.loc
from
    emp as e,
	dept10 as d
where
    e.deptno = d.deptno
order by
    e.deptno
;

 empno | ename  | e_deptno | d_deptno |   loc
-------+--------+----------+----------+----------
  7934 | MILLER |       10 |       10 | NEW YORK
  7782 | CLARK  |       10 |       10 | NEW YORK
  7839 | KING   |       10 |       10 | NEW YORK
(3 rows)

```

さらに`where`結合が`inner join`と等価なのか確認するために、`dept=10`が重複しているテーブルを紐付ける。`emp`の`dept=10`は 3 レコードなので、３レコード重複しているテーブルを紐付けるので、結果は 9 行返される。

```sql
select
    e.empno,
	e.ename,
	e.deptno as e_deptno,
	d.deptno as d_deptno,
	d.job
from
    emp as e,
	dept_duplication as d
where
    e.deptno = d.deptno
order by
    e.deptno
;

 empno | ename  | e_deptno | d_deptno |    job
-------+--------+----------+----------+-----------
  7934 | MILLER |       10 |       10 | PRESIDENT
  7934 | MILLER |       10 |       10 | MANAGER
  7934 | MILLER |       10 |       10 | CLERK
  7782 | CLARK  |       10 |       10 | PRESIDENT
  7782 | CLARK  |       10 |       10 | MANAGER
  7782 | CLARK  |       10 |       10 | CLERK
  7839 | KING   |       10 |       10 | PRESIDENT
  7839 | KING   |       10 |       10 | MANAGER
  7839 | KING   |       10 |       10 | CLERK
  (9 rows)
```

## :closed_book: Reference

- [SQL Cookbook](https://www.oreilly.com/library/view/sql-cookbook/0596009763/)
- [SQL クックブック 第 2 版](https://www.oreilly.co.jp/books/9784873119779/)
