## :memo: Overview

`left join`の`on`には結合条件を記述し、条件判定のために、等号、不等号などを使うが、実は`in`も使えるらしい。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`left join`, `on`, `in`

## :pencil2: Example

まずはサンプルデータを用意する。よくあるような横広のテーブルデータがあったとする。

```sql
create table ljoin(id varchar(5), c1 integer, c2 integer, c3 integer);
insert into ljoin(id, c1, c2, c3)
values
('a','1','2','3'),
('b','4',null,null),
('c','5','6',null),
('d',null,null,null);

select * from ljoin;
 id | c1 | c2 | c3
----+----+----+----
 a  |  1 |  2 |  3
 b  |  4 |    |
 c  |  5 |  6 |
 d  |    |    |
(4 rows)

CREATE TABLE t10 (ID INTEGER);
INSERT INTO t10 VALUES (1);
INSERT INTO t10 VALUES (2);
INSERT INTO t10 VALUES (3);
INSERT INTO t10 VALUES (4);
INSERT INTO t10 VALUES (5);
INSERT INTO t10 VALUES (6);
INSERT INTO t10 VALUES (7);
INSERT INTO t10 VALUES (8);
INSERT INTO t10 VALUES (9);
INSERT INTO t10 VALUES (10);

CREATE TABLE t10dummy (ID INTEGER);
INSERT INTO t10dummy VALUES (2);
INSERT INTO t10dummy VALUES (4);
INSERT INTO t10dummy VALUES (6);
INSERT INTO t10dummy VALUES (8);
INSERT INTO t10dummy VALUES (10);
```

ここから各`id`が持っている値を Wide 型ではなく Long 型で持たせるようにしたい。1 列ごとに`union`して積み上げていく方法があるが、今回はそれを利用せず、`join`で解決する。下記のように、`on`で`t10.id in (ljoin.c1, ljoin.c2, ljoin.c3)`と記述することで、テーブルを変形できる。

```sql
select
  ljoin.id as ljoinid,
  t10.id as t10id
from
  ljoin
left join
  t10
on
  t10.id in (ljoin.c1, ljoin.c2, ljoin.c3)
;

ljoinid | t10id
---------+-------
 a       |     1
 a       |     2
 a       |     3
 b       |     4
 c       |     5
 c       |     6
 d       |
(7 rows)
```

注意点として、マスタのテーブルに不備があると、思ったように結果は得られない。`t10dummy`が偶数しか持っていない。

```sql

select
  ljoin.id as ljoinid,
  t10dummy.id as t10id
from
  ljoin
left join
  t10dummy
on
  t10dummy.id in (ljoin.c1, ljoin.c2, ljoin.c3)
;

 ljoinid | t10id
---------+-------
 a       |     2
 b       |     4
 c       |     6
 d       |
(4 rows)
```

`inner join`でも勿論利用できる。

```sql
select
  ljoin.id as ljoinid,
  t10dummy.id as t10id
from
  ljoin
inner join
  t10dummy
on
  t10dummy.id in (ljoin.c1, ljoin.c2, ljoin.c3)
;

 ljoinid | t10id
---------+-------
 a       |     2
 b       |     4
 c       |     6
(3 rows)
```

この方法を使えば、忌まわしい履歴テーブルのように管理すればいいものを、マスタのように管理しているフラグテーブルを対処できると思ったかもしれないが、それはできない。下記の結果を見ればわかるが、`in`なので思っているとおりの結果は返ってこない。`a`と`k`は`1`か`0`しか値を持っていないので、1 行しか返らない。

```sql
create table ljoin2(id varchar(5),
 f1 integer, f2 integer, f3 integer, f4 integer, f5 integer,
 f6 integer, f7 integer, f8 integer, f9 integer, f10 integer
);

insert into ljoin2(id, f1, f2, f3, f4, f5, f6, f7, f8, f9, f10)
values
('a','1','1','1','1','1','1','1','1','1','1'),
('b','1','1','1','1','1','1','1','1','1','0'),
('c','1','1','1','1','1','1','1','1','0','0'),
('d','1','1','1','1','1','1','1','0','0','0'),
('e','1','1','1','1','1','1','0','0','0','0'),
('f','1','1','1','1','1','0','0','0','0','0'),
('g','1','1','1','1','0','0','0','0','0','0'),
('h','1','1','1','0','0','0','0','0','0','0'),
('i','1','1','0','0','0','0','0','0','0','0'),
('j','1','0','0','0','0','0','0','0','0','0'),
('k','0','0','0','0','0','0','0','0','0','0')
;

create table onezero(num integer);
insert into onezero(num)
values
('0'),
('1');

select
  *
from
  ljoin2
left join
  onezero
on
  onezero.num in (
    ljoin2.f1,
    ljoin2.f2,
    ljoin2.f3,
    ljoin2.f4,
    ljoin2.f5,
    ljoin2.f6,
    ljoin2.f7,
    ljoin2.f8,
    ljoin2.f9,
    ljoin2.f10)
;

 id | f1 | f2 | f3 | f4 | f5 | f6 | f7 | f8 | f9 | f10 | num
----+----+----+----+----+----+----+----+----+----+-----+-----
 a  |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  1 |   1 |   1
 b  |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  1 |   0 |   0
 b  |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  1 |   0 |   1
 c  |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  0 |   0 |   0
 c  |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  0 |   0 |   1
 d  |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  0 |  0 |   0 |   0
 d  |  1 |  1 |  1 |  1 |  1 |  1 |  1 |  0 |  0 |   0 |   1
 e  |  1 |  1 |  1 |  1 |  1 |  1 |  0 |  0 |  0 |   0 |   0
 e  |  1 |  1 |  1 |  1 |  1 |  1 |  0 |  0 |  0 |   0 |   1
 f  |  1 |  1 |  1 |  1 |  1 |  0 |  0 |  0 |  0 |   0 |   0
 f  |  1 |  1 |  1 |  1 |  1 |  0 |  0 |  0 |  0 |   0 |   1
 g  |  1 |  1 |  1 |  1 |  0 |  0 |  0 |  0 |  0 |   0 |   0
 g  |  1 |  1 |  1 |  1 |  0 |  0 |  0 |  0 |  0 |   0 |   1
 h  |  1 |  1 |  1 |  0 |  0 |  0 |  0 |  0 |  0 |   0 |   0
 h  |  1 |  1 |  1 |  0 |  0 |  0 |  0 |  0 |  0 |   0 |   1
 i  |  1 |  1 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |   0 |   0
 i  |  1 |  1 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |   0 |   1
 j  |  1 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |   0 |   0
 j  |  1 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |   0 |   1
 k  |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |  0 |   0 |   0
(20 rows)
```

## :closed_book: Reference

None
