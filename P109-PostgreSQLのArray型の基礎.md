## :memo: Overview

ここでは PostgreSQL で配列を使用する場合の基本的なクエリの使い方をまとめておく。PostgreSQL では、テーブルの列を可変長の多次元配列として定義できる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`array`, ``,

## :pencil2: Example

```sql
create table array_tbl (
  rownum serial primary key,
  products text array
  );

insert into
  array_tbl(products)
values
  (array['apple', 'banana']),
  (array['orange', 'peach']),
  (array['apple', 'banana', 'orange']),
  (array['apple', 'banana', 'peach']),
  (array[]::text[]),
  (array['']::text[]),
  (array['banana', 'peach']);

select
  *
from
  array_tbl;

 rownum |       products
--------+-----------------------
      1 | {apple,banana}
      2 | {orange,peach}
      3 | {apple,banana,orange}
      4 | {apple,banana,peach}
      5 | {}
      6 | {""}
      7 | {banana,peach}
(7 rows)
```

配列へのアクセスは他のプログラミング言語同様、インデックスを利用する。スライスも利用できる。

```sql
select
  rownum,
  products,
  products[1:2]
from
  array_tbl
;
 rownum |       products        |    products
--------+-----------------------+----------------
      1 | {apple,banana}        | {apple,banana}
      2 | {orange,peach}        | {orange,peach}
      3 | {apple,banana,orange} | {apple,banana}
      4 | {apple,banana,peach}  | {apple,banana}
      5 | {}                    | {}
      6 | {""}                  | {""}
      7 | {banana,peach}        | {banana,peach}
(7 rows)

select
  rownum,
  products,
  products[2:]
from
  array_tbl
;
 rownum |       products        |    products
--------+-----------------------+-----------------
      1 | {apple,banana}        | {banana}
      2 | {orange,peach}        | {peach}
      3 | {apple,banana,orange} | {banana,orange}
      4 | {apple,banana,peach}  | {banana,peach}
      5 | {}                    | {}
      6 | {""}                  | {}
      7 | {banana,peach}        | {peach}
(7 rows)
```

`-1`で末尾を取得する、とはならないので注意。

```sql
select
  rownum,
  products[-1]
from
  array_tbl
;

 rownum | products
--------+----------
      1 |
      2 |
      3 |
      4 |
      5 |
      6 |
      7 |
(7 rows)
```

`array_position`は値の位置を教えてくれる。

```sql

select
  rownum,
  array_position(products, 'banana')
from
  array_tbl
;

 rownum | array_position
--------+----------------
      1 |              2
      2 |
      3 |              2
      4 |              2
      5 |
      6 |
      7 |              1
(7 rows)
```

`array_dims`は配列の次元を返してくれる。

```sql
select
  rownum,
  products,
  array_dims(products)
from
  array_tbl
;

 rownum |       products        | array_dims
--------+-----------------------+------------
      1 | {apple,banana}        | [1:2]
      2 | {orange,peach}        | [1:2]
      3 | {apple,banana,orange} | [1:3]
      4 | {apple,banana,peach}  | [1:3]
      5 | {}                    |
      6 | {""}                  | [1:1]
      7 | {banana,peach}        | [1:2]
(7 rows)
```

`array_upper`でも次元を取ることはできる。

```sql
select
  rownum,
  products,
  array_dims(products),
  array_upper(products,1)
from
  array_tbl
;

 rownum |       products        | array_dims | array_upper
--------+-----------------------+------------+-------------
      1 | {apple,banana}        | [1:2]      |           2
      2 | {orange,peach}        | [1:2]      |           2
      3 | {apple,banana,orange} | [1:3]      |           3
      4 | {apple,banana,peach}  | [1:3]      |           3
      5 | {}                    |            |
      6 | {""}                  | [1:1]      |           1
      7 | {banana,peach}        | [1:2]      |           2
(7 rows)
```

値の長さは`cardinality`や`array_length`で調べることができる。ただ、`null`に対する扱いが少し異なるので注意。`cardinality`は 0 を返すが、`array_length`は`null`を返す。

```sql
select
  rownum,
  products,
  array_length(products,1),
  cardinality(products)
from
  array_tbl
;

 rownum |       products        | array_length | cardinality
--------+-----------------------+--------------+-------------
      1 | {apple,banana}        |            2 |           2
      2 | {orange,peach}        |            2 |           2
      3 | {apple,banana,orange} |            3 |           3
      4 | {apple,banana,peach}  |            3 |           3
      5 | {}                    |              |           0
      6 | {""}                  |            1 |           1
      7 | {banana,peach}        |            2 |           2
(7 rows)
```

配列のネストを解除する場合は、`unnest`を利用する。配列の長さ分を立てに積んだ状態で結果を返す。`rownum=5`は`null`なので、結果には含まれないが、`rownum=6`は`''`空文字なので、結果には含まれる。事実上、`cardinality`が生成する行数を返す。

```sql
select
  rownum,
  unnest(products)
from
  array_tbl
;

 rownum | unnest
--------+--------
      1 | apple
      1 | banana
      2 | orange
      2 | peach
      3 | apple
      3 | banana
      3 | orange
      4 | apple
      4 | banana
      4 | peach
      6 |
      7 | banana
      7 | peach
(13 rows)
```

ネストを解除したものは、`array_agg`で配列に再パッキングできる。

```sql
with tmp as (
select
  rownum,
  unnest(products) as ps
from
  array_tbl
)
select
  rownum,
  array_agg(ps)
from
  tmp
group by
  rownum
;

 rownum |       array_agg
--------+-----------------------
      3 | {apple,banana,orange}
      4 | {apple,banana,peach}
      6 | {""}
      2 | {orange,peach}
      7 | {banana,peach}
      1 | {apple,banana}
(6 rows)

```

`array_cat`や`array_prepend`、`array_append`で配列に値を追加できる。`array_cat`は配列を値として渡すが、他のはそうでもないので注意。

```sql
select
  rownum,
  array_cat('{fruits}', products),
  array_prepend('fruits',products),
  array_append(products, 'fruits')
from
  array_tbl
;

 rownum |          array_cat           |        array_prepend         |         array_append
--------+------------------------------+------------------------------+------------------------------
      1 | {fruits,apple,banana}        | {fruits,apple,banana}        | {apple,banana,fruits}
      2 | {fruits,orange,peach}        | {fruits,orange,peach}        | {orange,peach,fruits}
      3 | {fruits,apple,banana,orange} | {fruits,apple,banana,orange} | {apple,banana,orange,fruits}
      4 | {fruits,apple,banana,peach}  | {fruits,apple,banana,peach}  | {apple,banana,peach,fruits}
      5 | {fruits}                     | {fruits}                     | {fruits}
      6 | {fruits,""}                  | {fruits,""}                  | {"",fruits}
      7 | {fruits,banana,peach}        | {fruits,banana,peach}        | {banana,peach,fruits}
(7 rows)

select array_cat(array[1,2], array[3,4]);
 array_cat
-----------
 {1,2,3,4}
(1行)

select array_prepend(1, array[2,3]);
 array_prepend
---------------
 {1,2,3}
(1行)

select array_append(array[1,2], 3);
 array_append
--------------
 {1,2,3}
(1行)
```

検索を行う場合は`any`や`all`を使って行う。

```sql
select
  rownum,
  products
from
  array_tbl
where
  'apple' = any(products)
;

 rownum |       products
--------+-----------------------
      1 | {apple,banana}
      3 | {apple,banana,orange}
      4 | {apple,banana,peach}
(3 rows)
```

`@>`で`hoge`を含む検索を行うことができる。`contains`のようなもの。

```sql

select
  rownum,
  products
from
  array_tbl
where
  products @> array['peach']
;

 rownum |       products
--------+----------------------
      2 | {orange,peach}
      4 | {apple,banana,peach}
      7 | {banana,peach}
(3 rows)

select
  rownum,
  products
from
  array_tbl
where
  products @> array['orange', 'peach']
;

 rownum |    products
--------+----------------
      2 | {orange,peach}
(1 row)
```

「含まない」を検索するときは`not`を利用する。

```sql
select
  rownum,
  products
from
  array_tbl
where
  not(products @> array['apple'])
;

 rownum |    products
--------+----------------
      2 | {orange,peach}
      5 | {}
      6 | {""}
      7 | {banana,peach}
(4 rows)

```

`all`を使うためにテーブルを再作成する。

```sql
create table array_tbl2 (
  rownum serial primary key,
  products text array,
  prices integer array
  );

insert into
  array_tbl2(products, prices)
values
  (array['apple', 'banana'], array[100, 150]),
  (array['orange', 'peach'], array[50, 200]),
  (array['apple', 'banana', 'orange'], array[100, 150, 50]),
  (array['apple', 'banana', 'peach'], array[100, 150, 300]),
  (array[]::text[], array[]::integer[]),
  (array['']::text[], array[]::integer[]),
  (array['banana_sp', 'peach'], array[300, 300]); -- for all

select
  *
from
  array_tbl2
where
  300 = all(prices)
;

 rownum |     products      |  prices
--------+-------------------+-----------
      5 | {}                | {}
      6 | {""}              | {}
      7 | {banana_sp,peach} | {300,300}
(3 rows)
```

## :closed_book: Reference

- [8.15. Arrays](https://www.postgresql.org/docs/15/arrays.html)
