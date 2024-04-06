## :memo: Overview

Databricksをはじめてさわる機会があったので、自己学習用にAzureでDatabricks環境を構築して、Databricks SQLで遊んでみた。Databricks SQLに関しては、データブリックス・ジャパンの方が書かれた下記の記事がわかりやすい。

- [Databricks SQLとは何か？](https://qiita.com/taka_yayoi/items/5b6e4537b086775a408a)

今回は`array`を中心に遊んでみる。 `array`をそのまま扱うことはないと思うので、テーブルの値から`array`型に変換するパターン、DBの値が`array`型で定義されている場合の2つのケースで遊んでみる。


## :floppy_disk: Database

Databricks

## :bookmark: Tag

`array_agg`,`array_agg()`,`array_append()`,`array_compact()`,`array_contains()`,`array_distinct()`,`array_except()`,`array_insert()`,`array_intersect()`,`array_join()`,`array_max()`,`array.min()`,`array_position()`,`array_prepend()`,`array_remove()`,`array_repeat()`,`array_size()`,`array_sort()`,`array_union()`,`arrays_overlap()`,

## :pencil2: テーブルの値から`array`型に変換するパターン
 
テーブルの値から`array`型に変換するパターンで遊んでみる。実際のデータでは、順序関係が正しく積み上げられている保証はないテーブルから、特定のカラムの時系列をもとに並び替え、遷移順を作るとかはありそう。例えば、商品名の購入順で遷移順を作るとか。

まずは、`array_agg()`関数は値を配列としてまとめ上げる。配列内の要素の順序は「非決定論的」なので、順序の保証はなく、`null`は除外される。`array_agg(DISTINCT col)`という使い方もできて、この場合は重複を削除できる。`collect_list()`関数とも同じ。

```sql
with array_tmp as (
    select * from (values 
      (1, 1, 'a'), (1, 3, 'c'), (1, 2, 'b'),
      (2, 1, 'a'), (2, 2, 'b') , (2, 3, null),
      (3, 1, 'a'), (3, 2, null), (3, 3, 'b'),
      (4, 1, 'a')
      ) AS tab(cid, hist, alf)
      order by cid asc, hist asc
)
select cid, 
array_join(array_agg(alf), ' -> ') as move
, array_join(array_agg(hist), ' -> ') as move_hist
from array_tmp 
group by cid
order by cid asc;

```

|cid|move|move_hist|
|:----|:----|:----|
|1|a -> b -> c|1 -> 2 -> 3|
|2|a -> b|1 -> 2 -> 3|
|3|a -> b|1 -> 2 -> 3|
|4|a|1|


下記のケースでもうまく行っているように見えるが、`cid=1`の結果がおかしくなっている。配列内の要素の順序は非決定論的で、制御ができない。他の環境のSQLであれば、`array_agg(alf order by hist asc)`と書けるので、順序をコントロールできるが、Databricksはそうではないので、注意が必要。さきほどの例では、`order by`で並び替えているが、あれで順序を保証できるかは定かではない・・・。

```sql
with array_tmp as (
    select * from (values 
      (1, 1, 'a'), (1, 3, 'c'), (1, 2, 'b'),
      (2, 1, 'a'), (2, 2, 'b') , (2, 3, null),
      (3, 1, 'a'), (3, 2, null), (3, 3, 'b'),
      (4, 1, 'a')
      ) AS tab(cid, hist, alf)
      -- order by cid asc, hist asc --コメントアウトしている
)
select cid, 
array_join(array_agg(alf), ' -> ') as move
, array_join(array_agg(hist), ' -> ') as move_hist
from array_tmp 
group by cid
order by cid asc;
```

|cid|move|move_hist|
|:----|:----|:----|
|1|a -> c -> b|1 -> 3 -> 2|
|2|a -> b|1 -> 2 -> 3|
|3|a -> b|1 -> 2 -> 3|
|4|a|1|


今回のように1→2→3という順番で配列の値をを結合したのであれば、`array_sort()`を利用して確認する方法が良いかもしれない。`array_sort()`は配列の値を並び替えるので、その前の段階で、並び替えた文字列を、順序を保持して結合に持っていけるわけではない。ただ、並べたい順序(1→2→3という順番)での並べ方は、`array_sort()`で並べた順番と同じである。下記の通り、並び順を制御することを忘れ、意図していない並び方になっていると、このように`move_hist_sort`と`move_hist`の順番が合わない。このように検証しながら使うのは良いかもしれない。

```sql
with array_tmp as (
    select * from (values 
      (1, 1, 'a'), (1, 3, 'c'), (1, 2, 'b'),
      (2, 1, 'a'), (2, 2, 'b') , (2, 3, null),
      (3, 1, 'a'), (3, 2, null), (3, 3, 'b'),
      (4, 1, 'a')
      ) AS tab(cid, hist, alf)
      -- order by cid asc, hist asc
)
select cid, 
array_join(array_agg(alf), ' -> ') as move
, array_join(array_sort(array_agg(hist)), ' -> ') as move_hist_sort
, array_join(array_agg(hist), ' -> ') as move_hist
, if(move_hist_sort = move_hist, TRUE, FALSE) as move_check
from array_tmp 
group by cid
order by cid asc;
```
|cid|move|move_hist_sort|move_hist|move_check|
|:----|:----|:----|:----|:----|
|1|a -> c -> b|1 -> 2 -> 3|1 -> 3 -> 2|false|
|2|a -> b|1 -> 2 -> 3|1 -> 2 -> 3|true|
|3|a -> b|1 -> 2 -> 3|1 -> 2 -> 3|true|
|4|a|1|1|true|

チェックして問題なさそうとなっても、実際は`move`と`move_hist`が評価されて実行されるときに、同じ順序になるかどうかの保証は得られてない・・・。これは`move_hist_sort`と`move_hist`が一致しているということでしかない。

```sql
with array_tmp as (
    select * from (values 
      (1, 1, 'a'), (1, 3, 'c'), (1, 2, 'b'),
      (2, 1, 'a'), (2, 2, 'b') , (2, 3, null),
      (3, 1, 'a'), (3, 2, null), (3, 3, 'b'),
      (4, 1, 'a')
      ) AS tab(cid, hist, alf)
      order by cid asc, hist asc
)
select cid, 
array_join(array_agg(alf), ' -> ') as move
, array_join(array_sort(array_agg(hist)), ' -> ') as move_hist_sort
, array_join(array_agg(hist), ' -> ') as move_hist
, if(move_hist_sort = move_hist, TRUE, FALSE) as move_check
from array_tmp 
group by cid
order by cid asc;
```

|cid|move|move_hist_sort|move_hist|move_check|
|:----|:----|:----|:----|:----|
|1|a -> b -> c|1 -> 2 -> 3|1 -> 2 -> 3|true|
|2|a -> b|1 -> 2 -> 3|1 -> 2 -> 3|true|
|3|a -> b|1 -> 2 -> 3|1 -> 2 -> 3|true|
|4|a|1|1|true|

最初から実際の実行計画を見ればよいのだが、実行順序は・・・配列にまとめる前に、意図的にテーブル作成時に`order by`しているが、その後にShuffleされている。やはり上記のやり方ではデータのサイズが大きくなった時にはよろしくなさそう。

1: Result Query Stage
2: Object Hash Aggregate
3: Shuffle
4: Shuffle
5: Sort
6: Shuffle
7: Shuffle
8: Local Table Scan

![plan](https://github.com/SugiAki1989/sql_note/blob/main/image/p145-plan.png)

Shuffleの意味を取り違えている可能性があるが、DatabricksはApache Sparkのデータ処理エンジンを使っているっぽいので、異なるワーカーノード間でデータをShuffleまたはリパーティションするはず。今回のような特定のキーでデータをグループ化する場合、同じキーのデータが同じワーカーノードに集められるようにデータがリパーティションされるはずで、並び順も変わるかもしれない。このあたりの理解を深めることができれば、確信をもって回答できそうではあるが、現状、生半可なのでなんともわからない。

ただ、実践で利用するのであれば、他の解決策もある。少し面倒ではあるが、理解があやふやな以上、下記のほうが安全かも。

```sql
with array_tmp as (
    select * from (values 
      (1, 1, 'a'), (1, 3, 'c'), (1, 2, 'b'),
      (2, 1, 'a'), (2, 2, 'b') , (2, 3, null),
      (3, 1, 'a'), (3, 2, null), (3, 3, 'b'),
      (4, 1, 'a')
      ) as tab(cid, hist, alf)
), tmp2 as (
select 
  cid
  , max(case when hist = 1 then alf else null end) as hist1  
  , max(case when hist = 2 then alf else null end) as hist2
  , max(case when hist = 3 then alf else null end) as hist3  
from array_tmp 
group by cid
)
select
  cid
  , ifnull(hist1, '') || ' → ' || ifnull(hist2, '') || ' → ' || ifnull(hist3, '') as move_miss
  , concat_ws(' → ', hist1, hist2, hist3) as  move
from tmp2
;
```

|cid|move_miss|move|
|:----|:----|:----|
|1|a → b → c|a → b → c|
|2|a → b → |a → b|
|3|a →  → b|a → b|
|4|a →  → |a|

`concat_ws(seq, expr1, expr2,...,exprN)`関数は`sep`で区切られた連結文字列を返すが、`null`の`exprN`は無視される。

```sql
SELECT concat_ws(',', 'Spark', array('S', 'Q', NULL, 'L'), NULL);
-> Spark,S,Q,L
```

## :pencil2: DBの値が`array`型で定義されているパターン

関数名を見れば挙動はわかると思うので説明は割愛。

```sql
with cust as (
              select 1 as idx, 'mike'  as name, 'm' as gen, array(20, 10, 30) as coupons, array('tag1', 'tag2') as tags, array('tag1', 'tag9') as expired
    union all select 2 as idx, 'tom'   as name, 'm' as gen, array(10, 10)     as coupons, array('tag1', 'tag2', 'tag3') as tags, array('tag1') as expired
    union all select 3 as idx, 'keny'  as name, 'f' as gen, array(100, 20, 15) as coupons, array('tag1') as tags, array('tag1', 'tag9') as expired
    union all select 4 as idx, 'kathy' as name, 'f' as gen, array(null)       as coupons, array(null) as tags, array('tag1') as expired
)
select
  idx, name, gen, coupons, tags,
  --　array_prepend()もある
  array_append(tags, 'new') as a_append, 
  array_insert(tags, 1, 'new') as a_insert,
  array_compact(tags) as a_compact,
  array_contains(coupons, 10) as a_contains,
  slice(coupons, 2, 1) as a_slice,
  filter(coupons, x -> x = 10) as a_filter,
  array_distinct(coupons) as a_distinct,
  array_except(tags, expired) as a_except,
  array_intersect(tags, expired) as a_intersect,
  array_union(tags, expired) as a_union,
  arrays_overlap(tags, expired) as a_overlap,
  flatten(array(tags, expired)) as a_flatten,
  --　array_min()もある
  array_max(coupons) as a_max, 
  aggregate(coupons, 0, (acc, x) -> acc + x) as a_aggregate,
  --　element_at()もある
  array_position(coupons, 10) as a_position,
  array_remove(tags, 'tag1') as a_remove,
  array_repeat(tags, 2) as a_repeat,
  -- cardinality()もある
  array_size(tags) as a_size,
  array_sort(coupons) as a_sort
from cust;
```

|idx|name|gen|coupons|tags|a_append|a_insert|a_compact|a_contains|a_slice|a_filter|a_distinct|a_except|a_intersect|a_union|a_overlap|a_flatten|a_max|a_aggregate|a_position|a_remove|a_repeat|a_size|a_sort|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|1|mike|m|[20,10,30]|["tag1","tag2"]|["tag1","tag2","new"]|["new","tag1","tag2"]|["tag1","tag2"]|true|[10]|[10]|[20,10,30]|["tag2"]|["tag1"]|["tag1","tag2","tag9"]|true|["tag1","tag2","tag1","tag9"]|30|60|2|["tag2"]|[["tag1","tag2"],["tag1","tag2"]]|2|[10,20,30]|
|2|tom|m|[10,10]|["tag1","tag2","tag3"]|["tag1","tag2","tag3","new"]|["new","tag1","tag2","tag3"]|["tag1","tag2","tag3"]|true|[10]|[10,10]|[10]|["tag2","tag3"]|["tag1"]|["tag1","tag2","tag3"]|true|["tag1","tag2","tag3","tag1"]|10|20|1|["tag2","tag3"]|[["tag1","tag2","tag3"],["tag1","tag2","tag3"]]|3|[10,10]|
|3|keny|f|[100,20,15]|["tag1"]|["tag1","new"]|["new","tag1"]|["tag1"]|false|[20]|[]|[100,20,15]|[]|["tag1"]|["tag1","tag9"]|true|["tag1","tag1","tag9"]|100|135|0|[]|[["tag1"],["tag1"]]|1|[15,20,100]|
|4|kathy|f|[null]|[null]|[null,"new"]|["new",null]|[]| |[]|[]|[null]|[null]|[]|[null,"tag1"]| |[null,"tag1"]| | |0|[null]|[[null],[null]]|1|[null]|


## :closed_book: Reference

- [組み込み関数のアルファベット順の一覧](https://learn.microsoft.com/ja-jp/azure/databricks/sql/language-manual/sql-ref-functions-builtin-alpha)
