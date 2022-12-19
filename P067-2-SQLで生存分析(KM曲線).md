## :memo: Overview

サービス利用履歴データからユーザーのカプランマイヤー曲線(生存曲線)を計算する方法を 1 つ前のノートにまとめたが、より簡単な方法に気がついたので、その方法をまとめておく。

カプランマイヤー曲線(生存曲線)の数理については、[生存分析をまとめたノート](https://github.com/SugiAki1989/survival-analysis-note)でまとめている下記のページを参照する。

- [Introduction to Survival Analysis Note01](https://github.com/SugiAki1989/survival-analysis-note/blob/main/001_intro/001_intro.md)
- [Introduction to Survival Analysis Note02](https://github.com/SugiAki1989/survival-analysis-note/blob/main/001_intro/002_intro.md)
- [Kaplan-Meier Survival Curves and the Log-Rank Test Note01](https://github.com/SugiAki1989/survival-analysis-note/blob/main/002_KmMethod_LogRankTest/001_km_logrank.md)
- [Kaplan-Meier Survival Curves and the Log-Rank Test Note02](https://github.com/SugiAki1989/survival-analysis-note/blob/main/002_KmMethod_LogRankTest/002_km_logrank.md)
- [Kaplan-Meier Survival Curves and the Log-Rank Test Note03](https://github.com/SugiAki1989/survival-analysis-note/blob/main/002_KmMethod_LogRankTest/003_km_logrank.md)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

R の`survival`パッケージの`gehan`データの`treat == '6-MP'`に限定したデータを利用している。計算過程の説明ははこちらにまとめている。

- [Introduction to Survival Analysis Note02](https://github.com/SugiAki1989/survival-analysis-note/blob/main/001_intro/002_intro.md)

R での実行結果は下記の通り。

```R
library(survival)
library(tidyverse)

fit <- survival::survfit(survival::Surv(time, cens) ~ 1, data = df_6MP)
df_fit <- data.frame(time   = c(0, fit$time),
                     risk   = c(max(fit$n.risk), fit$n.risk),
                     event  = c(0, fit$n.event),
                     censor = c(0, fit$n.censor),
                     surv   = c(1, fit$surv))

df_fit
   time risk event censor      surv
1     0   21     0      0 1.0000000
2     6   21     3      1 0.8571429
3     7   17     1      0 0.8067227
4     9   16     0      1 0.8067227
5    10   15     1      1 0.7529412
6    11   13     0      1 0.7529412
7    13   12     1      0 0.6901961
8    16   11     1      0 0.6274510
9    17   10     0      1 0.6274510
10   19    9     0      1 0.6274510
11   20    8     0      1 0.6274510
12   22    7     1      0 0.5378151
13   23    6     1      0 0.4481793
14   25    5     0      1 0.4481793
15   32    4     0      2 0.4481793
16   34    2     0      1 0.4481793
17   35    1     0      1 0.4481793
```

SQL で再現する前に csv データをエクスポートしておく。

```sql
-- Rからエクスポート
df_6MP <- gehan %>% dplyr::filter(treat == '6-MP')
write_csv(df_6MP, '~/Desktop/df_6mp.csv')

-- エクスポートしたcsvをインサート
create table mp(
    time integer,
    censor integer
);

\copy mp from '/Users/aki/Desktop/df_6mp.csv' (encoding 'utf8', format csv, header true);
```

SQL で再現する前にサンプルデータを確認しておく。各行が 1 つの個人を表し、生存時間とイベントが発生したかどうかを管理するカラムの 2 列をもつ。ログデータなどをこの形式にまとめる方法は別のノートを参照。

```sql
select * from mp limit 10;
 time | censor
------+--------
   10 |      1
    7 |      1
   32 |      0
   23 |      1
   22 |      1
    6 |      1
   16 |      1
   34 |      0
   32 |      0
   25 |      0
(10 rows)
```

前回のようにこまこま計算しなくても「個人-時間形式」に加工すれば処理が楽かもしれない。KM 曲線の計算方法を理解すると、「個人-時間形式」のデータは非常に都合がよい。生存関数$S_{ij}$は、個人$i$が期間$j$を超えても生存している確率として定義される。生存するためには、個人$i$は対象のイベントを$j$番目の期間やそれ以前に経験してはいけないことになる。つまり、個人$i$が期間$j$の終わりに生存していることを意味し、時間$T_{i}$が$j$を超えるということ。

$$
S(t_{ij}) = P(T_{i} > j)
$$

生存確率は定義より 100％で始まり、時間経過とともにイベントが起こり、確率は低下していく。ハザードが高い期間はイベントが多く起こっているので、曲線は急激に変化し、ハザードが低い期間では、曲線は緩やかにしか変化しない。ハザードはイベントが発生したことを表すが、反転させれば、イベントが起こっていないことを表現できる。つまり、各期間でハザードを計算し、反転させることで、生存についてわかるようになる。あとは下記の通り、ハザードを総乗していけば、生存確率が計算できる。

$$
\hat{S}(t_{j}) = \hat{S}(t_{j-1}) * (1-\hat{h(t_{j})})
$$

この計算をするにあたって、各期間でのイベントの発生を計算する必要があるため、「個人-時間形式」に加工すればハザードの計算が簡単に行なえ、ハザードがあれば、生存確率が手に入る。

では、SQL で KM 曲線を計算する。まずは、`generate_series`を使って生存時間の値まで、レコードを拡張する。

```sql
with tally as (
select generate_series(1,100,1) as day
), mp2 as (
select 0 as start, time as end, censor from mp
), tmp1 as (
select m.start, m.end, t.day, m.censor from mp2 as m
inner join tally as t
on t.day >= m.start and m.end >= t.day
) select * from tmp1 limit 10;

 start | end | day | censor
-------+-----+-----+--------
     0 |  10 |   1 |      1
     0 |  10 |   2 |      1
     0 |  10 |   3 |      1
     0 |  10 |   4 |      1
     0 |  10 |   5 |      1
     0 |  10 |   6 |      1
     0 |  10 |   7 |      1
     0 |  10 |   8 |      1
     0 |  10 |   9 |      1
     0 |  10 |  10 |      1
(10 rows)
```

このままでは KM 曲線の計算ができないので、`censor`の値を修正する必要がある。イベントが起きたのであれば、起きた時点では`1`をとり、それ以外は`0`を取るようにする。

```sql
with tally as (
select generate_series(1,100,1) as day
), mp2 as (
select 0 as start, time as end, censor from mp
), tmp1 as (
select m.start, m.end, t.day, m.censor,
case when m.end = t.day then m.censor else 0 end as censor2
from mp2 as m
inner join tally as t
on t.day >= m.start and m.end >= t.day
) select * from tmp1 limit 10;

 start | end | day | censor | censor2
-------+-----+-----+--------+---------
     0 |  10 |   1 |      1 |       0
     0 |  10 |   2 |      1 |       0
     0 |  10 |   3 |      1 |       0
     0 |  10 |   4 |      1 |       0
     0 |  10 |   5 |      1 |       0
     0 |  10 |   6 |      1 |       0
     0 |  10 |   7 |      1 |       0
     0 |  10 |   8 |      1 |       0
     0 |  10 |   9 |      1 |       0
     0 |  10 |  10 |      1 |       1
(10 rows)
```

そして、このテーブルを時間とイベントでグループ化して集計する。

```sql
with tally as (
select generate_series(0,100,1) as day
), mp2 as (
select 0 as start, time as end, censor from mp
), tmp1 as (
select m.start, m.end, t.day, m.censor,
case when m.end = t.day then m.censor else 0 end as censor2
from mp2 as m
inner join tally as t
on t.day >= m.start and m.end >= t.day
), tmp2 as (
select day, censor2, count(1) as cnt
from tmp1
group by day, censor2
order by day asc, censor2 asc
)
select * from tmp2;

 day | censor2 | cnt
-----+---------+-----
   0 |       0 |  42
   1 |       0 |  42
   2 |       0 |  42
   3 |       0 |  42
   4 |       0 |  42
   5 |       0 |  42
   6 |       0 |  36
   6 |       1 |   6
   7 |       0 |  32
   7 |       1 |   2
   8 |       0 |  32
   9 |       0 |  32
  10 |       0 |  28
  10 |       1 |   2
  11 |       0 |  26
  12 |       0 |  24
  13 |       0 |  22
  13 |       1 |   2
  14 |       0 |  22
  15 |       0 |  22
  16 |       0 |  20
  16 |       1 |   2
  17 |       0 |  20
  18 |       0 |  18
  19 |       0 |  18
  20 |       0 |  16
  21 |       0 |  14
  22 |       0 |  12
  22 |       1 |   2
  23 |       0 |  10
  23 |       1 |   2
  24 |       0 |  10
  25 |       0 |  10
  26 |       0 |   8
  27 |       0 |   8
  28 |       0 |   8
  29 |       0 |   8
  30 |       0 |   8
  31 |       0 |   8
  32 |       0 |   8
  33 |       0 |   4
  34 |       0 |   4
  35 |       0 |   2
(43 rows)
```

これを Long 型から Wide 型に変換して、

```sql
with tally as (
select generate_series(0,100,1) as day
), mp2 as (
select 0 as start, time as end, censor from mp
), tmp1 as (
select m.start, m.end, t.day, m.censor,
case when m.end = t.day then m.censor else 0 end as censor2
from mp2 as m
inner join tally as t
on t.day >= m.start and m.end >= t.day
), tmp2 as (
select day, censor2, count(1) as cnt
from tmp1
group by day, censor2
order by day asc, censor2 asc
), tmp3 as (
select
day,
max(case when censor2 = 0 then cnt else 0 end) as event0,
max(case when censor2 = 1 then cnt else 0 end) as event1
from tmp2
group by day
)
select * from tmp3;

 day | event0 | event1
-----+--------+--------
   0 |     42 |      0
   1 |     42 |      0
   2 |     42 |      0
   3 |     42 |      0
   4 |     42 |      0
   5 |     42 |      0
   6 |     36 |      6
   7 |     32 |      2
   8 |     32 |      0
   9 |     32 |      0
  10 |     28 |      2
  11 |     26 |      0
  12 |     24 |      0
  13 |     22 |      2
  14 |     22 |      0
  15 |     22 |      0
  16 |     20 |      2
  17 |     20 |      0
  18 |     18 |      0
  19 |     18 |      0
  20 |     16 |      0
  21 |     14 |      0
  22 |     12 |      2
  23 |     10 |      2
  24 |     10 |      0
  25 |     10 |      0
  26 |      8 |      0
  27 |      8 |      0
  28 |      8 |      0
  29 |      8 |      0
  30 |      8 |      0
  31 |      8 |      0
  32 |      8 |      0
  33 |      4 |      0
  34 |      4 |      0
  35 |      2 |      0
(36 rows)
```

合計を計算する。合計とイベントが発生した人数を使ってハザードを計算する。ハザードを使えば、その時点での生存確率が計算できる。

```sql
with tally as (
select generate_series(0,100,1) as day
), mp2 as (
select 0 as start, time as end, censor from mp
), tmp1 as (
select m.start, m.end, t.day, m.censor,
case when m.end = t.day then m.censor else 0 end as censor2
from mp2 as m
inner join tally as t
on t.day >= m.start and m.end >= t.day
), tmp2 as (
select day, censor2, count(1) as cnt
from tmp1
group by day, censor2
order by day asc, censor2 asc
), tmp3 as (
select
day,
max(case when censor2 = 0 then cnt else 0 end) as event0,
max(case when censor2 = 1 then cnt else 0 end) as event1
from tmp2
group by day
)
select
day, event0, event1, event0+event1 as total,
(event1::real/(event0+event1)::real) as hazard,
1 - (event1::real/(event0+event1)::real) as hazard2
from tmp3;

 day | event0 | event1 | total |   hazard    |      hazard2
-----+--------+--------+-------+-------------+--------------------
   0 |     42 |      0 |    42 |           0 |                  1
   1 |     42 |      0 |    42 |           0 |                  1
   2 |     42 |      0 |    42 |           0 |                  1
   3 |     42 |      0 |    42 |           0 |                  1
   4 |     42 |      0 |    42 |           0 |                  1
   5 |     42 |      0 |    42 |           0 |                  1
   6 |     36 |      6 |    42 |  0.14285715 | 0.8571428507566452
   7 |     32 |      2 |    34 |  0.05882353 | 0.9411764703691006
   8 |     32 |      0 |    32 |           0 |                  1
   9 |     32 |      0 |    32 |           0 |                  1
  10 |     28 |      2 |    30 |  0.06666667 | 0.9333333298563957
  11 |     26 |      0 |    26 |           0 |                  1
  12 |     24 |      0 |    24 |           0 |                  1
  13 |     22 |      2 |    24 | 0.083333336 | 0.9166666641831398
  14 |     22 |      0 |    22 |           0 |                  1
  15 |     22 |      0 |    22 |           0 |                  1
  16 |     20 |      2 |    22 |  0.09090909 | 0.9090909063816071
  17 |     20 |      0 |    20 |           0 |                  1
  18 |     18 |      0 |    18 |           0 |                  1
  19 |     18 |      0 |    18 |           0 |                  1
  20 |     16 |      0 |    16 |           0 |                  1
  21 |     14 |      0 |    14 |           0 |                  1
  22 |     12 |      2 |    14 |  0.14285715 | 0.8571428507566452
  23 |     10 |      2 |    12 |  0.16666667 | 0.8333333283662796
  24 |     10 |      0 |    10 |           0 |                  1
  25 |     10 |      0 |    10 |           0 |                  1
  26 |      8 |      0 |     8 |           0 |                  1
  27 |      8 |      0 |     8 |           0 |                  1
  28 |      8 |      0 |     8 |           0 |                  1
  29 |      8 |      0 |     8 |           0 |                  1
  30 |      8 |      0 |     8 |           0 |                  1
  31 |      8 |      0 |     8 |           0 |                  1
  32 |      8 |      0 |     8 |           0 |                  1
  33 |      4 |      0 |     4 |           0 |                  1
  34 |      4 |      0 |     4 |           0 |                  1
  35 |      2 |      0 |     2 |           0 |                  1
(36 rows)
```

ただ、KM 曲線は各時点では、打ち切りやイベントが発生した人たちは処理して計算するため、この`hazard2`を総乗する必要がある。

```sql
with tally as (
select generate_series(0,100,1) as day
), mp2 as (
select 0 as start, time as end, censor from mp
), tmp1 as (
select m.start, m.end, t.day, m.censor,
case when m.end = t.day then m.censor else 0 end as censor2
from mp2 as m
inner join tally as t
on t.day >= m.start and m.end >= t.day
), tmp2 as (
select day, censor2, count(1) as cnt
from tmp1
group by day, censor2
order by day asc, censor2 asc
), tmp3 as (
select
day,
max(case when censor2 = 0 then cnt else 0 end) as event0,
max(case when censor2 = 1 then cnt else 0 end) as event1
from tmp2
group by day
), tmp4 as (
select
day, event0, event1, event0+event1 as total,
(event1::real/(event0+event1)::real) as hazard,
1 - (event1::real/(event0+event1)::real) as hazard2
from tmp3
)
select
day, event0, event1, total, hazard, hazard2,
round(
        exp(
            sum(
                ln(hazard2)
                ) over (order by day asc rows between unbounded preceding and current row)
            )::numeric
    , 4) as survival_prob
from tmp4;

 day | event0 | event1 | total |   hazard    |      hazard2       | survival_prob
-----+--------+--------+-------+-------------+--------------------+---------------
   0 |     42 |      0 |    42 |           0 |                  1 |        1.0000
   1 |     42 |      0 |    42 |           0 |                  1 |        1.0000
   2 |     42 |      0 |    42 |           0 |                  1 |        1.0000
   3 |     42 |      0 |    42 |           0 |                  1 |        1.0000
   4 |     42 |      0 |    42 |           0 |                  1 |        1.0000
   5 |     42 |      0 |    42 |           0 |                  1 |        1.0000
   6 |     36 |      6 |    42 |  0.14285715 | 0.8571428507566452 |        0.8571
   7 |     32 |      2 |    34 |  0.05882353 | 0.9411764703691006 |        0.8067
   8 |     32 |      0 |    32 |           0 |                  1 |        0.8067
   9 |     32 |      0 |    32 |           0 |                  1 |        0.8067
  10 |     28 |      2 |    30 |  0.06666667 | 0.9333333298563957 |        0.7529
  11 |     26 |      0 |    26 |           0 |                  1 |        0.7529
  12 |     24 |      0 |    24 |           0 |                  1 |        0.7529
  13 |     22 |      2 |    24 | 0.083333336 | 0.9166666641831398 |        0.6902
  14 |     22 |      0 |    22 |           0 |                  1 |        0.6902
  15 |     22 |      0 |    22 |           0 |                  1 |        0.6902
  16 |     20 |      2 |    22 |  0.09090909 | 0.9090909063816071 |        0.6275
  17 |     20 |      0 |    20 |           0 |                  1 |        0.6275
  18 |     18 |      0 |    18 |           0 |                  1 |        0.6275
  19 |     18 |      0 |    18 |           0 |                  1 |        0.6275
  20 |     16 |      0 |    16 |           0 |                  1 |        0.6275
  21 |     14 |      0 |    14 |           0 |                  1 |        0.6275
  22 |     12 |      2 |    14 |  0.14285715 | 0.8571428507566452 |        0.5378
  23 |     10 |      2 |    12 |  0.16666667 | 0.8333333283662796 |        0.4482
  24 |     10 |      0 |    10 |           0 |                  1 |        0.4482
  25 |     10 |      0 |    10 |           0 |                  1 |        0.4482
  26 |      8 |      0 |     8 |           0 |                  1 |        0.4482
  27 |      8 |      0 |     8 |           0 |                  1 |        0.4482
  28 |      8 |      0 |     8 |           0 |                  1 |        0.4482
  29 |      8 |      0 |     8 |           0 |                  1 |        0.4482
  30 |      8 |      0 |     8 |           0 |                  1 |        0.4482
  31 |      8 |      0 |     8 |           0 |                  1 |        0.4482
  32 |      8 |      0 |     8 |           0 |                  1 |        0.4482
  33 |      4 |      0 |     4 |           0 |                  1 |        0.4482
  34 |      4 |      0 |     4 |           0 |                  1 |        0.4482
  35 |      2 |      0 |     2 |           0 |                  1 |        0.4482
(36 rows)
```

これで生存曲線の計算は終了。前回計算したものはイベントや打ち切りが発生した時点のみが表示されていたが、今回のものは、イベントがない期間は生存確率は変わらず、時間だけが増えている形になっている。ということで、少し手を加えて、前回計算した`survival_prob`と R の計算結果`Surv`と同じになるか検算しておく。

```sql
with tally as (
select generate_series(0,100,1) as day
), mp2 as (
select 0 as start, time as end, censor from mp
), tmp1 as (
select m.start, m.end, t.day, m.censor,
case when m.end = t.day then m.censor else 0 end as censor2
from mp2 as m
inner join tally as t
on t.day >= m.start and m.end >= t.day
), tmp2 as (
select day, censor2, count(1) as cnt
from tmp1
group by day, censor2
order by day asc, censor2 asc
), tmp3 as (
select
day,
max(case when censor2 = 0 then cnt else 0 end) as event0,
max(case when censor2 = 1 then cnt else 0 end) as event1
from tmp2
group by day
), tmp4 as (
select
day, event0, event1, event0+event1 as total,
(event1::real/(event0+event1)::real) as hazard,
1 - (event1::real/(event0+event1)::real) as hazard2
from tmp3
)
select
day, event0, event1, total, hazard, hazard2,
round(
        exp(
            sum(
                ln(hazard2)
                ) over (order by day asc rows between unbounded preceding and current row)
            )::numeric
    , 4) as survival_prob2
from tmp4
where day in (0,6,7,9,10,11,13,16,17,19,20,22,23,25,32,34,35);

-- 右端のsurvival_probは前回計算したものを手でコピペしたもの
 day | event0 | event1 | total |   hazard    |      hazard2       | survival_prob2 | survival_prob　|  Surv　
-----+--------+--------+-------+-------------+--------------------+--------------  +--------------　| ----------
   0 |     42 |      0 |    42 |           0 |                  1 |        1.0000  |        1.0000　| 1.0000000　　
   6 |     36 |      6 |    42 |  0.14285715 | 0.8571428507566452 |        0.8571  |        0.8571　| 0.8571429
   7 |     32 |      2 |    34 |  0.05882353 | 0.9411764703691006 |        0.8067  |        0.8067　| 0.8067227
   9 |     32 |      0 |    32 |           0 |                  1 |        0.8067  |        0.8067　| 0.8067227
  10 |     28 |      2 |    30 |  0.06666667 | 0.9333333298563957 |        0.7529  |        0.7529　| 0.7529412
  11 |     26 |      0 |    26 |           0 |                  1 |        0.7529  |        0.7529　| 0.7529412
  13 |     22 |      2 |    24 | 0.083333336 | 0.9166666641831398 |        0.6902  |        0.6902　| 0.6901961
  16 |     20 |      2 |    22 |  0.09090909 | 0.9090909063816071 |        0.6275  |        0.6275　| 0.6274510
  17 |     20 |      0 |    20 |           0 |                  1 |        0.6275  |        0.6275　| 0.6274510
  19 |     18 |      0 |    18 |           0 |                  1 |        0.6275  |        0.6275　| 0.6274510
  20 |     16 |      0 |    16 |           0 |                  1 |        0.6275  |        0.6275　| 0.6274510
  22 |     12 |      2 |    14 |  0.14285715 | 0.8571428507566452 |        0.5378  |        0.5378　| 0.5378151
  23 |     10 |      2 |    12 |  0.16666667 | 0.8333333283662796 |        0.4482  |        0.4482　| 0.4481793
  25 |     10 |      0 |    10 |           0 |                  1 |        0.4482  |        0.4482　| 0.4481793
  32 |      8 |      0 |     8 |           0 |                  1 |        0.4482  |        0.4482　| 0.4481793
  34 |      4 |      0 |     4 |           0 |                  1 |        0.4482  |        0.4482　| 0.4481793
  35 |      2 |      0 |     2 |           0 |                  1 |        0.4482  |        0.4482　| 0.4481793
(17 rows)
```

## :closed_book: Reference

None
