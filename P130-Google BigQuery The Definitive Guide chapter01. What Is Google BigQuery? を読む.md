## :memo: Overview

下記の書籍の内容を各章に分けてメモを残しておく。特段、BigQuery自体について調べたことがなく、あまり詳しくないので、メモをまとめておく。600 ページほどあるが、非常に実践的な内容が多く、ありがたい一冊。ただ、日本語翻訳版の書籍がなく、見返すのに時間がかかるので、メモを残している。

- [Google BigQuery: The Definitive Guide](https://www.oreilly.com/library/view/google-bigquery-the/9781492044451/)

書籍自体は基本的な内容もカバーしているが、書籍を頭から読んでいって、気になった箇所をメモしているだけなので、特に内容ごとに整理しているわけでもない。

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`BigQuery`

## :pencil2: Chapter1 What Is Google BigQuery?

サンプルで登場する`bigquery-public-data.new_york_citibike.citibike_trips`データは、行数が58,937,715で、合計論理バイト数は7.47 GB。BigQueryでは、ご法度ではあるが、`select *`すると7.47 GBスキャンされる。

プレビューでデータの中身を見ればわかるが、`NULL`のみのレコードが非常に多い。下記のSQLで件数を調べると、58,937,715 - 53,108,721 = 5,828,994は、`tripduration`が`NULL`の行である。

```sql
SELECT
  count(1)
FROM
  `bigquery-public-data.new_york_citibike.citibike_trips`
WHERE
  tripduration IS NOT NULL
;
```
テーブル名の部分は、`bigquery-public-data.new_york_citibike.citibike_trips`となっており、順に`projectid.dataset.table`という並びで記述される。バックティックが必要なのかどうかは、名前によるが、今回のケースはプロジェクト名に`-`が入っており、マイナス演算子と解釈されるので、バックティックが必要となる。下記の形式でも問題ない。

```sql
`bigquery-public-data`.new_york_citibike.citibike_trips.
```

データセットである`new_york_citibike`は、テーブルやビューをコントロールするトップレベルのコンテナ。また、テーブルである`citibike_trips`は何らかのデータセットに属する必要があるので、単体では存在することができない。

## :pencil2: Chapter2 Query Essentials

BigQueryではレコードを取得するタスクを複数のワーカーに分散させて、各ワーカーのシャードを読み取るため、同じクエリを実行しても、異なる結果が得られる。キャッシュが残っている場合、同じ結果を返すが、そうではない場合、異なる結果が返される。

例えば、`gender`と`tripduration`を入れ替えて実行すればキャッシュが効かないので異なる結果が返されることを確認できる。

```sql
SELECT
  gender,
  tripduration
FROM
  `bigquery-public-data.new_york_citibike.citibike_trips`
WHERE
  tripduration IS NOT NULL
LIMIT 3
;
```

よく言われていることではあるが、`LIMIT`句は、結果の表示レコード数を制限できるが、スキャン量には影響しないので注意。

BiqQueryには`EXCEPT`が利用できるため、不要なカラム以外を指定して、その他のカラムを選択できる。`*`の後にカンマは不要なので注意。

```sql
SELECT 
  * EXCEPT(tripduration, gender)
FROM
  `bigquery-public-data.new_york_citibike.citibike_trips`
```

BigQueryは実行したクエリの履歴を保存している。これは成功したクエリだけではなく、エラーのクエリも含まれる。クエリの結果は、約24時間後に期限切れとなる一時テーブルに保存され、キャッシュとしても使用される。クエリが重複しているかどうかは文字列マッチングによって判定されるので、冒頭に書いたように少しでも変更すればキャッシュは利用されない。

## :pencil2: Chapter3　Data Types, Functions,　and Operators

0除算エラーは`IEEE_Divide`関数で回避できる。`val1/val2`だとエラーで計算が実行できない。`IEEE_DIVIDE`の[ドキュメント](https://cloud.google.com/bigquery/docs/reference/standard-sql/mathematical_functions#ieee_divide)　には
分子分母のパターンで、どのような値を返すかが書かれている。

```sql
WITH test AS (
  SELECT 10 AS val1, 3 AS val2
  UNION ALL SELECT 20, 5
  UNION ALL SELECT 10, 0
)
SELECT
  val1,
  val2,
  -- val1/val2 AS frac,
  IEEE_Divide(val1, val2) AS safe_frac,
  if(val2 = 0, null, val1/val2) AS safe_frac2
from
  test
;
```
|val1|val2|safe_frac|safe_frac2|
|:----|:----|:----|:----|
|10|3|3.3333333333333335|3.3333333333333335|
|20|5|4.0|4.0|
|10|0|Infinity| |

`SAFE`関数という便利な関数にあるようで、`SAFE`を付けることで、どんなスカラー関数でも`null`を返すようにすることができる。

```sql
SELECT SAFE.DIV(10, 0) as safe_frac;
```

|safe_frac|
|:----|
| null |

`null`でない値になるまで式を評価し続ける便利な関数`COALESCE`を使う方法もある。`null`でない結果が得られたら、後の式は評価されない。

```sql
WITH test AS (
SELECT 10 AS x, 5 AS y, 3 AS z
UNION ALL SELECT NULL, 5, 3
UNION ALL SELECT 10, NULL, 3
UNION ALL SELECT 10, 5, NULL
UNION ALL SELECT 10, NULL, NULL
)
SELECT
  x, y, z,
  COALESCE(
    x * y * z,
    x * 5 * z,
    x * y * 3,
    NULL
    ) AS coalesce_prod
FROM test
;
```
|x|y|z|coalesce_prod|
|:----|:----|:----|:----|
|10|5|3|150|
|null |5|3| null|
|10|null |3|150|
|10|5|null |150|
|10| null| null|null |

`IFNULL(a, b)`は`COALESCE(a, b)`や`IFNULL(a IS NULL, b, a)`と同じとも言える。`null`はキャストする際にも意図的に利用できる。下記のように、数字を文字型として保存しているナンセンスなテーブルがあったとする。笑えないが。このようなテーブルは実際、ビジネスではよくある。`CAST`関数だと、下記の例では`a`が邪魔してキャストできずエラーになるが、`SAFE_CAST`関数であれば、`null`として扱うことが出来る。

```sql
WITH test AS (
SELECT '1' as val1
UNION ALL SELECT '2'
UNION ALL SELECT 'a'
)
SELECT 
  SUM(SAFE_CAST(val1 as INT64)) as sum_val 
FROM test
```

|sum_val|
|:----|
|3|

ここまではPostgreSQLやMySQLで記述するような形でサンプルデータを作っていたが、1列だけであれば、配列を使った方がBigQueryでは簡単かもしれない。

```sql
WITH
  test AS (
  SELECT *
  FROM 
    UNNEST([ 'apple', 'avocado', 'orange', 'raspberry', 'grapefruit', 'kiwi', 'coconut', 'cherry', 'pineapple', 'banana' ]) AS fruits )
SELECT
  fruits
FROM
  test;
```
|fruits|
|:----|
|apple|
|avocado|
|orange|
|raspberry|
|grapefruit|
|kiwi|
|coconut|
|cherry|
|pineapple|
|banana|

複数の列になると、なんとも言えないが、`UNION ALL`を重ね続けるよりかはマシかも知れない。

```sql
WITH
  test AS (
  SELECT 
    ['apple', 'avocado', 'orange', 'raspberry', 'grapefruit', 'kiwi', 'coconut', 'cherry', 'pineapple', 'banana'] AS fruits,
    ['a', 'a', 'o', 'r', 'g', 'k', 'c', 'c', 'p', 'b'] AS prefix
  )
SELECT
  fruits[offset] AS fruits,
  prefix[offset] AS prefix
FROM
  test,
  UNNEST(fruits) WITH OFFSET AS offset;
```

|fruits|prefix|
|:----|:----|
|apple|a|
|avocado|a|
|orange|o|
|raspberry|r|
|grapefruit|g|
|kiwi|k|
|coconut|c|
|cherry|c|
|pineapple|p|
|banana|b|

日本語と英語、記号を扱う際に便利なUnicode系の文字列関数ももちろんある。ここではサンプルデータとして波ダッシュ・全角チルダ問題を例にする。BigQueryはUnicode文字の配列、バイトの配列、Unicodeコードポイント(INT64) の配列の3種類の方法で文字列を表現できる。

```sql
WITH test AS (
  SELECT * from unnest([
    '〜', '～', '~', '〰'
  ]) AS char
)
SELECT 
  char
  , TO_CODE_POINTS(char)[OFFSET(0)] as first_code_point
  , CAST (char AS BYTES) as bytes
FROM test
```
|char|first_code_point|bytes|
|:----|:----|:----|
|〜|12316|44Cc|
|～|65374|772e|
|~|126|fg==|
|〰|12336|44Cw|

## :pencil2: Chapter4 Loading Data into BigQuery

`bq`コマンドを利用してローカルからcsvをロードする。書籍の従って、リポジトリをクローンして、圧縮して、データセット`ch04`を作成してロードする。

```
$ git clone https://github.com/GoogleCloudPlatform/bigquery-oreilly-book.git
$ cd bigquery-oreilly-book/04_load
$ ZLESS college_scorecard.csv.gz

$ bq --location=asia-northeast1 mk ch04
Dataset 'sqlsandbox376108:ch04' successfully created.

$ bq ls
  datasetId
 -----------
  ch04

$ bq --location=asia-northeast1 \
    load \
    --source_format=CSV --autodetect \
    ch04.college_scorecard \
    ./college_scorecard.csv.gz
Upload complete.
Waiting on bqjob_r3855aa911ea73559_00000189c6ef7b0d_1 ... (35s) Current status: DONE
```

`replace=false`を指定すると、既存のテーブルにレコードを追加出来る。また、スキーマを自動検出させたくなければ、一旦自動検出でスキーマの定義ファイルを書き出して、再度テーブルを作り直す方法がある。

```
$ bq show --format prettyjson --schema ch04.college_scorecard > schema.json
[
  {
    "mode": "NULLABLE",
    "name": "UNITID",
    "type": "INTEGER"
  },
  {
    "mode": "NULLABLE",
    "name": "OPEID",
    "type": "INTEGER"
  },
  {
    "mode": "NULLABLE",
    "name": "OPEID6",
    "type": "INTEGER"
  },
(SNIP)
  {
    "mode": "NULLABLE",
    "name": "OMENRAP8_PTNFT_POOLED_SUPP",
    "type": "STRING"
  },
  {
    "mode": "NULLABLE",
    "name": "OMENRUP8_PTNFT_POOLED_SUPP",
    "type": "STRING"
  }
]
```

ここでは、数字列に記録されている`PrivacySuppressed`を`null`に置き換えて、スキーマ定義ファイルを使用してロードする。

```
$ zless ./college_scorecard.csv.gz | \
    sed 's/PrivacySuppressed/NULL/g' | \
    gzip > ./college_scorecard2.csv.gz

$ bq load \
  --location=asia-northeast1 \
  --null_marker=NULL --replace \
  --source_format=CSV \
  --schema=schema.json \
  --skip_leading_rows=1 \
  ch04.college_scorecard \
  ./college_scorecard2.csv.gz
```

他の方法として、スキーマ情報は下記のクエリで取得できる。

```sql
SELECT
  table_name
  , column_name
  , ordinal_position
  , is_nullable
  , data_type
FROM
  ch04.INFORMATION_SCHEMA.COLUMNS

-- SQLでJSONをつくる方法もある
SELECT
  TO_JSON_STRING(
  ARRAY_AGG(STRUCT(
    IF(is_nullable = 'YES', 'NULLABLE', 'REQUIRED') AS mode,
    column_name AS name,
    data_type AS type)
    ORDER BY ordinal_position), TRUE) AS schema
FROM
  ch04.INFORMATION_SCHEMA.COLUMNS
WHERE
  table_name = 'college_scorecard'
```

BiqQueryはデータを保存しておくのに料金がかかるので、一定期間経過した後に削除する設定を追加することもできる。このテーブルは2023年8月6日に作ったので、デフォルトでは2023年10月5日となっている。

- [ALTER TABLE SET OPTIONS statement](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-definition-language#examples_14)

```
ALTER TABLE ch04.college_scorecard
SET OPTIONS (
  expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY),
  description = "College Scorecard table that expires seven days from now"
)
```
このクエリを実行すると、7日後の2023年8月13日まで有効なテーブルに設定が変更されている。また、読み込んだテーブルにはカラムが非常に多く、使用しにくいので部分的にデータを取り出して、異なるテーブルとして保存することもできる。BigQuery固有の方法があるわけではなく、よく使われる`CTAS`クエリを使用すればよい。

```sql
CREATE OR REPLACE TABLE ch04.college_scorecard_etl AS
SELECT
INSTNM
, ADM_RATE_ALL
, FIRST_GEN
, MD_FAMINC
, SAT_AVG
, MD_EARN_WNE_P10
FROM ch04.college_scorecard
```

## :pencil2: Chapter5

## :pencil2: Chapter6

## :pencil2: Chapter7

## :pencil2: Chapter8

## :pencil2: Chapter9

## :closed_book: Reference

- [Google BigQuery: The Definitive Guide](https://www.oreilly.com/library/view/google-bigquery-the/9781492044451/)
- [bigquery-oreilly-book](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book)