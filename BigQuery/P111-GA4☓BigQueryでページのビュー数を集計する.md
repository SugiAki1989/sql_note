## :memo: Overview

ここでは、BigQuery で [ga4_obfuscated_sample_ecommerce](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja) データを使って、GA4 のエクスポートされたデータを BigQuery で集計する。基本的には下記の SQL を参考に、各指標の集計を実際に行い、その際に気づいたことをまとめている。

- [GA4 用の BigQuery クエリ集](https://www.ga4.guide/related-service/big-query/query-writing/)

GA4 および GA4 からエクスポートされるデータについては素人なので、まとめている内容に誤りがある可能性がある点は注意が必要。

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`unnest`

## :pencil2: Example

今回はページビューのイベントを計測する。ただ、ページビューのイベントには不思議な点があるらしい。まずは計測されないページビューについて確認しておく。

下記は特定のユーザーのセッションごとに、ページビューイベントが発生しているかを確認するための SQL。たしかに、セッションが開始されているにも関わらず、ページビューイベントが記録されていないものがいくつかあることがわかる。

```
select
  user_pseudo_id,
  format_timestamp('%F %H:%M:%S', timestamp_micros(event_timestamp),"Asia/Tokyo") as event_datetime,
  (select value.int_value from unnest(event_params) where key = 'ga_session_id') as session_id,
  event_name,
  max(if(event_name = 'page_view',1,0)) over (partition by user_pseudo_id, (select value.int_value from unnest(event_params) where key="ga_session_id") rows between unbounded preceding and unbounded following) as has_page_view
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "7739461143.5712388145" and
  _table_suffix between '20201201' and '20201231'
order by
  event_datetime asc
;
```

| user_pseudo_id        | event_datetime      | session_id | event_name      | has_page_view |
| --------------------- | ------------------- | ---------- | --------------- | ------------- |
| 7739461143.5712388145 | 2020-12-09 08:28:14 | 605643330  | first_visit     | 1             |
| 7739461143.5712388145 | 2020-12-09 08:28:14 | 605643330  | session_start   | 1             |
| 7739461143.5712388145 | 2020-12-09 08:28:14 | 605643330  | page_view       | 1             |
| 7739461143.5712388145 | 2020-12-09 08:28:19 | 605643330  | view_promotion  | 1             |
| 7739461143.5712388145 | 2020-12-09 10:36:39 | 6272824984 | session_start   | 0             |
| 7739461143.5712388145 | 2020-12-09 10:36:39 | 6272824984 | user_engagement | 0             |
| 7739461143.5712388145 | 2020-12-09 13:41:09 | 7648610621 | session_start   | 1             |
| 7739461143.5712388145 | 2020-12-09 13:41:09 | 7648610621 | page_view       | 1             |
| (snip)                | (snip)              | (snip)     | (snip)          | (snip)        |
| 7739461143.5712388145 | 2020-12-16 06:41:11 | 5415361510 | page_view       | 1             |
| 7739461143.5712388145 | 2020-12-16 06:41:11 | 5415361510 | view_promotion  | 1             |
| 7739461143.5712388145 | 2020-12-16 08:13:53 | 8737016676 | user_engagement | 0             |
| 7739461143.5712388145 | 2020-12-16 08:13:53 | 8737016676 | session_start   | 0             |

ページビューの集計の前に、GA4 の半構造化データの扱いについて確認しておく。GA4 の半構造化データは RECORD 型のカラムにネストしてデータを保持している。そのため、下記のように`unnest()`関数を使用してネストをアンネストする必要がある。そうしなければ、下記のような構造化データのようにデータを利用できない。

```
select
  user_pseudo_id,
  date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  event_name,
  (select value.string_value from unnest(event_params) where key = 'page_title') as page_title,
  (select value.string_value from unnest(event_params) where key = 'page_location') as page_location
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "86253338.5702333040" and
  event_timestamp = 1608035472816783 and
  event_name = 'page_view' and
  _table_suffix between '20201201' and '20201231'
order by
  event_date asc
;

```

| user_pseudo_id      | event_date | event_name | event_timestamp  | page_title    | page_location                                       |
| ------------------- | ---------- | ---------- | ---------------- | ------------- | --------------------------------------------------- |
| 86253338.5702333040 | 2020-12-15 | page_view  | 1608035472816783 | Shopping Cart | https://shop.googlemerchandisestore.com/basket.html |

この例では、RECORD 型のカラムは`event_params`であり、`unnest()`関数を利用しなければ下記のようなネストしたまま取り出される。ただ、規則性があるので、理解すればそこまで面倒でもない。下記がわかり良い。

- [BigQuery 活用術: UNNEST 関数](https://developers-jp.googleblog.com/2017/04/bigquery-tip-unnest-function.html)

`event_params.key`に取り出したいカラム名が格納されており、`event_params.value.*_value`に値が記録されている。`*`にカラムに対応するデータ型が書かれている。

```
select
  user_pseudo_id,
  date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  event_name,
  event_params
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "86253338.5702333040" and
  event_timestamp = 1608035472816783 and
  event_name = 'page_view' and
  _table_suffix between '20201201' and '20201231'
order by
  event_date asc
;

```

![nesteddata](https://github.com/SugiAki1989/sql_note/blob/main/image/p111-nesteddata.png)

例えば、`event_params`カラムの中にある`event_params.key`の`page_title`をカラムとして、`value`の`string_value`を値として取り出したいのであれば、下記のように記述する。

```
(select value.string_value from unnest(event_params) where key = 'page_title') as page_title
```

他のカラムも取り出したいのであれば、同じように`select`文に追加すればよい。

```
select
  user_pseudo_id,
  date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  event_name,
  (select value.string_value from unnest(event_params) where key = 'page_title') as page_title,
  (select value.string_value from unnest(event_params) where key = 'page_referrer') as page_referrer,
  (select value.int_value from unnest(event_params) where key = 'ga_session_id') as ga_session_id
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  user_pseudo_id = "86253338.5702333040" and
  event_timestamp = 1608035472816783 and
  event_name = 'page_view' and
  _table_suffix between '20201201' and '20201231'
order by
  event_date asc
;

```

| user_pseudo_id      | event_date | event_name | page_title    | page_referrer                                        | ga_session_id |
| ------------------- | ---------- | ---------- | ------------- | ---------------------------------------------------- | ------------- |
| 86253338.5702333040 | 2020-12-15 | page_view  | Shopping Cart | https://shop.googlemerchandisestore.com/basket.html? | 8686662490    |

E コマースのデータであれば、他にも `items`という RECORD 型のカラムがあるが、こちらは先程と違って、キーごとに値を取り出したというよりは、1 列まるごと取り出したい。

![nesteddata2](https://github.com/SugiAki1989/sql_note/blob/main/image/p111-nesteddata2.png)

その場合、`from`句に`unnest(items) as unnested_items`という形で指定することでネストを解除できる。

```
select
  user_pseudo_id,
  date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  event_name,
  (select value.int_value from unnest(event_params) where key = 'ga_session_id') as ga_session_id,
  unnested_items.item_id,
  unnested_items.item_name,
  unnested_items.price,
  unnested_items.quantity,
  unnested_items.item_list_index
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`,
  unnest(items) as unnested_items
where
  user_pseudo_id = "86253338.5702333040" and
  event_timestamp = 1607989990133999 and
  event_name = 'add_to_cart' and
  _table_suffix between '20201101' and '20201231'
order by
  event_date asc
;
```

| user_pseudo_id      | event_date | event_name  | ga_session_id | item_id        | item_name                             | price | quantity | item_list_index |
| ------------------- | ---------- | ----------- | ------------- | -------------- | ------------------------------------- | ----- | -------- | --------------- |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKH152299 | Google Mountain View Campus Sticker   | 2.0   |          | 10              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEAFKA087599 | Android Large Removable Sticker Sheet | 3.0   |          | 12              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKB140599 | Google NYC Campus Sticker             | 2.0   |          | 3               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKR149499 | Google Kirkland Campus Sticker        | 2.0   |          | 8               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGDWC152799 | Google Sunnyvale Campus Mug           | 7.0   | 10       | 14              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKC146399 | Google LA Campus Sticker              | 2.0   |          | 5               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKH148999 | Google Seattle Campus Sticker         | 2.0   |          | 7               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKR133899 | Google Cambridge Campus Sticker       | 2.0   |          | 2               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKC152399 | Google Sunnyvale Campus Sticker       | 2.0   |          | 11              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKT149599 | Google Bellevue Campus Sticker        | 2.0   |          | 9               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKC144299 | Google Boulder Campus Sticker         | 2.0   |          | 4               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKH148399 | Google PNW Campus Sticker             | 2.0   |          | 6               |

複数のイベントがある場合、ネストが解除された状態で積み上げあれ、レコードに応じて、その他のカラムの値が紐づくことになる。`ga_session_id`がわかりやすい。

```
select
  user_pseudo_id,
  date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  event_name,
  (select value.int_value from unnest(event_params) where key = 'ga_session_id') as ga_session_id,
  unnested_items.item_id,
  unnested_items.item_name,
  unnested_items.price,
  unnested_items.quantity,
  unnested_items.item_list_index
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`,
  unnest(items) as unnested_items
where
  user_pseudo_id = "86253338.5702333040" and
  event_timestamp in (1607989990133999, 1608057500859194, 1608035225564603) and
  event_name = 'add_to_cart' and
  _table_suffix between '20201101' and '20201231'
order by
  event_date asc
;
```

| user_pseudo_id      | event_date | event_name  | ga_session_id | item_id        | item_name                             | price | quantity | item_list_index |
| ------------------- | ---------- | ----------- | ------------- | -------------- | ------------------------------------- | ----- | -------- | --------------- |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKH152299 | Google Mountain View Campus Sticker   | 2.0   |          | 10              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEAFKA087599 | Android Large Removable Sticker Sheet | 3.0   |          | 12              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKB140599 | Google NYC Campus Sticker             | 2.0   |          | 3               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKR149499 | Google Kirkland Campus Sticker        | 2.0   |          | 8               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGDWC152799 | Google Sunnyvale Campus Mug           | 7.0   | 10       | 14              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKC146399 | Google LA Campus Sticker              | 2.0   |          | 5               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKH148999 | Google Seattle Campus Sticker         | 2.0   |          | 7               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKR133899 | Google Cambridge Campus Sticker       | 2.0   |          | 2               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKC152399 | Google Sunnyvale Campus Sticker       | 2.0   |          | 11              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKT149599 | Google Bellevue Campus Sticker        | 2.0   |          | 9               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKC144299 | Google Boulder Campus Sticker         | 2.0   |          | 4               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    | GGOEGCKH148399 | Google PNW Campus Sticker             | 2.0   |          | 6               |
| (hr)                | (hr)       | (hr)        | (hr)          | (hr)           | (hr)                                  | (hr)  | (hr)     | (hr)            |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKC146399 | Google LA Campus Sticker              | 2.0   |          | 5               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKR149499 | Google Kirkland Campus Sticker        | 2.0   |          | 8               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKC152399 | Google Sunnyvale Campus Sticker       | 2.0   |          | 11              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEAFKA087599 | Android Large Removable Sticker Sheet | 3.0   |          | 12              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKR133899 | Google Cambridge Campus Sticker       | 2.0   |          | 2               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKH148399 | Google PNW Campus Sticker             | 2.0   |          | 6               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKT149599 | Google Bellevue Campus Sticker        | 2.0   |          | 9               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKC144299 | Google Boulder Campus Sticker         | 2.0   |          | 4               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKH152299 | Google Mountain View Campus Sticker   | 2.0   |          | 10              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKH148999 | Google Seattle Campus Sticker         | 2.0   |          | 7               |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEADHH129499 | Android Iconic Glass Bottle Green     | 17.0  |          | 22              |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    | GGOEGCKB140599 | Google NYC Campus Sticker             | 2.0   |          | 3               |
| (hr)                | (hr)       | (hr)        | (hr)          | (hr)           | (hr)                                  | (hr)  | (hr)     | (hr)            |
| 86253338.5702333040 | 2020-12-16 | add_to_cart | 2955978236    | 9181149        | Gift Card - $25.00                    | 31.0  | 1        | 2               |

ネストを解除しなければ、このレコードは 3 行しかない。

```
select
  user_pseudo_id,
  date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  event_name,
  (select value.int_value from unnest(event_params) where key = 'ga_session_id') as ga_session_id
  -- unnested_items.item_id,
  -- unnested_items.item_name,
  -- unnested_items.price,
  -- unnested_items.quantity,
  -- unnested_items.item_list_index
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  -- unnest(items) as unnested_items
where
  user_pseudo_id = "86253338.5702333040" and
  event_timestamp in (1607989990133999, 1608057500859194, 1608035225564603) and
  event_name = 'add_to_cart' and
  _table_suffix between '20201101' and '20201231'
order by
  event_date asc
;
```

| user_pseudo_id      | event_date | event_name  | ga_session_id |
| ------------------- | ---------- | ----------- | ------------- |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 8686662490    |
| 86253338.5702333040 | 2020-12-15 | add_to_cart | 5766097767    |
| 86253338.5702333040 | 2020-12-16 | add_to_cart | 2955978236    |

このような形で集計したい内容やカラムに応じてネストを解除する必要があるので、準備として構造化データに変換する手間が発生する。
そこさえ理解できれば、ページのビュー数を集計する SQL が書ける。細かい要件や GA4 のデータに対して正しい仕様をまだ理解できてないので、下記ももしかすると雑で粗い SQL かもしれない。

```
select
  -- date(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_date,
  (select value.string_value from unnest(event_params) where key = 'page_title') as page_title,
  count(event_name) as cnt_pageview,
  count(distinct user_pseudo_id) as cntd_user
from
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
  event_name = 'page_view' and
  _table_suffix between '20201230' and '20201231'
group by
  --event_date,
  page_title
order by
  --event_date asc,
  cnt_pageview desc
;
```

| page_title                                           | cnt_pageview | cntd_user |
| ---------------------------------------------------- | ------------ | --------- |
| Home                                                 | 3802         | 1739      |
| Apparel \| Google Merchandise Store                  | 2455         | 1255      |
| Google Online Store                                  | 2118         | 959       |
| YouTube \| Shop by Brand \| Google Merchandise Store | 1188         | 549       |
| Shopping Cart                                        | 756          | 229       |
| (snip)                                               | (snip)       | (snip)    |
| Google PNW Campus Unisex Tee                         | 1            | 1         |
| Google LA Campus Unisex Tee                          | 1            | 1         |
| Google Sunnyvale Campus Unisex Tee                   | 1            | 1         |
| Google Kirkland Campus Unisex Tee                    | 1            | 1         |

## :closed_book: Reference

- [Google アナリティクス 4 e コマースウェブ実装向けの BigQuery サンプル データセット](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja)
- [JSON to Markdown Table](https://kdelmonte.github.io/json-to-markdown-table/)
