## :memo: Overview

ここでは、下記を参考に RFV 分析を計算する方法をまとめておく。

- [SaaS アナリティクス・ワークショップ 第 5 回： エンゲージメント Part 3 - RFV 分析](https://exploratory.io/note/BWz1Bar4JF/SaaS-5-Part-3-RFV-CLJ4efS3jU)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

集計方針は下記の通り。

1. ユーザーの利用日数、利用時間、初回、最終利用日を計算
2. アクティブピリオドを計算
3. 各種スコアを計算
4. 統合スコアとしてエンゲージメントを作成

サンプルデータを用意する。

```sql
create table rfv(
timestamp timestamp with time zone,
userid varchar(255),
time float
);

COPY rfv FROM '/Users/aki/Desktop/rfv.csv' WITH CSV HEADER;

select * from rfv where userid like '0158153c2';
        timestamp        |  userid   | time
------------------------+-----------+-------
 2017-01-18 16:13:39+09 | 0158153c2 | 212.1
 2017-01-19 19:12:10+09 | 0158153c2 | 190.6
 2017-01-20 02:30:48+09 | 0158153c2 | 115.5
 2017-01-20 19:23:28+09 | 0158153c2 | 211.3
 2017-01-21 05:36:40+09 | 0158153c2 | 124.2
 2017-01-23 15:05:43+09 | 0158153c2 | 226.6
 2017-01-24 10:20:08+09 | 0158153c2 |  59.5
 2017-01-24 18:39:26+09 | 0158153c2 |    96
 2017-01-25 02:56:42+09 | 0158153c2 |  41.7
 2017-01-25 05:52:54+09 | 0158153c2 | 124.4
 2017-01-25 21:21:17+09 | 0158153c2 | 193.2
 2017-01-26 09:09:02+09 | 0158153c2 |  88.2
 2017-01-27 15:54:13+09 | 0158153c2 | 170.6
 2017-01-28 00:27:54+09 | 0158153c2 | 135.1
 2017-01-29 16:16:14+09 | 0158153c2 |  45.4
 2017-01-30 03:23:28+09 | 0158153c2 |  43.9
 2017-01-31 20:03:55+09 | 0158153c2 |  80.3
 2017-02-03 19:52:55+09 | 0158153c2 | 230.4
(18 rows)
```

まずは、ユーザーの利用日数、利用時間、初回、最終利用日を計算する。

```sql
with param as (
    select max(date_trunc('day', timestamp))::date as paramdate from rfv
), tmp as (
select
    userid,
    count(distinct date_trunc('day', timestamp)) as freq,
    sum(time) as volume,
    min(date_trunc('day', timestamp)::date) as min_ymd,
    max(date_trunc('day', timestamp)::date) as max_ymd
from
    rfv
-- where userid = '00149e578'
group by
    userid
) select * from tmp limit 10;


  userid   | freq |       volume       |  min_ymd   |  max_ymd
-----------+------+--------------------+------------+------------
 00149e578 |    7 |                772 | 2016-11-04 | 2016-11-15
 001c36deb |    5 |  752.9000000000001 | 2017-01-11 | 2017-01-19
 003c8c570 |    8 | 1725.4999999999998 | 2016-11-16 | 2016-11-24
 00740e802 |    1 |              199.3 | 2017-01-26 | 2017-01-26
 010c14919 |    2 | 238.79999999999998 | 2017-01-28 | 2017-02-01
 0158153c2 |   14 | 2388.9999999999995 | 2017-01-18 | 2017-02-03
 019129b5b |   16 | 3182.6999999999994 | 2017-01-09 | 2017-01-30
 01cabb832 |   36 |  5771.900000000001 | 2016-11-04 | 2017-02-08
 01f4d539z |   34 |             7130.5 | 2017-01-04 | 2017-02-09
 01fcb59f8 |    6 |              957.2 | 2016-12-01 | 2016-12-11
(10 rows)
```

```sql
with param as (
    select max(date_trunc('day', timestamp))::date as paramdate from rfv
), tmp as (
select
    userid,
    count(distinct date_trunc('day', timestamp)) as freq,
    sum(time) as volume,
    min(date_trunc('day', timestamp)::date) as min_ymd,
    max(date_trunc('day', timestamp)::date) as max_ymd
from
    rfv
-- where userid = '00149e578'
group by
    userid
) -- select * from tmp limit 10;
, tmp2 as (
select
    t.userid,
    t.min_ymd,
    t.max_ymd,
    p.paramdate,
    t.freq,
    t.volume,
    p.paramdate - t.max_ymd as recency_score,
    max_ymd - min_ymd as active_period
from
    tmp as t
cross join
    param as p
where freq > 1
--  userid   |  min_ymd   |  max_ymd   | paramdate  | freq | volume | recency | active_period
-- -----------+------------+------------+------------+------+--------+---------+---------------
-- 00740e802 | 2017-01-26 | 2017-01-26 | 2017-02-12 |    1 |  199.3 |      17 |             0
-- 0f9a2b307 | 2017-01-25 | 2017-01-25 | 2017-02-12 |    1 |  162.9 |      18 |             0
) select * from tmp2 limit 10;

  userid   |  min_ymd   |  max_ymd   | paramdate  | freq |       volume       | recency_score | active_period
-----------+------------+------------+------------+------+--------------------+---------------+---------------
 00149e578 | 2016-11-04 | 2016-11-15 | 2017-02-12 |    7 |                772 |            89 |            11
 001c36deb | 2017-01-11 | 2017-01-19 | 2017-02-12 |    5 |  752.9000000000001 |            24 |             8
 003c8c570 | 2016-11-16 | 2016-11-24 | 2017-02-12 |    8 | 1725.4999999999998 |            80 |             8
 010c14919 | 2017-01-28 | 2017-02-01 | 2017-02-12 |    2 | 238.79999999999998 |            11 |             4
 0158153c2 | 2017-01-18 | 2017-02-03 | 2017-02-12 |   14 | 2388.9999999999995 |             9 |            16
 019129b5b | 2017-01-09 | 2017-01-30 | 2017-02-12 |   16 | 3182.6999999999994 |            13 |            21
 01cabb832 | 2016-11-04 | 2017-02-08 | 2017-02-12 |   36 |  5771.900000000001 |             4 |            96
 01f4d539z | 2017-01-04 | 2017-02-09 | 2017-02-12 |   34 |             7130.5 |             3 |            36
 01fcb59f8 | 2016-12-01 | 2016-12-11 | 2017-02-12 |    6 |              957.2 |            63 |            10
 02bf3632z | 2016-11-07 | 2017-02-06 | 2017-02-12 |   46 | 18256.300000000003 |             6 |            91
(10 rows)
```

実際のサービス利用期間であるアクティブピリオドを使って、各種スコアを計算する。ユーザーによって利用期間が異なるので、アクティブピリオドを利用することで、日あたり何時間利用しているのか、アクティブピリオドの期間内で平均的にどれだけ利用しているのかを計算する。

`recency_score`は 0 日に近いほうが直近利用していることを表し、`volume_score`は大きいほうがたくさん利用しており、`freq_score`は 1 に近いと毎日利用していることがわかる。

```sql
with param as (
    select max(date_trunc('day', timestamp))::date as paramdate from rfv
), tmp as (
select
    userid,
    count(distinct date_trunc('day', timestamp)) as freq,
    sum(time) as volume,
    min(date_trunc('day', timestamp)::date) as min_ymd,
    max(date_trunc('day', timestamp)::date) as max_ymd
from
    rfv
-- where userid = '00149e578'
group by
    userid
) -- select * from tmp limit 10;
, tmp2 as (
select
    t.userid,
    t.min_ymd,
    t.max_ymd,
    p.paramdate,
    t.freq,
    t.volume,
    p.paramdate - t.max_ymd as recency_score,
    max_ymd - min_ymd as active_period
from
    tmp as t
cross join
    param as p
where freq > 1
--  userid   |  min_ymd   |  max_ymd   | paramdate  | freq | volume | recency | active_period
-- -----------+------------+------------+------------+------+--------+---------+---------------
-- 00740e802 | 2017-01-26 | 2017-01-26 | 2017-02-12 |    1 |  199.3 |      17 |             0
-- 0f9a2b307 | 2017-01-25 | 2017-01-25 | 2017-02-12 |    1 |  162.9 |      18 |             0
) -- select * from tmp2 limit 10;
select
    *,
    round((volume / active_period)::numeric, 2) as volume_score,
    round(freq::numeric / active_period, 2) as freq_score
from
    tmp2
limit 10
;
  userid   |  min_ymd   |  max_ymd   | paramdate  | freq |       volume       | active_period | recency_score | volume_score | freq_score
-----------+------------+------------+------------+------+--------------------+---------------+---------------+--------------+------------
 00149e578 | 2016-11-04 | 2016-11-15 | 2017-02-12 |    7 |                772 |            11 |            89 |        70.18 |       0.64
 001c36deb | 2017-01-11 | 2017-01-19 | 2017-02-12 |    5 |  752.9000000000001 |             8 |            24 |        94.11 |       0.63
 003c8c570 | 2016-11-16 | 2016-11-24 | 2017-02-12 |    8 | 1725.4999999999998 |             8 |            80 |       215.69 |       1.00
 010c14919 | 2017-01-28 | 2017-02-01 | 2017-02-12 |    2 | 238.79999999999998 |             4 |            11 |        59.70 |       0.50
 0158153c2 | 2017-01-18 | 2017-02-03 | 2017-02-12 |   14 | 2388.9999999999995 |            16 |             9 |       149.31 |       0.88
 019129b5b | 2017-01-09 | 2017-01-30 | 2017-02-12 |   16 | 3182.6999999999994 |            21 |            13 |       151.56 |       0.76
 01cabb832 | 2016-11-04 | 2017-02-08 | 2017-02-12 |   36 |  5771.900000000001 |            96 |             4 |        60.12 |       0.38
 01f4d539z | 2017-01-04 | 2017-02-09 | 2017-02-12 |   34 |             7130.5 |            36 |             3 |       198.07 |       0.94
 01fcb59f8 | 2016-12-01 | 2016-12-11 | 2017-02-12 |    6 |              957.2 |            10 |            63 |        95.72 |       0.60
 02bf3632z | 2016-11-07 | 2017-02-06 | 2017-02-12 |   46 | 18256.300000000003 |            91 |             6 |       200.62 |       0.51
(10 rows)
```

この指標を使って、統合スコアであるエンゲージメントを計算してもよいし、RFM 分析と同様に各種スコアをビニングして利用する。

## :closed_book: Reference

- [SaaS アナリティクス・ワークショップ 第 5 回： エンゲージメント Part 3 - RFV 分析](https://exploratory.io/note/BWz1Bar4JF/SaaS-5-Part-3-RFV-CLJ4efS3jU)
