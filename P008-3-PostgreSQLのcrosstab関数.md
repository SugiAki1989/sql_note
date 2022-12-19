## :memo: Overview

PostgreSQL の`crosstab`関数を利用するとピボット集計テーブルが簡単に作ることができるが、使い方に癖があるので、`case`文で対応するのが良いかも。

- [F.40. tablefunc](https://www.postgresql.org/docs/14/tablefunc.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`crosstab`

## :pencil2: Example

サンプルのテーブルを用意しておく。このテーブルを Onehot エンコーディングしたいとする。

```sql
create table onehot(cuid int, goods varchar(10));
insert into onehot(cuid, goods)
values
  ('1','a'),
  ('1','b'),
  ('1','c'),
  ('1','d'),
  ('1','e'),
  ('2','a'),
  ('2','b'),
  ('2','c'),
  ('2','d'),
  ('3','a'),
  ('3','b'),
  ('3','c'),
  ('4','a'),
  ('4','b'),
  ('5','a'),
  ('6','a'),
  ('6','a');
```

`crosstab`は拡張機能なので、下記を実行してロードする必要がある。

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;
```

`tablefunc`には他にも、正規分布から乱数を生成する`Normal_rand`や

```sql
select * from normal_rand(5, 100, 10);

    normal_rand
--------------------
 106.92818549168467
  83.82559175332881
  73.14899691506155
  99.62481956615093
  89.98807929796102
(5 rows)

```

階層ツリー構造の表現を生成する`connectby`などもある。

```sql
CREATE TABLE connectby_tree(keyid text, parent_keyid text, pos int);

INSERT INTO connectby_tree VALUES('row1',NULL, 0);
INSERT INTO connectby_tree VALUES('row2','row1', 0);
INSERT INTO connectby_tree VALUES('row3','row1', 0);
INSERT INTO connectby_tree VALUES('row4','row2', 1);
INSERT INTO connectby_tree VALUES('row5','row2', 0);
INSERT INTO connectby_tree VALUES('row6','row4', 0);
INSERT INTO connectby_tree VALUES('row7','row3', 0);
INSERT INTO connectby_tree VALUES('row8','row6', 0);
INSERT INTO connectby_tree VALUES('row9','row5', 0);

-- with branch, without orderby_fld (order of results is not guaranteed)
SELECT * FROM connectby('connectby_tree', 'keyid', 'parent_keyid', 'row2', 0, '~')
 AS t(keyid text, parent_keyid text, level int, branch text);

 keyid | parent_keyid | level |       branch
-------+--------------+-------+---------------------
 row2  |              |     0 | row2
 row4  | row2         |     1 | row2~row4
 row6  | row4         |     2 | row2~row4~row6
 row8  | row6         |     3 | row2~row4~row6~row8
 row5  | row2         |     1 | row2~row5
 row9  | row5         |     2 | row2~row5~row9
(6 rows)
```

`crosstab`に話を戻すと、普通に SQL で Onehot エンコーディングしようとすると、展開したい列の数分、`case`文を書けば良い。ここではアルファベット順で展開することにする。

```sql
select
  cuid,
  max(case when goods = 'a' then 1 else null end) as f_a,
  max(case when goods = 'b' then 1 else null end) as f_b,
  max(case when goods = 'c' then 1 else null end) as f_c,
  max(case when goods = 'd' then 1 else null end) as f_d,
  max(case when goods = 'e' then 1 else null end) as f_e
from onehot
group by cuid
order by cuid
;

 cuid | f_a | f_b | f_c | f_d | f_e
------+-----+-----+-----+-----+-----
    1 |   1 |   1 |   1 |   1 |   1
    2 |   1 |   1 |   1 |   1 |
    3 |   1 |   1 |   1 |     |
    4 |   1 |   1 |     |     |
    5 |   1 |     |     |     |
    6 |   1 |     |     |     |
(6 rows)
```

`crosstab`を使うためには、`from`で`crosstab`を利用する。`as`のところで返ってくるテーブルの定義を行う。ここでは意図的にアルファベット順の逆順で定義している。

```sql
select *
from crosstab(
  'select cuid, goods, max(1::int)
   from onehot
   group by cuid, goods
   order by cuid'
) as pivot(
  cuid int,
  f_e int,
  f_d int,
  f_c int,
  f_b int,
  f_a int
);

 cuid | f_e | f_d | f_c | f_b | f_a
------+-----+-----+-----+-----+-----
    1 |   1 |   1 |   1 |   1 |   1
    2 |   1 |   1 |   1 |   1 |
    3 |   1 |   1 |   1 |     |
    4 |   1 |   1 |     |     |
    5 |   1 |     |     |     |
    6 |   1 |     |     |     |
(6 rows)

```

できたけど、`case`文の結果と異なっている。`f_a`は全員にフラグが立っている必要があるがそうなっていない。つまり、PostgreSQL が `pivot`で定義したカラム名と値を比較して、適切に展開してくれるわけではない。`order by`で並び替えてもうまく行かない。

```sql

select *
from crosstab(
  'select cuid, goods, max(1::int)
   from onehot
   group by cuid, goods
   order by cuid, goods desc'
) as pivot(
  cuid int,
  f_e int,
  f_d int,
  f_c int,
  f_b int,
  f_a int
);

 cuid | f_e | f_d | f_c | f_b | f_a
------+-----+-----+-----+-----+-----
    1 |   1 |   1 |   1 |   1 |   1
    2 |   1 |   1 |   1 |   1 |
    3 |   1 |   1 |   1 |     |
    4 |   1 |   1 |     |     |
    5 |   1 |     |     |     |
    6 |   1 |     |     |     |
(6 rows)
```

サブクエリで予め並び替えてもだめ。

```sql
select *
from crosstab(
  'select cuid, goods, max(1::int)
   from (select * from onehot order by goods desc) as s
   group by cuid, goods
   order by cuid'
) as pivot(
  cuid int,
  f_e int,
  f_d int,
  f_c int,
  f_b int,
  f_a int
);

 cuid | f_e | f_d | f_c | f_b | f_a
------+-----+-----+-----+-----+-----
    1 |   1 |   1 |   1 |   1 |   1
    2 |   1 |   1 |   1 |   1 |
    3 |   1 |   1 |   1 |     |
    4 |   1 |   1 |     |     |
    5 |   1 |     |     |     |
    6 |   1 |     |     |     |
(6 rows)
```

`goods`が昇順にソートされている前提で、合わせて`as`でテーブルを定義する必要がある。これで正しい結果が得られる。

```sql
select *
from crosstab(
  'select cuid, goods, max(1::int)
   from onehot
   group by cuid, goods
   order by cuid'
) as pivot(
  cuid int,
  f_a int,
  f_b int,
  f_c int,
  f_d int,
  f_e int
);

 cuid | f_a | f_b | f_c | f_d | f_e
------+-----+-----+-----+-----+-----
    1 |   1 |   1 |   1 |   1 |   1
    2 |   1 |   1 |   1 |   1 |
    3 |   1 |   1 |   1 |     |
    4 |   1 |   1 |     |     |
    5 |   1 |     |     |     |
    6 |   1 |     |     |     |
(6 rows)
```

実際使うときには、何らかの方法で機械にソートしてもらう必要がある。例えば SQL 分のいち部分を SQL で生成する必要があるかも。

```sql
with tmp as (
select distinct goods from onehot order by goods asc
), tmp2 as (
  select *, row_number() over (order by goods asc) as ind from tmp
)
select
  case when
  ind != (select max(ind) from tmp2)
  then 'f_' || goods::text || ' integer,'
  else 'f_' || goods::text || ' integer'
  end as sql_statement
from tmp2
;

 sql_statement
---------------
 f_a integer,
 f_b integer,
 f_c integer,
 f_d integer,
 f_e integer
(5 rows)
```

## :closed_book: Reference

None
