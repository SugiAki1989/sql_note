## :memo: Overview

Databricksをはじめてさわる機会があったので、自己学習用にAzureでDatabricks環境を構築して、Databricks SQLで遊んでみた。Databricks SQLに関しては、データブリックス・ジャパンの方が書かれた下記の記事がわかりやすい。

- [Databricks SQLとは何か？](https://qiita.com/taka_yayoi/items/5b6e4537b086775a408a)

DDL、DCLは見ておらず、DMLの関数系を中心に遊んでいる(=私の独断と偏見でまとめている)。Databricks SQLでは、標準の ANSI SQL がサポートされているとのこと。サンプルデータはDatabricksにデフォルで用意されている`tpch`のスキーマのデータまたは一時的に作成したテーブルを使用している。TPC-Hベンチマークのデータが含まれているとのこと。リレーションはこんな感じ([TPC ベンチマークH標準仕様](chrome-extension://efaidnbmnnnibpcajpcglclefindmkaj/https://www.tpc.org/tpc_documents_current_versions/pdf/tpc-h_v2.17.1.pdf))。

![formula](https://github.com/SugiAki1989/sql_note/blob/main/image/p144-relation.png)

```sql
use catalog samples;
use schema tpch;
```

## :floppy_disk: Database

Databricks

## :bookmark: Tag

`qualify`,`offset`,`pivot`,`unpivot`,`tablesample`,`range`,`any`,`approx_*`,`bool_and`,`bool_or`,`collect_set`,`collect_list`,`count_if`,`date_format`,`cube`,`grouping`,`if`,`ifnull`,`last_day`,`next_day`,`months_between`,`printf`,`try_*`,`lambda`

## :pencil2: `qualify`句 
 

BigQueryやSnowflakeなどで使用できた`qualify`句がサポートされているので、ウインドウ関数の結果をフィルターできる。サブクエリや`with`句をかませる必要がないので、これは便利。

```sql
SELECT [ hints ] [ ALL | DISTINCT ] { named_expression | star_clause } [, ...]
  FROM table_reference [, ...]
  [ LATERAL VIEW clause ]
  [ WHERE clause ]
  [ GROUP BY clause ]
  [ HAVING clause]
  [ QUALIFY clause ]

select 
  o_custkey
  , o_orderdate
  , row_number() over(partition by o_custkey order by o_orderdate desc) as ind
from
  orders
qualify
  ind in (1,2,3)
;
```

|o_custkey|o_orderdate|ind|
|:---:|:---:|:---:|
|28|1998-02-18|1|
|28|1997-05-31|2|
|28|1997-04-21|3|
|29|1998-03-21|1|
|29|1997-01-18|2|
|29|1995-07-30|3|
|151|1998-06-06|1|
|151|1998-05-30|2|
|151|1996-12-10|3|
|200|1997-10-06|1|
|200|1996-06-27|2|
|200|1995-10-30|3|
|(snip)|(snip)|(snip)|


## :pencil2: `offset`句 

地味に便利な`offset`句をサポートされている。下記は、5️番目までをオフセットして、6,7,8番目のレコードを取得するクエリ。`offset`してからの`limit`なので気をつける。

```sql
select * from customer order by c_custkey asc limit 3 offset 5;
```

|c_custkey|c_name|
|:---:|:---:|
|6|Customer#000000006|
|7|Customer#000000007|
|8|Customer#000000008|


## :pencil2: `pivot`句 / `unpivot`句

`pivot`句もサポートされているので、テーブル展開系の操作が簡単にできる。下記の注文履歴を横展開したい。つまり、顧客ごとに1行に注文日を横に並べていきたい。

```sql
select 
  o_custkey
  , o_orderkey
  , o_orderdate
  , row_number() over(partition by o_custkey order by o_orderdate asc) as ind
from
  orders
where
  o_custkey in (126260, 171734, 476747, 568715)
;

```

|o_custkey|o_orderkey|o_orderdate|ind|
|:---:|:---:|:---:|:---:|
|126260|10373220|1993-12-07|1|
|126260|11403141|1995-11-09|2|
|126260|12798916|1996-09-27|3|
|126260|29836612|1998-04-13|4|
|171734|20913543|1992-04-02|1|
|171734|959556|1992-08-07|2|
|171734|11396992|1993-12-07|3|
|171734|8489636|1994-12-19|4|
|171734|3428768|1997-03-07|5|
|476747|12827362|1992-12-05|1|
|476747|11435623|1993-04-25|2|
|476747|8449318|1993-05-30|3|
|476747|13536390|1993-12-25|4|
|476747|13040962|1995-05-20|5|
|476747|9564902|1997-04-18|6|
|568715|24625216|1993-01-19|1|
|568715|7712096|1995-07-04|2|
|568715|20926087|1995-07-28|3|
|568715|11398695|1995-11-29|4|
|568715|4863749|1996-05-14|5|
|568715|15066403|1997-05-05|6|
|568715|3200293|1998-03-13|7|


`pivot`句を使わなくても、頑張ればできる。

```sql
with tmp as (
  select 
    o_custkey
    , o_orderkey
    , o_orderdate
    , row_number() over(partition by o_custkey order by o_orderdate asc) as ind
  from
    orders
  where
    o_custkey in (126260, 171734, 476747, 568715)
)
select o_custkey,
  max(case when ind = 1 then o_orderdate end) as o1,
  max(case when ind = 2 then o_orderdate end) as o2,
  max(case when ind = 3 then o_orderdate end) as o3,
  max(case when ind = 4 then o_orderdate end) as o4,
  max(case when ind = 5 then o_orderdate end) as o5,
  max(case when ind = 6 then o_orderdate end) as o6,
  max(case when ind = 7 then o_orderdate end) as o7
from tmp
group by o_custkey;
```

|o_custkey|o1|o2|o3|o4|o5|o6|o7|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|126260|1993-12-07|1995-11-09|1996-09-27|1998-04-13| | | |
|171734|1992-04-02|1992-08-07|1993-12-07|1994-12-19|1997-03-07| | |
|476747|1992-12-05|1993-04-25|1993-05-30|1993-12-25|1995-05-20|1997-04-18| |
|568715|1993-01-19|1995-07-04|1995-07-28|1995-11-29|1996-05-14|1997-05-05|1998-03-13|

もう1つのの方法もある。

```sql
with tmp as (
  select 
    o_custkey
    , o_orderkey
    , o_orderdate
    , row_number() over(partition by o_custkey order by o_orderdate asc) as ind
  from
    orders
  where
    o_custkey in (126260, 171734, 476747, 568715)
)
select o_custkey,
  max(o_orderdate) filter(where ind = 1) as o1,
  max(o_orderdate) filter(where ind = 2) as o2,
  max(o_orderdate) filter(where ind = 3) as o3,
  max(o_orderdate) filter(where ind = 4) as o4,
  max(o_orderdate) filter(where ind = 5) as o5,
  max(o_orderdate) filter(where ind = 6) as o6,
  max(o_orderdate) filter(where ind = 7) as o7
from tmp
group by o_custkey;
```

|o_custkey|o1|o2|o3|o4|o5|o6|o7|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|126260|1993-12-07|1995-11-09|1996-09-27|1998-04-13| | | |
|171734|1992-04-02|1992-08-07|1993-12-07|1994-12-19|1997-03-07| | |
|476747|1992-12-05|1993-04-25|1993-05-30|1993-12-25|1995-05-20|1997-04-18| |
|568715|1993-01-19|1995-07-04|1995-07-28|1995-11-29|1996-05-14|1997-05-05|1998-03-13|

`pivot`句を使うと、このような感じ。`pivot(agg_func as alias for col in (cond as alias))`みたいな感じ。`pivot`句をつかっても、データに合わせてカラム数を自動で調整してくれるわけではないので、結局のところ同じかも。

```sql
with tmp as (
select 
  o_custkey
  , o_orderdate
  , row_number() over(partition by o_custkey order by o_orderdate asc) as ind
from
  orders
where
  o_custkey in (126260, 171734, 476747, 568715)
)
select o_custkey, o1, o2, o3, o4, o5, o6, o7
from tmp
pivot (min(o_orderdate) as order_dt
for ind
in (1 as o1, 2 as o2, 3 as o3, 4 as o4, 5 as o5, 6 as o6, 7 as o7))
;
```

|o_custkey|o1|o2|o3|o4|o5|o6|o7|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|126260|1993-12-07|1995-11-09|1996-09-27|1998-04-13| | | |
|171734|1992-04-02|1992-08-07|1993-12-07|1994-12-19|1997-03-07| | |
|476747|1992-12-05|1993-04-25|1993-05-30|1993-12-25|1995-05-20|1997-04-18| |
|568715|1993-01-19|1995-07-04|1995-07-28|1995-11-29|1996-05-14|1997-05-05|1998-03-13|

`pivot`句は変換前のテーブル構造に依存するので、`o_orderkey`があると階段状に展開される。SQLアートとかでは便利かも。

```sql
with tmp as (
select 
  o_custkey
  , o_orderkey
  , o_orderdate
  , row_number() over(partition by o_custkey order by o_orderdate asc) as ind
from
  orders
where
  o_custkey in (126260, 171734, 476747, 568715)
)
select o_custkey, o1, o2, o3, o4, o5, o6, o7
from tmp
pivot (min(o_orderdate) as order_dt
for ind
in (1 as o1, 2 as o2, 3 as o3, 4 as o4, 5 as o5, 6 as o6, 7 as o7))
;
```

|o_custkey|o1|o2|o3|o4|o5|o6|o7|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|126260|1993-12-07| | | | | | |
|126260| |1995-11-09| | | | | |
|126260| | |1996-09-27| | | | |
|126260| | | |1998-04-13| | | |
|171734|1992-04-02| | | | | | |
|171734| |1992-08-07| | | | | |
|171734| | |1993-12-07| | | | |
|171734| | | |1994-12-19| | | |
|171734| | | | |1997-03-07| | |
|476747|1992-12-05| | | | | | |
|476747| |1993-04-25| | | | | |
|476747| | |1993-05-30| | | | |
|476747| | | |1993-12-25| | | |
|476747| | | | |1995-05-20| | |
|476747| | | | | |1997-04-18| |
|568715|1993-01-19| | | | | | |
|568715| |1995-07-04| | | | | |
|568715| | |1995-07-28| | | | |
|568715| | | |1995-11-29| | | |
|568715| | | | |1996-05-14| | |
|568715| | | | | |1997-05-05| |
|568715| | | | | | |1998-03-13|

ここでは練習のためにピボットしたものを運ピボットしておく。`INCLUDE nulls`を含めると、アンピボットしたときに`null`行を含められる。不要であれば除外すればOK。

```sql
with tmp as (
select 
  o_custkey
  , o_orderdate
  , row_number() over(partition by o_custkey order by o_orderdate asc) as ind
from
  orders
where
  o_custkey in (126260, 171734, 476747, 568715)
), pivot_tbl as (
select o_custkey, o1, o2, o3, o4, o5, o6, o7
from tmp
pivot (min(o_orderdate) as order_dt
for ind
in (1 as o1, 2 as o2, 3 as o3, 4 as o4, 5 as o5, 6 as o6, 7 as o7))
)
select o_custkey, o_orderdate, ind
from pivot_tbl
unpivot INCLUDE nulls
(o_orderdate 
for ind 
in (
  o1 as `1`,
  o2 as `2`,
  o3 as `3`,
  o4 as `4`,
  o5 as `5`,
  o6 as `6`,
  o7 as `7`
))
order by o_custkey, ind asc;
```

|o_custkey|o_orderdate|ind|
|:---:|:---:|:---:|
|126260|1993-12-07|1|
|126260|1995-11-09|2|
|126260|1996-09-27|3|
|126260|1998-04-13|4|
|126260| |5|
|126260| |6|
|126260| |7|
|171734|1992-04-02|1|
|171734|1992-08-07|2|
|171734|1993-12-07|3|
|171734|1994-12-19|4|
|171734|1997-03-07|5|
|171734| |6|
|171734| |7|
|476747|1992-12-05|1|
|476747|1993-04-25|2|
|476747|1993-05-30|3|
|476747|1993-12-25|4|
|476747|1995-05-20|5|
|476747|1997-04-18|6|
|476747| |7|
|568715|1993-01-19|1|
|568715|1995-07-04|2|
|568715|1995-07-28|3|
|568715|1995-11-29|4|
|568715|1996-05-14|5|
|568715|1997-05-05|6|
|568715|1998-03-13|7|


## :pencil2: `tablesample`句 

ざっくりテーブルを確認したいときに便利な`tablesample`句もサポートされている。

- `percentage PERCENT`: サンプリング行割合を指定。0から100の間の実数。
- `num_rows ROWS`: サンプリング行の絶対数を指定。
- `BUCKET fraction OUT OF total`: バケットを指定して、その中からサンプリングするもの？
- `REPEATABLE ( seed )`: 乱数種

``テーブルは750000行あるので、0.01%くらいが返される。

```sql
select c_custkey, c_name, c_phone from customer tablesample (0.0001 percent) repeatable(1234);
```

|c_custkey|c_name|c_phone|
|:---:|:---:|:---:|
|417855|Customer#000417855|34-634-539-2383|
|206606|Customer#000206606|30-893-401-2489|
|213781|Customer#000213781|26-164-767-9959|
(snip)
|609866|Customer#000609866|31-188-705-5902|
|611840|Customer#000611840|17-932-979-5503|


こっちは5レコードをサンプリングする。

```sql
select c_custkey, c_name, c_phone from customer tablesample (5 rows) repeatable(1234);
```

|c_custkey|c_name|c_phone|
|:---:|:---:|:---:|
|412445|Customer#000412445|31-421-403-4333|
|412446|Customer#000412446|30-487-949-7942|
|412447|Customer#000412447|17-797-466-6308|
|412448|Customer#000412448|16-541-510-4964|
|412449|Customer#000412449|24-710-983-5536|

カラムの値をランダムに返す`any_value()`関数もあるとのこと。

```sql
with tmp as (
    select * from (values 
      (1, 10), (1, 5), (1, 20),
      (2, 1) , (2, 4), (2, 3)
      ) AS tab(id, col)
)
select any_value(col) as any_val from tmp;
```

|any_val|
|:---:|
|10|


## :pencil2: `range`関数

テーブル値関数の`range`関数が使えるので数列や数列の値を用いたタリーテーブルが簡単に作れる。今回のDatabricks環境はUTCなので、日本時間に変換している。

```sql
select 
  id as num 
  , convert_timezone('Asia/Tokyo', current_date())::date + id::int as dat 
  , convert_timezone('Asia/Tokyo', current_date()) as conv_c_date
  , convert_timezone('Asia/Tokyo', current_timestamp()) as conv_c_time
  , current_timestamp() -- UTC
  -- DATE'2024-04-06';
  -- TIMESTAMP'2024-04-06 12:20:31';
from range(0, 10);
```

|num|dat|conv_c_date|conv_c_time|current_date()|current_timestamp()|
|:---:|:---:|:---:|:---:|:---:|:---:|
|0|2024-04-06|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|1|2024-04-07|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|2|2024-04-08|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|3|2024-04-09|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|4|2024-04-10|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|5|2024-04-11|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|6|2024-04-12|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|7|2024-04-13|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|8|2024-04-14|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|
|9|2024-04-15|2024-04-06 09:00:00|2024-04-06 11:59:22.377|2024-04-06|2024-04-06 02:59:22.377000|

## :pencil2: `any()`関数

postgreSQLだと下記のように配列で扱う必要があった`any()`関数が、Databricksでは普通に使用できる。機能は同じで、グループ内に少なくとも1つの値が`true`の場合に`true`を返す。

- `ANY(array expression)`
- `ALL(array expression)`

```sql
with tmp as (
    select * from (values 
      (1, 10, TRUE), (1, 5, TRUE), (1, 20, FALSE),
      (2, 1, FALSE) , (2, 4, FALSE), (2, 3, FALSE)
      ) AS tab(id, col, flg)
)
select id, any(flg) from tmp group by id;
```

|id|any(flg)|
|:---:|:---:|
|1|true|
|2|false|


## :pencil2: `approx_*()`関数

BigQueryにあったような`approx_*()`関数も利用可能。5%程度以内の誤差で計算できるとのこと。`approx_percentile()`関数、`approx_top_k()`関数などもサポートされている。

```sql
select 
  approx_count_distinct(c_custkey), 
  count(distinct c_custkey)   
from 
  customer
;
```

|approx_count_distinct(c_custkey)|count(DISTINCT c_custkey)|
|:---:|:---:|
|725800|750000|

## :pencil2: `bool_and()`関数　 / `bool_or()`関数

`bool_and()`関数(=`every()`関数)は、すべての値がグループ内で`true`の場合は`true`を返し、`bool_or()`関数(=`some()`関数)は、少なくとも1つの値が`true`の場合は`true`を返す。

```sql
with tmp as (
    select * from (values 
      (0, 1, TRUE), (0, 3, TRUE), (0, 10, TRUE),
      (1, 10, TRUE), (1, 5, TRUE), (1, 20, FALSE),
      (2, 1, FALSE) , (2, 4, FALSE), (2, 3, FALSE),
      (3, 10, null), (3, 4, FALSE), (3, 3, TRUE)
      ) AS tab(id, col, flg)
)
select 
  id 
  , bool_and(flg) 
  , bool_or(flg) 
from 
  tmp 
group by 
  id
;
```

|id|bool_and(flg)|bool_or(flg)|
|:---:|:---:|:---:|
|0|true|true|
|1|false|true|
|2|false|false|
|3|false|true|



## :pencil2: `collect_set()`関数 / `collect_list()`関数


`collect_set()`関数は関数名の通り集合を作成する。集合には重複という考え方はないので、ユニークに配列でまとめられている。一方、`collect_list()`関数は配列として値をまとめてくれる。いずれの関数も`null`はまとめられないのと、配列内の要素の順序は非決定論的なので、順序に保証はないっぽい。

```sql
with tmp as (
    select * from (values 
      (0,  1,  TRUE, 'a'), (0, 3, TRUE , 'a'), (0, 10,  TRUE, 'a'),
      (1, 10,  TRUE, 'a'), (1, 5, TRUE , 'b'), (1, 20, FALSE, 'b'),
      (2,  1, FALSE, 'a'), (2, 4, FALSE, 'b'), (2,  3, FALSE, 'c'),
      (3, 10,  null, 'd'), (3, 4, FALSE, 'd'), (3,  3,  TRUE, 'd')
      ) AS tab(id, col, flg, str)
)
select 
  id 
  , collect_set(flg)
  , collect_set(str)
  , collect_list(flg)
  , collect_list(str)
from 
  tmp 
group by 
  id
;
```

|id|collect_set(flg)|collect_set(str)|collect_list(flg)|collect_list(str)|
|:---:|:---:|:---:|:---:|:---:|
|1|[true,false]|["a","b"]|[true,true,false]|["a","b","b"]|
|3|[false,true]|["d"]|[false,true]|["d","d","d"]|
|2|[false]|["a","c","b"]|[false,false,false]|["a","b","c"]|
|0|[true]|["a"]|[true,true,true]|["a","a","a"]|


`collect_list()`関数


## :pencil2: `count_if()`関数

地味に便利な`count_if()`関数もサポートされている。

```sql

with tmp as (
    select * from (values 
      (0,  1,  TRUE, 'a'), (0, 3, TRUE , 'a'), (0, 10,  TRUE, 'a'),
      (1, 10,  TRUE, 'a'), (1, 5, TRUE , 'b'), (1, 20, FALSE, 'b'),
      (2,  1, FALSE, 'a'), (2, 4, FALSE, 'b'), (2,  3, FALSE, 'c'),
      (3, 10,  null, 'd'), (3, 4, FALSE, 'd'), (3,  3,  TRUE, 'd')
      ) AS tab(id, col, flg, str)
)
select 
  id 
  , count_if(flg is null)
  , count_if(str = 'a')
from 
  tmp 
group by 
  id
;
```

|id|count_if((flg IS NULL))|count_if((str = a))|
|:---:|:---:|:---:|
|0|0|3|
|1|0|1|
|2|0|1|
|3|1|0|


## :pencil2: `()`関数

PostgreSQL、MySQLだとフォーマット関数は下記の通り。

PostgreSQL: `to_char(time,'YYYY')`
MySQL: `date_format(time,"%Y")`	

DatabricksだとMySQLと似ているが、フォーマットの指定方法が異なる。

```sql

select 
  current_timestamp()
  , date_format(current_timestamp(), 'y') as year
  , date_format(current_timestamp(), 'MM') as month
  , date_format(current_timestamp(), 'd') as day
  , date_format(current_timestamp(), 'h') as hour
  , date_format(current_timestamp(), 'm') as min
  , date_format(current_timestamp(), 's') as sec
  , date_format(current_timestamp(), 'yyyy-MM-01') as yyyymmdd  
;
```

|current_timestamp()|year|month|day|hour|min|sec|yyyymmdd|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|2024-04-06 07:48:53.643000|2024|04|6|7|48|53|2024-04-01|


## :pencil2: `cube()`関数 / `grouping()`関数

`grouping()`関数は、`GROUPING SET`、`ROLLUP`、`CUBE`の指定した列が小計を表すかどうかを示す。`cube()`関数は多次元キューブ=集計表
を作成する関数。

```sql
with tmp as (
    select * from (values 
      (0,  1,  TRUE, 'a'), (0, 3, TRUE , 'a'), (0, 10,  TRUE, 'a'),
      (1, 10,  TRUE, 'a'), (1, 5, TRUE , 'b'), (1, 20, FALSE, 'b'),
      (2,  1, FALSE, 'a'), (2, 4, FALSE, 'b'), (2,  3, FALSE, 'c'),
      (3, 10,  null, 'd'), (3, 4, FALSE, 'd'), (3,  3,  TRUE, 'd')
      ) AS tab(id, col, flg, str)
)
select 
  id 
  , grouping(id)
  , count(1)
  , sum(col)
  , count_if(str = 'a')
from 
  tmp 
group by 
  cube(id)
order by 
  id asc
;
```

|id|grouping(id)|count(1)|sum(col)|count_if((str = a))|
|:---:|:---:|:---:|:---:|:---:|
|null|1|12|74|5|
|0|0|3|14|3|
|1|0|3|35|1|
|2|0|3|8|1|
|3|0|3|17|0|



## :pencil2: `if()`関数 / `ifnull()`関数

地味にありがたい`if()`関数、`ifnull()`関数もサポートされている。これでわざわざ`case`文を書かなくても良くなる。`iff()`関数もサポートされているが、これは`if()`関数と同じ。

`ifnull()`関数は`expr1`が`null`の場合は`expr2`を返す。それ以外の場合は`expr1`を返す。`coalesce(expr1, expr2)`と同じ。

```sql
with tmp as (
    select * from (values 
      (0,  1,  TRUE, 'a'), (0, 3, TRUE , 'a'), (0, 10,  TRUE, 'a'),
      (1, 10,  TRUE, 'a'), (1, 5, TRUE , 'b'), (1, 20, FALSE, 'b'),
      (2,  1, FALSE, 'a'), (2, 4, FALSE, 'b'), (2,  3, FALSE, 'c'),
      (3, 10,  null, 'd'), (3, 4, FALSE, 'd'), (3,  3,  TRUE, 'd')
      ) AS tab(id, col, flg, str)
)
select 
  id
  , col
  , if(col > 5, '6-', '5-') as if_func
  , flg
  , ifnull(flg, TRUE) as ifnull_func
from 
  tmp 
order by 
  id asc
;

```
|id|col|if_func|flg|ifnull_func|
|:---:|:---:|:---:|:---:|:---:|
|0|1|5-|true|true|
|0|3|5-|true|true|
|0|10|6-|true|true|
|1|10|6-|true|true|
|1|5|5-|true|true|
|1|20|6-|false|false|
|2|1|5-|false|false|
|2|4|5-|false|false|
|2|3|5-|false|false|
|3|10|6-| |true|
|3|4|5-|false|false|
|3|3|5-|true|true|


## :pencil2: `last_day()`関数　 / `next_day()`関数

これも地味に嬉しい`last_day()`関数と`next_day()`関数。`last_day()`関数月に最後の日を取得できる。これで、次の月の1日を計算して、そこから1日戻す作業が必要なくなる。`next_day()`関数は、指定した日付の指定曜日の日を返してくれる。

```sql
select current_date(), last_day(current_date());
```
|current_date()|last_day(current_date())|
|:---:|:---:|
|2024-04-06|2024-04-30|


- 'SU', 'SUN', 'SUNDAY'
- 'MO', 'MON', 'MONDAY'
- 'TU', 'TUE', 'TUESDAY'
- 'WE', 'WED', 'WEDNESDAY'
- 'TH', 'THU', 'THURSDAY'
- 'FR', 'FRI', 'FRIDAY'
- 'SA', 'SAT', 'SATURDAY'


```sql
select
  next_day('2024-04-06', 'SUNDAY') as next_SUNDAY
  , next_day('2024-04-06', 'MONDAY') as next_MONDAY
  , next_day('2024-04-06', 'TUESDAY') as next_TUESDAY
  , next_day('2024-04-06', 'WEDNESDAY') as next_WEDNESDAY
  , next_day('2024-04-06', 'THURSDAY') as next_THURSDAY
  , next_day('2024-04-06', 'FRIDAY') as next_FRIDAY
  , next_day('2024-04-06', 'SATURDAY') as next_SATURDAY
;
```

|next_SUNDAY|next_MONDAY|next_TUESDAY|next_WEDNESDAY|next_THURSDAY|next_FRIDAY|next_SATURDAY|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|2024-04-07|2024-04-08|2024-04-09|2024-04-10|2024-04-11|2024-04-12|2024-04-13|


## :pencil2: `months_between()`関数

PostgreSQLの`age()`関数に似たような関数として、日付またはタイムスタンプの経過した月数を返す関数がある。

```sql
select
    months_between('2024-04-07'::date, '2024-04-06'::date) as months_between0
  , months_between('2024-05-05'::date, '2024-04-06'::date) as months_between1
  , months_between('2024-05-06'::date, '2024-04-06'::date) as months_between2
  , months_between('2024-05-07'::date, '2024-04-06'::date) as months_between3
  , months_between('2025-04-05'::date, '2024-04-06'::date) as months_between4
  , months_between('2025-04-06'::date, '2024-04-06'::date) as months_between5
  , months_between('2025-04-07'::date, '2024-04-06'::date) as months_between6
;
```

|months_between0|months_between1|months_between2|months_between3|months_between4|months_between5|months_between6|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|0.03225806|0.96774194|1.0|1.03225806|11.96774194|12.0|12.03225806|

PostgreSQLの`age()`関数はこちら。

```sql
select
    age('2024-04-07'::date, '2024-04-06'::date) as age0
  , age('2024-05-05'::date, '2024-04-06'::date) as age1
  , age('2024-05-06'::date, '2024-04-06'::date) as age2
  , age('2024-05-07'::date, '2024-04-06'::date) as age3
  , age('2025-04-05'::date, '2024-04-06'::date) as age4
  , age('2025-04-06'::date, '2024-04-06'::date) as age5
  , age('2025-04-07'::date, '2024-04-06'::date) as age6
;
 age0  |  age1   | age2  |    age3     |      age4       |  age5  |     age6
-------+---------+-------+-------------+-----------------+--------+--------------
 1 day | 29 days | 1 mon | 1 mon 1 day | 11 mons 29 days | 1 year | 1 year 1 day
(1 row)
```

## :pencil2: `printf()`関数

分析ではあまり使わないかもしれないが、`printf()`関数が使用できる模様。

```sql
with tmp as (
    select * from (values 
      (1, 'Brad', 'Pitt'), (2, 'Daisy', 'Dove'), (3, 'Michael', 'William')
      ) AS tab(id, nm1, nm2)
)
select printf('Hello! %s %s', nm1, nm2) from tmp;
```

|printf(Hello! %s %s| nm1| nm2)|
|:---:|:---:|:---:|
|Hello! Brad Pitt|
|Hello! Daisy Dove|
|Hello! Michael William|


## :pencil2: `try_*()`関数

`try_*()`関数もサポートされている。例えば下記の関数が使用できる。

- `try_add()`関数
- `try_aes_decrypt()`関数
- `try_avg()`関数
- `try_cast()`関数
- `try_divide()`関数
- `try_element_at()`関数
- `try_multiply()`関数
- `try_reflect()`関数
- `try_subtract()`関数
- `try_sum()`関数
- `try_to_binary()`関数
- `try_to_number()`関数
- `try_to_timestamp()`関数

下記では、`try_divide()`関数を使用してみた。0で割っても`Division by zero.`がでず、`null`になる。

```sql
with tmp as (
    select * from (values 
      (0,  1,  TRUE, 'a'), (0, 3, TRUE , 'a'), (0, 10,  TRUE, 'a'),
      (1, 0,  TRUE, 'a'), (1, 0, TRUE , 'b'), (1, null, FALSE, 'b')
      ) AS tab(id, col, flg, str)
)
select 
  id
  , col
  -- , id/col as div -- Division by zero.
  , try_divide(id, col) as try_div
from 
  tmp 
order by 
  id asc
;
```

|id|col|try_div|
|:---:|:---:|:---:|
|0|1|0.0|
|0|3|0.0|
|0|10|0.0|
|1|0| |
|1|0| |
|1| | |

## :pencil2: ラムダ関数

個人的にはあんまり使い道がわかってないが、ラムダ関数がつける。

```sql
SELECT 
array('Hello', 'World') as str, 
array_sort(array('Hello', 'World'),
  (p1, p2) -> CASE WHEN p1 = p2 THEN 0
              WHEN reverse(p1) < reverse(p2) THEN -1
              ELSE 1 END)  as inv_str;
```

|str|inv_str|
|:---:|:---:|
|["Hello","World"]|["World","Hello"]|

## :closed_book: Reference

- [組み込み関数のアルファベット順の一覧](https://learn.microsoft.com/ja-jp/azure/databricks/sql/language-manual/sql-ref-functions-builtin-alpha)





