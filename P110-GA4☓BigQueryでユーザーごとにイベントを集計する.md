## :memo: Overview

ここでは、BigQuery で [ga4_obfuscated_sample_ecommerce](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja) データを使って、GA4 のエクスポートされたデータを BigQuery で集計する。基本的には下記の SQL を参考に、各指標の集計を実際に行い、その際に気づいたことをまとめている。

- [GA4 用の BigQuery クエリ集](https://www.ga4.guide/related-service/big-query/query-writing/)

GA4 および GA4 からエクスポートされるデータについては素人なので、まとめている内容に誤りがある可能性がある点は注意が必要。

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`event`

## :pencil2: Example

今回はユーザーごとにイベントを計測する。まず、GA4 のユーザーには 2 種類ある。

- [[GA4] BigQuery Export スキーマ](https://support.google.com/firebase/answer/7029846?hl=ja#zippy=%2Cuser:~:text=event-,user,-user%20%E3%83%95%E3%82%A3%E3%83%BC%E3%83%AB%E3%83%89%E3%81%AB)

| カラム名         | 説明                                              |
| ---------------- | ------------------------------------------------- |
| `user_id`        | setUserId API によって設定されるユーザー ID。     |
| `user_pseudo_id` | ユーザーの仮の ID（アプリインスタンス ID など）。 |

`user_id` はログイン ID や会員 ID など設定することで、GA4 で`user_id`として利用でき、GA4 がアプリの場合 Instance ID、ウェブの場合は cookie などを利用して自動で付与する ID が`user_pseudo_id`。ここでは、`user_id` は設定してないので利用できないため、`user_pseudo_id`をユーザーを識別するカラムとして利用する。

イベントごとに集計したければ、`countif()`関数で特定のイベントをカウントすることで集計できる。`count(if())`の組み合わせだと誤りなので注意。`if()`関数を利用するのであれば、`sum()`関数で 1 を合計する必要がある。

```
select
　user_pseudo_id,
  countif(event_name = 'page_view') as cnt_page_view,
  -- これは誤りで、0,1 のレコード数をカウントしているため行数が返される。
  count(if(event_name = 'page_view', 1, 0)) as miss_page_view,
  sum(if(event_name = 'page_view', 1, 0)) as sum_page_view
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "86253338.5702333040" and
  _table_suffix between '20201201' and '20201231'
group by
  user_pseudo_id
;
```

| user_pseudo_id      | cnt_page_view | miss_page_view | sum_page_view |
| ------------------- | ------------- | -------------- | ------------- |
| 86253338.5702333040 | 208           | 640            | 208           |

このような横長でイベントを集計したいのであれば、`countif()`関数を使うことで、横長(WIDE 型)で集計できる。

```
select
　user_pseudo_id,
  countif(event_name = 'first_visit') as cnt_first_visit,
  countif(event_name = 'session_start') as cnt_session_start,
  countif(event_name = 'page_view') as cnt_page_view,
  countif(event_name = 'user_engagement') as cnt_user_engagement,
  countif(event_name = 'click') as cnt_click,
  countif(event_name = 'scroll') as cnt_scroll,
  countif(event_name = 'view_item') as cnt_view_item,
  countif(event_name = 'select_item') as cnt_select_item,
  countif(event_name = 'view_search_results') as cnt_view_search_results,
  countif(event_name = 'view_promotion') as cnt_view_promotion,
  countif(event_name = 'select_promotion') as cnt_select_promotion,
  countif(event_name = 'add_to_cart') as cnt_add_to_cart,
  countif(event_name = 'purchase') as cnt_purchase,
  countif(event_name = 'begin_checkout') as cnt_begin_checkout,
  countif(event_name = 'add_payment_info') as cnt_add_payment_info,
  countif(event_name = 'add_shipping_info') as cnt_add_shipping_info
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "86253338.5702333040" and
  _table_suffix between '20201201' and '20201231'
group by
  user_pseudo_id
;
```

| user_pseudo_id      | cnt_first_visit | cnt_session_start | cnt_page_view | cnt_user_engagement | cnt_click | cnt_scroll | cnt_view_item | cnt_select_item | cnt_view_search_results | cnt_view_promotion | cnt_select_promotion | cnt_add_to_cart | cnt_purchase | cnt_begin_checkout | cnt_add_payment_info | cnt_add_shipping_info |
| ------------------- | --------------- | ----------------- | ------------- | ------------------- | --------- | ---------- | ------------- | --------------- | ----------------------- | ------------------ | -------------------- | --------------- | ------------ | ------------------ | -------------------- | --------------------- |
| 86253338.5702333040 | 1               | 11                | 208           | 202                 | 4         | 73         | 44            | 1               | 1                       | 21                 | 2                    | 7               | 6            | 36                 | 10                   | 13                    |

縦長(LONG 型)で集計したければ、グループ化してカウントすれば集計できる。

```
select
　user_pseudo_id,
  event_name,
  count(1) as cnt
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "86253338.5702333040" and
  _table_suffix between '20201201' and '20201231'
group by
  user_pseudo_id,
  event_name
order by
  cnt desc
;
```

| user_pseudo_id      | event_name          | cnt |
| ------------------- | ------------------- | --- |
| 86253338.5702333040 | page_view           | 208 |
| 86253338.5702333040 | user_engagement     | 202 |
| 86253338.5702333040 | scroll              | 73  |
| 86253338.5702333040 | view_item           | 44  |
| 86253338.5702333040 | begin_checkout      | 36  |
| 86253338.5702333040 | view_promotion      | 21  |
| 86253338.5702333040 | add_shipping_info   | 13  |
| 86253338.5702333040 | session_start       | 11  |
| 86253338.5702333040 | add_payment_info    | 10  |
| 86253338.5702333040 | add_to_cart         | 7   |
| 86253338.5702333040 | purchase            | 6   |
| 86253338.5702333040 | click               | 4   |
| 86253338.5702333040 | select_promotion    | 2   |
| 86253338.5702333040 | first_visit         | 1   |
| 86253338.5702333040 | select_item         | 1   |
| 86253338.5702333040 | view_search_results | 1   |

## :closed_book: Reference

- [Google アナリティクス 4 e コマースウェブ実装向けの BigQuery サンプル データセット](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja)
- [JSON to Markdown Table](https://kdelmonte.github.io/json-to-markdown-table/)
