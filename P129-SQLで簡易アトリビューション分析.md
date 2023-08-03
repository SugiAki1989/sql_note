## :memo: Overview

ここでは BigQuery で配列を使用する場合の基本的なクエリの使い方をまとめておく。BigQuery でいう配列とは、0個以上の同じデータ型の値で構成された順序付きリストのこと、とのこと。

前回、配列の基本的な部分を下記の公式ドキュメントに従っておさらいしたので、今回は、簡易的なアトリビューション分析を行う方法をまとめる。ここで行うアトリブーション分析は簡単なものなので、先頭と末尾を評価するだけで、より複雑なアトリビューション分析とは異なるので注意。アトリビューション分析というよりも、BigQuery で配列を扱うための練習。

- [配列の操作](https://cloud.google.com/bigquery/docs/arrays?hl=ja)

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`array`

## :pencil2: Example

今回使用するデータは都合よくパスの推移とコンバージョン、収益がまとまっているテーブルデータを利用する。また、結果の表示の都合上、`limit`をつけたまま行うことにする。

```sql
select 
  path,
  conversions,
  revenue
from 
  attribution_table
limit 
  5
;
```

|path|conversions|revenue|
|:----|:----|:----|
|google / organic > l.messenger.com / referral > google / organic|2.0|0.0|
|(not set) / (not set) > yahoo / organic|2.0|0.0|
|analytics.google.com / referral > developers.google.com / referral > (not set) / (not set) > analytics.google.com / referral|2.0|0.0|
|google / organic > analytics.google.com / referral > google / organic > analytics.google.com / referral > google / organic|2.0|0.0|
|google / organic > google / cpc|2.0|0.0|

まずは`split`関数を利用して、パスの推移を配列に分解して、押し込める。`array_length`関数を利用して、配列内の要素が1よりも大きいものを対象にする。1つしかないと、そもそもパスの遷移が存在してない。

```sql
select 
  split(path, '>') as path,
  conversions
from 
  attribution_table
where 
  array_length(split(path, '>')) > 1
limit 5
;
```
|path|conversions|
|:----|:----|
|[google / organic , l.messenger.com / referral , google / organic]|2.0|
|[(not set) / (not set) , yahoo / organic]|2.0|
|[analytics.google.com / referral , developers.google.com / referral , (not set) / (not set) , analytics.google.com / referral]|2.0|
|[google / organic , analytics.google.com / referral , google / organic , analytics.google.com / referral , google / organic]|2.0|
|[google / organic , google / cpc]|2.0|

配列に押し込めたものを`unnest`演算子で行として展開し、`index`をうまく使って、パスの識別子(0 ~ `nbr_in_path - 1`)を作る。

```sql
with base_table as (
select 
  split(path, '>') as path,
  conversions
from 
  attribution_table
where 
  array_length(split(path, '>')) > 1
limit 5
), tmp1 as (
select
  lower(trim(row_tmp)) as row,
  index,
  conversions,
  array_length(path) as nbr_in_path
from
  base_table, unnest(path) as row_tmp with offset as index
)
select * from tmp1
;
```

|row|index|conversions|nbr_in_path|
|:----|:----|:----|:----|
|google / organic|0|2.0|3|
|l.messenger.com / referral|1|2.0|3|
|google / organic|2|2.0|3|
|(not set) / (not set)|0|2.0|2|
|yahoo / organic|1|2.0|2|
|analytics.google.com / referral|0|2.0|4|
|developers.google.com / referral|1|2.0|4|
|(not set) / (not set)|2|2.0|4|
|analytics.google.com / referral|3|2.0|4|
|google / organic|0|2.0|5|
|analytics.google.com / referral|1|2.0|5|
|google / organic|2|2.0|5|
|analytics.google.com / referral|3|2.0|5|
|google / organic|4|2.0|5|
|google / organic|0|2.0|2|
|google / cpc|1|2.0|2|

あとは最初と最後を評価したいので、`touch_type`として識別出来るように処理して、

```sql
with base_table as (
select 
  split(path, '>') as path,
  conversions
from 
  attribution_table
where 
  array_length(split(path, '>')) > 1
limit 5
), tmp1 as (
select
  lower(trim(row_tmp)) as row,
  index,
  conversions,
  array_length(path) as nbr_in_path
from
  base_table, unnest(path) as row_tmp with offset as index
), tmp2 as (
select *,
  case 
    when index = 0 then 'first'
    when nbr_in_path - 1 = index then 'last'
    else 'mid' end as touch_type
from
  tmp1
) 
select * from tmp2
;
```

|row|index|conversions|nbr_in_path|touch_type|
|:----|:----|:----|:----|:----|
|google / organic|0|2.0|3|first|
|l.messenger.com / referral|1|2.0|3|mid|
|google / organic|2|2.0|3|last|
|(not set) / (not set)|0|2.0|2|first|
|yahoo / organic|1|2.0|2|last|
|analytics.google.com / referral|0|2.0|4|first|
|developers.google.com / referral|1|2.0|4|mid|
|(not set) / (not set)|2|2.0|4|mid|
|analytics.google.com / referral|3|2.0|4|last|
|google / organic|0|2.0|5|first|
|analytics.google.com / referral|1|2.0|5|mid|
|google / organic|2|2.0|5|mid|
|analytics.google.com / referral|3|2.0|5|mid|
|google / organic|4|2.0|5|last|
|google / organic|0|2.0|2|first|
|google / cpc|1|2.0|2|last|

グループ化して合計すれば終わり。

```sql
with base_table as (
select 
  split(path, '>') as path,
  conversions
from 
  attribution_table
where 
  array_length(split(path, '>')) > 1
limit 5
), tmp1 as (
select
  lower(trim(row_tmp)) as row,
  index,
  conversions,
  array_length(path) as nbr_in_path
from
  base_table, unnest(path) as row_tmp with offset as index
), tmp2 as (
select 
  *,
  case 
    when index = 0 then 'first'
    when nbr_in_path - 1 = index then 'last'
    else 'mid' end as touch_type
from
  tmp1
)
select
  row,
  touch_type,
  sum(conversions) as sum_conversions
from 
  tmp2 
where 
  touch_type != 'mid'
group by
  row,
  touch_type
order by
  touch_type asc,
  sum_conversions desc
;

```
|row|touch_type|sum_conversions|
|:----|:----|:----|
|google / organic|first|6.0|
|(not set) / (not set)|first|2.0|
|analytics.google.com / referral|first|2.0|
|google / organic|last|4.0|
|yahoo / organic|last|2.0|
|analytics.google.com / referral|last|2.0|
|google / cpc|last|2.0|

ちなみにCLIの`bq`コマンドからでもSQLは実行できるが、SQLを1行にするか、バックスラッシュ付きで改行しないとエラーになる。

```sql
bq query \
--use_legacy_sql=false \
" \
with base_table as ( \
select  \
  split(path, '>') as path, \
  conversions \
from  \
  attribution_table \
where  \
  array_length(split(path, '>')) > 1 \
limit 5 \
), tmp1 as ( \
select \
  lower(trim(row_tmp)) as row, \
  index, \
  conversions, \
  array_length(path) as nbr_in_path \
from \
  base_table, unnest(path) as row_tmp with offset as index \
), tmp2 as ( \
select  \
  *, \
  case  \
    when index = 0 then 'first' \
    when nbr_in_path - 1 = index then 'last' \
    else 'mid' end as touch_type \
from \
  tmp1 \
) \
select \
  row, \
  touch_type, \
  sum(conversions) as sum_conversions \
from  \
  tmp2  \
where  \
  touch_type != 'mid' \
group by \
  row, \
  touch_type \
order by \
  touch_type asc, \
  sum_conversions desc \
;  \
"
+---------------------------------+------------+-----------------+
|               row               | touch_type | sum_conversions |
+---------------------------------+------------+-----------------+
| google / organic                | first      |             6.0 |
| google / organic                | last       |             4.0 |
| (not set) / (not set)           | first      |             2.0 |
| yahoo / organic                 | last       |             2.0 |
| analytics.google.com / referral | first      |             2.0 |
| analytics.google.com / referral | last       |             2.0 |
| google / cpc                    | last       |             2.0 |
+---------------------------------+------------+-----------------+
```

なので、長いSQLは`.sql`ファイルとして保存し、リダイレクトの標準入力で渡す方法が便利。先程のSQLを`attribution_query.sql`として保存して実行すると、同じ結果が得られる。

```sql
$ bq query --use_legacy_sql=false < ~/Desktop/attribution_query.sql
+---------------------------------+------------+-----------------+
|               row               | touch_type | sum_conversions |
+---------------------------------+------------+-----------------+
| google / organic                | first      |             6.0 |
| google / organic                | last       |             4.0 |
| (not set) / (not set)           | first      |             2.0 |
| yahoo / organic                 | last       |             2.0 |
| analytics.google.com / referral | first      |             2.0 |
| analytics.google.com / referral | last       |             2.0 |
| google / cpc                    | last       |             2.0 |
+---------------------------------+------------+-----------------+

```

## :closed_book: Reference

- [配列の操作](https://cloud.google.com/bigquery/docs/arrays?hl=ja)
