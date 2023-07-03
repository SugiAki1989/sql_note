## :memo: Overview

ここでは、BigQuery で [ga4_obfuscated_sample_ecommerce](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja) データを使って、GA4 のエクスポートされたデータを BigQuery で集計する。基本的には下記の SQL を参考に、各指標の集計を実際に行い、その際に気づいたことをまとめている。

- [GA4 用の BigQuery クエリ集](https://www.ga4.guide/related-service/big-query/query-writing/)

GA4 および GA4 からエクスポートされるデータについては素人なので、まとめている内容に誤りがある可能性がある点は注意が必要。

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`date_add`、`date_sub`、`format_date`、`extract()`

## :pencil2: Example

今回は SQL で集計する際の日付の設定方法についてまとめておく。たとえば、本日 2023 年 3 月 5 日を基準として、本日の 365 日前から 1 日前までを集計対象としたいのであれば、`current_date()`関数と`date_sub()`関数を組み合わせることで、本日から n 日前という指定が可能になる。

```
select
  current_date() as today,
  date_sub(current_date(), interval 365 day) as current_pre365,
  format_date('%Y%m%d', date_sub(current_date(), interval 365 day)) as current_pre365_formatted,
  date_sub(current_date(), interval 1 day) as current_pre1,
  format_date('%Y%m%d', date_sub(current_date(), interval 1 day)) as current_pre1_formatted,
;
```

| today      | current_pre365 | current_pre365_formatted | current_pre1 | current_pre1_formatted |
| ---------- | -------------- | ------------------------ | ------------ | ---------------------- |
| 2023-03-05 | 2022-03-05     | 20220305                 | 2023-03-04   | 20230304               |

`_table_suffix`の指定が`yyyymmdd`というハイフンやスラッシュがない文字列を渡す必要があるので、`format_date()`関数で表示を整えている。こうすることで、テーブルの集計期間は`_table_suffix between 20220305 and 20230304`となり、2022 年 3 月 5 日から 2023 年 3 月 4 日までが対象となる。

```
where
  _table_suffix between
    format_date('%Y%m%d', date_sub(current_date(), interval 365 day)) and
    format_date('%Y%m%d', date_sub(current_date(), interval 1 day))
```

他にも、日時から時間要素を取り出して集計したい場面はよくある。日時から時間要素を取り出して集計する場合は`extract([hour] from timestamp [at time zone 'Asia/Tokyo'])`とすれば時間要素を抽出できる。`extract()`関数には様々な時間要素を利用できる。

```
select
  1677983184000000 as epochsec,
  datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo') as jst,
  extract(date from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as date,
  extract(year from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as year,
  extract(quarter from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as quarter,
  extract(month from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as month,
  extract(day from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as day,
  -- dayofyear	1月1日を1として何日目かを表す
  extract(dayofyear from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as dayofyear,
  extract(week from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as week,
  extract(hour from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as hour,
  extract(minute from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as minute,
  extract(second from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as second
;
```

| epochsec         | jst                 | date       | year | quarter | month | day | dayofyear | week | hour | minute | second |
| ---------------- | ------------------- | ---------- | ---- | ------- | ----- | --- | --------- | ---- | ---- | ------ | ------ |
| 1677983184000000 | 2023-03-05T11:26:24 | 2023-03-05 | 2023 | 1       | 3     | 5   | 64        | 10   | 11   | 26     | 24     |

下記は、同じ結果を返すが、タイムゾーンの変換を実行している場所が異なる SQL。

```
-- 1677983184000000は2023-03-05 11:26:24
select
  1677983184000000 as epochsec,
  -- utc > jst > extract
  datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo') as jst1,
  extract(hour from datetime(timestamp_micros(1677983184000000), 'Asia/Tokyo')) as jst1_hour,
  -- utc > extract&convert
  timestamp_micros(1677983184000000) as jst2,
  extract(hour from timestamp_micros(1677983184000000) at time zone "Asia/Tokyo") as jst2_hour
;
```

タイムゾーンの変換をどこでするかはさておき、時間ごとの集計は`extract()`関数で抽出した単位でグループ化すればよい。

```
select
    date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
    extract(hour from datetime(timestamp_micros(event_timestamp), 'Asia/Tokyo')) as hour,
    count(event_name) as cnt_event
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "86253338.5702333040" and
  _table_suffix between '20201201' and '20201231'
group by
  event_date,
  hour
order by
  event_date asc,
  hour asc
;

```

| event_date | hour | cnt_event |
| ---------- | ---- | --------- |
| 2020-12-15 | 8    | 75        |
| 2020-12-15 | 21   | 63        |
| 2020-12-15 | 22   | 2         |
| 2020-12-16 | 3    | 31        |
| 2020-12-17 | 10   | 5         |
| 2020-12-18 | 3    | 2         |
| 2020-12-19 | 9    | 250       |
| 2020-12-19 | 10   | 111       |
| 2020-12-20 | 2    | 24        |
| 2020-12-22 | 22   | 29        |
| 2020-12-24 | 0    | 6         |
| 2020-12-28 | 1    | 42        |

他にも時間関係の処理として、初回アクセス日のユーザー数を集計する際には便利かもしれないカラムがある。それは`user_first_touch_timestamp`というカラムで、このカラムには、ユーザーが初めてアプリを起動したか、サイトに訪れた時刻（マイクロ秒単位）が記録されている。

ただよくわからない点があって、2020 年 11 月 01 日から 2020 年 12 月 31 日の間にイベントがあるユーザーの`user_first_touch_timestamp`単位で集計してみると、`user_first_touch_timestamp`が`null`のユーザーがそれなりに出現する。この期間に何かしらのイベントを発火させているが、`user_first_touch_timestamp`を持ってない。考えられそうな理由として、GA4 のデータを GA タグを利用して記録しだした前後で、記録後のみ確実に初回アクセスと判定できるユーザーにのみ`user_first_touch_timestamp`が記録され、タグ利用前からアクセスしているユーザーは、何かしらの方法で判定して、`user_first_touch_timestamp`を付与してないのか・・・？

また、`user_first_touch_timestamp`を持っている`user_pseudo_id`は、`first_visit`したイベントのときの同じ`user_first_touch_timestamp`の値を持つレコードとなるはず。そして、同じ`user_pseudo_id`では`first_visit`は 1 回のはずなので、集計期間内の 2020 年 11 月 01 日から 2020 年 12 月 31 日ではあれば下記の SQL の結果は一致しそうであるが合わない。

```
select
  extract(date from datetime(timestamp_micros(user_first_touch_timestamp),"Asia/Tokyo")) as day_first_touch,
  count(distinct user_pseudo_id) as first_users1,
  -- こちらの値のほうが同じか、大きくなる
  sum(if(event_name = 'first_visit', 1, 0)) as first_users2
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  _table_suffix between '20201101' and '20201231'
group by
  day_first_touch
order by
  day_first_touch asc
;
```

| day_first_touch | first_users1 | first_users2 |
| --------------- | ------------ | ------------ |
|                 | 2909         | 0            |
| 2019-10-15      | 2            | 0            |
| 2019-10-16      | 2            | 0            |
| 2019-10-17      | 1            | 0            |
| 2019-10-18      | 1            | 0            |
| 2019-10-19      | 2            | 0            |
| 2019-10-20      | 5            | 0            |
| (snip)          | (snip)       | (snip)       |
| 2020-12-26      | 2202         | 2203         |
| 2020-12-27      | 2150         | 2150         |
| 2020-12-28      | 2461         | 2462         |
| 2020-12-29      | 2567         | 2567         |
| 2020-12-30      | 2614         | 2614         |
| 2020-12-31      | 2340         | 2340         |
| 2021-01-01      | 678          | 678          |

調べてみると、同じユーザーと識別されているにも関わらず、`first_visit`イベントを異なるタイミングで複数回、発火させているユーザーがちらほらいるため、`date_first_touch`が同じであっても、集計期間内であっても、さきほどの SQL では違いがでてしまう。前者の SQL の方がおそらく実態に即していると思われる。また、初めてサイトにアクセスしたときの`event_timestamp = date_first_touch`だと思っていたが、そうでもないケースがある模様。

```
select
  user_pseudo_id,
  datetime(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_timestamp,
  event_name,
  (select value.int_value from unnest(event_params) where key = 'ga_session_id') as session_id,
  datetime(timestamp_micros(user_first_touch_timestamp),"Asia/Tokyo") as date_first_touch
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id in ('5661311.7262579481', '70077886.1506277727') and
  event_name ='first_visit' and
  _table_suffix between '20201205' and '20201212'
order by
  user_pseudo_id desc,
  event_timestamp asc
;
```

| user_pseudo_id      | event_timestamp            | event_name  | session_id | date_first_touch           |
| ------------------- | -------------------------- | ----------- | ---------- | -------------------------- |
| 70077886.1506277727 | 2020-12-10T14:16:11.518192 | first_visit | 5988720141 | 2020-12-07T13:30:20.751916 |
| 70077886.1506277727 | 2020-12-10T15:16:41.001167 | first_visit | 7658623401 | 2020-12-07T13:30:20.751916 |
| 5661311.7262579481  | 2020-12-10T12:37:24.754851 | first_visit | 7442718469 | 2020-12-06T13:04:27.561559 |
| 5661311.7262579481  | 2020-12-10T12:37:27.140097 | first_visit | 7442718469 | 2020-12-06T13:04:27.561559 |

先ほどと似ているが、ユーザーの初回アクセスから n 日以内のイベントを集計したいといことであれば、下記のようにユーザー単位で、`date_sub()`関数を利用しながら集計範囲のデータをフラグづするなりして、レコードをフィルタリングすれば OK。あくまでも説明のためにセッションイベントのみにしているので、この SQL がそのまま要件を満たすわけではない。

```
with event_history as (
select
  distinct
  user_pseudo_id,
  event_name,
  date(TIMESTAMP_MICROS(event_timestamp), 'Asia/Tokyo') as date,
  min(date(TIMESTAMP_MICROS(event_timestamp), 'Asia/Tokyo'))
    over(partition by user_pseudo_id order by date(TIMESTAMP_MICROS(event_timestamp), 'Asia/Tokyo') rows between unbounded preceding and unbounded following) as min_date
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = '86253338.5702333040' and
  event_name = 'session_start' and
  _table_suffix between '20201201' and '20201231'
order by
  date asc
)
select
  *,
  -- 初回イベント日を起算日として3日
  date_add(min_date, interval 2 day) as min_date_plus3,
  if(date <= date_add(min_date, interval 2 day), 1, 0) as target_date
from
  event_history
;
```

| user_pseudo_id      | event_name    | date       | min_date   | min_date_plus3 | target_date |
| ------------------- | ------------- | ---------- | ---------- | -------------- | ----------- |
| 86253338.5702333040 | session_start | 2020-12-15 | 2020-12-15 | 2020-12-17     | 1           |
| 86253338.5702333040 | session_start | 2020-12-16 | 2020-12-15 | 2020-12-17     | 1           |
| 86253338.5702333040 | session_start | 2020-12-17 | 2020-12-15 | 2020-12-17     | 1           |
| 86253338.5702333040 | session_start | 2020-12-18 | 2020-12-15 | 2020-12-17     | 0           |
| 86253338.5702333040 | session_start | 2020-12-19 | 2020-12-15 | 2020-12-17     | 0           |
| 86253338.5702333040 | session_start | 2020-12-20 | 2020-12-15 | 2020-12-17     | 0           |
| 86253338.5702333040 | session_start | 2020-12-22 | 2020-12-15 | 2020-12-17     | 0           |
| 86253338.5702333040 | session_start | 2020-12-24 | 2020-12-15 | 2020-12-17     | 0           |
| 86253338.5702333040 | session_start | 2020-12-28 | 2020-12-15 | 2020-12-17     | 0           |

## :closed_book: Reference

- [Google アナリティクス 4 e コマースウェブ実装向けの BigQuery サンプル データセット](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja)
- [JSON to Markdown Table](https://kdelmonte.github.io/json-to-markdown-table/)
