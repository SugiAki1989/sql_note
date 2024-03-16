## :memo: Overview

面白い問題が[SQL 実践入門 ── 高速でわかりやすいクエリの書き方](https://gihyo.jp/book/2015/978-4-7741-7301-6)の P261 にのっていたので、その問題を参考にさせていただきました。問題の内容は、`null`を`update`で埋める問題だけど簡単には埋められない問題。ここでは`update`するわけではなく、単純に`select`で`null`を埋める例をまとめる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`subquery`

## :pencil2: Example

この問題のようなデータは、紙媒体を電子化する際に、人間のタイピング入力作業を減らすことで生まれたデータと説明されている。ですが、私個人としては、なぜかは知らないが EC の注文データでこうなっているものを見たことがある。

```sql
create table fillup(id varchar(5), seq integer, val integer);
insert into fillup
    (id, seq, val)
values
    ('a', '1', '10'),
    ('a', '2', null),
    ('a', '3', null),
    ('a', '4', '20'),
    ('a', '5', null),
    ('a', '6', '30'),
    ('b', '1', '10'),
    ('b', '2', '20'),
    ('b', '3', null),
    ('b', '4', '30'),
    ('b', '5', null),
    ('b', '6', null)
;
```

下記のようなデータで、同じ値は頭だけ入力し、同じ値の入力は避ける、値が変われば入力し、同じ値は避けるという記録のされ方をしている。このままでは集計で値が利用できないので、埋める必要がある。

```sql
select * from fillup;
 id | seq | val
----+-----+-----
 a  |   1 |  10
 a  |   2 |
 a  |   3 |
 a  |   4 |  20
 a  |   5 |
 a  |   6 |  30
 b  |   1 |  10
 b  |   2 |  20
 b  |   3 |
 b  |   4 |  30
 b  |   5 |
 b  |   6 |
```

この`null` を埋めるためにはカレント行だけでは難しく、行をまたいで値を絞り込無必要がある。例えば、2,3 行目は 1 行目の 10 を取ってくる必要がある。条件を書き下すと下記のようになる。

- 同じ`id`で
- カレント行`seq`より小さく
- `val`が`null`ではない

相関サブクエリで、この条件で値を取ってくれば、うまいこと`null`を埋めることができる。

```sql
select
    id, seq, val,
    case when val is null then
        (select max(val)
        from fillup as f2
        where
            f2.id = f1.id and
            f2.seq < f1.seq and
            f2.val is not null
        )
    else f1.val end as fillval
from
    fillup as f1
;

 id | seq | val | fillval
----+-----+-----+---------
 a  |   1 |  10 |      10
 a  |   2 |     |      10
 a  |   3 |     |      10
 a  |   4 |  20 |      20
 a  |   5 |     |      20
 a  |   6 |  30 |      30
 b  |   1 |  10 |      10
 b  |   2 |  20 |      20
 b  |   3 |     |      20
 b  |   4 |  30 |      30
 b  |   5 |     |      30
 b  |   6 |     |      30
```

SQL だけではイメージしづらいので、動作イメージをのせておく。あくまでもイメージなので実際の動作とは異なる。

![nullうめ](https://user-images.githubusercontent.com/65038325/182362199-0863faec-29f3-4a82-a266-787f66fbd569.png)

実行計画を見るとわかりますが、`f2`から実行されているので、イメージ図のように外側の`f1`から実行されているわけではないので注意が必要。

```sql
                                           QUERY PLAN
-------------------------------------------------------------------------------------------------
 Seq Scan on fillup f1  (cost=0.00..41388.00 rows=1360 width=36)
   SubPlan 1
     ->  Aggregate  (cost=30.40..30.41 rows=1 width=4)
           ->  Seq Scan on fillup f2  (cost=0.00..30.40 rows=2 width=4)
                 Filter: ((val IS NOT NULL) AND (seq < f1.seq) AND ((id)::text = (f1.id)::text))
```

このようなクエリが思いつくような頭になりたい…orz

ただ、このクエリはこのテーブルだから上手く行ってるだけで、必ず`null`の一番近い上の値を取得しているわけではない。下記のように`null`の一番近い上の値よりも小さい値をいれると、想定通り動かなくなる。

```sql
create table fillup2(id varchar(5), seq integer, val integer);
insert into fillup2
    (id, seq, val)
values
    ('a', '1', '10'),
    ('a', '2', null),
    ('a', '3', null),
    ('a', '4',  '5'), -- 10よりも小さい
    ('a', '5', null),
    ('a', '6', '30'),
    ('b', '1', '10'),
    ('b', '2', '20'),
    ('b', '3', null),
    ('b', '4', '15'), -- ２0よりも小さい
    ('b', '5', null),
    ('b', '6', null)
;
```

同じクエリを実行すると、思ったようには動かない。これは`max()`を利用しているからである。

```sql
select
    id, seq, val,
    case when val is null then
        (select max(val)
        from fillup2 as f2
        where
            f2.id = f1.id and
            f2.seq < f1.seq and
            f2.val is not null
        )
    else f1.val end as fillval
from
    fillup2 as f1
;

 id | seq | val  | fillval
----+-----+------+---------
 a  |   1 |   10 |      10
 a  |   2 | NULL |      10
 a  |   3 | NULL |      10
 a  |   4 |    5 |       5
 a  |   5 | NULL |      10 --本当は5がほしい
 a  |   6 |   30 |      30
 b  |   1 |   10 |      10
 b  |   2 |   20 |      20
 b  |   3 | NULL |      20
 b  |   4 |   15 |      15
 b  |   5 | NULL |      20 --本当は15がほしい
 b  |   6 | NULL |      20 --本当は15がほしい
(12 rows)
```

区切り線を入れると修正の方向性がみえやすい。`id`、`seq`で条件はついているが、この条件を満たす場合、区切り線を超えて、値を取得してしまう。そのため、複数の値が存在すると、集計関数の性質により、ほしい値とは異なる値が取得されてしまう。そのため、区切り線をまたがないようにする条件を追加する。

```sql
 id | seq | val  | fillval
----+-----+------+---------
 a  |   1 |   10 |      10
 a  |   2 | NULL |      10
 a  |   3 | NULL |      10
--------------------------------------
 a  |   4 |    5 |       5
 a  |   5 | NULL |      10 --本当は5がほしい
--------------------------------------
 a  |   6 |   30 |      30
--------------------------------------
 b  |   1 |   10 |      10
 --------------------------------------
 b  |   2 |   20 |      20
 b  |   3 | NULL |      20
 --------------------------------------
 b  |   4 |   15 |      15
 b  |   5 | NULL |      20 --本当は15がほしい
 b  |   6 | NULL |      20 --本当は15がほしい
 --------------------------------------
(12 rows)
```

ということで、区切りがないので作ればよい。それから、相関サブクエリを利用すればよさそう。

```sql
with tmp as (
select
    id,
    seq,
    val,
    case when val is not null then 1 else 0 end as flag,
    lag(case when val is not null then 1 else 0 end, 1, 0) over(order by id asc, seq asc) as lagflag
from
    fillup2
)
select
    id,
    seq,
    val,
    flag,
    lagflag,
    case when (flag = 1 and lagflag = 0) then seq else 0 end as g,
    max(case when (flag = 1 and lagflag = 0) then seq else 0 end) over(partition by id order by seq asc) as ggg
from tmp;

 id | seq | val  | flag | lagflag | g | ggg
----+-----+------+------+---------+---+-----
 a  |   1 |   10 |    1 |       0 | 1 |   1
 a  |   2 | NULL |    0 |       1 | 0 |   1
 a  |   3 | NULL |    0 |       0 | 0 |   1
 a  |   4 |    5 |    1 |       0 | 4 |   4
 a  |   5 | NULL |    0 |       1 | 0 |   4
 a  |   6 |   30 |    1 |       0 | 6 |   6
 b  |   1 |   10 |    1 |       1 | 0 |   0
 b  |   2 |   20 |    1 |       1 | 0 |   0
 b  |   3 | NULL |    0 |       1 | 0 |   0
 b  |   4 |   15 |    1 |       0 | 4 |   4
 b  |   5 | NULL |    0 |       1 | 0 |   4
 b  |   6 | NULL |    0 |       0 | 0 |   4
```

区切りができたので、あとは相関サブクエリの条件を追加すれば`max()`でも問題なく`null`を思った通りの値で埋めることができる。

```sql
with tmp as (
select
    id,
    seq,
    val,
    case when val is not null then 1 else 0 end as flag,
    -- lag(flag, 1, 0) over(order by id asc, seq asc) as lagflag
    lag(case when val is not null then 1 else 0 end, 1, 0) over(order by id asc, seq asc) as lagflag
from
    fillup2
), tmp2 as (
select
    id,
    seq,
    val,
    flag,
    lagflag,
    case when (flag = 1 and lagflag = 0) then seq else 0 end as g,
    -- max(g) over(partition by id order by seq asc) as ggg
    max(case when (flag = 1 and lagflag = 0) then seq else 0 end) over(partition by id order by seq asc) as ggg
from tmp
)
select
    id, seq, val, flag, lagflag, g, ggg,
    case when val is null then
        (select max(val)
        from tmp2 as f2
        where
            f2.id = f1.id and
            f2.seq < f1.seq and
            f2.val is not null and
            f2.ggg = f1.ggg
        )
    else f1.val end as fillval
from
    tmp2 as f1
;

 id | seq | val  | flag | lagflag | g | ggg | fillval
----+-----+------+------+---------+---+-----+---------
 a  |   1 |   10 |    1 |       0 | 1 |   1 |      10
 a  |   2 | NULL |    0 |       1 | 0 |   1 |      10
 a  |   3 | NULL |    0 |       0 | 0 |   1 |      10
 a  |   4 |    5 |    1 |       0 | 4 |   4 |       5
 a  |   5 | NULL |    0 |       1 | 0 |   4 |       5
 a  |   6 |   30 |    1 |       0 | 6 |   6 |      30
 b  |   1 |   10 |    1 |       1 | 0 |   0 |      10
 b  |   2 |   20 |    1 |       1 | 0 |   0 |      20
 b  |   3 | NULL |    0 |       1 | 0 |   0 |      20
 b  |   4 |   15 |    1 |       0 | 4 |   4 |      15
 b  |   5 | NULL |    0 |       1 | 0 |   4 |      15
 b  |   6 | NULL |    0 |       0 | 0 |   4 |      15
(12 rows)
```

## :closed_book: Reference

- [SQL 実践入門 ── 高速でわかりやすいクエリの書き方](https://gihyo.jp/book/2015/978-4-7741-7301-6)


