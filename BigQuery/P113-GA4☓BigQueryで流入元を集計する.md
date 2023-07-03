## :memo: Overview

ここでは、BigQuery で [ga4_obfuscated_sample_ecommerce](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja) データを使って、GA4 のエクスポートされたデータを BigQuery で集計する。基本的には下記の SQL を参考に、各指標の集計を実際に行い、その際に気づいたことをまとめている。

- [GA4 用の BigQuery クエリ集](https://www.ga4.guide/related-service/big-query/query-writing/)

GA4 および GA4 からエクスポートされるデータについては素人なので、まとめている内容に誤りがある可能性がある点は注意が必要。

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`traffic_source.name`、`traffic_source.medium`、`traffic_source.source`

## :pencil2: Example

今回は初回流入元やセッションの参照元を集計する際の SQL についてまとめておく。トラフィック関係は[こちら](https://support.google.com/firebase/answer/7029846?hl=ja#zippy=%2Ctraffic-source:~:text=app_info-,traffic_source,-traffic_source%20RECORD%20%E3%81%AB)のドキュメントにあるように、`traffic_source.*`カラムを使用すればよい模様。ただ、ドキュメントにも記載されているように、このカラムにはユーザーを最初に獲得した際の値が記録されるようなので、その点は注意。

- `traffic_source.name`: ユーザーを最初に獲得したマーケティング キャンペーンの名前。
- `traffic_source.medium`: ユーザーを最初に獲得したメディアの名前（有料検索、オーガニック検索、メールなど）。
- `traffic_source.source`: ユーザーを最初に獲得したネットワークの名前。

テーブルの中身を確認すると、下記のような値を持っていることがわかる。

```
select
    user_pseudo_id,
    datetime(timestamp_micros(event_timestamp),"Asia/Tokyo") as event_timestamp,
    event_name,
    (select value.int_value from unnest(event_params) where key = 'ga_session_id') as session_id,
    traffic_source.source,
    traffic_source.medium,
    traffic_source.name
from
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
    user_pseudo_id = "86253338.5702333040" and
    _table_suffix between '20201215' and '20201231'
order by
    event_timestamp asc
;
```

| user_pseudo_id      | event_timestamp            | event_name      | session_id | source                          | medium         | campaign       |
| ------------------- | -------------------------- | --------------- | ---------- | ------------------------------- | -------------- | -------------- |
| 86253338.5702333040 | 2020-12-15T21:26:07.814308 | session_start   | 8686662490 | google                          | organic        | (organic)      |
| 86253338.5702333040 | 2020-12-15T21:26:07.814308 | page_view       | 8686662490 | google                          | organic        | (organic)      |
| 86253338.5702333040 | 2020-12-15T21:26:13.029799 | view_promotion  | 8686662490 | google                          | organic        | (organic)      |
| 86253338.5702333040 | 2020-12-15T21:26:17.607393 | user_engagement | 8686662490 | google                          | organic        | (organic)      |
| (snip)              | (snip)                     | (snip)          | (snip)     | (snip)                          | (snip)         | (snip)         |
| 86253338.5702333040 | 2020-12-16T03:36:44.884080 | session_start   | 2955978236 | (data deleted)                  | (data deleted) | (data deleted) |
| 86253338.5702333040 | 2020-12-16T03:36:44.884080 | page_view       | 2955978236 | (data deleted)                  | (data deleted) | (data deleted) |
| (snip)              | (snip)                     | (snip)          | (snip)     | (snip)                          | (snip)         | (snip)         |
| 86253338.5702333040 | 2020-12-17T10:35:29.741176 | user_engagement | 5955001669 | <Other>                         | <Other>        | <Other>        |
| 86253338.5702333040 | 2020-12-17T10:35:35.777181 | page_view       | 5955001669 | <Other>                         | <Other>        | <Other>        |
| 86253338.5702333040 | 2020-12-18T03:37:05.322032 | session_start   | 1666647159 | shop.googlemerchandisestore.com | referral       | (referral)     |
| 86253338.5702333040 | 2020-12-18T03:37:05.322032 | page_view       | 1666647159 | shop.googlemerchandisestore.com | referral       | (referral)     |
| 86253338.5702333040 | 2020-12-19T09:08:00.035448 | session_start   | 2599448025 | google                          | organic        | (organic)      |
| 86253338.5702333040 | 2020-12-19T09:08:00.035448 | user_engagement | 2599448025 | google                          | organic        | (organic)      |
| (snip)              | (snip)                     | (snip)          | (snip)     | (snip)                          | (snip)         | (snip)         |
| 86253338.5702333040 | 2020-12-22T22:25:54.683111 | scroll          | 7059463959 | <Other>                         | <Other>        | <Other>        |
| 86253338.5702333040 | 2020-12-22T22:25:54.683111 | user_engagement | 7059463959 | <Other>                         | <Other>        | <Other>        |
| (snip)              | (snip)                     | (snip)          | (snip)     | (snip)                          | (snip)         | (snip)         |
| 86253338.5702333040 | 2020-12-24T00:53:22.405631 | session_start   | 5984060833 | (data deleted)                  | (data deleted) | (data deleted) |

`traffic_source`でグループ化すれば、初回流入元ごとのユーザー数を集計できる。ただ、初回流入元の集計をする場合、`event_name`を限定する必要はないのだろうか…。ちなみに`traffic_source.term`や`traffic_source.utm_content`というカラムは存在しない模様。

```
select
    traffic_source.source,
    traffic_source.medium,
    traffic_source.name as campaign,　
    count(distinct user_pseudo_id) as cnt_users
from
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
    _table_suffix between '20201225' and '20201231'
group by
    source,
    medium,
    campaign
order by
    cnt_users desc
;
```

| source                          | medium         | campaign       | cnt_users |
| ------------------------------- | -------------- | -------------- | --------- |
| google                          | organic        | (organic)      | 6514      |
| (direct)                        | (none)         | (direct)       | 4652      |
| <Other>                         | <Other>        | <Other>        | 3100      |
| <Other>                         | referral       | (referral)     | 1930      |
| shop.googlemerchandisestore.com | referral       | (referral)     | 1409      |
| google                          | cpc            | <Other>        | 955       |
| (data deleted)                  | (data deleted) | (data deleted) | 836       |
| <Other>                         | organic        | (organic)      | 628       |
| (data deleted)                  | (data deleted) | <Other>        | 15        |
| <Other>                         | (data deleted) | (data deleted) | 12        |
| google                          | <Other>        | <Other>        | 1         |

次は、セッションの参照元を集計する。`with`句内では、ネストを解除しながら必要なカラムを取り出して、グループ化しながら整形する、という SQL が参考にしている[こちらのサイト](https://www.ga4.guide/related-service/big-query/query-writing/)には記載されている。`user_pseudo_id, session_id`でグループ化して`max()`関数で値を取得している(テキストは`first_value()`で画像は`max()`関数)が、これでいいのかは私には GA4 の知識がないので、わからない。また、セッションの参照元の集計をする場合、`event_name`を限定する必要はないのだろうか…。`with`句内の SQL は下記のように機能する。

```
with tmp as (
select
    user_pseudo_id,
    (select value.int_value from unnest(event_params) where key = 'ga_session_id') as session_id,　
    max(concat(user_pseudo_id,"-", (select value.int_value from unnest(event_params) where key = 'ga_session_id'))) as strict_session_id,
    max((select value.string_value from unnest(event_params) where key = 'session_engaged')) as session_engaged,
    max((select value.string_value from unnest(event_params) where key = 'medium')) as medium,
    max((select value.string_value from unnest(event_params) where key = 'source')) as source
from
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
    event_name = 'page_view' and
    _table_suffix between '20201225' and '20201231'
group by
    user_pseudo_id,
    session_id)
select * from tmp;
```

| user_pseudo_id      | session_id | strict_session_id              | session_engaged | medium    | source   |
| ------------------- | ---------- | ------------------------------ | --------------- | --------- | -------- |
| 27609376.6610693817 | 6810687663 | 27609376.6610693817-6810687663 | 1               | affiliate | Partners |
| 6624764.2030956955  | 3108132284 | 6624764.2030956955-3108132284  | 1               | affiliate | Partners |
| 6987294.8319087872  | 9076201595 | 6987294.8319087872-9076201595  | 1               | affiliate | Partners |
| 6987294.8319087872  | 2564531688 | 6987294.8319087872-2564531688  | 0               | affiliate | Partners |
| 8027618.3772313264  | 2799298177 | 8027618.3772313264-2799298177  | 1               | affiliate | Partners |
| 8027618.3772313264  | 7832240390 | 8027618.3772313264-7832240390  | 0               | affiliate | Partners |
| 44740186.8456869937 | 8337590507 | 44740186.8456869937-8337590507 | 1               | affiliate | Partners |
| 42426159.3842258280 | 1906900762 | 42426159.3842258280-1906900762 | 0               | affiliate | Partners |
| (snip)              | (snip)     | (snip)                         | (snip)          | (snip)    | (snip)   |
| 43161807.3508995650 | 7122815453 | 43161807.3508995650-7122815453 | 1               | affiliate | Partners |
| 62544394.8871546960 | 5572611600 | 62544394.8871546960-5572611600 | 1               | affiliate | Partners |
| 88961085.2972630577 | 3112019243 | 88961085.2972630577-3112019243 | 1               | affiliate | Partners |
| 1005538.3649404164  | 4664322296 | 1005538.3649404164-4664322296  | 1               |           |          |
| 1012230.4994195474  | 9049733054 | 1012230.4994195474-9049733054  | 1               |           |          |
| 1028911.7693473709  | 8624814225 | 1028911.7693473709-8624814225  | 1               |           |          |
| 1037158.8475831089  | 5454336917 | 1037158.8475831089-5454336917  | 1               |           |          |
| (snip)              | (snip)     | (snip)                         | (snip)          | (snip)    | (snip)   |

`user_pseudo_id, session_id`でグループ化して`max()`関数で値を取得するということは、下記の通り、`user_pseudo_id, session_id`でグループ単位で、`session_engaged`、`medium`、`source`の最大値を取ることになるため、このような前処理で要件を満たしているのか不明。これは私の GA4 の知識がないだけで、正しい SQL なのかもしれない。誤りかどうかも判断できない。

```
with tmp as (
select
    user_pseudo_id,
    event_name,
    (select value.int_value from unnest(event_params) where key = 'ga_session_id') as session_id,　
    concat(user_pseudo_id,"-", (select value.int_value from unnest(event_params) where key = 'ga_session_id')) as strict_session_id,
    (select value.string_value from unnest(event_params) where key = 'session_engaged') as session_engaged,
    (select value.string_value from unnest(event_params) where key = 'medium') as medium,
    (select value.string_value from unnest(event_params) where key = 'source') as source
from
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
where
    user_pseudo_id in ("27609376.6610693817", '6624764.2030956955', '6987294.8319087872') and
    --event_name = 'page_view' and
    _table_suffix between '20201225' and '20201231'
)
select * from tmp;

```

| user_pseudo_id      | event_name        | session_id | strict_session_id              | session_engaged | medium    | source                          |
| ------------------- | ----------------- | ---------- | ------------------------------ | --------------- | --------- | ------------------------------- |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | scroll            | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | view_item         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | scroll            | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | scroll            | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | affiliate | Partners                        |
| 6624764.2030956955  | view_promotion    | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | view_item         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | scroll            | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | session_start     | 3108132284 | 6624764.2030956955-3108132284  |                 |           |                                 |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 0               | affiliate | Partners                        |
| 6624764.2030956955  | scroll            | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | add_shipping_info | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | view_promotion    | 3108132284 | 6624764.2030956955-3108132284  | 0               | affiliate | Partners                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | affiliate | Partners                        |
| 6624764.2030956955  | view_promotion    | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | add_shipping_info | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | user_engagement   | 3108132284 | 6624764.2030956955-3108132284  | 1               | affiliate | Partners                        |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | scroll            | 3108132284 | 6624764.2030956955-3108132284  | 0               | affiliate | Partners                        |
| 6624764.2030956955  | scroll            | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | page_view         | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | view_promotion    | 3108132284 | 6624764.2030956955-3108132284  | 1               |           |                                 |
| 6624764.2030956955  | scroll            | 3108132284 | 6624764.2030956955-3108132284  | 1               | (none)    | (direct)                        |
| 6987294.8319087872  | user_engagement   | 9076201595 | 6987294.8319087872-9076201595  | 1               | affiliate | Partners                        |
| 6987294.8319087872  | view_promotion    | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | scroll            | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | session_start     | 2564531688 | 6987294.8319087872-2564531688  |                 |           |                                 |
| 6987294.8319087872  | user_engagement   | 9076201595 | 6987294.8319087872-9076201595  | 1               | referral  | shop.googlemerchandisestore.com |
| 6987294.8319087872  | view_promotion    | 2564531688 | 6987294.8319087872-2564531688  | 0               | affiliate | Partners                        |
| 6987294.8319087872  | page_view         | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | page_view         | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | page_view         | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | page_view         | 2564531688 | 6987294.8319087872-2564531688  | 0               | affiliate | Partners                        |
| 6987294.8319087872  | scroll            | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | scroll            | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | user_engagement   | 9076201595 | 6987294.8319087872-9076201595  | 1               | referral  | shop.googlemerchandisestore.com |
| 6987294.8319087872  | page_view         | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | user_engagement   | 2564531688 | 6987294.8319087872-2564531688  | 1               | affiliate | Partners                        |
| 6987294.8319087872  | user_engagement   | 2564531688 | 6987294.8319087872-2564531688  | 1               | referral  | shop.googlemerchandisestore.com |
| 6987294.8319087872  | user_engagement   | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | page_view         | 9076201595 | 6987294.8319087872-9076201595  | 1               |           |                                 |
| 6987294.8319087872  | session_start     | 9076201595 | 6987294.8319087872-9076201595  |                 |           |                                 |
| 6987294.8319087872  | user_engagement   | 9076201595 | 6987294.8319087872-9076201595  | 1               | referral  | shop.googlemerchandisestore.com |
| 6987294.8319087872  | page_view         | 9076201595 | 6987294.8319087872-9076201595  | 0               | affiliate | Partners                        |
| 27609376.6610693817 | session_start     | 6810687663 | 27609376.6610693817-6810687663 |                 |           |                                 |
| 27609376.6610693817 | page_view         | 846952999  | 27609376.6610693817-846952999  | 0               | organic   | google                          |
| 27609376.6610693817 | user_engagement   | 846952999  | 27609376.6610693817-846952999  | 1               |           |                                 |
| 27609376.6610693817 | session_start     | 846952999  | 27609376.6610693817-846952999  |                 |           |                                 |
| 27609376.6610693817 | page_view         | 6810687663 | 27609376.6610693817-6810687663 | 0               | affiliate | Partners                        |
| 27609376.6610693817 | first_visit       | 6810687663 | 27609376.6610693817-6810687663 |                 |           |                                 |
| 27609376.6610693817 | page_view         | 846952999  | 27609376.6610693817-846952999  | 1               |           |                                 |
| 27609376.6610693817 | scroll            | 846952999  | 27609376.6610693817-846952999  | 1               |           |                                 |
| 27609376.6610693817 | page_view         | 6810687663 | 27609376.6610693817-6810687663 | 1               |           |                                 |
| 27609376.6610693817 | view_promotion    | 6810687663 | 27609376.6610693817-6810687663 | 1               |           |                                 |

流入元ごとのセッション数を取得する SQL を[参考にしているサイト](https://www.ga4.guide/related-service/big-query/query-writing/)の SQL を下記にメモしておく。`first_value()`関数をテキストの SQL では利用しているが、画像では`max()`関数になっている。`first_value()`を下記のようにそのまま使用するとエラーがでる。

![sql](https://github.com/SugiAki1989/sql_note/blob/main/image/p113-sql.png)

```
with predata as ( --select文に名称をつける。ここではpredata。「(」を忘れないように
select
    user_pseudo_id, --ユーザーのCookieIDを取得
    (select value.int_value from unnest(event_params) where key = 'ga_session_id') as session_id,　--ユーザーのセッションIDを取得
    first_value((select value.string_value from unnest(event_params) where key = 'session_engaged')) as session_engaged, --セッションエンゲージのパラメータ取得
    first_value((select value.string_value from unnest(event_params) where key = 'medium')) as medium, --イベントパラメータからメディアを取得。session_startのイベントには流入元は記録されていないため、該当セッションID内に含まれているメディアを取得してくる
    first_value((select value.string_value from unnest(event_params) where key = 'source')) as source --イベントパラメータから参照元を取得。session_startのイベントには流入元は記録されていないため、該当セッションID内に含まれているメディアを取得してくる
from
    `ha-ga4.analytics_227084301.events_*`   　-- データの選択範囲。ここでは全期間とし、whereの部分で日付を指定する
where
    _table_suffix between '20220201' and '20220205' 　-- 日付の指定
group by
    user_pseudo_id, --ユーザーのCookieIDでグルーピング
    session_id) --セッションIDでグルーピング。クエリの最後でpredataを閉じるため「)」を忘れないように

select
    concat(ifnull(source,'(direct)'),' / ',ifnull(medium,'(none)')) as session_source_medium, --セッションとメディアを「/」でつなげる。参照元に値が入っていない場合は参照元を(direct)に変換、メディアに値が入っていない場合はメディアを(none)に変換
    count(distinct concat(user_pseudo_id,"-",session_id)) as sessions,　--CookieIDとセッションIDをつなげユニーク数をカウントし、厳密なセッション数を取得
    count(distinct case when session_engaged = '1' then concat(user_pseudo_id,"-",session_id) else null end) as engaged_sessions --session_engagedの値が1の時に、CookieIDとセッションIDをつなげたユニーク数をカウントし、エンゲージメントセッション数を算出。1以外の場合はnullに変換

from
    predata --上記の2つのcountをpredataのクエリ結果から取得
group by
    session_source_medium　 --参照元メディアでグルーピング
order by
    sessions desc　 --セッション数、降順で並び替え
```

ちなみに`session_engaged`は下記のサイトに書かれている内容のカラムだと思われる。

- [【GA4 解説】「エンゲージのあったセッション」でアクセスの質をみる](https://webad.cgsc.info/blog/ga4_engagement_session/)

> 「エンゲージのあったセッション」における、「エンゲージ」とは何でしょうか。
> ここでいうエンゲージ（メント）は、以下の 3 つのどれかというふうに定義されています。
>
> - 10 秒以上滞在
> - 設定したコンバージョンが発生
> - 2 ページ以上閲覧
>
>   上記のどれかが達成されたセッションを「エンゲージのあったセッション」として、通常のセッションと区別しているわけです。
>   つまり「エンゲージのないセッション」は、コンバージョンもなく 10 秒未満で直帰したセッションということになります。

これとは関係ないが、おそらく… BigQuery のようなカラムナ型のデータベースは、理論的なイベント発火順序とデータベースの記録順序が一致しているとは限らないので、`first_value()`、`last_value()`を使う際は注意が必要という自戒をメモしておく。

## :closed_book: Reference

- [Google アナリティクス 4 e コマースウェブ実装向けの BigQuery サンプル データセット](https://developers.google.com/analytics/bigquery/web-ecommerce-demo-dataset?hl=ja)
- [JSON to Markdown Table](https://kdelmonte.github.io/json-to-markdown-table/)
