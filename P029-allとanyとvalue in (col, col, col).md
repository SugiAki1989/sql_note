## :memo: Overview

「いずれか(すべて)のカラムの値が `1`(`null`)」みたいな条件で大量のカラムから抽出する場合、`all`や`any`が役立つ。
また、`where`で`in`を使う場合、`col in (value, value,...)`という使い方が一般的だが、反対に`value in (col, col,...)`という使い方もできる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`in`, `all`, `any`

## :pencil2: Example

こんなサンプルテーブルがあるとする。とにかく実務ではよく見る、Long 型でテーブル管理してほしいやつ。

```sql
create table allany(id integer, c1 integer, c2 integer, c3 integer, c4 integer, c5 integer);
insert into allany
    (id, c1, c2, c3, c4, c5)
values
    ('2','1','0','99','0','0'),
    ('1','1','1','1','1','1'),
    ('3','1','99','0','0','0'),
    ('4','1',null,null,null,null),
    ('5',null,null,null,null,null)
;

 select * from allany;
 id | c1 | c2 | c3 | c4 | c5
----+----+----+----+----+----
  2 |  1 |  0 | 99 |  0 |  0
  1 |  1 |  1 |  1 |  1 |  1
  3 |  1 | 99 |  0 |  0 |  0
  4 |  1 |    |    |    |
  5 |    |    |    |    |
(5 rows)
```

このテーブルに対して、「いずれかのカラムの値が 1」というレコードを抜き出したいとすると、愚直に書くと…

```sql
select
    *
from
    allany
where
    c1 = 1 or
    c2 = 1 or
    c3 = 1 or
    c4 = 1 or
    c5 = 1
;
 id | c1 | c2 | c3 | c4 | c5
----+----+----+----+----+----
  2 |  1 |  0 | 99 |  0 |  0
  1 |  1 |  1 |  1 |  1 |  1
  3 |  1 | 99 |  0 |  0 |  0
  4 |  1 |    |    |    |
(4 rows)
```

このようになり、`value in (col, col,...)`ともかけるのでこう書く。

```sql
select
    *
from
    allany
where
    1 in (c1, c2, c3, c4, c5)
;

 id | c1 | c2 | c3 | c4 | c5
----+----+----+----+----+----
  2 |  1 |  0 | 99 |  0 |  0
  1 |  1 |  1 |  1 |  1 |  1
  3 |  1 | 99 |  0 |  0 |  0
  4 |  1 |    |    |    |
(4 rows)
```

次に「すべてのカラムの値が 1」というレコードを抜き出したいとすると、ふたたび愚直に書くとこうなる。

```sql
select
    *
from
    allany
where
    c1 = 1 and
    c2 = 1 and
    c3 = 1 and
    c4 = 1 and
    c5 = 1
;

 id | c1 | c2 | c3 | c4 | c5
----+----+----+----+----+----
  1 |  1 |  1 |  1 |  1 |  1
(1 row)
```

どの方法もすごく力づくでメンテナンス性がない。これを`all`や`any`が解決してくれる(あまり変わってない気もしなくもない)。`all`や`any`は、PostgeSQL では`any(array[]), all(array[])`と書く必要がある。まずは`any`から。`any`は`c1 = 1 or c2 = 1 or ... or c5 = 1`のショートカットである。

```sql
select
    *
from
    allany
where
    1 = any(array[c1, c2, c3, c4, c5])
;

 id | c1 | c2 | c3 | c4 | c5
----+----+----+----+----+----
  2 |  1 |  0 | 99 |  0 |  0
  1 |  1 |  1 |  1 |  1 |  1
  3 |  1 | 99 |  0 |  0 |  0
  4 |  1 |    |    |    |
(4 rows)
```

次は`all`。

```sql
select
    *
from
    allany
where
    1 = all(array[c1, c2, c3, c4, c5])
;

 id | c1 | c2 | c3 | c4 | c5
----+----+----+----+----+----
  1 |  1 |  1 |  1 |  1 |  1
(1 row)
```

`all`は`c1 = 1 and c2 = 1 and ... and c5 = 1`のショートカットなので、連続している`null`を抜き出す場合は`coalesce`を使って調整が必要。

```sql
select
    *
from
    allany
where
    null = all(array[c1, c2, c3, c4, c5])
;
 id | c1 | c2 | c3 | c4 | c5
----+----+----+----+----+----
(0 rows)

select
    *
from
    allany
where
    coalesce(c1, c2, c3, c4, c5) is null
;

 id | c1 | c2 | c3 | c4 | c5
----+----+----+----+----+----
  5 |    |    |    |    |
(1 row)
```

`all`や`any`の使い方をもう少し見ておく。`sal`は、`ALLEN:1600`、`SCOTT:3000`である。`any`だといずれかを満たせばよいので、この場合`1600`より大きいレコードが検索される。

```sql
select * from emp
where sal > any(select sal from emp where ename = 'ALLEN' or ename = 'SCOTT');

 empno | ename |    job    | mgr  |  hiredate  | sal  | comm | deptno
-------+-------+-----------+------+------------+------+------+--------
  7566 | JONES | MANAGER   | 7839 | 1981-04-02 | 2975 | NULL |     20
  7698 | BLAKE | MANAGER   | 7839 | 1981-05-01 | 2850 | NULL |     30
  7782 | CLARK | MANAGER   | 7839 | 1981-06-09 | 2450 | NULL |     10
  7788 | SCOTT | ANALYST   | 7566 | 1982-12-09 | 3000 | NULL |     20
  7839 | KING  | PRESIDENT | NULL | 1981-11-17 | 5000 | NULL |     10
  7902 | FORD  | ANALYST   | 7566 | 1981-12-03 | 3000 | NULL |     20
(6 rows)
```

`all`だとすべてを満たす必要があるので、この場合`3000`以上でなければならない。

```sql
select * from emp
where sal >= all(select sal from emp where ename = 'ALLEN' or ename = 'SCOTT');

 empno | ename |    job    | mgr  |  hiredate  | sal  | comm | deptno
-------+-------+-----------+------+------------+------+------+--------
  7788 | SCOTT | ANALYST   | 7566 | 1982-12-09 | 3000 | NULL |     20
  7839 | KING  | PRESIDENT | NULL | 1981-11-17 | 5000 | NULL |     10
  7902 | FORD  | ANALYST   | 7566 | 1981-12-03 | 3000 | NULL |     20
(3 rows)
```

`in`はスカラやベクトルだけではなく、テーブルを使って条件付けることもできる。あまり使う頻度は高くないと思うが、忘れた時のメモとして記録しておく。`where`で選択したカラムと`select`で選択したカラムが一致している必要がある。

```sql
select
    ename, deptno
from
    emp
where
    (ename, deptno) in 
    (select ename, deptno from emp where ename = 'MILLER' or ename = 'SCOTT')
;

 ename  | deptno
--------+--------
 SCOTT  |     20
 MILLER |     10
(2 rows)
```

## :closed_book: Reference

None
