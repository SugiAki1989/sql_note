## :memo: Overview

PostgreSQL には`rollup`関数、`grouping sets`関数という小計を計算してくれる関数や、柔軟な集合を作って集計できる関数が用意されている。ここでは、`rollup`関数、`grouping sets`関数の基本的な使い方をまとめていおく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`rollup`, `grouping sets`, `nulls last`

## :pencil2: Example

カテゴリごとの集計値と合計を 1 つのテーブルに出力しようとすると、`union`を利用する必要がある。

```sql
select 'total' as species, sum(flipper_length_mm) as sum from penguins
union all
select
    species,
    sum(flipper_length_mm) as sum
from
    penguins
group by
    species
;

  species  |  sum
-----------+-------
 total     | 68713
 Adelie    | 28683
 Gentoo    | 26714
 Chinstrap | 13316
(4 rows)

```

これと同じことは`rollup`を利用すれば簡潔に計算できる。`coalesce`は`rollup`が`null`を返すので、変換している。

```sql
select
    coalesce(species, 'Rollup-total') as species,
    sum(flipper_length_mm) as sum
from
    penguins
group by
    rollup(species)
;

   species    |  sum
--------------+-------
 Rollup-total | 68713
 Adelie       | 28683
 Gentoo       | 26714
 Chinstrap    | 13316
(4 rows)
```

`rollup`には複数のカラムを利用でき、こうすることで小計と合計を計算できる。

```sql
select
    species,
    island,
    sum(flipper_length_mm) as sum
from
    penguins
group by
    rollup(species, island)
order by
    species asc
;

  species  |  island   |  sum
-----------+-----------+-------
 Adelie    |           | 28683
 Adelie    | Torgersen |  9751
 Adelie    | Dream     | 10625
 Adelie    | Biscoe    |  8307
 Chinstrap |           | 13316
 Chinstrap | Dream     | 13316
 Gentoo    | Biscoe    | 26714
 Gentoo    |           | 26714
           |           | 68713
(9 rows)
```

`grouping sets`はより柔軟な部分集合の集計結果を返す。

```sql
select
    species,
    year,
    island,
    sum(flipper_length_mm) as sum
from
    penguins
group by
    grouping sets((species), (year), (island), ())
;

  species  | year |  island   |  sum
-----------+------+-----------+-------
           |      |           | 68713
 Adelie    |      |           | 28683
 Gentoo    |      |           | 26714
 Chinstrap |      |           | 13316
           | 2008 |           | 23119
           | 2009 |           | 24134
           | 2007 |           | 21460
           |      | Dream     | 23941
           |      | Torgersen |  9751
           |      | Biscoe    | 35021
(10 rows)
```

PostgreSQL で`nulls last`という書き方もできることに初めて気づいた。

```sql
select
    species,
    year,
    island,
    sum(flipper_length_mm) as sum
from
    penguins
group by
    grouping sets((species), (year), (island), ())
order by
    species, year, island nulls last
;

  species  | year |  island   |  sum
-----------+------+-----------+-------
 Adelie    |      |           | 28683
 Chinstrap |      |           | 13316
 Gentoo    |      |           | 26714
           | 2007 |           | 21460
           | 2008 |           | 23119
           | 2009 |           | 24134
           |      | Biscoe    | 35021
           |      | Dream     | 23941
           |      | Torgersen |  9751
           |      |           | 68713
(10 rows)
```

## :closed_book: Reference

- [7.2.4. GROUPING SETS、CUBE、ROLLUP](https://www.postgresql.jp/document/13/html/queries-table-expressions.html#QUERIES-GROUPING-SETS)
