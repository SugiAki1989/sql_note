## :memo: Overview

Preppin' Data challenge の「2019: Week 11」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/04/2019-week-11.html)
- [Answer](https://preppindata.blogspot.com/2019/04/2019-week-11-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。よくわからないが JSON が元のデータで CSV に変換することで、フォーマットがくずれてる状態からスタート。株式の関するデータに最終的に整形する、というお題。

```sql
select * from p2019w11t1 limit 30;
                         json_name                          | json_valuestring
------------------------------------------------------------+------------------
 chart.result.0.meta.currency                               | USD
 chart.result.0.meta.symbol                                 | DATA
 chart.result.0.meta.exchangeName                           | NYQ
 chart.result.0.meta.instrumentType                         | EQUITY
 chart.result.0.meta.firstTradeDate                         | 1368777600
 chart.result.0.meta.gmtoffset                              | -14400
 chart.result.0.meta.timezone                               | EDT
 chart.result.0.meta.exchangeTimezoneName                   | America/New_York
 chart.result.0.meta.chartPreviousClose                     | 54.09
 chart.result.0.meta.priceHint                              | 2
 chart.result.0.meta.currentTradingPeriod.pre.timezone      | EDT
 chart.result.0.meta.currentTradingPeriod.pre.start         | 1556006400
 chart.result.0.meta.currentTradingPeriod.pre.end           | 1556026200
 chart.result.0.meta.currentTradingPeriod.pre.gmtoffset     | -14400
 chart.result.0.meta.currentTradingPeriod.regular.timezone  | EDT
 chart.result.0.meta.currentTradingPeriod.regular.start     | 1556026200
 chart.result.0.meta.currentTradingPeriod.regular.end       | 1556049600
 chart.result.0.meta.currentTradingPeriod.regular.gmtoffset | -14400
 chart.result.0.meta.currentTradingPeriod.post.timezone     | EDT
 chart.result.0.meta.currentTradingPeriod.post.start        | 1556049600
 chart.result.0.meta.currentTradingPeriod.post.end          | 1556064000
 chart.result.0.meta.currentTradingPeriod.post.gmtoffset    | -14400
 chart.result.0.meta.dataGranularity                        | 1d
 chart.result.0.meta.validRanges.0                          | 1d
 chart.result.0.meta.validRanges.1                          | 5d
 chart.result.0.meta.validRanges.2                          | 1mo
 chart.result.0.meta.validRanges.3                          | 3mo
 chart.result.0.meta.validRanges.4                          | 6mo
 chart.result.0.meta.validRanges.5                          | 1y
 chart.result.0.meta.validRanges.6                          | 2y
(30 rows)
```

必要な部分にある情報を取り出しておく。

```sql
select
    split_part(json_name, '.', 4) as json_name4,
    split_part(json_name, '.', 5) as json_name5,
    split_part(json_name, '.', 7) as json_name7,
    split_part(json_name, '.', 8) as json_name8,
    json_valuestring as value
from
    p2019w11t1
where
    split_part(json_name, '.', 4) = 'indicators' or
    split_part(json_name, '.', 4) = 'timestamp'
limit 10
;
 json_name4 | json_name5 | json_name7 | json_name8 |   value
------------+------------+------------+------------+------------
 timestamp  | 0          |            |            | 1493040600
 timestamp  | 1          |            |            | 1493127000
 timestamp  | 2          |            |            | 1493213400
 timestamp  | 3          |            |            | 1493299800
 timestamp  | 4          |            |            | 1493386200
 timestamp  | 5          |            |            | 1493645400
 timestamp  | 6          |            |            | 1493731800
 timestamp  | 7          |            |            | 1493818200
 timestamp  | 8          |            |            | 1493904600
 timestamp  | 9          |            |            | 1493991000
(10 rows)
```

必要な部分にある情報を取り出しておく。カラムの値に応じて`data_type`と`row_`を作る。

```sql
with tmp as (
select
    split_part(json_name, '.', 4) as json_name4,
    split_part(json_name, '.', 5) as json_name5,
    split_part(json_name, '.', 7) as json_name7,
    split_part(json_name, '.', 8) as json_name8,
    json_valuestring as value
from
    p2019w11t1
where
    split_part(json_name, '.', 4) = 'indicators' or
    split_part(json_name, '.', 4) = 'timestamp'
), tmp2 as (
select
    json_name4,
    json_name5,
    json_name7,
    json_name8,
    value,
    case when json_name7 = '' then json_name4 else json_name7 end as data_type,
    case when json_name8 = '' then json_name5 else json_name8 end as row_
from
    tmp
)
select * from tmp2;

 json_name4 | json_name5 | json_name7 | json_name8 |      value       | data_type | row_
------------+------------+------------+------------+------------------+-----------+------
 timestamp  | 0          |            |            | 1493040600       | timestamp | 0
 timestamp  | 1          |            |            | 1493127000       | timestamp | 1
 timestamp  | 2          |            |            | 1493213400       | timestamp | 2
 timestamp  | 3          |            |            | 1493299800       | timestamp | 3
 timestamp  | 4          |            |            | 1493386200       | timestamp | 4
[snip]
 indicators | adjclose   | adjclose   | 497        | 125.849998474121 | adjclose  | 497
 indicators | adjclose   | adjclose   | 498        | 124.599998474121 | adjclose  | 498
 indicators | adjclose   | adjclose   | 499        | 121.309997558594 | adjclose  | 499
 indicators | adjclose   | adjclose   | 500        | 118.569999694824 | adjclose  | 500
 indicators | adjclose   | adjclose   | 501        | 119.430000305176 | adjclose  | 501
 indicators | adjclose   | adjclose   | 502        | 118.959999084473 | adjclose  | 502
(3521 rows)

```

あとは LongToWide 変換をしたいので、`case`文で処理してから、

```sql
with tmp as (
select
    split_part(json_name, '.', 4) as json_name4,
    split_part(json_name, '.', 5) as json_name5,
    split_part(json_name, '.', 7) as json_name7,
    split_part(json_name, '.', 8) as json_name8,
    json_valuestring as value
from
    p2019w11t1
where
    split_part(json_name, '.', 4) = 'indicators' or
    split_part(json_name, '.', 4) = 'timestamp'
), tmp2 as (
select
    json_name4,
    json_name5,
    json_name7,
    json_name8,
    value,
    case when json_name7 = '' then json_name4 else json_name7 end as data_type,
    case when json_name8 = '' then json_name5 else json_name8 end as row_
from
    tmp
)
select
    row_,
    data_type,
    value,
    case when data_type = 'adjclose' then value else null end as adjclose,
    case when data_type = 'close' then value else null end as close,
    case when data_type = 'high' then value else null end as high,
    case when data_type = 'low' then value else null end as low,
    case when data_type = 'open' then value else null end as open,
    case when data_type = 'timestamp' then value else null end as timestamp,
    case when data_type = 'volume' then value else null end as volume
from
    tmp2
;

 row_ | data_type |      value       |     adjclose     |      close       |       high       |       low        |       open       | timestamp  | volume
------+-----------+------------------+------------------+------------------+------------------+------------------+------------------+------------+---------
 0    | timestamp | 1493040600       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493040600 | (null)
 1    | timestamp | 1493127000       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493127000 | (null)
 2    | timestamp | 1493213400       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493213400 | (null)
 3    | timestamp | 1493299800       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493299800 | (null)
 4    | timestamp | 1493386200       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493386200 | (null)
 5    | timestamp | 1493645400       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493645400 | (null)
 6    | timestamp | 1493731800       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493731800 | (null)
 7    | timestamp | 1493818200       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493818200 | (null)
 8    | timestamp | 1493904600       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493904600 | (null)
 9    | timestamp | 1493991000       | (null)           | (null)           | (null)           | (null)           | (null)           | 1493991000 | (null)
 10   | timestamp | 1494250200       | (null)           | (null)           | (null)           | (null)           | (null)           | 1494250200 | (null)
[snip]
```

グループ化を行なう。これで横に広がるので、あとは UNIX 秒を修正して完了。

```sql

with tmp as (
select
    split_part(json_name, '.', 4) as json_name4,
    split_part(json_name, '.', 5) as json_name5,
    split_part(json_name, '.', 7) as json_name7,
    split_part(json_name, '.', 8) as json_name8,
    json_valuestring as value
from
    p2019w11t1
where
    split_part(json_name, '.', 4) = 'indicators' or
    split_part(json_name, '.', 4) = 'timestamp'
), tmp2 as (
select
    json_name4,
    json_name5,
    json_name7,
    json_name8,
    value,
    case when json_name7 = '' then json_name4 else json_name7 end as data_type,
    case when json_name8 = '' then json_name5 else json_name8 end as row_
from
    tmp
), tmp3 as (
select
    row_,
    data_type,
    value,
    case when data_type = 'adjclose' then value else null end as adjclose,
    case when data_type = 'close' then value else null end as close,
    case when data_type = 'high' then value else null end as high,
    case when data_type = 'low' then value else null end as low,
    case when data_type = 'open' then value else null end as open,
    case when data_type = 'timestamp' then value else null end as timestamp,
    case when data_type = 'volume' then value else null end as volume
from
    tmp2
)
select
    row_,
    max(adjclose) as max_adjclose,
    to_timestamp(max(timestamp)::int) as max_timestamp,
    max(high) as max_high,
    max(low) as max_low,
    max(open) as max_open,
    max(close) as max_close,
    max(volume) as max_volume
from
    tmp3
group by
    row_
order by
    row_::int asc
;

 row_ |   max_adjclose   |     max_timestamp      |     max_high     |     max_low      |     max_open     |    max_close     | max_volume
------+------------------+------------------------+------------------+------------------+------------------+------------------+------------
 0    | 53.9799995422363 | 2017-04-24 22:30:00+09 | 54.5699996948242 | 53               | 54.3600006103516 | 53.9799995422363 | 1349500
 1    | 53.6300010681152 | 2017-04-25 22:30:00+09 | 54.4199981689453 | 53.6199989318848 | 54.3699989318848 | 53.6300010681152 | 777400
 2    | 53.6500015258789 | 2017-04-26 22:30:00+09 | 53.9500007629395 | 53.0299987792969 | 53.6199989318848 | 53.6500015258789 | 591900
 3    | 53.4799995422363 | 2017-04-27 22:30:00+09 | 54.1300010681152 | 53.3499984741211 | 53.7999992370605 | 53.4799995422363 | 739100
 4    | 53.6800003051758 | 2017-04-28 22:30:00+09 | 53.8800010681152 | 53.2200012207031 | 53.4300003051758 | 53.6800003051758 | 585300
 5    | 54.1100006103516 | 2017-05-01 22:30:00+09 | 54.2400016784668 | 53.5             | 53.75            | 54.1100006103516 | 817500
 6    | 54.7799987792969 | 2017-05-02 22:30:00+09 | 54.8499984741211 | 53.6500015258789 | 54.1599998474121 | 54.7799987792969 | 1203100
 7    | 54.7000007629395 | 2017-05-03 22:30:00+09 | 55.1199989318848 | 54.2099990844727 | 54.4799995422363 | 54.7000007629395 | 1918000
 8    | 58.0499992370605 | 2017-05-04 22:30:00+09 | 63.3400001525879 | 57.6500015258789 | 61.8899993896484 | 58.0499992370605 | 5166300
 9    | 60.439998626709  | 2017-05-05 22:30:00+09 | 60.75            | 57.9900016784668 | 58.2299995422363 | 60.439998626709  | 2228500
 10   | 60.0499992370605 | 2017-05-08 22:30:00+09 | 60.5499992370605 | 59.439998626709  | 59.5299987792969 | 60.0499992370605 | 2401000
```

## :closed_book: Reference

None
