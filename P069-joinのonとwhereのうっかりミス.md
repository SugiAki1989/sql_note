## :memo: Overview

`lrft join`の`on`と`where`のどちらに条件を書くかで、当たり前だが返ってくる結果が異なるので、うっかりミスに注意する。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`on`, `where`

## :pencil2: Example

`on`は結合条件、`where`は抽出条件という違いがある。例えば、`emp`に紐づく`emp_score`というテーブルがある。

```sql
create table emp_score(empno int, score varchar(1), deleteflag boolean);
insert into emp_score(empno, score, deleteflag)
values
    ('7369','S', 'false'),
    ('7499','A', 'true'),
    ('9999','B', 'false');
```

これを`emp`全員に紐付けた際に、`deleteflag`は`false`だけを紐付けたい。つまり、この SQL は誤り。

```sql
select
    e.empno,
    e.ename,
    es.score,
    es.deleteflag
from
    emp as e
left join
    emp_score as es
on
    e.empno = es.empno
;

 empno | ename  | score | deleteflag
-------+--------+-------+------------
  7369 | SMITH  | S     | f
  7499 | ALLEN  | A     | t
  7839 | KING   |       |
  7698 | BLAKE  |       |
  7934 | MILLER |       |
  7844 | TURNER |       |
  7876 | ADAMS  |       |
  7654 | MARTIN |       |
  7900 | JAMES  |       |
  7521 | WARD   |       |
  7788 | SCOTT  |       |
  7782 | CLARK  |       |
  7566 | JONES  |       |
  7902 | FORD   |       |
(14 rows)
```

この SQL も誤り。

```sql
select
    e.empno,
    e.ename,
    es.score,
    es.deleteflag
from
    emp as e
left join
    emp_score as es
on
    e.empno = es.empno
where
    es.deleteflag is false
;

 empno | ename | score | deleteflag
-------+-------+-------+------------
  7369 | SMITH | S     | f
(1 row)
```

この SQL は正しく、`on`に書くことで目的の結果が得られる。うっかりしてると結果は返ってくるのでバグにつながる。

```sql
select
    e.empno,
    e.ename,
    es.score,
    es.deleteflag
from
    emp as e
left join
    emp_score as es
on
    e.empno = es.empno and
    es.deleteflag is false
;

 empno | ename  | score | deleteflag
-------+--------+-------+------------
  7369 | SMITH  | S     | f
  7839 | KING   |       |
  7499 | ALLEN  |       |
  7698 | BLAKE  |       |
  7934 | MILLER |       |
  7844 | TURNER |       |
  7876 | ADAMS  |       |
  7654 | MARTIN |       |
  7900 | JAMES  |       |
  7521 | WARD   |       |
  7788 | SCOTT  |       |
  7782 | CLARK  |       |
  7566 | JONES  |       |
  7902 | FORD   |       |
(14 rows)
```

他の例も見ておく。`dept`テーブルに対して、条件`on d.deptno = e.deptno and e.deptno = 20`で`join`し、`deptno`をすべて残した結果がほしいとする。

```sql
select
    d.deptno,
    d.dname,
    e.ename,
    e.sal
from
    dept as d
left join
    emp as e
on
    d.deptno = e.deptno and
    e.deptno = 20
;

 deptno |   dname    | ename | sal
--------+------------+-------+------
     10 | ACCOUNTING |       |
     20 | RESEARCH   | FORD  | 3000
     20 | RESEARCH   | ADAMS | 1100
     20 | RESEARCH   | SCOTT | 3000
     20 | RESEARCH   | JONES | 2975
     20 | RESEARCH   | SMITH |  800
     30 | SALES      |       |
     40 | OPERATIONS |       |
(8 rows)
```

複雑な分析系 SQL を私は一発で作れず、試行錯誤や追加修正対応などしてるうちに、うっかり下記のような SQL を書いたとする。これはわかりやすいデータと例なので目検でも対応可能レベルだけど、ビックなデータで、条件に一致しないレコードが少ない場合、何もおかしなことがあっても見逃す可能性がある。

```sql
select
    d.deptno,
    d.dname,
    e.ename,
    e.sal
from
    dept as d
left join
    emp as e
on
    d.deptno = e.deptno
where
    e.deptno = 20
;

 deptno |  dname   | ename | sal
--------+----------+-------+------
     20 | RESEARCH | SMITH |  800
     20 | RESEARCH | JONES | 2975
     20 | RESEARCH | SCOTT | 3000
     20 | RESEARCH | ADAMS | 1100
     20 | RESEARCH | FORD  | 3000
(5 rows)
```

結局、何が起こっていたのかというと、前者の条件`on d.deptno = e.deptno and e.deptno = 20`では、`on`の評価に合わせて、`e.deptno = 20`も評価された結果を`dept`テーブルに紐付ける。後者は紐付け終わったものに対して、`where`で条件づけを行うので、`deptno=20`しか存在しなくなる。

これなんかは`deptno`が取りうる値の範囲や値を知らないと、問題ないように見える。

```sql
select
    d.deptno,
    d.dname,
    e.ename,
    e.sal
from
    dept as d
left join
    emp as e
on
    d.deptno = e.deptno
where
    e.sal > 1000
;

 deptno |   dname    | ename  | sal
--------+------------+--------+------
     30 | SALES      | ALLEN  | 1600
     30 | SALES      | WARD   | 1250
     20 | RESEARCH   | JONES  | 2975
     30 | SALES      | MARTIN | 1250
     30 | SALES      | BLAKE  | 2850
     10 | ACCOUNTING | CLARK  | 2450
     20 | RESEARCH   | SCOTT  | 3000
     10 | ACCOUNTING | KING   | 5000
     30 | SALES      | TURNER | 1500
     20 | RESEARCH   | ADAMS  | 1100
     20 | RESEARCH   | FORD   | 3000
     10 | ACCOUNTING | MILLER | 1300
(12 rows)
```

しかし、本来残すべき`deptno=40`の下記のレコードが誤って除外されている。

```sql
 deptno |   dname    | ename  | sal
--------+------------+--------+------
     20 | RESEARCH   | SMITH  |  800
     30 | SALES      | JAMES  |  950
     40 | OPERATIONS |        |
```

実際は、こっちの結果がほしい。

```sql
select
    d.deptno,
    d.dname,
    e.ename,
    e.sal
from
    dept as d
left join
    emp as e
on
    d.deptno = e.deptno and
    e.sal > 1000
;

 deptno |   dname    | ename  | sal
--------+------------+--------+------
     30 | SALES      | ALLEN  | 1600
     30 | SALES      | WARD   | 1250
     20 | RESEARCH   | JONES  | 2975
     30 | SALES      | MARTIN | 1250
     30 | SALES      | BLAKE  | 2850
     10 | ACCOUNTING | CLARK  | 2450
     20 | RESEARCH   | SCOTT  | 3000
     10 | ACCOUNTING | KING   | 5000
     30 | SALES      | TURNER | 1500
     20 | RESEARCH   | ADAMS  | 1100
     20 | RESEARCH   | FORD   | 3000
     10 | ACCOUNTING | MILLER | 1300
     40 | OPERATIONS |        |
(13 rows)
```

うっかりミスには注意・・・Orz。

## :closed_book: Reference

None
