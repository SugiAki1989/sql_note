## :memo: Overview

[SQL クックブック 第 2 版](https://www.oreilly.co.jp/books/9784873119779/)の p52、レシピ 3.9「集約の使用時に結合を実行する」では `sum(distinct)`の使用例が記載されている。結合することによって、レコードが重複し、集計の妨げになるので、`sum(distinct)`を使用して回避するという内容だが、アドホックな SQL の例としておそらく記載されているものだと思うのですが、運用段階では集計誤りを起こす可能性がありそうなので、そのまとめ。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`sum(distinct)`

## :pencil2: Example

クエリの目的は、`deptno=10`に限定し、`emp`テーブルの`sal`の合計と、`emp_bonus`テーブルの`bonus`の合計を計算すること。ただし、`emp_bonus`テーブルを`empno`で紐付けると、複数回ボーナスを貰っている社員がいるので、`emp`テーブルのレコードが重複し、`emp`テーブルの`sal`が重複した分ダブルカウントしてしまい、集計誤りが発生するというもの。

```sql
-- これはtotal_salがダブルカウントされていて誤り
 deptno | total_sal | total_bonus
--------+-----------+-------------
     10 |     10050 |      2135.0
```

下記では、書籍に記載されている解決策のクエリを記載する。

```sql
CREATE TABLE emp_bonus (EMPNO integer NOT NULL, RECEIVED DATE, TYPE integer);
INSERT INTO emp_bonus VALUES(7934, '1981-3-17', 1);
INSERT INTO emp_bonus VALUES(7934, '1981-2-15', 2);
INSERT INTO emp_bonus VALUES(7839, '1981-2-15', 3);
INSERT INTO emp_bonus VALUES(7782, '1981-2-15', 1);

select
    deptno,
	sum(distinct sal) as total_sal,
	sum(bonus) as total_bonus
from
    (
	select
	    e.empno,
		e.ename,
		e.sal,
		e.deptno,
		e.sal*case when eb.type = 1 then 0.1
                   when eb.type = 2 then 0.2
		else 0.3 end as bonus
	from
		emp as e, emp_bonus as eb
	where e.empno = eb.empno
		and e.deptno = 10
	) as x
group by
    deptno
;

 deptno | total_sal | total_bonus
--------+-----------+-------------
     10 |      8750 |      2135.0
```

レコードが重複するのであれば、`sal`を`sum(distinct sal)`として、ダブルカウントを回避している。ただこの SQL は、この`emp`テーブルであれば問題が起きないが、下記の例のように、`sal`が同じ別の社員`TANAKA`が追加されると、その社員の`sal`が合計されなくなる。

```sql
-- テーブル定義をそのままコピーして、1行追加
create table emp_addrow (like emp);
insert into emp_addrow select * from emp;
INSERT INTO emp_addrow VALUES(9999, 'TANAKA', 'CLERK', 7782,'1982-11-23', 1300, NULL, 10);

select * from emp_addrow order by deptno asc;
 empno | ename  |    job    | mgr  |  hiredate  | sal  | comm | deptno
-------+--------+-----------+------+------------+------+------+--------
  7839 | KING   | PRESIDENT |      | 1981-11-17 | 5000 |      |     10
  7782 | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450 |      |     10
  7934 | MILLER | CLERK     | 7782 | 1982-01-23 | 1300 |      |     10
  9999 | TANAKA | CLERK     | 7782 | 1982-11-23 | 1300 |      |     10  ★追加
  7788 | SCOTT  | ANALYST   | 7566 | 1982-12-09 | 3000 |      |     20
  7566 | JONES  | MANAGER   | 7839 | 1981-04-02 | 2975 |      |     20
  7369 | SMITH  | CLERK     | 7902 | 1980-12-17 |  800 |      |     20
  7876 | ADAMS  | CLERK     | 7788 | 1983-01-12 | 1100 |      |     20
  7902 | FORD   | ANALYST   | 7566 | 1981-12-03 | 3000 |      |     20
  7844 | TURNER | SALESMAN  | 7698 | 1981-09-08 | 1500 |    0 |     30
  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250 |  500 |     30
  7900 | JAMES  | CLERK     | 7698 | 1981-12-03 |  950 |      |     30
  7698 | BLAKE  | MANAGER   | 7839 | 1981-05-01 | 2850 |      |     30
  7499 | ALLEN  | SALESMAN  | 7698 | 1981-02-20 | 1600 |  300 |     30
  7654 | MARTIN | SALESMAN  | 7698 | 1981-09-28 | 1250 | 1400 |     30
```

テーブルを`emp_addrow`に変更して先程のクエリを発行すると、`total_sal`が合計されていない。

```sql
select
    deptno,
	sum(distinct sal) as total_sal,
	sum(bonus) as total_bonus
from
    (
	select
	    e.empno,
		e.ename,
		e.sal,
		e.deptno,
		e.sal*case when eb.type = 1 then 0.1
                   when eb.type = 2 then 0.2
		else 0.3 end as bonus
	from
		emp_addrow as e, emp_bonus as eb -- テーブルをemp_addrowに変更
	where e.empno = eb.empno
		and e.deptno = 10
	) as x
group by
    deptno
;

 deptno | total_sal | total_bonus
--------+-----------+-------------
     10 |      8750 |      2135.0
```

他にもたくさんやり方はあると思うが、レコードに重複が発生するのであれば`sal`、`bonus`を別々に集計すればよいが、あまり良い方法が思いつかず、愚直な方法法で集計するクエリをメモ程度に記載しておく。

```sql
with tmp1 as (
select
    deptno,
    sum(sal) as total_sal
from
    emp_addrow
where
    deptno = 10
group by
    deptno
), tmp2 as (
select
	deptno,
	e.sal*case when eb.type = 1 then 0.1
               when eb.type = 2 then 0.2
		       else 0.3 end as bonus
from
	emp as e
left join
	emp_bonus as eb
on
	e.empno = eb.empno
where
	e.deptno = 10
), tmp2_g as (
select
	deptno,
	sum(bonus) as total_bonus
from
	tmp2
group by
	deptno
)
select
	t1.deptno,
    t1.total_sal,
	t2g.total_bonus
from
    tmp1 as t1
left join
    tmp2_g as t2g
on
    t1.deptno = t2g.deptno
;

 deptno | total_sal | total_bonus
--------+-----------+-------------
     10 |     10050 |      2135.0
```

## :closed_book: Reference

- [SQL Cookbook](https://www.oreilly.com/library/view/sql-cookbook/0596009763/)
- [SQL クックブック 第 2 版](https://www.oreilly.co.jp/books/9784873119779/)
