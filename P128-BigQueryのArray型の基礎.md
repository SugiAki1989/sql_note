## :memo: Overview

ここでは BigQuery で配列を使用する場合の基本的なクエリの使い方をまとめておく。BigQuery でいう配列とは、0個以上の同じデータ型の値で構成された順序付きリストのこと、とのこと。

ここでは配列の基本的な部分を下記の公式ドキュメントに従っておさらいし、次回、簡易的なアトリビューション分析を配列を利用して行う方法をまとめる。まずは、基礎編。

- [配列の操作](https://cloud.google.com/bigquery/docs/arrays?hl=ja)

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`array`,`offset`,`array_length`,`unnest`,`array_agg`,`array_to_string`,`array_concat`

## :pencil2: Example

配列の`ARRAY`データ型は`[]`で値をラップすれば生成できる。

```sql
WITH
  Sequences AS (
    SELECT [0, 1, 2, 3] AS some_numbers UNION ALL
    SELECT [1, 2, 3] UNION ALL
    SELECT [2, 4, 6] UNION ALL
    SELECT [1, 1, 1]
  )
SELECT * FROM Sequences;
```
|some_numbers|
|:----|
|[0,1,2,3]|
|[1,2,3]|
|[2,4,6]|
|[1,1,1]|

UIで返される見た目は、1行の中に縦積みで表示されるため、上記とは少し見た目が異なる点は注意。配列の要素にアクセスしたいときは`arr[OFFSET(index)]`とすることで、アクセスできる。0インデックス始まりの点は注意する。1インデックス始まりで操作したければ、`arr[ORDINAL(index)]`を利用する。


```sql
WITH
  Sequences AS (
    SELECT [0, 1, 2, 3] AS some_numbers UNION ALL
    SELECT [1, 2, 3] UNION ALL
    SELECT [2, 4, 6] UNION ALL
    SELECT [1, 1, 1]
  )
SELECT 
  some_numbers,
  some_numbers[OFFSET(0)] as offset0,
  some_numbers[OFFSET(1)] as offset1,
  some_numbers[OFFSET(2)] as offset2
FROM 
  Sequences;
```
|some_numbers|offset0|offset1|offset2|
|:----|:----|:----|:----|
|[0,1,2,3]|0|1|2|
|[1,2,3]|1|2|3|
|[2,4,6]|2|4|6|
|[1,1,1]|1|1|1|

配列の要素数は`ARRAY_LENGTH`関数で取得できる。


```sql
WITH
  Sequences AS (
    SELECT [0, 1, 2, 3] AS some_numbers UNION ALL
    SELECT [1, 2, 3] UNION ALL
    SELECT [2, 4, 6] UNION ALL
    SELECT [1, 1, 1]
  )
SELECT 
  some_numbers,
  some_numbers[OFFSET(0)] as offset0,
  some_numbers[OFFSET(1)] as offset1,
  some_numbers[OFFSET(2)] as offset2
FROM 
  Seq
```

|some_numbers|len|
|:----|:----|
|[0,1,2,3]|4|
|[1,2,3]|3|
|[2,4,6]|3|
|[1,1,1]|3|


配列の要素を展開して行にする場合は、`UNNEST`関数を利用することで、要素を展開できる。つまり、配列を展開してフラット化する。

`UNNEST`関数ではなく、`UNNEST`演算子のほうが適切な表現かもしれない。また、少し`UNNEST`関数は見慣れない記述方法になる(個人的に)。例えば、下記の様な配列を含むデータがあったとする。

```
WITH tbl AS (
    SELECT ['banana', 'orange'] AS name, 'fruit' as category UNION ALL
    SELECT ['tomato', 'asparagus', 'okra', 'pea'] AS name, 'vegetable' as category
  )
SELECT 
  *
FROM 
  tbl
;
```
|name|category|
|:----|:----|
|[banana,orange]|fruit|
|[tomato,asparagus,okra,pea]|vegetable|

これを`UNNEST`関数を使うことで、扱いやすい形に変換できる。また、展開する際に`WITH OFFSET`とつけることで、配列のインデックスを取得できる。

```
WITH tbl AS (
    SELECT ['banana', 'orange'] AS name, 'fruit' as category UNION ALL
    SELECT ['tomato', 'asparagus', 'okra', 'pea'] AS name, 'vegetable' as category
  )
SELECT 
  category, 
  name,
  offset
FROM 
  tbl, UNNEST(name) AS name WITH OFFSET AS offset;
```

|category|name|offset|
|:----|:----|:----|
|fruit|banana|0|
|fruit|orange|1|
|vegetable|tomato|0|
|vegetable|asparagus|1|
|vegetable|okra|2|
|vegetable|pea|3|

似たような例として、ドキュメントには下記の`CROSS JOIN`する例も記載されている。相関クロス結合の場合、`UNNEST`関数は省略可。

```
WITH Sequences AS
  (SELECT 1 AS id, [0, 1, 1, 2, 3, 5] AS some_numbers
   UNION ALL SELECT 2 AS id, [2, 4, 8, 16, 32] AS some_numbers
   UNION ALL SELECT 3 AS id, [5, 10] AS some_numbers)
SELECT id, flattened_numbers
FROM Sequences
CROSS JOIN UNNEST(Sequences.some_numbers) AS flattened_numbers;

WITH Sequences AS
  (SELECT 1 AS id, [0, 1, 1, 2, 3, 5] AS some_numbers
   UNION ALL SELECT 2 AS id, [2, 4, 8, 16, 32] AS some_numbers
   UNION ALL SELECT 3 AS id, [5, 10] AS some_numbers)
SELECT id, flattened_numbers
FROM Sequences, Sequences.some_numbers AS flattened_numbers;
```

公式ドキュメントには`STRUCT`の`ARRAY`を含むデータをフラット化する例も記載されているが、あまり使用頻度が高くないので、ここでは省略する。

次は配列の要素に対する処理を行う方法について。BigQueryでは、配列の要素に対して、何らかの処理を行うことが出来る。JavaScriptの`map`メソッドのようなイメージ。公式ドキュメントにあるように、要素を2倍したければ、下記の通り記述する。これは、フラット化したものを配列に再結合するために、`UNNEST`関数で展開して、処理して、`ARRAY`関数で配列に変換しているようなイメージ。

```
WITH Sequences AS
  (SELECT [0, 1, 1, 2, 3, 5] AS some_numbers
  UNION ALL SELECT [2, 4, 8, 16, 32] AS some_numbers
  UNION ALL SELECT [5, 10] AS some_numbers)
SELECT some_numbers,
  ARRAY(SELECT x * 2
        FROM UNNEST(some_numbers) AS x) AS doubled
FROM Sequences;
```

|some_numbers|doubled|
|:----|:----|
|[0,1,1,2,3,5]|[0,2,2,4,6,10]|
|[2,4,8,16,32]|[4,8,16,32,64]|
|[5,10]|[10,20]|

```
WITH tbl AS (
    SELECT ['banana', 'orange'] AS name, 'fruit' as category UNION ALL
    SELECT ['tomato', 'asparagus', 'okra', 'pea'] AS name, 'vegetable' as category
  )
SELECT 
  category, 
  name,
  ARRAY(SELECT CONCAT(name, '!') FROM UNNEST(name) AS name) AS new_name
FROM 
  tbl;
```
|category|name|new_name|
|:----|:----|:----|
|fruit|[banana,orange]|[banana!,orange!]|
|vegetable|[tomato,asparagus,okra,pea]|[tomato!,asparagus!,okra!,pea!]|

この2つの配列を行としてフラット化したい場合は、下記のように書けばよさそうではあるが、これでは意図していない結果となる。`UNNEST`演算子を複数書くと、直積のような計算となってしまう。

```
WITH tbl AS (
    SELECT ['banana', 'orange'] AS name, 'fruit' as category UNION ALL
    SELECT ['tomato', 'asparagus', 'okra', 'pea'] AS name, 'vegetable' as category
  ), tbl2 AS (
SELECT 
  category, 
  name,
  ARRAY(SELECT concat(name, '!') FROM UNNEST(name) AS name) AS new_name
FROM 
  tbl
)
SELECT
  category,
  name,
  new_name,
  offset
FROM
  tbl2, 
  UNNEST(name) AS name WITH OFFSET AS offset, 
  UNNEST(new_name) AS new_name
;
```
|category|name|new_name|offset|
|:----|:----|:----|:----|
|fruit|banana|banana!|0|
|fruit|banana|orange!|0|
|fruit|orange|banana!|1|
|fruit|orange|orange!|1|
|vegetable|tomato|tomato!|0|
|vegetable|tomato|asparagus!|0|
|vegetable|tomato|okra!|0|
|vegetable|tomato|pea!|0|
|vegetable|asparagus|tomato!|1|
|vegetable|asparagus|asparagus!|1|
|vegetable|asparagus|okra!|1|
|vegetable|asparagus|pea!|1|
|vegetable|okra|tomato!|2|
|vegetable|okra|asparagus!|2|
|vegetable|okra|okra!|2|
|vegetable|okra|pea!|2|
|vegetable|pea|tomato!|3|
|vegetable|pea|asparagus!|3|
|vegetable|pea|okra!|3|
|vegetable|pea|pea!|3|

意図したように配列をフラット化したい場合は下記のようなクエリを記述する。もちろん他の方法でも書ける。`name`を
`offset`でインデックス付きで展開し、そのインデックスで`new_name`の値を取得する。

```
WITH tbl AS (
  SELECT ['banana', 'orange'] AS name, 'fruit' as category UNION ALL
  SELECT ['tomato', 'asparagus', 'okra', 'pea'] AS name, 'vegetable' as category
), tbl2 AS (
SELECT 
  category, 
  name,
  (SELECT ARRAY(SELECT CONCAT(name, '!') FROM UNNEST(name) AS name)) AS new_name
FROM 
  tbl
)
SELECT
  category,
  name,
  new_name[offset] AS new_name
FROM
  tbl2,
  UNNEST(name) AS name WITH OFFSET AS offset
;
```
|category|name|new_name|
|:----|:----|:----|
|fruit|banana|banana!|
|fruit|orange|orange!|
|vegetable|tomato|tomato!|
|vegetable|asparagus|asparagus!|
|vegetable|okra|okra!|
|vegetable|pea|pea!|

他にも配列の値に条件をつけて抽出したり、

```
WITH Sequences AS
  (SELECT [0, 1, 1, 2, 3, 5] AS some_numbers
   UNION ALL SELECT [2, 4, 8, 16, 32] AS some_numbers
   UNION ALL SELECT [5, 10] AS some_numbers)
SELECT
  some_numbers,
  ARRAY(SELECT x * 2
        FROM UNNEST(some_numbers) AS x
        WHERE x < 5) AS doubled_less_than_five
FROM Sequences;

```
|some_numbers|doubled_less_than_five|
|:----|:----|
|[0,1,1,2,3,5]|[0,2,2,4,6]|
|[2,4,8,16,32]|[4,8]|
|[5,10]|[]|

重複を除外したりできる。

```
WITH Sequences AS
  (SELECT [0, 1, 1, 2, 3, 5] AS some_numbers)
SELECT 
  some_numbers,
  ARRAY(SELECT DISTINCT x FROM UNNEST(some_numbers) AS x) AS unique_numbers
FROM Sequences;

```
|some_numbers|unique_numbers|
|:----|:----|
|[0,1,1,2,3,5]|[0,1,2,3,5]|


また、配列の要素を`WHERE`句でスキャンすることも出来るので、条件の値を含むレコードのみを残したい場合など便利に利用できる。

```
WITH Sequences AS
  (SELECT 1 AS id, [0, 1, 1, 2, 3, 5] AS some_numbers
   UNION ALL SELECT 2 AS id, [2, 4, 8, 16, 32] AS some_numbers
   UNION ALL SELECT 3 AS id, [5, 10] AS some_numbers)
SELECT 
  id, some_numbers
FROM 
  Sequences
WHERE 
  2 IN UNNEST(Sequences.some_numbers);

-- このように書く必要はない
WITH Sequences AS
  (SELECT 1 AS id, [0, 1, 1, 2, 3, 5] AS some_numbers
   UNION ALL SELECT 2 AS id, [2, 4, 8, 16, 32] AS some_numbers
   UNION ALL SELECT 3 AS id, [5, 10] AS some_numbers)
, tmp as (
SELECT id AS matching_rows
FROM Sequences
WHERE 2 IN UNNEST(Sequences.some_numbers)
)
SELECT 
  * 
FROM 
  Sequences
INNER JOIN
  tmp
ON
 Sequences.id = tmp.matching_rows
;
```

|id|some_numbers|matching_rows|
|:----|:----|:----|
|1|[0,1,1,2,3,5]|1|
|2|[2,4,8,16,32]|2|


他にも、`ARRAY_AGG`関数で行を配列に集約できる。

```
WITH alphabets AS
  (SELECT "a" AS alphabet
   UNION ALL SELECT "c" AS alphabet
   UNION ALL SELECT "b" AS alphabet)
SELECT ARRAY_AGG(alphabet ORDER BY alphabet) AS ordered_alphabets
FROM alphabets;
```

|ordered_alphabets|
|:----|
|[a,b,c]|

これは`GROUP BY`句とも併用できるので、組み合わせると非常に便利。

```
WITH alphabets AS
  (SELECT "a" AS alphabet, "g1" AS category,
   UNION ALL SELECT "c" AS alphabet, "g1" AS category,
   UNION ALL SELECT "b" AS alphabet, "g1" AS category,
   UNION ALL SELECT "e" AS alphabet, "g2" AS category,
   UNION ALL SELECT "d" AS alphabet, "g2" AS category
   )
SELECT 
  category,
  ARRAY_AGG(alphabet ORDER BY alphabet) AS ordered_alphabets
FROM 
  alphabets
GROUP BY
  category;
```

|category|ordered_alphabets|
|:----|:----|
|g1|[a,b,c]|
|g2|[d,e]|


`ARRAY_TO_STRING`関数を利用して、配列から文字列への変換できたり、`ARRAY_CONCAT`関数で配列を結合できたりする。

```
SELECT
  ARRAY_TO_STRING(arr, ".", "N") AS non_empty_string,
  ARRAY_TO_STRING(arr, ".", "") AS empty_string,
  ARRAY_TO_STRING(arr, ".") AS omitted
FROM (SELECT ["a", NULL, "b", NULL, "c", NULL] AS arr);

/*------------------+--------------+---------*
 | non_empty_string | empty_string | omitted |
 +------------------+--------------+---------+
 | a.N.b.N.c.N      | a..b..c.     | a.b.c   |
 *------------------+--------------+---------*/

 SELECT ARRAY_CONCAT([1, 2], [3, 4], [5, 6]) AS count_to_six;

/*--------------------------------------------------*
 | count_to_six                                     |
 +--------------------------------------------------+
 | [1, 2, 3, 4, 5, 6]                               |
 *--------------------------------------------------*/
```

その他にも更新(`UPDATE`)や圧縮(`UNNEST`と`WITH OFFSET`)、配列の配列を作れたりもするが、あまり使用頻度が高くないので、ここではまとめていない。

次回は簡易的なアトリビューション分析を行う予定。


## :closed_book: Reference

- [配列の操作](https://cloud.google.com/bigquery/docs/arrays?hl=ja)
