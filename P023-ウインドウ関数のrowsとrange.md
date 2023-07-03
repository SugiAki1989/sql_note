## :memo: Overview

ウインドウ関数のフレーム句には`rows`と`range`が使用することができ、`rows`は行単位での指定ができる一方で、`range`はカラムの値単位で指定ができる。これらの`rows`と`range`の違いについてまとめておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`range`, `rows`, `over`

## :pencil2: Example

ウインドウ関数は下記の構文となっており、`frame`を調整することで計算対象の範囲を調整できる。`frame`には`rows`と`range`が使用できる。

```sql
sum({col}) over(
    partition by {col}
    order by {col}
    frame
)
```

`rows`と`range`の違いを見るために、`2022-08-04`が少しおかしいサンプルデータを用意する。

```sql
create table winframe(dt date, val integer);
insert into winframe
    (dt, val)
values
    ('2022-08-01', 1),
    ('2022-08-02', 2),
    ('2022-08-03', 3),
    ('2022-08-04', 4),
    ('2022-08-04', 5),
    ('2022-08-04', null),
    ('2022-08-04', 6),
    ('2022-08-05', 7),
    ('2022-08-06', 8)
;
```

`rows`は常に行を優先するため、日付や値が同じだろうと、行が変わるのであれば「1」と考えるので、`2022-08-04`は何行あっても各行ごとに計算が実行される。

```sql
select
    dt,
    val,
    sum(val) over(order by dt asc rows between 1 preceding and current row) as rowsframe
from
    winframe
;

     dt     | val | rowsframe |
------------+-----+-----------+
 2022-08-01 |   1 |         1 | = 1
 2022-08-02 |   2 |         3 | = 1 + 2
 2022-08-03 |   3 |         5 | = 2 + 3
 2022-08-04 |   4 |         7 | = 3 + 4
 2022-08-04 |   5 |         9 | = 4 + 5
 2022-08-04 |     |         5 | = 5 + null
 2022-08-04 |   6 |         6 | = null + 6
 2022-08-05 |   7 |        13 | = 6 + 7
 2022-08-06 |   8 |        15 | = 7 + 8
```

一方でこの例の`range`では、1 日(単位)は同じ日とみなしてフレームを調整するため、`2022-08-04`の部分の各行の計算は変わらない。

`2022-08-05`の部分の計算を見るとわかりやすいが、1 日(単位)は同じとみなすので、`2022-08-04`すべての行が計算対象になっている。

```sql
select
    dt,
    val,
    sum(val) over(order by dt asc range between interval '1' day preceding and current row) as rangeframe
from
    winframe
;

     dt     | val | rangeframe|
------------+-----+-----------|
 2022-08-01 |   1 |          1| = 1
 2022-08-02 |   2 |          3| = 1 + 2
 2022-08-03 |   3 |          5| = 2 + 3
 2022-08-04 |   4 |         18| = 3 + (4 + 5 + null + 6)
 2022-08-04 |   5 |         18| = 3 + (4 + 5 + null + 6)
 2022-08-04 |     |         18| = 3 + (4 + 5 + null + 6)
 2022-08-04 |   6 |         18| = 3 + (4 + 5 + null + 6)
 2022-08-05 |   7 |         22| = (4 + 5 + null + 6) + 7
 2022-08-06 |   8 |         15| = 7 + 8
```

## :closed_book: Reference

None
