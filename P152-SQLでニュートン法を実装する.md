## :memo: Overview

ここでは、SQLでニュートン法を実装する。`RECURSIVE`クエリを使う機会があまりないので、定期的に頭の体操をしておかないと、本番で使えないのでボケ防止の一貫でSQLでニュートン法を実装する。本来、ニュートン法をSQLで実装するケースは教育目的以外に絶対存在しないと思う。ニュートン法については下記を参照願います。

- [ニュートン法とは何か？？ニュートン法で解く方程式の近似解](https://qiita.com/PlanetMeron/items/09d7eb204868e1a49f49)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`RECURSIVE`

## :pencil2: Example

ここでは √2 をニュートン法で求めてみる。8回の反復で収束しており、1.4142...と計算できていることがわかる。

```sql
-- パラメタ初期値
with RECURSIVE params as (
  select
    2.0 :: DOUBLE PRECISION as c       -- function f(x)=x²−c
    , 10.0 :: DOUBLE PRECISION as x0   -- initial value
    , 1e-15 :: DOUBLE PRECISION as eps -- convergence |x_{k+1}-x_k|
    , 100 :: INT as max_iter           -- max iteration
),
newton as (
  -- k = 0 
  select 
    0 as iter
    , p.x0 as xk
    , p.x0 * p.x0 - p.c as fk
    , 2 * p.x0 as dfk -- f(x) = x²-c
    , null :: DOUBLE PRECISION as diff -- f'(x) = 2x -- 差分（最初は無）
  from
    params p
  union all
  -- k → k+1 → ...
  select
    iter + 1 as iter
    , nx as xk
    , nx * nx - p.c as fk
    , 2 * nx as dfk -- f(nx)
    , ABS(nx - xk) as diff -- f'(nx)
  from
    (select n.*, (xk - fk/dfk) as nx from newton n) as n  -- 新しい推定値 
  cross join  
    params p 
  where
    iter < p.max_iter
    and (diff is null or diff > p.eps )
)
select
  iter as iteration
  , xk as approximate_solution
from
  newton
order by
  iter asc
;

 iteration | approximate_solution |
-----------+-----------------------
         0 |                   10 |
         1 |                  5.1 |
         2 |   2.7460784313725486 |
         3 |   1.7371948743795982 |
         4 |    1.444238094866232 |
         5 |   1.4145256551487377 |
         6 |   1.4142135968022693 |
         7 |   1.4142135623730954 |
         8 |   1.4142135623730951 |
(9 rows)

```

実行部分の説明を書いておく。再帰部のサブクエリが実際に数値を更新している部分で、`update: x_{n+1} = x_{n} - (f(x_{n})/f'(x_{n}))`の計算をしている。

```sql
(select n.*, (xk - fk / dfk) as nx from newton n) as n  -- 新しい推定値 
```
なので、パラメタ初期値から受け取った値をアンカー部で受け取って、このサブクエリを通して、`update`を`nx`として計算し、それを外側の再帰部の`select`が受け取って計算が反復する。
`WHERE`の打ち切り条件が反復回数か収束判定なので、この条件が満たされる限り反復計算が実行される。`xk`を`nx`に更新して、新しい`nx`で`xk`を計算して…という感じに繰り返しをイメージするとわかり良いかもしれない。

ニュートン法の更新式でサブクエリを単独で実行すると確認しやすいというか、コンパクトに書けるので、その都合で分割している。サブクエリを使わない場合、こんな感じになると思われる。

```sql
select
    n.iter + 1                                         as iter,
    (n.xk - n.fk / n.dfk)                              as xk,
    (n.xk - n.fk / n.dfk)*(n.xk - n.fk / n.dfk) - p.c  as fk,
    2*(n.xk - n.fk / n.dfk)                            as dfk,
    ABS((n.xk - n.fk / n.dfk) - n.xk)                  as diff
from 
  newton as n
```

おしまい。

## :closed_book: Reference

- none