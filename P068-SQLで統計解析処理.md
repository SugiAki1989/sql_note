## :memo: Overview

ここでは、平均、分散、共分散、相関係数、単回回帰のパラメタを SQL で手計算する方法をまとめておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`variance`, `covariance`, `correlation`

## :pencil2: Example

SQL で計算する必要はないく、関数が用意されておりので、こちらを使うほうが簡便。

```sql
select
    corr(x, y) as correlation,
    covar_pop(x, y) as covariance_pop,
    covar_samp(x, y) as covariance,
    var_pop(x) as variance_pop,
    var_samp(x) as variance,
    -- 向きがあるのでy,x
    regr_slope(y, x) as slope,
    regr_intercept (y, x) as intercept
from
    cor
;

    correlation     | covariance_pop | covariance | variance_pop | variance |  slope   | intercept
--------------------+----------------+------------+--------------+----------+----------+-----------
 0.6335175425702007 |            188 |        235 |          256 |      320 | 0.734375 |   24.0625
(1 row)
```

まずはサンプルデータを作成する。

```sql
create table cor(x real, y real);
insert into cor(x, y)
values
    ('50','50'),
    ('50','70'),
    ('80','60'),
    ('70','90'),
    ('90','100');
```

まずは平均。

```sql
select sum(x) / count(*) as avg from cor;
 avg
-----
  68
(1 row)

select avg(x) as avg from cor;
 avg
-----
  68
(1 row)
```

次は分散。

```sql
with tmp as (
select
    count(*) over () as n,
    x,
    avg(x) over () as x_avg
from cor
)
select
    (1.0 / n) * sum((x - x_avg)^2) as variance
from tmp
group by n;

 variance
----------
      256
(1 row)
```

そして、共分散。

```sql
with tmp as (
select
    count(*) over () as n,
    x, avg(x) over () as x_avg,
    y, avg(y) over () as y_avg
from
    cor
)
select
    -- {(x - E[X])(y - E[Y])}/ n
    ((1.0 / n) * sum((x - x_avg) * (y - y_avg))) as covariance
from tmp
group by n;
;
 covariance
------------
        188
(1 row)
```

これらを組み合わせて、共分散を計算。

```sql
with tmp as (
select
    count(*) as n,
    avg(x) as x_avg,
    avg(y) as y_avg,
    sum(x*y) as xy
from
    cor
)
select
    -- E[XY] - E[X]E[Y]
    1.0/n * xy - (x_avg * y_avg) as covariance
from tmp
;
 covariance
------------
        188
(1 row)
```

頑張ればワンライナー。

```sql
select 1.0/count(*) * sum(x*y) - (avg(x) * avg(y)) as covariance from cor;
 covariance
------------
        188
(1 row)
```

これらを更に組み合わせ、相関係数を計算できる。

```sql
with tmp as
(select
    count(*) as n,
    sum(x * y) as xy_sum,
    sum(x) as x_sum,
    sum(y) as y_sum,
    sum(x * x) as x2_sum,
    sum(y * y) as y2_sum
from cor)
select
    ((n * xy_sum) - (x_sum * y_sum)) as numerator,
    (sqrt ((n * x2_sum -(x_sum * x_sum)) * (n * y2_sum -(y_sum * y_sum)))) as denominator,
    ((n * xy_sum) - (x_sum * y_sum)) /
        (sqrt ((n * x2_sum -(x_sum * x_sum)) * (n * y2_sum -(y_sum * y_sum)))) as r
from tmp;
         r

 numerator |    denominator    |         r
-----------+-------------------+--------------------
      4700 | 7418.894796396563 | 0.6335175425702007
(1 row)
```

これも頑張ればワンライナー。

```sql
select ((count(*) * sum(x * y)) - (sum(x) * sum(y))) / (sqrt( (count(*) * sum(x * x) - (sum(x)*sum(x))) * (count(*) * sum(y * y) - (sum(y)*sum(y))))) as r from cor;
         r
--------------------
 0.6335175425702007
(1 row)
```

長くなるので、関数で置き換えて、回帰係数のパラメタを計算する。

```sql
select
    covar_samp(x, y)/var_samp(x) as slope,
    avg(y) - (covar_samp(x, y)/var_samp(x)) * avg(x) as intercept
from
    cor
;

  slope   | intercept
----------+-----------
 0.734375 |   24.0625
(1 row)
```

やっぱり書いた。

```sql
with tmp as (
select
    x,
    avg(x) over () as x_bar,
    y as y,
    avg(y) over () as y_bar
from
    cor
where
    x is not null and y is not null
), tmp2 as (
select
    sum((x - x_bar) * (y - y_bar)) / sum((x - x_bar) * (x - x_bar)) as slope,
    max(x_bar) as x_bar_max,
    max(y_bar) as y_bar_max
from
    tmp
)
select
    slope,
    y_bar_max - x_bar_max * slope as intercept
from
    tmp2
;

  slope   | intercept
----------+-----------
 0.734375 |   24.0625
(1 row)
```

## :closed_book: Reference

- [9.18. 集約関数](https://www.postgresql.jp/document/8.3/html/functions-aggregate.html)
