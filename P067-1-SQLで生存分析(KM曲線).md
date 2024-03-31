## :memo: Overview

サービス利用履歴データからユーザーのカプランマイヤー曲線(生存曲線)を計算する方法を 2 回にわけてまとめている後編。前回は、まずサービス利用履歴からカプランマイヤー曲線を計算するためのベースとなるテーブルを作成したので、そのテーブルを利用してカプランマイヤー曲線を計算する。

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

生存曲線の計算で必要な情報は、各時点でのイベント発生数と打ち切りの発生数がまとまったテーブルなのでこれを作成する。前回作成したテーブル`km_v`から計算を始める。

```sql
select * from km_v limit 10;
       user_id        |  min_date  |  max_date  | censor_date | duration_days | cond_days | churn
----------------------+------------+------------+-------------+---------------+-----------+-------
 0014163a53056db3f10e | 2016-08-11 | 2016-10-31 | 2017-02-21  |            81 |       113 |     1
 00164e47967b306239e8 | 2016-05-19 | 2016-05-26 | 2017-02-21  |             7 |       271 |     1
 0016e5b445a29e9cbfcc | 2016-06-07 | 2017-02-13 | 2017-02-21  |           251 |         8 |     0
 002e4faffad3be6b0f95 | 2016-11-14 | 2017-02-13 | 2017-02-21  |            91 |         8 |     0
 003ef43de19023179562 | 2016-06-06 | 2017-02-13 | 2017-02-21  |           252 |         8 |     0
 00402661d6fb30a9755b | 2016-12-09 | 2017-02-13 | 2017-02-21  |            66 |         8 |     0
 0044a682e415caab74b3 | 2016-07-23 | 2017-02-13 | 2017-02-21  |           205 |         8 |     0
 004536fa45ac76c2642e | 2016-12-23 | 2017-02-02 | 2017-02-21  |            41 |        19 |     0
 00539bac05fbc17ca1f7 | 2016-07-25 | 2017-02-13 | 2017-02-21  |           203 |         8 |     0
 005a8cc5a6d05c6230df | 2016-09-16 | 2017-02-13 | 2017-02-21  |           150 |         8 |     0
(10 rows)
```

下記、計算過程での注意点。

- 計算内容を確認しながら作ったので、冗長な SQL になっている。利用時は必要最低限にまとめる
- `risk_set`では打ち切りが発生すると、次の時点のリスクセットからは除外される必要がある
- 積極限式(Product Limit Formula)は SQL では対応できないので、対数に変換して累計和を取ってから指数に戻して計算している
- 最初の方で`survival_prob`が 1 を少し超えているのは、`ln`に`0`を入れれないので微修正した影響。

```sql
create table kmcurve as (
with tmp as (
select
    duration_days,
    churn,
    case when churn = 1 then 1 else 0 end as is_event,
    case when churn = 0 then 1 else 0 end as is_cens,
    count(1) over () as n
from
    km_v
), tmp2 as (
select
    duration_days,
    sum(is_event) as sum_event,
    sum(is_cens) as sum_cens,
    max(n) as n
from
    tmp
group by
    duration_days
order by
    duration_days asc
), tmp3 as (
select
    duration_days,
    sum_event,
    sum_cens,
    n,
    lag((sum_event + sum_cens), 1, 0) over () as lag_sum_eventcens
from
    tmp2
), tmp4 as (
select
    duration_days,
    sum_event,
    sum_cens,
    n,
    lag_sum_eventcens,
    sum(lag_sum_eventcens) over (order by duration_days asc rows between unbounded preceding and current row) as cumsum_lag_sum_eventcens
from
    tmp3
), tmp5 as (
select
    duration_days,
    sum_event,
    sum_cens,
    n,
    lag_sum_eventcens,
    cumsum_lag_sum_eventcens,
    n - cumsum_lag_sum_eventcens as risk_set
from
    tmp4
), tmp6 as (
-- 場合によっては不要
select
    0 as duration_days,
    (select count(1) from km )as risk_set,
    0 as sum_event,
    0 as sum_cens
union all
select
    duration_days,
    risk_set,
    sum_event,
    sum_cens
from
    tmp5
)
select
    duration_days,
    risk_set,
    sum_event,
    sum_cens,
    risk_set - sum_event as t1,
    ((risk_set - sum_event) / risk_set) as t2,
        exp(
            sum(
                ln(
                    ((risk_set - sum_event) / risk_set) + 0.0000000001
                )) over (order by duration_days asc rows between unbounded preceding and current row)
            )::numeric as survival_prob
from
    tmp6
)
;

select
    duration_days,
    risk_set,
    sum_event,
    sum_cens,
    survival_prob
from
    kmcurve;

 duration_days | risk_set | sum_event | sum_cens |        survival_prob
---------------+----------+-----------+----------+------------------------------
             0 |     3390 |         0 |        0 | 1.00000000010000000000000000　-- 場合によっては不要
             0 |     3390 |       424 |      829 | 0.87492625387480825959910628
             1 |     2137 |        45 |        8 | 0.85650244421762780511648971
             2 |     2084 |        30 |        9 | 0.84417275460724693909412257
             3 |     2045 |        19 |        4 | 0.83632958484445751927694758
             4 |     2022 |        23 |        4 | 0.82681643930424155444028994
             5 |     1995 |        14 |        3 | 0.82101421876022676640414429
             6 |     1978 |        14 |        5 | 0.81520319808265014241567751
             7 |     1959 |       438 |       79 | 0.63293724575978007816009102
             8 |     1442 |        12 |        1 | 0.62767008427722265090903507
【略】
           262 |       90 |         0 |       22 | 0.42663117503531277888179669
           263 |       68 |         0 |        4 | 0.42663117507797589638532796
           264 |       64 |         0 |        7 | 0.42663117512063901389312555
           265 |       57 |         0 |        6 | 0.42663117516330213140518946
           266 |       51 |         0 |        8 | 0.42663117520596524892151967
           267 |       43 |         0 |        5 | 0.42663117524862836644211619
           268 |       38 |         0 |        8 | 0.42663117529129148396697903
           269 |       30 |         0 |       12 | 0.42663117533395460149610818
           270 |       18 |         0 |       12 | 0.42663117537661771902950364
           271 |        6 |         0 |        6 | 0.42663117541928083656716541
(269 rows)
```

R での実行結果は下記の通り。

```r
df <- read_csv('~/Desktop/km_v.csv')
fit <- survival::survfit(survival::Surv(duration_days, churn) ~ 1, data = df)
df_fit <- data.frame(duration_days   = c(0, fit$time),
                     risk   = c(max(fit$n.risk), fit$n.risk),
                     event  = c(0, fit$n.event),
                     censor = c(0, fit$n.censor),
                     surv   = c(1, fit$surv))

df_fit
    time risk event censor      surv
1      0 3390     0      0 1.0000000　-- 場合によっては不要
2      0 3390   424    829 0.8749263
3      1 2137    45      8 0.8565024
4      2 2084    30      9 0.8441728
5      3 2045    19      4 0.8363296
6      4 2022    23      4 0.8268164
7      5 1995    14      3 0.8210142
8      6 1978    14      5 0.8152032
9      7 1959   438     79 0.6329372
10     8 1442    12      1 0.6276701
【略】
260  262   90     0     22 0.4266312
261  263   68     0      4 0.4266312
262  264   64     0      7 0.4266312
263  265   57     0      6 0.4266312
264  266   51     0      8 0.4266312
265  267   43     0      5 0.4266312
266  268   38     0      8 0.4266312
267  269   30     0     12 0.4266312
268  270   18     0     12 0.4266312
269  271    6     0      6 0.4266312
```

出力結果を CSV エクスポートして、スプレッドシートで簡易検証しておく。SQL の結果と R での結果をスプレッドシートで簡単に検証してみたが、小数点以下でズレているだけなので、ほぼ等しい結果が得られている。

```sql
-- DBからcsvエクスポート
\copy kmcurve to '~/Desktop/kmcurve_sql.csv' WITH CSV DELIMITER ',' HEADER;

-- Rからcsvエクスポート
write_csv(df_fit, '~/Desktop/survival_pack_survfit.csv')
```

![比較](https://user-images.githubusercontent.com/65038325/186905486-06c0beb1-db9f-4702-9ab1-b9417c1be442.png)

下記はおまけ。

ここではサイズの小さいテーブルを計算式のデバック用にのせておく。R の`survival`パッケージの`gehan`データの`treat == '6-MP'`に限定したデータを利用している。計算過程の説明ははこちらにまとめている。

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

SQL で再現する。

```sql
with tmp as (
select
    time,
    censor,
    case when censor = 1 then 1 else 0 end as is_event,
    case when censor = 0 then 1 else 0 end as is_cens,
    count(1) over () as n
from
    mp
), tmp2 as (
select
    time,
    sum(is_event) as sum_event,
    sum(is_cens) as sum_cens,
    max(n) as n
from
    tmp
group by
    time
order by
    time asc
), tmp3 as (
select
    time,
    sum_event,
    sum_cens,
    n,
    lag((sum_event + sum_cens), 1, 0) over () as lag_sum_eventcens
from
    tmp2
), tmp4 as (
select
    time,
    sum_event,
    sum_cens,
    n,
    lag_sum_eventcens,
    sum(lag_sum_eventcens) over (order by time asc rows between unbounded preceding and current row) as cumsum_lag_sum_eventcens
from
    tmp3
), tmp5 as (
select
    time,
    sum_event,
    sum_cens,
    n,
    lag_sum_eventcens,
    cumsum_lag_sum_eventcens,
    n - cumsum_lag_sum_eventcens as risk_set
from
    tmp4
), tmp6 as (
select
    0 as time,
    (select count(1) from mp )as risk_set,
    0 as sum_event,
    0 as sum_cens
union all
select
    time,
    risk_set,
    sum_event,
    sum_cens
from
    tmp5
)
select
    time,
    risk_set,
    sum_event,
    sum_cens,
    risk_set - sum_event as t1,
    ((risk_set - sum_event) / risk_set) as t2,
    round(
        exp(
            sum(
                ln(
                    ((risk_set - sum_event) / risk_set)
                )) over (order by time asc rows between unbounded preceding and current row)
            )::numeric
    , 4) as survival_prob
from
    tmp6
;

 time | risk_set | sum_event | sum_cens | t1 |   t2  | survival_prob
------+----------+-----------+----------+----+-------+---------------
    0 |       42 |         0 |        0 | 42 | 1.000 |        1.0000
    6 |       42 |         6 |        2 | 36 | 0.857 |        0.8571
    7 |       34 |         2 |        0 | 32 | 0.941 |        0.8067
    9 |       32 |         0 |        2 | 32 | 1.000 |        0.8067
   10 |       30 |         2 |        2 | 28 | 0.933 |        0.7529
   11 |       26 |         0 |        2 | 26 | 1.000 |        0.7529
   13 |       24 |         2 |        0 | 22 | 0.916 |        0.6902
   16 |       22 |         2 |        0 | 20 | 0.909 |        0.6275
   17 |       20 |         0 |        2 | 20 | 1.000 |        0.6275
   19 |       18 |         0 |        2 | 18 | 1.000 |        0.6275
   20 |       16 |         0 |        2 | 16 | 1.000 |        0.6275
   22 |       14 |         2 |        0 | 12 | 0.857 |        0.5378
   23 |       12 |         2 |        0 | 10 | 0.833 |        0.4482
   25 |       10 |         0 |        2 | 10 | 1.000 |        0.4482
   32 |        8 |         0 |        4 |  8 | 1.000 |        0.4482
   34 |        4 |         0 |        2 |  4 | 1.000 |        0.4482
   35 |        2 |         0 |        2 |  2 | 1.000 |        0.4482
(17 rows)

【Rの出力参考】
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

## :closed_book: Reference

None
