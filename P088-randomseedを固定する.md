## :memo: Overview

ランダムにテストデータを生成したい場合があったりする。そして、可能であれば、再現性を担保するためにランダムに生成される値を固定したい。PostgreSQL ではできないと思っていたけど、シードを固定できたので、メモしておく。すごく簡単な話だった。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`random`, `seed`

## :pencil2: Example

`random`関数を利用すると、毎回異なる値が生成される。

```sql
select ceil(random() * 10) AS val;
 val
-----
   1
(1 row)

select ceil(random() * 10) AS val;
 val
-----
   6
(1 row)

select ceil(random() * 10) AS val;
 val
-----
   2
(1 row)

```

`setseed`関数を実行すれば一時的に固定できるが、あくまでも関数を実行したあとの結果は固定できるが、再度実行すると異なる値が生成される。

```sql
select setseed(0.1989);
 setseed
---------

(1 row)

select ceil(random() * 10) AS val from generate_series(1, 5);
 val
-----
   6
   7
   6
   6
   2
(5 rows)

select ceil(random() * 10) AS val from generate_series(1, 5);
 val
-----
   7
   9
  10
   3
   9
(5 rows)
```

ということであれば、セッションが変わっても、再実行時でも、`setseed`関数を先に実行しておけばよい。すごく単純なことに気が付かなかった。

```sql
select setseed(0.1989);
 setseed
---------

(1 row)

select ceil(random() * 10) AS val from generate_series(1, 5);

 val
-----
   6
   7
   6
   6
   2
(5 rows)

select setseed(0.1989);
 setseed
---------

(1 row)

select ceil(random() * 10) AS val from generate_series(1, 5);

 val
-----
   6
   7
   6
   6
   2
(5 rows)
```

何らかのテーブルがあったときに、毎回同じレコードを抽出することも可能。

```sql
select setseed(0.1989);
 setseed
---------

(1 row)

with tmp as (
select
    random(),
    rowid::text || '-' || species || '-' || island as penguin,
    row_number() over (order by random()) as ind
from
    penguins
order by
    random()
)
select * from tmp where ind <= 10;

        random         |       penguin       | ind
-----------------------+---------------------+-----
 0.0007780243303514567 | 280-Chinstrap-Dream |   1
 0.0015331067629489326 | 290-Chinstrap-Dream |   2
 0.0023207163582057433 | 300-Chinstrap-Dream |   3
  0.009880334624092768 | 304-Chinstrap-Dream |   4
  0.011869371227916758 | 34-Adelie-Dream     |   5
  0.012049122590404693 | 308-Chinstrap-Dream |   6
     0.018820064746496 | 145-Adelie-Dream    |   7
  0.019691009578327368 | 337-Chinstrap-Dream |   8
  0.021628641961743966 | 77-Adelie-Torgersen |   9
  0.022619479449204505 | 138-Adelie-Dream    |  10
(10 rows)

select setseed(0.1989);
 setseed
---------

(1 row)

        random         |       penguin       | ind
-----------------------+---------------------+-----
 0.0007780243303514567 | 280-Chinstrap-Dream |   1
 0.0015331067629489326 | 290-Chinstrap-Dream |   2
 0.0023207163582057433 | 300-Chinstrap-Dream |   3
  0.009880334624092768 | 304-Chinstrap-Dream |   4
  0.011869371227916758 | 34-Adelie-Dream     |   5
  0.012049122590404693 | 308-Chinstrap-Dream |   6
     0.018820064746496 | 145-Adelie-Dream    |   7
  0.019691009578327368 | 337-Chinstrap-Dream |   8
  0.021628641961743966 | 77-Adelie-Torgersen |   9
  0.022619479449204505 | 138-Adelie-Dream    |  10
(10 rows)
```

おまけとして`aaray`を利用してランダムに値を生成する方法をまとめておく。

```sql
select
    (array['A', 'B', 'C', 'D', 'E'])[ceil(random() * 5)] as alphabet
from
    generate_series(1, 10);

 alphabet
----------
 A
 B
 D
 A
 D
 D
 A
 E
 D
 D
(10 rows)
```

## :closed_book: Reference

None
