## :memo: Overview

ここではPostgreSQLの行コンストラクタについて理解を深める。下記、公式ドキュメントを参考にしている。

- [PostgreSQL: 4.2.12. 行コンストラクタ](https://www.postgresql.jp/document/8.4/html/sql-expressions.html#:~:text=%E3%81%A6%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82-,4.2.12.%20%E8%A1%8C%E3%82%B3%E3%83%B3%E3%82%B9%E3%83%88%E3%83%A9%E3%82%AF%E3%82%BF,-%E8%A1%8C%E3%82%B3%E3%83%B3%E3%82%B9%E3%83%88%E3%83%A9%E3%82%AF%E3%82%BF)


## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`row`

## :pencil2: Example

そもそも行コンストラクタとはなにか。公式ドキュメントによると、

> 行コンストラクタは、そのメンバフィールドに対する値を用いて行値（複合値とも呼ばれます）を構築する式です。 

とのことで、行値を作成する式のことらしく、`row()`関数がその役割を果たす。ドキュメントにある例をそのままお借りするが、本来は3カラムとして扱われるものが、1カラムで扱われており、カンマ区切りの括弧で囲われている。

```sql
SELECT ROW(1,2.5,'this is a test');

           row
--------------------------
 (1,2.5,"this is a test")
(1 row)
```

行コンストラクタには`rowvalue.*`構文でも利用でき、下記は等価である。

```sql
 select row(e.empno, e.ename, e.job, e.mgr, e.hiredate, e.sal, e.comm, e.deptno) from emp as e;

                         row
-----------------------------------------------------
 (7369,SMITH,CLERK,7902,1980-12-17,800,,20)
 (7499,ALLEN,SALESMAN,7698,1981-02-20,1600,300,30)
 (7521,WARD,SALESMAN,7698,1981-02-22,1250,500,30)
 (7566,JONES,MANAGER,7839,1981-04-02,2975,,20)
 (7654,MARTIN,SALESMAN,7698,1981-09-28,1250,1400,30)
 (7698,BLAKE,MANAGER,7839,1981-05-01,2850,,30)
 (7782,CLARK,MANAGER,7839,1981-06-09,2450,,10)
 (7788,SCOTT,ANALYST,7566,1982-12-09,3000,,20)
 (7839,KING,PRESIDENT,,1981-11-17,5000,,10)
 (7844,TURNER,SALESMAN,7698,1981-09-08,1500,0,30)
 (7876,ADAMS,CLERK,7788,1983-01-12,1100,,20)
 (7900,JAMES,CLERK,7698,1981-12-03,950,,30)
 (7902,FORD,ANALYST,7566,1981-12-03,3000,,20)
 (7934,MILLER,CLERK,7782,1982-01-23,1300,,10)
(14 rows)

select row(e.*) from emp as e;
                         row
-----------------------------------------------------
 (7369,SMITH,CLERK,7902,1980-12-17,800,,20)
 (7499,ALLEN,SALESMAN,7698,1981-02-20,1600,300,30)
 (7521,WARD,SALESMAN,7698,1981-02-22,1250,500,30)
 (7566,JONES,MANAGER,7839,1981-04-02,2975,,20)
 (7654,MARTIN,SALESMAN,7698,1981-09-28,1250,1400,30)
 (7698,BLAKE,MANAGER,7839,1981-05-01,2850,,30)
 (7782,CLARK,MANAGER,7839,1981-06-09,2450,,10)
 (7788,SCOTT,ANALYST,7566,1982-12-09,3000,,20)
 (7839,KING,PRESIDENT,,1981-11-17,5000,,10)
 (7844,TURNER,SALESMAN,7698,1981-09-08,1500,0,30)
 (7876,ADAMS,CLERK,7788,1983-01-12,1100,,20)
 (7900,JAMES,CLERK,7698,1981-12-03,950,,30)
 (7902,FORD,ANALYST,7566,1981-12-03,3000,,20)
 (7934,MILLER,CLERK,7782,1982-01-23,1300,,10)
(14 rows)
```

行コンストラクタ同士を比較させることもできる。

```sql
select
  e.*
from
  emp as e
where
  row(e.deptno) = row(20)
;

 empno | ename |   job   | mgr  |  hiredate  | sal  | comm | deptno
-------+-------+---------+------+------------+------+------+--------
  7369 | SMITH | CLERK   | 7902 | 1980-12-17 |  800 |      |     20
  7566 | JONES | MANAGER | 7839 | 1981-04-02 | 2975 |      |     20
  7788 | SCOTT | ANALYST | 7566 | 1982-12-09 | 3000 |      |     20
  7876 | ADAMS | CLERK   | 7788 | 1983-01-12 | 1100 |      |     20
  7902 | FORD  | ANALYST | 7566 | 1981-12-03 | 3000 |      |     20
(5 rows)
```

複数の値を利用して比較することも可能。

```sql
select
  e.*
from
  emp as e
where
  row(e.empno, e.sal) >= row(7876, 2500)
;
 empno | ename  |   job   | mgr  |  hiredate  | sal  | comm | deptno
-------+--------+---------+------+------------+------+------+--------
  7900 | JAMES  | CLERK   | 7698 | 1981-12-03 |  950 |      |     30
  7902 | FORD   | ANALYST | 7566 | 1981-12-03 | 3000 |      |     20
  7934 | MILLER | CLERK   | 7782 | 1982-01-23 | 1300 |      |     10
(3 rows)
```

注意が必要な点として、`OR`演算の「ような」動きになる点がある。`e.empno`が7886より小さく、`e.sal`が2500より小さい比較を意図した下記の通り書くと、上手くいかない。

```sql
select
  e.*
from
  emp as e
where
  row(e.empno, e.sal) < row(7876, 2500)
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
(11 rows)
```

「おそらく」ではあるが、まず`e.empno`で比較して、さらに`and`の結果を`or`で判定するような仕様になっていると思われる。

```sql
SELECT
  e.*
FROM
  emp AS e
WHERE
  e.empno < 7876 OR (e.empno = 7876 AND e.sal < 2500)
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
(11 rows)
```

前回のノートで参考にさせていただいた、こちらの記事で紹介されているグループごとに先頭行を表示するSQLでも利用できる。

- [Selecting N records for each group: PostgreSQL](https://explainextended.com/2009/04/29/selecting-n-records-for-each-group-postgresql/#more-1160)

```sql
with dlo as (
    select grouper,
           (
               select li
               from t_limiter li
               where li.grouper = dl.grouper
               order by grouper, ts, id
               offset 2 limit 1
           ) as mid
    from (
             select grouper
             from t_limiter
             group by grouper
         ) as dl
)
select l.*
from t_limiter l
join dlo 
on l.grouper = dlo.grouper
  and row(l.ts, l.id) < row(coalesce((dlo.mid).ts, 'infinity'::timestamp), (dlo.mid).id);

   id   | grouper |             ts             |    value
--------+---------+----------------------------+--------------
 652430 |       0 | 2024-03-05 10:13:31.070914 | Value 652430
 879640 |       0 | 2024-03-05 10:13:36.139145 | Value 879640
 929801 |       1 | 2024-03-05 10:13:32.376964 | Value 929801
 763081 |       1 | 2024-03-05 10:13:39.375441 | Value 763081
 384552 |       2 | 2024-03-05 10:13:33.337483 | Value 384552
  46732 |       2 | 2024-03-05 10:13:35.835395 | Value 46732
 690233 |       3 | 2024-03-05 10:13:39.049219 | Value 690233
  39363 |       3 | 2024-03-05 10:13:44.319483 | Value 39363
 851674 |       4 | 2024-03-05 10:13:25.974246 | Value 851674
 859994 |       4 | 2024-03-05 10:13:31.265885 | Value 859994
   6085 |       5 | 2024-03-05 10:13:22.391663 | Value 6085
 881095 |       5 | 2024-03-05 10:13:25.694444 | Value 881095
 660986 |       6 | 2024-03-05 10:13:29.275853 | Value 660986
 803096 |       6 | 2024-03-05 10:13:43.47626  | Value 803096
 156037 |       7 | 2024-03-05 10:13:33.738189 | Value 156037
 883417 |       7 | 2024-03-05 10:13:46.849142 | Value 883417
 420998 |       8 | 2024-03-05 10:13:32.471879 | Value 420998
 499968 |       8 | 2024-03-05 10:13:36.187704 | Value 499968
  99499 |       9 | 2024-03-05 10:13:21.918183 | Value 99499
 860289 |       9 | 2024-03-05 10:13:22.040125 | Value 860289
(20 rows)
```

先程の話通りであれば、`WHERE l.ts < dlo.ts OR (l.ts = dlo.ts AND l.id < dlo.id);`という条件で判定されているので、分解すると下記の通りになるはず（あまり自信ない）。

```sql
-- l table
   id   | grouper |             ts             |    value     | ind
--------+---------+----------------------------+--------------+-----
 652430 |       0 | 2024-03-05 10:13:31        | Value 652430 |   1
 879640 |       0 | 2024-03-05 10:13:36        | Value 879640 |   2
 778220 |       0 | 2024-03-05 10:13:41        | Value 778220 |   3
 859880 |       0 | 2024-03-05 10:13:54        | Value 859880 |   4
 972930 |       0 | 2024-03-05 10:13:56        | Value 972930 |   5

-- dlo
 grouper |                          mid
---------+--------------------------------------------------------
       0 | (778220,0,"2024-03-05 10:13:41.748538","Value 778220")

   l.ts  |  dlo.ts  |        |    l.ts  |  dlo.ts |   |  l.id < dlo.id |                              |  
--------------------------------------------------------------------------------------------------------------------------------
10:13:31 < 10:13:41 -> TRUE  | 10:13:31 = 10:13:41 AND 652430 < 778220 | -> FALSE AND TRUE  -> FALSE  | TRUE  OR FALSE -> TRUE
10:13:36 < 10:13:41 -> TRUE  | 10:13:36 = 10:13:41 AND 879640 < 778220 | -> FALSE AND FALSE -> FALSE  | TRUE  OR FALSE -> TRUE
10:13:41 < 10:13:41 -> FALSE | 10:13:41 = 10:13:41 AND 778220 < 778220 | -> TRUE AND FALSE  -> FALSE  | FALSE OR FALSE -> FALSE
10:13:54 < 10:13:41 -> FALSE |          |         |   |                | 
10:13:56 < 10:13:41 -> FALSE |          |         |   |                | 
```


## :closed_book: Reference

- [EXPLAIN EXTENDED](https://explainextended.com/)





