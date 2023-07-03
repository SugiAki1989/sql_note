## :memo: Overview

コホート分析を SQL で行う。ユーザーを特定の条件でグループに分け、時間経過に伴う行動の有無を分析する手法のこと。ここでは初回利用月というコホートを作成する例をまとめておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

サンプルデータは[こちら](https://mysql.hatenablog.jp/entry/2021/12/27/171200)でデータを生成したものを利用する。半年間のソーシャルメディアのログデータで、程よく大きいサイズ。

```sql
select count(1), min(event_time), max(event_time) from event;

  count   |         min         |         max
----------+---------------------+---------------------
 16995997 | 2020-01-01 00:00:15 | 2020-06-30 23:59:50
(1 row)
```

集計の手順としては下記の通り。

1. ユーザーごとのアクションを月単位にまとめる
2. 初回利用月、利用月ごとにユーザーをカウントする
3. 各コホートの初回利用月のユーザー数を計算する
4. 体裁を整える

まずはユーザーごとに毎日のアクションログが記録されているので、これをまずは月単位にまとめて小さくする。合わせて、ユーザーのクションの初回実行時を初回利用月として計算する。

```sql
with tmp as (
select
    account_id,
    date_trunc('month', event_time)::date as ym,
    min(date_trunc('month', event_time)) over (partition by account_id order by date_trunc('month', event_time) asc)::date as firstym
from
    event
group by
    account_id,
    date_trunc('month', event_time)
)
select
    account_id,
    ym,
    firstym
from
    tmp
limit 20;

 account_id |     ym     |  firstym
------------+------------+------------
          1 | 2020-01-01 | 2020-01-01
          1 | 2020-02-01 | 2020-01-01
          1 | 2020-03-01 | 2020-01-01
          1 | 2020-04-01 | 2020-01-01
          1 | 2020-05-01 | 2020-01-01
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
          2 | 2020-01-01 | 2020-01-01
          2 | 2020-02-01 | 2020-01-01
          2 | 2020-03-01 | 2020-01-01
          2 | 2020-04-01 | 2020-01-01
          2 | 2020-05-01 | 2020-01-01
          2 | 2020-06-01 | 2020-01-01
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
          3 | 2020-01-01 | 2020-01-01
          3 | 2020-02-01 | 2020-01-01
          3 | 2020-03-01 | 2020-01-01
          3 | 2020-04-01 | 2020-01-01
          3 | 2020-05-01 | 2020-01-01
          3 | 2020-06-01 | 2020-01-01
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
          4 | 2020-01-01 | 2020-01-01
          4 | 2020-02-01 | 2020-01-01
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
          5 | 2020-01-01 | 2020-01-01
(20 rows)
```

次は、初回利用月、利用月ごとにユーザーをカウントしていく。

```sql
with tmp as (
select
    account_id,
    date_trunc('month', event_time)::date as ym,
    min(date_trunc('month', event_time)) over (partition by account_id order by date_trunc('month', event_time) asc)::date as firstym
from
    event
group by
    account_id,
    date_trunc('month', event_time)
), tmp1 as (
select
    firstym,
    ym,
    count(account_id) as cnt
from
    tmp
group by
    firstym,
    ym
)
select * from tmp1 order by firstym asc, ym asc
;

  firstym   |     ym     | cnt
------------+------------+------
 2020-01-01 | 2020-01-01 | 9972
 2020-01-01 | 2020-02-01 | 9945
 2020-01-01 | 2020-03-01 | 9253
 2020-01-01 | 2020-04-01 | 8738
 2020-01-01 | 2020-05-01 | 8354
 2020-01-01 | 2020-06-01 | 8018
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2020-02-01 | 2020-02-01 | 1025
 2020-02-01 | 2020-03-01 | 1009
 2020-02-01 | 2020-04-01 |  926
 2020-02-01 | 2020-05-01 |  878
 2020-02-01 | 2020-06-01 |  833
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2020-03-01 | 2020-03-01 | 1101
 2020-03-01 | 2020-04-01 | 1099
 2020-03-01 | 2020-05-01 | 1029
 2020-03-01 | 2020-06-01 |  974
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2020-04-01 | 2020-04-01 | 1208
 2020-04-01 | 2020-05-01 | 1199
 2020-04-01 | 2020-06-01 | 1104
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2020-05-01 | 2020-05-01 | 1331
 2020-05-01 | 2020-06-01 | 1326
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 2020-06-01 | 2020-06-01 |    4
(21 rows)
```

コホートごとの残存率を計算したいので、初回利用ユーザー数を利用して残存率を計算し、体裁を整えるために連番を付与しておく。

```sql
with tmp as (
select
    account_id,
    date_trunc('month', event_time)::date as ym,
    min(date_trunc('month', event_time)) over (partition by account_id order by date_trunc('month', event_time) asc)::date as firstym
from
    event
group by
    account_id,
    date_trunc('month', event_time)
), tmp1 as (
select
    firstym,
    ym,
    count(account_id) as cnt
from
    tmp
group by
    firstym,
    ym
)
select
    firstym,
    ym,
    cnt,
    first_value(cnt) over (partition by firstym order by ym asc) as total,
    row_number() over (partition by firstym order by ym asc) as rownum,
    round(
    cnt::numeric / first_value(cnt) over (partition by firstym order by ym asc)::numeric
    ,3) as ratio
from
    tmp1
order by
    firstym asc,
    ym asc
;

  firstym   |     ym     | cnt  | total | rownum | ratio
------------+------------+------+-------+--------+-------
 2020-01-01 | 2020-01-01 | 9972 |  9972 |      1 | 1.000
 2020-01-01 | 2020-02-01 | 9945 |  9972 |      2 | 0.997
 2020-01-01 | 2020-03-01 | 9253 |  9972 |      3 | 0.928
 2020-01-01 | 2020-04-01 | 8738 |  9972 |      4 | 0.876
 2020-01-01 | 2020-05-01 | 8354 |  9972 |      5 | 0.838
 2020-01-01 | 2020-06-01 | 8018 |  9972 |      6 | 0.804
 2020-02-01 | 2020-02-01 | 1025 |  1025 |      1 | 1.000
 2020-02-01 | 2020-03-01 | 1009 |  1025 |      2 | 0.984
 2020-02-01 | 2020-04-01 |  926 |  1025 |      3 | 0.903
 2020-02-01 | 2020-05-01 |  878 |  1025 |      4 | 0.857
 2020-02-01 | 2020-06-01 |  833 |  1025 |      5 | 0.813
 2020-03-01 | 2020-03-01 | 1101 |  1101 |      1 | 1.000
 2020-03-01 | 2020-04-01 | 1099 |  1101 |      2 | 0.998
 2020-03-01 | 2020-05-01 | 1029 |  1101 |      3 | 0.935
 2020-03-01 | 2020-06-01 |  974 |  1101 |      4 | 0.885
 2020-04-01 | 2020-04-01 | 1208 |  1208 |      1 | 1.000
 2020-04-01 | 2020-05-01 | 1199 |  1208 |      2 | 0.993
 2020-04-01 | 2020-06-01 | 1104 |  1208 |      3 | 0.914
 2020-05-01 | 2020-05-01 | 1331 |  1331 |      1 | 1.000
 2020-05-01 | 2020-06-01 | 1326 |  1331 |      2 | 0.996
 2020-06-01 | 2020-06-01 |    4 |     4 |      1 | 1.000
(21 rows)
```

このまま BI などに連携して横展してもよいが、SQL で行う場合は、`rownum`を使って`case`で横展を行う。

```sql
with tmp as (
select
    account_id,
    date_trunc('month', event_time)::date as ym,
    min(date_trunc('month', event_time)) over (partition by account_id order by date_trunc('month', event_time) asc)::date as firstym
from
    event
group by
    account_id,
    date_trunc('month', event_time)
), tmp1 as (
select
    firstym,
    ym,
    count(account_id) as cnt
from
    tmp
group by
    firstym,
    ym
), tmp2 as (
select
    firstym,
    ym,
    cnt,
    first_value(cnt) over (partition by firstym order by ym asc) as total,
    row_number() over (partition by firstym order by ym asc) as rownum,
    round(
    cnt::numeric / first_value(cnt) over (partition by firstym order by ym asc)::numeric
    ,3) as ratio
from
    tmp1
)
select
    firstym,
    max(case when rownum = 1 then ratio else null end) as m0,
    max(case when rownum = 2 then ratio else null end) as m1,
    max(case when rownum = 3 then ratio else null end) as m2,
    max(case when rownum = 4 then ratio else null end) as m3,
    max(case when rownum = 5 then ratio else null end) as m4,
    max(case when rownum = 6 then ratio else null end) as m5
from
    tmp2
group by
    firstym
order by
    firstym asc
;

  firstym   |  m0   |  m1   |  m2   |  m3   |  m4   |  m5
------------+-------+-------+-------+-------+-------+-------
 2020-01-01 | 1.000 | 0.997 | 0.928 | 0.876 | 0.838 | 0.804
 2020-02-01 | 1.000 | 0.984 | 0.903 | 0.857 | 0.813 |
 2020-03-01 | 1.000 | 0.998 | 0.935 | 0.885 |       |
 2020-04-01 | 1.000 | 0.993 | 0.914 |       |       |
 2020-05-01 | 1.000 | 0.996 |       |       |       |
 2020-06-01 | 1.000 |       |       |       |       |
(6 rows)
```

## :closed_book: Reference

None
