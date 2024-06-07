## :memo: Overview

ここでは join の key と and 条件を同時に利用する場合の SQL の挙動をおさらいしておく。下記を参考にしている。

- [新人 SE のための SQL 外部結合 | 新人 SE のための製薬会社の IT 研究](https://pharma-it.net/left-join/)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`join`, `and`

## :pencil2: Example

まずはサンプルテーブルを用意する。

```sql
create table jt1 (key varchar(1));
INSERT INTO jt1 values ('8'), ('9'), ('a');

create table jt2 (key varchar(1));
INSERT INTO jt2 values ('7'), ('8'), ('9');

create table jt3 (key varchar(1));
INSERT INTO jt3 values ('9'), ('a'), ('b');

select key from jt1;
 key
-----
 8
 9
 a
(3 rows)

select key from jt2;
 key
-----
 7
 8
 9
(3 rows)

select key from jt3;
 key
-----
 9
 a
 b
(3 rows)
```

`親 - 2子`という関係であれば理解しやすい。特に何も考えずに返される結果を想像できる。

```sql
explain
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
left join jt2 on jt1.key = jt2.key
left join jt3 on jt1.key = jt3.key
;

 t1_key | t2_key |
--------+--------+
 8      | 8      |
 9      | 9      |
 a      | (null) |
↓
↓
↓
 t1_key | t2_key | t3_key
--------+--------+--------
 8      | 8      | (null)
 9      | 9      | 9
 a      | (null) | a
(3 rows)

                                  QUERY PLAN
-------------------------------------------------------------------------------
 Merge Left Join  (cost=475.52..5209.87 rows=288579 width=24)
   Merge Cond: ((jt1.key)::text = (jt3.key)::text)
   ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
         Sort Key: jt3.key
         ->  Seq Scan on jt3  (cost=0.00..32.60 rows=2260 width=8)
   ->  Materialize  (cost=317.01..775.23 rows=25538 width=16)
         ->  Merge Left Join  (cost=317.01..711.38 rows=25538 width=16)
               Merge Cond: ((jt1.key)::text = (jt2.key)::text)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt1.key
                     ->  Seq Scan on jt1  (cost=0.00..32.60 rows=2260 width=8)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt2.key
                     ->  Seq Scan on jt2  (cost=0.00..32.60 rows=2260 width=8)
(14 rows)
```

## `親 - 子 - 孫`のパターン 1

問題は、`親 - 子 - 孫`のパターン。つまり、「親をベースに子と孫」を紐づけるが、「親と孫」の紐づけは「子と孫」の関係を利用して紐づける。下記のようなイメージである。

[join-key](https://github.com/SugiAki1989/sql_note/blob/main/image/p147-1.png)


この後に類似ケースを記載するが、肝となるのは`left join jt3　on jt2.key = jt3.key`であり、`jt1`に`jt3`に情報を紐づけるには、`jt2`が処理される必要があるため、実行計画の最下部には`jt2`を含めた処理が行われる。

実行計画を見る限り、`jt2 - Merge Left Join - jt3`を処理してから、`MaterializeTable`を作成し、`jt1 - Merge Left Join - MaterializeTable`が紐づけられる。

```sql
explain
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
left join jt2 on jt1.key = jt2.key
left join jt3 on jt2.key = jt3.key
;

-- MaterializeTable
| t2_key | t3_key
+--------+-------
| 7      | (null)
| 8      | (null)
| 9      | 9
↓
↓
↓
 t1_key | t2_key | t3_key
--------+--------+--------
 8      | 8      | (null)
 9      | 9      | 9
 a      | (null) | (null)
(3 rows)

                                  QUERY PLAN
-------------------------------------------------------------------------------
 Merge Left Join  (cost=475.52..5209.87 rows=288579 width=24)
   Merge Cond: ((jt1.key)::text = (jt2.key)::text)
   ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
         Sort Key: jt1.key
         ->  Seq Scan on jt1  (cost=0.00..32.60 rows=2260 width=8)
   ->  Materialize  (cost=317.01..775.23 rows=25538 width=16)
         ->  Merge Left Join  (cost=317.01..711.38 rows=25538 width=16)
               Merge Cond: ((jt2.key)::text = (jt3.key)::text)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt2.key
                     ->  Seq Scan on jt2  (cost=0.00..32.60 rows=2260 width=8)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt3.key
                     ->  Seq Scan on jt3  (cost=0.00..32.60 rows=2260 width=8)
(14 rows)
```

## `親 - 子 - 孫`のパターン 2

2 つ目の `join` を `inner` に変更すると、結果は下記の通りとなる。実行計画を見る限り、指定した`left join`ではなく`jt1 - Merge Join - jt2`、`jt1.key = jt2.key`という条件で処理してから、`MaterializeTable`を作成し、`jt3 - Merge Join - MaterializeTable`が`jt3.key = jt1.key`という条件で紐づけられる。

```sql
explain
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
left join jt2 on jt1.key = jt2.key
inner join jt3 on jt2.key = jt3.key
;

-- MaterializeTable
| t1_key | t2_key
+--------+-------
| 8      | 8
| 9      | 9
↓
↓
↓
 t1_key | t2_key | t3_key
--------+--------+--------
 9      | 9      | 9
(1 row)

                                  QUERY PLAN
-------------------------------------------------------------------------------
 Merge Join  (cost=475.52..5209.87 rows=288579 width=24)
   Merge Cond: ((jt3.key)::text = (jt1.key)::text)
   ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
         Sort Key: jt3.key
         ->  Seq Scan on jt3  (cost=0.00..32.60 rows=2260 width=8)
   ->  Materialize  (cost=317.01..775.23 rows=25538 width=16)
         ->  Merge Join  (cost=317.01..711.38 rows=25538 width=16)
               Merge Cond: ((jt1.key)::text = (jt2.key)::text)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt1.key
                     ->  Seq Scan on jt1  (cost=0.00..32.60 rows=2260 width=8)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt2.key
                     ->  Seq Scan on jt2  (cost=0.00..32.60 rows=2260 width=8)
(14 rows)
```

なぜ指定した`jt1 - jt2`の紐づけは、`leftjoin`が利用されないのか。`leftjoin`は、`jt1`のすべての行と、`jt1.key = jt2.key`で一致する`jt2`の行を結合する。`jt2` に一致する行がない場合、`jt2` の列は `null` になる。すでに結合された結果に対して、`jt2.key = jt3.key` で一致する `jt1 - jt3` の行を結合する。`inner join` は、`jt2.key` が `null` の場合、その行は結果から除外する。つまり、最初から除外されるのであれば、`left join`する必要ないため、SQL のオプティマイザが実行計画をこのように組み立てたと思われる。実行計画通りに書き直すと同じ結果が得られる。

```
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
inner join jt2 on jt1.key = jt2.key
inner join jt3 on jt1.key = jt3.key
;

 t1_key | t2_key | t3_key
--------+--------+--------
 9      | 9      | 9
(1 row)
```

この結果がほしいのであれば下記パターン 3 で記述するほうがわかりやすい。

## `親 - 子 - 孫`のパターン 3

すべて `inner` で紐づけた場合はこのようになる。実行計画を見る限り、`jt1 - Merge Join - jt2`を処理してから、`MaterializeTable`を作成し、`jt3 - Merge Join - MaterializeTable`が紐づけられる。

```sql
explain
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
inner join jt2 on jt1.key = jt2.key
inner join jt3 on jt2.key = jt3.key
;

-- MaterializeTable
 t1_key | t2_key |
--------+--------+
 8      | 8      |
 9      | 9      |
↓
↓
↓
 t1_key | t2_key | t3_key
--------+--------+--------
 9      | 9      | 9
(1 row)

                                  QUERY PLAN
-------------------------------------------------------------------------------
 Merge Join  (cost=475.52..5209.87 rows=288579 width=24)
   Merge Cond: ((jt3.key)::text = (jt1.key)::text)
   ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
         Sort Key: jt3.key
         ->  Seq Scan on jt3  (cost=0.00..32.60 rows=2260 width=8)
   ->  Materialize  (cost=317.01..775.23 rows=25538 width=16)
         ->  Merge Join  (cost=317.01..711.38 rows=25538 width=16)
               Merge Cond: ((jt1.key)::text = (jt2.key)::text)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt1.key
                     ->  Seq Scan on jt1  (cost=0.00..32.60 rows=2260 width=8)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt2.key
                     ->  Seq Scan on jt2  (cost=0.00..32.60 rows=2260 width=8)
(14 rows)
```

ちなみに PostgreSQL では、逆だとエラーとなる模様。`jt2`テーブルがまだ参照される前に、`jt2.key`が`jt3`テーブルとの結合条件として使用されているためだと思われる。

```sql
explain
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
inner join jt3 on jt2.key = jt3.key
inner join jt2 on jt1.key = jt2.key
;
ERROR:  missing FROM-clause entry for table "jt2"
LINE 4: inner join jt3 on jt2.key = jt3.key
```

## `親 - 子 - 孫`のパターン 4

先ほどとは反対で、2 つ目の `join` を `left` に変更すると、返される結果は異なる。実行計画を見る限り、`jt1 - jt2`を処理してから、`MaterializeTable`を作成し、`jt3 - MaterializeTable`が紐づけられる。

```sql
explain
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
inner join jt2 on jt1.key = jt2.key
left join jt3 on jt2.key = jt3.key
;

-- MaterializeTable
| t1_key | t2_key
+--------+-------
| 8      | 8
| 9      | 9
↓
↓
↓
 t1_key | t2_key | t3_key
--------+--------+--------
 8      | 8      | (null)
 9      | 9      | 9
(2 rows)

                                  QUERY PLAN
-------------------------------------------------------------------------------
 Merge Left Join  (cost=475.52..5209.87 rows=288579 width=24)
   Merge Cond: ((jt2.key)::text = (jt3.key)::text)
   ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
         Sort Key: jt3.key
         ->  Seq Scan on jt3  (cost=0.00..32.60 rows=2260 width=8)
   ->  Materialize  (cost=317.01..775.23 rows=25538 width=16)
         ->  Merge Join  (cost=317.01..711.38 rows=25538 width=16)
               Merge Cond: ((jt1.key)::text = (jt2.key)::text)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt1.key
                     ->  Seq Scan on jt1  (cost=0.00..32.60 rows=2260 width=8)
               ->  Sort  (cost=158.51..164.16 rows=2260 width=8)
                     Sort Key: jt2.key
                     ->  Seq Scan on jt2  (cost=0.00..32.60 rows=2260 width=8)
(14 rows)
```

## まとめ

まとめると、`親 - 子 - 孫`のパターンを記述したいのであれば、`inner`か`left`で統一するのが望ましいが、要件的に困難なのであれば、`with`でステップを刻んで可読性を高めるとかしたほうがいいかも。今回のケース用のようにマスタテーブル同士ではなく、トランザクション系のテーブルを利用するのであれば、なおさら何をやっているのか理解しにくくなる。

# 応用

## `親 - 子 - 孫`のパターン 2(応用)

2 つ目の `join` を `inner` に変更するパターンに`and`条件を追加すると、結果は下記の通りとなる。

```sql
explain
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
left join jt2 on jt1.key = jt2.key and jt2.key = '8'
inner join jt3 on jt2.key = jt3.key
;

-- Nested Loop
| t1_key | t2_key
+--------+-------
| 8      | 8
↓
↓
↓
 t1_key | t2_key | t3_key
--------+--------+--------
(0 rows)

                              QUERY PLAN
-----------------------------------------------------------------------
 Nested Loop  (cost=0.00..132.95 rows=1331 width=24)
   ->  Nested Loop  (cost=0.00..78.04 rows=121 width=16)
         ->  Seq Scan on jt1  (cost=0.00..38.25 rows=11 width=8)
               Filter: ((key)::text = '8'::text)
         ->  Materialize  (cost=0.00..38.30 rows=11 width=8)
               ->  Seq Scan on jt2  (cost=0.00..38.25 rows=11 width=8)
                     Filter: ((key)::text = '8'::text)
   ->  Materialize  (cost=0.00..38.30 rows=11 width=8)
         ->  Seq Scan on jt3  (cost=0.00..38.25 rows=11 width=8)
               Filter: ((key)::text = '8'::text)
(10 rows)


-------------------------------------
-- and条件を入れる前
-------------------------------------

 t1_key | t2_key | t3_key
--------+--------+--------
 9      | 9      | 9
(1 row)
```

## `親 - 子 - 孫`のパターン 4(応用)

先ほどとは反対で、2 つ目の `join` を `left` に変更するパターンに`and`条件を追加すると、返される結果は下記の通り。。

```sql
explain
select jt1.key as t1_key, jt2.key as t2_key, jt3.key as t3_key
from jt1
inner join jt2 on jt1.key = jt2.key and jt2.key = '8'
left join jt3 on jt2.key = jt3.key
;

-- Nested Loop
| t1_key | t2_key
+--------+-------
| 8      | 8
↓
↓
↓
 t1_key | t2_key | t3_key
--------+--------+--------
 8      | 8      | (null)
(1 row)

                              QUERY PLAN
-----------------------------------------------------------------------
 Hash Left Join  (cost=38.39..130.19 rows=1331 width=24)
   Hash Cond: ((jt2.key)::text = (jt3.key)::text)
   ->  Nested Loop  (cost=0.00..78.04 rows=121 width=16)
         ->  Seq Scan on jt1  (cost=0.00..38.25 rows=11 width=8)
               Filter: ((key)::text = '8'::text)
         ->  Materialize  (cost=0.00..38.30 rows=11 width=8)
               ->  Seq Scan on jt2  (cost=0.00..38.25 rows=11 width=8)
                     Filter: ((key)::text = '8'::text)
   ->  Hash  (cost=38.25..38.25 rows=11 width=8)
         ->  Seq Scan on jt3  (cost=0.00..38.25 rows=11 width=8)
               Filter: ((key)::text = '8'::text)
(11 rows)

-------------------------------------
-- and条件を入れる前
-------------------------------------

 t1_key | t2_key | t3_key
--------+--------+--------
 8      | 8      | (null)
 9      | 9      | 9
(2 rows)
```

## :closed_book: Reference

- [新人 SE のための SQL 外部結合 | 新人 SE のための製薬会社の IT 研究](https://pharma-it.net/left-join/)
