## :memo: Overview

ここでは、BigQuery で [ga4_obfuscated_sample_ecommerce](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja) データを使って、GA4 のエクスポートされたデータを BigQuery で集計する。基本的には下記の SQL を参考に、各指標の集計を実際に行い、その際に気づいたことをまとめている。

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

日毎にイベントを集計する場合は下記の通り`countif()`関数や`count(distinct)`のように関数を組み合わせて集計すれば OK。ただ、12 月 31 日までしか指定していないのに、1 月 1 日の結果を表示されている。これは、`event_timestamp`が UTC で記録されていて、テーブルのパーティショニングもそれ基準であるため。そのため、日本時間に変換すると、9 時間が時差の関係で集計されることになる。そのため、12 月 20 日は反対に 9 時間少ないので数値が小さくなる。これを回避するためには、12 月 19 日から指定すればよい。

```
select
　date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  countif(event_name = 'page_view') as cnt_page_view,
  count(distinct user_pseudo_id) as cntd_user_pseudo_id,
  count(1) as cnt
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  _table_suffix between '20201220' and '20201231'
group by
  event_date
order by
  event_date asc
;
```

| event_date | cnt_page_view | cntd_user_pseudo_id | cnt   |
| ---------- | ------------- | ------------------- | ----- |
| 2020-12-20 | 5475          | 1845                | 19167 |
| 2020-12-21 | 10108         | 3413                | 35314 |
| 2020-12-22 | 13909         | 3494                | 48264 |
| 2020-12-23 | 11154         | 3267                | 39130 |
| 2020-12-24 | 9268          | 2836                | 32901 |
| 2020-12-25 | 6385          | 2508                | 22871 |
| 2020-12-26 | 5908          | 2473                | 20864 |
| 2020-12-27 | 6231          | 2447                | 22395 |
| 2020-12-28 | 8220          | 2849                | 28506 |
| 2020-12-29 | 11466         | 3087                | 34304 |
| 2020-12-30 | 11081         | 3055                | 31667 |
| 2020-12-31 | 9600          | 2735                | 27606 |
| 2021-01-01 | 2652          | 801                 | 7525  |

実際に`2020-12-25`を指定して値の検算をしたいのであれば、前後 1 日を範囲指定して集計対象にしてから日付を指定する必要がある。

```
select
  countif(event_name = 'page_view') as cnt_page_view,
  count(distinct user_pseudo_id) as cntd_user_pseudo_id,
  count(1) as cnt
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  _table_suffix between '20201224' and '20201226' and
  date(timestamp_micros(event_timestamp),"Asia/Tokyo") = date('2020-12-25')
;
```

| cnt_page_view | cntd_user_pseudo_id | cnt   |
| ------------- | ------------------- | ----- |
| 6385          | 2508                | 22871 |

## :closed_book: Reference

- [Google アナリティクス 4 e コマースウェブ実装向けの BigQuery サンプル データセット](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja)
- [JSON to Markdown Table](https://kdelmonte.github.io/json-to-markdown-table/)
