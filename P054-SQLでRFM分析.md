## :memo: Overview

RFM 分析を SQL で行う。RFM 分析は、ユーザーを最新購入日(Recency)、購入回数(Frequency)、購入金額合計(monetary)ごとにグループ化し、会員を特徴を把握する分析手法のこと。売上金額が高く、かつ、売上回数も多い優良顧客層などを知ることができる。

場合によっては、3 次元だと 125 通りの組み合わせが存在するので、購入回数(Frequency)、最新購入日(Recency)の二次元のみを利用することも多いかも。また、ランクを利用して総合ランクを計算する場合もある。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

RFM 分析の計算方法は下記の通り。

1. 特定の時点からの会員の最新購入日からの経過日数を計算する
2. 会員の購入回数を計算する
3. 会員の購入金額を計算する
4. 各集計値を利用して会員を分類する

まずは会員の経過日数、購入回数、売上合計を計算し、分類するための集計値を計算する。

```sql
with tmp as (
select
    customerid,
    (select max(date_trunc('day', invoicedate)) from onlinestore) as specificpoint,
    max(date_trunc('day', invoicedate)) as max_date,
    (select max(date_trunc('day', invoicedate)) from onlinestore) - max(date_trunc('day', invoicedate)) as r,
    sum(unitprice) as m,
    count(distinct invoiceno) as f
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid
)
select
*
from
    tmp
;

 customerid |    specificpoint    |      max_date       |    r     |     m     | f
------------+---------------------+---------------------+----------+-----------+----
 12346      | 2011-12-09 00:00:00 | 2011-01-18 00:00:00 | 325 days |      2.08 |  2
 12347      | 2011-12-09 00:00:00 | 2011-12-07 00:00:00 | 2 days   | 481.21008 |  7
 12348      | 2011-12-09 00:00:00 | 2011-09-25 00:00:00 | 75 days  | 178.70998 |  4
 12349      | 2011-12-09 00:00:00 | 2011-11-21 00:00:00 | 18 days  |     605.1 |  1
 12350      | 2011-12-09 00:00:00 | 2011-02-02 00:00:00 | 310 days | 65.299995 |  1
 12352      | 2011-12-09 00:00:00 | 2011-11-03 00:00:00 | 36 days  | 2211.0999 | 11
 12353      | 2011-12-09 00:00:00 | 2011-05-19 00:00:00 | 204 days |      24.3 |  1
 12354      | 2011-12-09 00:00:00 | 2011-04-21 00:00:00 | 232 days | 261.21997 |  1
 12355      | 2011-12-09 00:00:00 | 2011-05-09 00:00:00 | 214 days |     54.65 |  1
 12356      | 2011-12-09 00:00:00 | 2011-11-17 00:00:00 | 22 days  | 188.86996 |  3
(10 rows)
 ~~~~略
```

ただ、このままだと`r`が`interval`型なので、このあとの`case`文での判定ができないので、データ型をキャストして日付にしておく。

```sql
with tmp as (
select
    customerid,
    (select max(invoicedate) from onlinestore)::date as specificpoint,
    max(invoicedate)::date as max_date,
    (select max(invoicedate) from onlinestore)::date - max(invoicedate)::date as r,
    sum(unitprice) as m,
    count(distinct invoiceno) as f
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid
)
select
*
from
    tmp
;

 customerid | specificpoint |  max_date  |  r  |     m     | f
------------+---------------+------------+-----+-----------+----
 12346      | 2011-12-09    | 2011-01-18 | 325 |      2.08 |  2
 12347      | 2011-12-09    | 2011-12-07 |   2 |  481.2099 |  7
 12348      | 2011-12-09    | 2011-09-25 |  75 | 178.70999 |  4
 12349      | 2011-12-09    | 2011-11-21 |  18 |     605.1 |  1
 12350      | 2011-12-09    | 2011-02-02 | 310 |      65.3 |  1
 12352      | 2011-12-09    | 2011-11-03 |  36 | 2211.0996 | 11
 12353      | 2011-12-09    | 2011-05-19 | 204 | 24.300001 |  1
 12354      | 2011-12-09    | 2011-04-21 | 232 | 261.21997 |  1
 12355      | 2011-12-09    | 2011-05-09 | 214 |     54.65 |  1
 12356      | 2011-12-09    | 2011-11-17 |  22 | 188.86996 |  3
(10 rows)
```

ここから各指標を扱いやすいカテゴリサイズにビニングしていく。ビンの範囲はデータや目的に応じて変更する。

```sql
with tmp as (
select
    customerid,
    (select max(invoicedate) from onlinestore)::date as specificpoint,
    max(invoicedate)::date as max_date,
    (select max(invoicedate) from onlinestore)::date - max(invoicedate)::date as r,
    sum(unitprice) as m,
    count(distinct invoiceno) as f
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid
)
select
    customerid,
    r,
    f,
    m,
    case
        when r < 14 then 5
        when r < 28 then 4
        when r < 60 then 3
        when r < 90 then 2
        else 1 end as r_rank,
    case
        when 20 <= f then 5
        when 10 <= f then 4
        when 5  <= f then 3
        when 2  <= f then 2
        else 1 end as f_rank,
    case
        when 10000 <= m then 5
        when 5000  <= m then 4
        when 1000  <= m then 3
        when 500   <= m then 2
        else 1 end as m_rank
from
    tmp
;

 customerid |  r  | f  |     m     | r_rank | f_rank | m_rank
------------+-----+----+-----------+--------+--------+--------
 12346      | 325 |  2 |      2.08 |      1 |      2 |      1
 12347      |   2 |  7 | 481.21014 |      5 |      3 |      1
 12348      |  75 |  4 | 178.70999 |      2 |      2 |      1
 12349      |  18 |  1 |  605.1002 |      4 |      1 |      2
 12350      | 310 |  1 |      65.3 |      1 |      1 |      1
 12352      |  36 | 11 | 2211.0984 |      3 |      4 |      3
 12353      | 204 |  1 |      24.3 |      1 |      1 |      1
 12354      | 232 |  1 | 261.21997 |      1 |      1 |      1
 12355      | 214 |  1 |     54.65 |      1 |      1 |      1
 12356      |  22 |  3 |    188.87 |      4 |      2 |      1
(10 rows)
```

この状態でマスタにランクを返して施策に利用するのもよし、更に集計をして、各ランクごとの人数などをみて分析してもよし。途中の`cross join`の部分を可視化しておく。

```sql
with tmp as (
select
    customerid,
    (select max(invoicedate) from onlinestore)::date as specificpoint,
    max(invoicedate)::date as max_date,
    (select max(invoicedate) from onlinestore)::date - max(invoicedate)::date as r,
    sum(unitprice) as m,
    count(distinct invoiceno) as f
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid
), tmp2 as (
select
    customerid,
    r,
    f,
    m,
    case
        when r < 14 then 5
        when r < 28 then 4
        when r < 60 then 3
        when r < 90 then 2
        else 1 end as r_rank,
    case
        when 20 <= f then 5
        when 10 <= f then 4
        when 5  <= f then 3
        when 2  <= f then 2
        else 1 end as f_rank,
    case
        when 10000 <= m then 5
        when 5000  <= m then 4
        when 1000  <= m then 3
        when 500   <= m then 2
        else 1 end as m_rank
from
    tmp
), tmp3 as (
select generate_series(1,5,1) as rfm_id
)
select
    *,
    case when rfm_id = r_rank then 1 else 0 end as r_flag,
    case when rfm_id = f_rank then 1 else 0 end as f_flag,
    case when rfm_id = m_rank then 1 else 0 end as m_flag
from
    tmp2
cross join
    tmp3
;

 customerid |  r  | f  |     m     | r_rank | f_rank | m_rank | rfm_id | r_flag | f_flag | m_flag
------------+-----+----+-----------+--------+--------+--------+--------+--------+--------+--------
 12346      | 325 |  2 |      2.08 |      1 |      2 |      1 |      1 |      1 |      0 |      1
 12346      | 325 |  2 |      2.08 |      1 |      2 |      1 |      2 |      0 |      1 |      0
 12346      | 325 |  2 |      2.08 |      1 |      2 |      1 |      3 |      0 |      0 |      0
 12346      | 325 |  2 |      2.08 |      1 |      2 |      1 |      4 |      0 |      0 |      0
 12346      | 325 |  2 |      2.08 |      1 |      2 |      1 |      5 |      0 |      0 |      0
 12347      |   2 |  7 |    481.21 |      5 |      3 |      1 |      1 |      0 |      0 |      1
 12347      |   2 |  7 |    481.21 |      5 |      3 |      1 |      2 |      0 |      0 |      0
 12347      |   2 |  7 |    481.21 |      5 |      3 |      1 |      3 |      0 |      1 |      0
 12347      |   2 |  7 |    481.21 |      5 |      3 |      1 |      4 |      0 |      0 |      0
 12347      |   2 |  7 |    481.21 |      5 |      3 |      1 |      5 |      1 |      0 |      0
 12348      |  75 |  4 | 178.70998 |      2 |      2 |      1 |      1 |      0 |      0 |      1
 12348      |  75 |  4 | 178.70998 |      2 |      2 |      1 |      2 |      1 |      1 |      0
 12348      |  75 |  4 | 178.70998 |      2 |      2 |      1 |      3 |      0 |      0 |      0
 12348      |  75 |  4 | 178.70998 |      2 |      2 |      1 |      4 |      0 |      0 |      0
 12348      |  75 |  4 | 178.70998 |      2 |      2 |      1 |      5 |      0 |      0 |      0
 12349      |  18 |  1 |  605.1001 |      4 |      1 |      2 |      1 |      0 |      1 |      0
 12349      |  18 |  1 |  605.1001 |      4 |      1 |      2 |      2 |      0 |      0 |      1
 12349      |  18 |  1 |  605.1001 |      4 |      1 |      2 |      3 |      0 |      0 |      0
 12349      |  18 |  1 |  605.1001 |      4 |      1 |      2 |      4 |      1 |      0 |      0
 12349      |  18 |  1 |  605.1001 |      4 |      1 |      2 |      5 |      0 |      0 |      0
 12350      | 310 |  1 | 65.299995 |      1 |      1 |      1 |      1 |      1 |      1 |      1
 12350      | 310 |  1 | 65.299995 |      1 |      1 |      1 |      2 |      0 |      0 |      0
 12350      | 310 |  1 | 65.299995 |      1 |      1 |      1 |      3 |      0 |      0 |      0
 12350      | 310 |  1 | 65.299995 |      1 |      1 |      1 |      4 |      0 |      0 |      0
 12350      | 310 |  1 | 65.299995 |      1 |      1 |      1 |      5 |      0 |      0 |      0
```

このテーブルに対して集計する。

```sql
with tmp as (
select
    customerid,
    (select max(invoicedate) from onlinestore)::date as specificpoint,
    max(invoicedate)::date as max_date,
    (select max(invoicedate) from onlinestore)::date - max(invoicedate)::date as r,
    sum(unitprice) as m,
    count(distinct invoiceno) as f
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid
), tmp2 as (
select
    customerid,
    r,
    f,
    m,
    case
        when r < 14 then 5
        when r < 28 then 4
        when r < 60 then 3
        when r < 90 then 2
        else 1 end as r_rank,
    case
        when 20 <= f then 5
        when 10 <= f then 4
        when 5  <= f then 3
        when 2  <= f then 2
        else 1 end as f_rank,
    case
        when 10000 <= m then 5
        when 5000  <= m then 4
        when 1000  <= m then 3
        when 500   <= m then 2
        else 1 end as m_rank
from
    tmp
), tmp3 as (
select generate_series(1,5,1) as rfm_id
), tmp4 as (
select
    *,
    case when rfm_id = r_rank then 1 else 0 end as r_flag,
    case when rfm_id = f_rank then 1 else 0 end as f_flag,
    case when rfm_id = m_rank then 1 else 0 end as m_flag
from
    tmp2
cross join
    tmp3
)
select
    rfm_id,
    sum(r_flag) as cnt_r_flag,
    sum(f_flag) as cnt_f_flag,
    sum(m_flag) as cnt_m_flag
from
    tmp4
group by
    rfm_id
order by
    rfm_id desc
;

 rfm_id | cnt_r_flag | cnt_f_flag | cnt_m_flag
--------+------------+------------+------------
      5 |        946 |        156 |          8
      4 |        633 |        381 |          9
      3 |        843 |        838 |        207
      2 |        496 |       1684 |        413
      1 |       1454 |       1313 |       3735
(5 rows)
```

横長であれば下記の SQL でも OK。

```sql
with tmp as (
select
    customerid,
    (select max(invoicedate) from onlinestore)::date as specificpoint,
    max(invoicedate)::date as max_date,
    (select max(invoicedate) from onlinestore)::date - max(invoicedate)::date as r,
    sum(unitprice) as m,
    count(distinct invoiceno) as f
from
    onlinestore
where
    customerid != 'NA'
group by
    customerid
), tmp2 as (
select
    customerid,
    r,
    f,
    m,
    case
        when r < 14 then 5
        when r < 28 then 4
        when r < 60 then 3
        when r < 90 then 2
        else 1 end as r_rank,
    case
        when 20 <= f then 5
        when 10 <= f then 4
        when 5  <= f then 3
        when 2  <= f then 2
        else 1 end as f_rank,
    case
        when 10000 <= m then 5
        when 5000  <= m then 4
        when 1000  <= m then 3
        when 500   <= m then 2
        else 1 end as m_rank
from
    tmp
), tmp3 as (
select 'r' as rfm, r_rank as rank, count(customerid) as cnt from tmp2 group by r_rank
union all select 'f' as rfm, f_rank as rank, count(customerid) as cnt from tmp2 group by f_rank
union all select 'm' as rfm, m_rank as rank, count(customerid) as cnt from tmp2 group by m_rank
order by rfm, rank desc
)
select
    rfm,
    max(case when rank = 5 then cnt else 0 end) as rank5,
    max(case when rank = 4 then cnt else 0 end) as rank4,
    max(case when rank = 3 then cnt else 0 end) as rank3,
    max(case when rank = 2 then cnt else 0 end) as rank2,
    max(case when rank = 1 then cnt else 0 end) as rank1
from
    tmp3
group by
    rfm
;

 rfm | rank5 | rank4 | rank3 | rank2 | rank1
-----+-------+-------+-------+-------+-------
 f   |   156 |   381 |   838 |  1684 |  1313
 m   |     8 |     9 |   207 |   413 |  3735
 r   |   946 |   633 |   843 |   496 |  1454
(3 rows)
```

## :closed_book: Reference

None
