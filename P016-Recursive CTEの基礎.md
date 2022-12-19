## :memo: Overview

再帰 SQL(Recursive SQL) の基本的な動きを理解する。今回は基本的な再帰 SQL の使い方を学び、次回は再帰 SQL を実践的な利用方法をまとめる。

ここで取り上げるのは、ネットで検索するとよく出てくるような再帰 SQL の事例をまとめている。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`with`, `recursive`,

## :pencil2: Example

### 基本的な再帰 SQL

まずは連番を生成する再帰 SQL。再帰 SQL は 3 つのパートに分かれている。非再帰クエリ、再帰クエリ、メインクエリの 3 つ。

```sql
with recursive recur(depth) as (
    -- 非再帰クエリ
    select 1
    union all
    -- 再帰クエリ
    select depth + 1
    from recur
    where depth + 1 <= 3)
-- メインクエリ
select * from recur;

 depth
-------
     1
     2
     3
```

厳密ではないが、イメージとして、再帰 SQL の動きをまとめると、まず、非再帰クエリが実行される。この時点で、非再帰クエリによって生成されたテーブルが利用可能になるので、再帰クエリがこれを呼び出す。再帰クエリが実行されて、実行結果を最初に非再帰クエリが生成したテーブルにユニオンする。そして、終了条件に引っかかるまでは、これを繰り返す。終了条件に引っかかったら、メインクエリにテーブルが渡されて、メインクエリが実行される。

```sql
-- これであれば再帰SQLでなくてもよい…
select depth from generate_series(1, 3) as depth;

 depth
-------
     1
     2
     3
```

### 総積を計算する再帰 SQL

先程の動作イメージがあれば総積が計算できることも理解できるはず。ウインドウ関数には総積を計算する関数がないので、対数の性質を利用する方法を以前まとめたが、再帰 SQL を使えば再現できる。

```sql
with recursive recur(depth, val) as (
    select 1, 1
    union all
    select depth + 1, val*(depth + 1)
    from recur
    where depth + 1 <= 10
)
select * from recur;

 depth |   val
-------+---------
     1 |       1
     2 |       2
     3 |       6
     4 |      24
     5 |     120
     6 |     720
     7 |    5040
     8 |   40320
     9 |  362880
    10 | 3628800
```

### 期間の穴を埋める再帰 SQL

次は終了条件をうまく使えば、各レコードに対して、異なる深さの再帰を実行できる。下記のような各`id`ごとに開始日と終了日だけを持っているテーブルをもとに、開始終了の期間を生成する場合に再帰 SQL は利用できる。先程の例と異なるのは各レコードごとに深さが異なるので、終了条件に数字を直打ちせず、`s`と`e`の値を利用して、深さをコントロールしている。

```sql
create table r (id integer, s date, e date);
insert into r
    (id, s, e)
values
    ('1', '2022-08-01', '2022-08-04'), --  4 days
    ('2', '2022-08-05', '2022-08-14'), -- 10 days
    ('3', '2022-08-21', '2022-08-21'); --  0 days

with recursive recur(id, s, e) as (
    select id, s, e
    from r
    union all
    select id, s+1, e
    from recur
    -- whereがないと無限ループになるので注意
    where s+1 <= e
)
select * from recur order by id asc, s asc;

 id |     s      |     e
----+------------+------------
  1 | 2022-08-01 | 2022-08-04
  1 | 2022-08-02 | 2022-08-04
  1 | 2022-08-03 | 2022-08-04
  1 | 2022-08-04 | 2022-08-04
  2 | 2022-08-05 | 2022-08-14
  2 | 2022-08-06 | 2022-08-14
  2 | 2022-08-07 | 2022-08-14
  2 | 2022-08-08 | 2022-08-14
  2 | 2022-08-09 | 2022-08-14
  2 | 2022-08-10 | 2022-08-14
  2 | 2022-08-11 | 2022-08-14
  2 | 2022-08-12 | 2022-08-14
  2 | 2022-08-13 | 2022-08-14
  2 | 2022-08-14 | 2022-08-14
  3 | 2022-08-21 | 2022-08-21
```

## 隣接リストモデル

基本的な使い方のまとめとして、 隣接リストモデルに対する再帰 SQL をみておく。

```sql
create table tree(id,parent_id) as
values(1, null),
      (2,   1),
      (3,   1),
      (4,   2),
      (5,   2),
      (6,   3),
      (7,   6),
      (8,   6);

1
├── 2
│   ├── 4
│   └── 5
└── 3
    └── 6
        ├── 7
        └── 8
```

やっていることは、ここまでで紹介した内容を使っているだけ。再帰 SQL がテーブルをユニオンする過程もあわせてメモしておく。

```sql
with recursive recur(id, parent_id, path, depth) as(
    select id, parent_id, array[id], 1 as depth
    from tree
    where parent_id is null
    union all
    select b.id, b.parent_id, path || b.id, depth + 1
    from recur
    join tree as b
    on recur.id = b.parent_id
    -- 枝切り
    -- where depth + 1 <= 1
)
select id, parent_id, path, depth
from recur order by path;

 id | parent_id | path | depth
----+-----------+------+-------
  1 |           | {1}  |     1

 id | parent_id | path  | depth
----+-----------+-------+-------
  1 |           | {1}   |     1
  2 |         1 | {1,2} |     2
  3 |         1 | {1,3} |     2

 id | parent_id |  path   | depth
----+-----------+---------+-------
  1 |           | {1}     |     1
  2 |         1 | {1,2}   |     2
  4 |         2 | {1,2,4} |     3
  5 |         2 | {1,2,5} |     3
  3 |         1 | {1,3}   |     2
  6 |         3 | {1,3,6} |     3

 id | parent_id |   path    | depth
----+-----------+-----------+-------
  1 |           | {1}       |     1
  2 |         1 | {1,2}     |     2
  4 |         2 | {1,2,4}   |     3
  5 |         2 | {1,2,5}   |     3
  3 |         1 | {1,3}     |     2
  6 |         3 | {1,3,6}   |     3
  7 |         6 | {1,3,6,7} |     4
  8 |         6 | {1,3,6,8} |     4
```

`array_*`系の関数を利用しても同じことはできる。

```sql
-- array_upper, array_lengthバージョン
with recursive recur(id, parent_id, path) as(
    select id, parent_id, array[id]
    from tree
    where parent_id is null
    union all
    select b.id, b.parent_id, path || b.id
    from recur
    join tree as b
    on recur.id = b.parent_id
    where array_length(recur.path, 1) + 1 <= 2
)
select id,parent_id, path, array_upper(path,1) as depth
from recur order by path;

 id | parent_id | path  | depth
----+-----------+-------+-------
  1 |           | {1}   |     1
  2 |         1 | {1,2} |     2
  3 |         1 | {1,3} |     2
```

## 単純なツリー探索

[プログラマのための SQL 第 4 版 すべてを知り尽くしたいあなたに](https://www.shoeisha.co.jp/book/detail/9784798128023)の p487 に記載されている例。
`assembly_nbr`は`subassembly_nbr`の部品から構築されているため、再帰 SQL を使えば、`assembly_nbr`に必要な`subassembly_nbr`の部品を探索できる。

```sql
create table billofmaterials
(part_name varchar(20) not null primary key,
 assembly_nbr integer, -- nullは完成品
 subassembly_nbr integer not null);

insert into billofmaterials values('a', 11, 12);
insert into billofmaterials values('b', 12, 10);
insert into billofmaterials values('c', 12,  9);
insert into billofmaterials values('d', 10,  8);
insert into billofmaterials values('e',  8,  7);
insert into billofmaterials values('f', 20, 11);
insert into billofmaterials values('g',  6,  5);
insert into billofmaterials values('h', 20,  7);
insert into billofmaterials values('i', 10,  4);

select * from billofmaterials;
 part_name | assembly_nbr | subassembly_nbr
-----------+--------------+-----------------
 a         |           11 |              12
 b         |           12 |              10
 c         |           12 |               9
 d         |           10 |               8
 e         |            8 |               7
 f         |           20 |              11
 g         |            6 |               5
 h         |           20 |               7
 i         |           10 |               4
(9 rows)
```

`assembly_nbr = 12`を起点に探索を開始するイメージは下記の通り。

![クイックノート P1](https://user-images.githubusercontent.com/65038325/187591365-d254da4b-b37b-4079-a9ed-92b2b61ceeaa.png)

```sql
with recursive explosion (assembly_nbr, subassembly_nbr, part_name) as (
 select bom.assembly_nbr, bom.subassembly_nbr, bom.part_name
 from billofmaterials as bom
 where bom.assembly_nbr = 12 -- 探索の開始点
 union
 select child.assembly_nbr, child.subassembly_nbr, child.part_name
 from explosion as parent, billofmaterials as child
 where parent.subassembly_nbr = child.assembly_nbr)
select
  assembly_nbr,
  subassembly_nbr,
  part_name
from explosion;

 assembly_nbr | subassembly_nbr | part_name
--------------+-----------------+-----------
           12 |              10 | b
           12 |               9 | c
           10 |               8 | d
           10 |               4 | i
            8 |               7 | e
(5 rows)
```

## :closed_book: Reference

- [再帰 SQL -図解-](https://qiita.com/Shoyu_N/items/f1786f99545fa5053b75)
- [Recursive CTE で多階層カテゴリのレコードを取得](https://www.wakuwakubank.com/posts/597-mysql-recursive-cte/)
- [PostgreSQL の再帰 SQL の使用例](https://oraclesqlpuzzle.ninja-web.net/postgresql-rec-with.html#1)
- [プログラマのための SQL 第 4 版 すべてを知り尽くしたいあなたに](https://www.shoeisha.co.jp/book/detail/9784798128023)
