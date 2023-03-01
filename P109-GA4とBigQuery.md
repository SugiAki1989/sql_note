## :memo: Overview

ここでは、BigQuery で `[ga4_obfuscated_sample_ecommerce](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja)` データを使って、GA4 のエクスポートされたデータを BigQuery で集計する。基本的には下記の SQL を参考に、各指標の集計を実際に行い、その際に気づいたことをまとめている。

- [GA4 用の BigQuery クエリ集](https://www.ga4.guide/related-service/big-query/query-writing/)

GA4 および GA4 からエクスポートされるデータについては素人なので、まとめている内容に誤りがある可能性がある点は注意が必要。

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`event`

## :pencil2: Example

GA4 はイベント単位で計測され、そのイベント単位のレコードが積み上げられたようなテーブル構造をしている。イベントごとに RECORD 型というネストしたカラムを持っており、これを展開しなければ、普通の Web アクセスログと似たようなデータである。展開してネストしている値を利用する場合は`unnest()`関数を利用する必要がある。

```
select
  user_pseudo_id,
  datetime(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_datetime,
  event_name
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_20201214`
where
  user_pseudo_id = "86253338.5702333040"
order by
  event_datetime asc
;
```

| user_pseudo_id      | event_datetime             | event_name       |
| ------------------- | -------------------------- | ---------------- |
| 86253338.5702333040 | 2020-12-15T08:45:53.853920 | first_visit      |
| 86253338.5702333040 | 2020-12-15T08:45:53.853920 | session_start    |
| 86253338.5702333040 | 2020-12-15T08:45:53.853920 | page_view        |
| 86253338.5702333040 | 2020-12-15T08:45:59.584592 | view_promotion   |
| 86253338.5702333040 | 2020-12-15T08:46:32.658280 | view_promotion   |
| 86253338.5702333040 | 2020-12-15T08:46:32.658280 | select_promotion |
| 86253338.5702333040 | 2020-12-15T08:46:39.008503 | page_view        |
| 86253338.5702333040 | 2020-12-15T08:46:44.334876 | user_engagement  |
| 86253338.5702333040 | 2020-12-15T08:46:50.665941 | page_view        |
| 86253338.5702333040 | 2020-12-15T08:48:07.060942 | user_engagement  |
| 86253338.5702333040 | 2020-12-15T08:48:13.647870 | page_view        |
| 86253338.5702333040 | 2020-12-15T08:49:17.843547 | view_item        |
| 86253338.5702333040 | 2020-12-15T08:49:27.561984 | user_engagement  |
| 86253338.5702333040 | 2020-12-15T08:49:27.561984 | add_to_cart      |
| 86253338.5702333040 | 2020-12-15T08:49:34.211393 | page_view        |
| 86253338.5702333040 | 2020-12-15T08:49:39.913179 | user_engagement  |
| -snip-              | - snip-                    | - snip-          |
| 86253338.5702333040 | 2020-12-15T08:58:18.363014 | view_item        |
| 86253338.5702333040 | 2020-12-15T08:58:22.341017 | user_engagement  |
| 86253338.5702333040 | 2020-12-15T08:58:29.498471 | user_engagement  |

このようなテーブル構造をしているので、イベントを単純に集計したければ SQL は下記の通りとなる。ここではユーザーを`86253338.5702333040`に限定している。イベントの数を計測する要望があるのかはわからないが、下記のように集計はできる。

```
select
  event_name,
  count(1) as cnt, -- nullはないので下記でも同じ
  count(event_name) as cnt2
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_20201214`
where
  user_pseudo_id = "86253338.5702333040"
group by
  event_name
order by
  cnt desc
;
```

| event_name       | cnt | cnt2 |
| ---------------- | --- | ---- |
| page_view        | 23  | 23   |
| user_engagement  | 22  | 22   |
| view_item        | 11  | 11   |
| scroll           | 5   | 5    |
| view_promotion   | 5   | 5    |
| add_to_cart      | 4   | 4    |
| select_promotion | 2   | 2    |
| first_visit      | 1   | 1    |
| session_start    | 1   | 1    |
| select_item      | 1   | 1    |

GA4 をエクスポートしたデータはイベントごと RECORD 型カラムと合わせて積み上げられているので、必要に応じてネストを解除しながら集計することになる。他にも日毎にセッション数を集計したければ、`_table_suffix`を利用して期間を設定し、あとはグループ化して集計すればよい。ちなみにこの方法では厳密なセッション数は計算できないので注意。

```
select
  date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  event_name,
  count(1) as cnt,
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "86253338.5702333040" and
  event_name = 'session_start' and
  _table_suffix between '20201201' and '20201231'
group by
  event_date,
  event_name
order by
  cnt desc
;
```

| event_date | event_name    | cnt |
| ---------- | ------------- | --- |
| 2020-12-15 | session_start | 3   |
| 2020-12-18 | session_start | 1   |
| 2020-12-16 | session_start | 1   |
| 2020-12-28 | session_start | 1   |
| 2020-12-22 | session_start | 1   |
| 2020-12-19 | session_start | 1   |
| 2020-12-24 | session_start | 1   |
| 2020-12-17 | session_start | 1   |
| 2020-12-20 | session_start | 1   |

## :closed_book: Reference

- [Google アナリティクス 4 e コマースウェブ実装向けの BigQuery サンプル データセット](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja)
- [JSON to Markdown Table](https://kdelmonte.github.io/json-to-markdown-table/)
