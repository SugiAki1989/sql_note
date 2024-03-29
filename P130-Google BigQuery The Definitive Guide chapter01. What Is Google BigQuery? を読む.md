## :memo: Overview

下記の書籍の内容を各章に分けてメモを残しておく。特段、BigQuery自体について調べたことがなく、あまり詳しくないので、メモをまとめておく。600 ページほどあるが、非常に実践的な内容が多く、ありがたい一冊。ただ、日本語翻訳版の書籍がなく、見返すのに時間がかかるので、メモを残している。

- [Google BigQuery: The Definitive Guide](https://www.oreilly.com/library/view/google-bigquery-the/9781492044451/)

書籍自体は基本的な内容もカバーしているが、書籍を頭から読んでいって、気になった箇所をメモしているだけなので、特に内容ごとに整理しているわけでもない。

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`BigQuery`

## :pencil2: Chapter1 What Is Google BigQuery?
### new_york_citibike

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

### Cache

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

### SAFE Function
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

### COALESCE

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

### Data Generation

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

### Unicode Function

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

### Load
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
このクエリを実行すると、7日後の2023年8月13日まで有効なテーブルに設定が変更されている。

### DDL,DML

読み込んだテーブルにはカラムが非常に多く、使用しにくいので部分的にデータを取り出して、異なるテーブルとして保存することもできる。BigQuery固有の方法があるわけではなく、よく使われる`CTAS`クエリを使用すればよい。

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

中身はこの通り。

```sql
SELECT *
FROM ch04.college_scorecard_etl
LIMIT 5;
```
|INSTNM|ADM_RATE_ALL|FIRST_GEN|MD_FAMINC|SAT_AVG|MD_EARN_WNE_P10|
|:----|:----|:----|:----|:----|:----|
|Rabbinical College of America|0.95833333333333|0.1162790698|35086.5| |25600|
|Spanish-American Institute| | | | | |
|Yeshiva and Kollel Harbotzas Torah| | |9135.0| | |
|Bullard-Havens Technical High School| | |21571.0| | |
|W F Kaynor Technical High School| | |22793.0| | |

DDL,DMLももちろん利用できるが、サンドボックス環境のBigQueryを利用しているため、利用ができない。クエリを書いておく必要はないかもしれないが、一応メモしておく。

```sql
-- DELETE
DELETE FROM ch04.college_scorecard_etl
WHERE INSTNM = 'Birthwise Midwifery School';

-- INSERT
INSERT ch04.college_scorecard_etl
(INSTNM
, ADM_RATE_ALL
, FIRST_GEN
, MD_FAMINC
, SAT_AVG
, MD_EARN_WNE_P10
)
VALUES 
('abc', 0.1, 0.3, 12345, 1234, 23456),
('def', 0.2, 0.2, 23451, 1232, 32456)
```

### Avro

効率的なロード方法としてAvro形式の解説がされているので、メモしておく。この書籍によると、BigQueryで最も効率的で表現力が豊かなフォーマットはAvro形式とのこと。Parquetも良いが、下記の点でAvro形式が優れているとのこと。

- ブロックに分割されたバイナリファイル
- ブロックごとに圧縮することができるため、並列的にロードできる
- ネストしたフィールドや繰り返されるフィールドを表現することができる
- Avroファイルは自己記述型なので、スキーマの指定が不要

### GoogleSpreadSheet

あとはお手軽にデータを管理、作成できて、BigQueryにロードする方法として、外部ソースとしてGoogleSpreadSheetがある。WebUIから下記を設定する。

- 外部データ設定
  - ソース URI: https://docs.google.com/spreadsheets/d/id
  ‐ スキーマの自動検出: true
  - 不明な値を無視: false
  - ソース形式: GOOGLE_SHEETS
  - 先頭行をスキップ: 1
  - シート範囲: co2!A:B

これでテーブルが出来上がるので、BigQueryからクエリできる。外部データセットに対する連携クエリの結果はキャッシュされないので、GoogleSpreadSheetが変更されれば、結果も変わることになる。また、SpreadSheetからBigQueryに接続してクエリを実行し、その結果をSpreadSheetに入力することもできる。基本的には可視化するために必要な最小粒度までデータをBigQueryで集計して、SpreadSheetで可視化することが多いと思うが、BigQueryはSpreadSheetに表示されるデータのクラウドバックエンドとして機能させることもできる。

### Scheduled queries

クエリはスケジュール設定を行うことで定期的に実行ができ、その結果をBigQueryテーブルへの保存をサポートしている。連携クエリを使用して外部データソースからデータを抽出し、変換してBigQueryにロードすることができ、DDL、DMLを含めることができるため、SQLだけでワークフローを構築できる。

ここではサンドボックスのプロジェクトを変更して、DDL、DMLを利用できるプロジェクトに変更して、簡単なワークフローを構築する。データは何度か、最近のノートでも登場している5分ごとに部屋のCO2を記録しているSpreadSheetを転送元データとする。

```sql
SELECT
  time,
  co2_ppm
FROM
  co2dataset.co2
ORDER BY
  time desc
LIMIT 30
```
|time|co2_ppm|
|:----|:----|
|2023-08-06 15:11:37.000000 UTC|635|
|2023-08-06 15:06:34.000000 UTC|625|
|2023-08-06 15:01:29.000000 UTC|672|
|2023-08-06 14:56:26.000000 UTC|665|
|2023-08-06 14:51:22.000000 UTC|658|
|2023-08-06 14:46:16.000000 UTC|682|
|2023-08-06 14:41:13.000000 UTC|680|
|2023-08-06 14:36:07.000000 UTC|693|
|2023-08-06 14:31:02.000000 UTC|661|
|2023-08-06 14:25:49.000000 UTC|672|
|2023-08-06 14:20:46.000000 UTC|686|
|2023-08-06 14:15:42.000000 UTC|688|
|2023-08-06 14:10:38.000000 UTC|688|
|2023-08-06 14:05:34.000000 UTC|678|
|2023-08-06 14:00:30.000000 UTC|680| -- 14時台の平均は677
|2023-08-06 13:55:26.000000 UTC|679|
|2023-08-06 13:50:22.000000 UTC|661|
|2023-08-06 13:45:18.000000 UTC|656|
|2023-08-06 13:40:10.000000 UTC|667|
|2023-08-06 13:35:05.000000 UTC|673|
|2023-08-06 13:30:02.000000 UTC|671|
|2023-08-06 13:24:57.000000 UTC|682|
|2023-08-06 13:19:54.000000 UTC|694|
|2023-08-06 13:14:22.000000 UTC|655|
|2023-08-06 13:09:18.000000 UTC|660|
|2023-08-06 13:04:13.000000 UTC|636| -- 13時台の平均は666
|2023-08-06 12:59:08.000000 UTC|600|
|2023-08-06 12:54:04.000000 UTC|533|
|2023-08-06 12:49:00.000000 UTC|489|
|2023-08-06 12:43:56.000000 UTC|551|

ちょっと面倒なことに、SpreadSheetの日時関係のカラムをBigQueryで読み込むと`TIMESTAMP`型になるようで、`DATETIME`型で読み直すとBigQueryの評価は問題なくでも、実行時に下記のエラーが発生する。また、SpreadSheet側はJSTで記録されており、UTCではない。

```
Error while reading table: ch04.co22, error message: Could not convert value to datetime. Error: Invalid datetime string "Time". Row 0; Col 0. 
```

[修正方法](https://itips.krsw.biz/bigquery-could-not-convert-value-to-datetime-spreadsheet/)はこちらに書かれているが、今回はSpreadSheet側をいじれないので、無理やり処理しているので、後に登場するSQLが面倒くさい感じになっている。

それはさておき、クエリ実行時の1時間前のCO2を集計する用の転送先のテーブルを作成する。転送先のテーブルの`timestamp_1hour`は`STRING`型としているのは、`TIMESTAMP`型で戻すとUTCになってしまい、それを表記のために戻すとSQLがさらに面倒くさいことになるので、文字型で定義している。

```sql
CREATE OR REPLACE TABLE
  co2dataset.co2_summary (
    timestamp_1hour STRING,
    avg_co2_ppm FLOAT64
    );
```

スケジュールするクエリは下記を利用する。`TIMESTAMP_SUB(TIMESTAMP_ADD())`するくらいなら最初から8時間調整すれば良いようにも思えるが、そこは愚直に記述している。というか、きっともっと転送元のデータ型とかBigQueryのことをもう少し理解していれば、もっと効率的なクエリが書けるはず。

```sql
INSERT INTO co2dataset.co2_summary (timestamp_1hour, avg_co2_ppm)
SELECT
  FORMAT_TIMESTAMP('%Y-%m-%d %H:00:00', TIMESTAMP_SUB(TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 9 HOUR), INTERVAL 1 HOUR)) AS timestamp_1hour,
  AVG(co2_ppm) AS avg_co2_ppm
FROM
  co2dataset.co2
WHERE
  time >= TIMESTAMP(FORMAT_TIMESTAMP('%Y-%m-%d %H:00:00', TIMESTAMP_SUB(TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 9 HOUR), INTERVAL 1 HOUR))) AND
  TIMESTAMP(FORMAT_TIMESTAMP('%Y-%m-%d %H:00:00', TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 9 HOUR))) < time
GROUP BY
  timestamp_1hour
;
```

とりあえず実行してインサートできるか確認しておく。

```sql
SELECT
  timestamp_1hour,
  avg_co2_ppm
FROM
  co2dataset.co2_summary
;
```
|timestamp_1hour|avg_co2_ppm|
|:----|:----|
|2023-08-06 13:00:00|666|


クエリをスケジュールするためには、BigQuery Data Transfer APIが有効でないといけないので、有効にしてからスケジュールボタンから設定を行う。

- 詳細とスケジュール
  - 名前: Query executed to insert hourly averages
  - 繰り返しの頻度: 時間
  - 繰り返し感覚: 1
  - 設定した時刻に開始: true 
  - 開始日と実行時間: 2023/08/06 15:05 
  - 終了時刻を設定: true 
  - 終了日: 2023/08/06 18:05 
  - クエリ結果の書き込み先: SQL内で記述ずみなので設定不要

あとは時間が経過してからテーブルの中身を確認すると、問題なくインサートされている。

```sql
SELECT
  timestamp_1hour,
  avg_co2_ppm
FROM
  co2dataset.co2_summary
;
```
|timestamp_1hour|avg_co2_ppm|
|:----|:----|
|2023-08-06 13:00:00|666|
|2023-08-06 14:00:00|677|
|2023-08-06 15:00:00|762|

なんか検索するとBigQueryの時間関係はややこしいみたいなので、重点的におさらいしておく必要がありそう。

## :pencil2: Chapter5　Developing with　BigQuery

### INFORMATION SCHEMA
`dataset.INFORMATION_SCHEMA.name`には各種データセットに関する情報が保存されているので、例えば`ch04.INFORMATION_SCHEMA.TABLES`というテーブルを選択すれば、テーブル情報を引き出すことが出来る。

```
SELECT
  * EXCEPT(ddl)
FROM
  ch04.INFORMATION_SCHEMA.TABLES
```

|table_catalog|table_schema|table_name|table_type|is_insertable_into|is_typed|creation_time|base_table_catalog|base_table_schema|base_table_name|snapshot_time_ms|default_collation_name|upsert_stream_apply_watermark|
|:----|:----|:----|:----|:----|:----|:----|:----|:----|:----|:----|:----|:----|
|sqlsandbox376108|ch04|college_scorecard_etl|BASE TABLE|YES|NO|2023-08-05 19:01:40.791000 UTC| | | | NULL| |
|sqlsandbox376108|ch04|co2|EXTERNAL|NO|NO|2023-08-06 03:11:30.073000 UTC| | | | |NULL| |


### shell script
BigQuery の SQL クエリを実行して結果を取得したければ、HTTP POST リクエストを発行するシェルスクリプトを作成すればよい。`read -d '' QUERY_TEXT << EOF`は`QUERY_TEXT`という名前の環境変数にSQLを格納するためのヒアドキュメントを開始する。そして、`bq query --use_legacy_sql=false $QUERY_TEXT`の部分で、`bq query`コマンドを使用して、BigQueryに対してSQLを実行する。`--use_legacy_sql=false`フラグは、Standard SQL構文を使用することを示し、`$QUERY_TEXT`は環境変数`QUERY_TEXT`の中身を展開して実際のSQLとして使用する。

```
#!/bin/bash

read -d '' QUERY_TEXT << EOF
SELECT 
  start_station_name
  , AVG(duration) as duration
  , COUNT(duration) as num_trips
FROM \`bigquery-public-data\`.london_bicycles.cycle_hire
GROUP BY start_station_name 
ORDER BY num_trips DESC 
LIMIT 5
EOF

bq query --use_legacy_sql=false $QUERY_TEXT

```

実行すると結果が得られる。

```
$ ./bq_query.sh
+-------------------------------------+--------------------+-----------+
|         start_station_name          |      duration      | num_trips |
+-------------------------------------+--------------------+-----------+
| Hyde Park Corner, Hyde Park         | 2475.2221245805335 |    671389 |
| Belgrove Street , King's Cross      | 1040.4981000777511 |    592919 |
| Waterloo Station 3, Waterloo        |  896.5945201320634 |    527020 |
| Albert Gate, Hyde Park              | 2212.7423641803734 |    460887 |
| Black Lion Gate, Kensington Gardens |  2602.048261964297 |    459513 |
+-------------------------------------+--------------------+-----------+
```

クエリの結果を得るのではなく、クエリの情報を得るためにはHTTP POST リクエストを発行するシェルスクリプトを作成すればよい。

```sh
#!/bin/bash

PROJECT=$(gcloud config get-value project)

read -d '' QUERY_TEXT << EOF
SELECT 
  start_station_name
  , AVG(duration) as duration
  , COUNT(duration) as num_trips
FROM \`bigquery-public-data\`.london_bicycles.cycle_hire
GROUP BY start_station_name 
ORDER BY num_trips DESC 
LIMIT 5
EOF

read -d '' request << EOF
{
 "useLegacySql": false,
 "useQueryCache": true,
 "query": \"${QUERY_TEXT}\"
}
EOF

request=$(echo "$request" | tr '\n' ' ')
access_token=$(gcloud auth application-default print-access-token)

curl -H "Authorization: Bearer $access_token"  \
    -H "Content-Type: application/json" \
    -X POST \
    -d "$request" \
    "https://www.googleapis.com/bigquery/v2/projects/$PROJECT/queries"
```

実行結果はこちら。

```
$ ./rest_query.sh
{
  "kind": "bigquery#queryResponse",
  "schema": {
    "fields": [
      {
        "name": "start_station_name",
        "type": "STRING",
        "mode": "NULLABLE"
      },
      {
        "name": "duration",
        "type": "FLOAT",
        "mode": "NULLABLE"
      },
      {
        "name": "num_trips",
        "type": "INTEGER",
        "mode": "NULLABLE"
      }
    ]
  },
  "jobReference": {
    "projectId": "sqlsandbox376108",
    "jobId": "job_bufMknrao50bnwAYXYb283tb6EBe",
    "location": "EU"
  },
  "totalRows": "5",
  "rows": [
    {
      "f": [
        {
          "v": "Hyde Park Corner, Hyde Park"
        },
        {
          "v": "2475.222124580533"
        },
        {
          "v": "671389"
        }
      ]
    },
    (snip),
    {
      "f": [
        {
          "v": "Black Lion Gate, Kensington Gardens"
        },
        {
          "v": "2602.0482619642976"
        },
        {
          "v": "459513"
        }
      ]
    }
  ],
  "totalBytesProcessed": "3101263659",
  "jobComplete": true,
  "cacheHit": false
}
```

これ以降の内容はPythonからBigQueryを利用するための方法が色々と記載されていたので、今は必要ないのでスキップ。

## :pencil2: Chapter6　Architecture of　BigQuery
### Query 
BigQueryは、Google Cloudの各リージョンにまたがる複数のアベイラビリティゾーンにある、相互に関連する数十のマイクロサービスに数十万の実行タスクを持つ大規模な分散システム、この章ではBigQueryの仕組みを簡略化して解説される。まずはクエリの仕組みから。

- 1. HTTP POST: 
クライアントはBigQueryエンドポイントにPOSTを送る。curlでも送れるので、大抵のプログラミング言語から利用できる。BigQuery APIのドキュメントは[こちら](https://cloud.google.com/bigquery/docs/reference/rest/)
- 2. Routing: 
URLの一部にプロジェクト名があるのでそれをヒントにする。あとはプロジェクトのリージョンの制限なども考慮される。データセットは場所と紐づくので、BigQueryはデータセットの地域を調べてその地域にルーティングする。
- 3. Job Server: 
ジョブサーバは、呼び出し元がジョブを含むプロジェクトに課金されるクエリの実行を許可されていることを確認するために認証を行う。問題なければジョブサーバはリクエストを適切なクエリサーバに送信する。
- 4. Query engine: 
クエリ・マスターはクエリ全体の実行を担当する。クエリー・マスターはメタデータ・サーバーと連絡を取り、物理データがどこにあり、どのようにパーティショニングされているかを確認する。クエリサーバーがクエリのデータ量を把握し、クエリプランを作成。その後、クエリマスターはスケジューラーにスロット(クエリワーカーシャード上の実行スレッド)を要求する。
- 5. Returning the query results: 
結果はクエリのメタデータとともに分散リレーショナルデータベースであるSpannerに格納されるSpannerのデータは、クエリが実行されているのと同じ 領域 に配置される。残りのデータはColossusに書き込まれる。BigQuery APIは再接続できるように設計されている。ジョブサーバはタイムアウトする前にジョブIDをクライアントに返し、クライアントはそのジョブを検索して結果を得ることができる。BigQueryの結果は24時間保存され、機能的にはテーブルと同等であり、テーブルのようにクエリできる。

これ以降はQuery EngineのDremelの細かい話や、クエリの実行計画の詳細がまとめられている。

### Storage
BigQueryは数エクサバイトのデータを、数十のリージョンにある数百万の物理ディスクに分散して保存する。Colossusは、Google全体で使用されている分散ストレージシステムであり、大規模分散ストレージシステムGoogle File System（GFS）を進化させたもの。ディスクが壊れてもデータを失わないようにするためにレプリカを作成するとのこと。

ここらへんの詳しい話(erasure encoding、Storage formatのCapacitor)は書籍に書いてあるのと、よくわからないのでパスする。つカラムナー・ストアの話も少し触れられている。

### Partitioning

BigQueryでパーティショニングを行うと、大きな論理テーブルを小さなパーティションに分割し、必要な部分だけをクエリすることができる。テーブルを日付でパーティショニングしておけば、パーティショニングされたテーブルを使用して、特定の日時のデータのみを効率的に読み取れる。パーティションは基本的に軽量テーブルであり、パーティションのデータは他のパーティションとは物理的に別の場所に保存される。

パーティションの有効期限を設定すると、日付ベースのパーティションは、一定期間が経過すると期限切れとなって削除される。このように、パーティションは、複数のテーブルを効率的に管理できる利点がある。

クエリでも使用でき、このパーテーションの日付範囲をまたいでデータを取得できる、不要なスキャンを回避できる。パーティションは 、カーディナリティの低い、異なる値の数が少ないカラムで利用する。カーディナリティが高いのであれば、クラスタリングを使うべき。クラスタリングは例えば会員IDのようなカラムを利用することで、特定の会員ID群をまとめて管理することで効率を高める。

パーテーションはメタデータを持っており、指定されたパーテーションのメタデータを見れば、物理データにアクセスする必要もなく、スキャンするサイズがわかる。

## :pencil2: Chapter7　Optimizing　　Performance and Cost

### Principles of Performance

パフォーマンスチューニングは開発段階の最後に行うべきで、わずかなパフォーマンスの向上のために、テーブルスキーマやクエリを難解にするよりも、柔軟なテーブルスキーマと読みやすく保守性の高いクエリを持たせる方がはるかに優れている。

BigQueryの料金体系には大きく2つあり、1つはオンデマンド料金で、もう1つは定額制。オンデマンドの料金プランは、コストはクエリによって処理されるデータ量に比例するため、クエリが処理するデータ量を少なくすることがコスト削減につながる。WebUIから実行前のスキャンする量を見積もれるので目安として利用する。

`bq`コマンドラインクライアントの場合は`-dry_run`を指定する。`-maximum_bytes_billed`パラメータを指定して、クエリが処理できるデータ量に制限をかけることもできる。下記のクエリでコストの高いクエリを見つけることが出来る。

```sql
SELECT
job_id
, query
, user_email
, total_bytes_processed
, total_slot_ms
FROM `pj`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE EXTRACT(YEAR FROM creation_time) = 2023
ORDER BY total_bytes_processed DESC
LIMIT 5
```

### Measuring Query Speed
クエリの処理時間を計測したければAPI経由でクエリを発行することで計測は可能。計測するSQLを定義、キャッシュを無効化して、必要なコンフィグ情報とともにループを回して、`time`で計測する。あとは計測時間を繰り返し回数でわる。

```bash
#!/bin/bash

NUM_TIMES=3

read -d '' QUERY_TEXT << EOF
SELECT 
  start_station_name
  , AVG(duration) as duration
  , COUNT(duration) as num_trips
FROM \`bigquery-public-data\`.london_bicycles.cycle_hire
GROUP BY start_station_name 
ORDER BY num_trips DESC 
LIMIT 5
EOF

read -d '' request_nocache << EOF
{
 "useLegacySql": false,
 "useQueryCache": false,
 "query": \"${QUERY_TEXT}\"
}
EOF
request_nocache=$(echo "$request_nocache" | tr '\n' ' ')

access_token=$(gcloud auth application-default print-access-token)
PROJECT=$(gcloud config get-value project)
echo "Running query repeatedly; please divide reported times by $NUM_TIMES"

time for i in $(seq 1 $NUM_TIMES); do
echo -en "\r ... $i / $NUM_NUMTIMES ..."
curl --silent \
    -H "Authorization: Bearer $access_token"  \
    -H "Content-Type: application/json" \
    -X POST \
    -d "$request" \
    "https://www.googleapis.com/bigquery/v2/projects/$PROJECT/queries" > /dev/null
done
```

大体2秒ほどなので、平均実行時間は0.6秒くらいとわかる。

```
real	0m2.015s
user	0m0.060s
sys  	0m0.035s
```

他にもクエリを効率化するための情報として`jobid`を利用できる。

```bash
#!/bin/bash

JOBID=job_wiXp6taIipYjWHzGO1m2LC6awDCR 

access_token=$(gcloud auth application-default print-access-token)
PROJECT=$(gcloud config get-value project)

echo "$request"
curl --silent \
    -H "Authorization: Bearer $access_token"  \
    -X GET \
    "https://www.googleapis.com/bigquery/v2/projects/$PROJECT/jobs/${JOBID}"
```

結果には、データ待ち時間、クエリにおける計算に対するI/Oの割合、読み込みデータ、シャッフルされたデ
ータ、書き出されたデータに関する情報が含まれる。

```
"waitRatioAvg": 0.012448132780082987,
"waitMsAvg": "3",
"waitRatioMax": 0.012448132780082987,
"waitMsMax": "3",
"readRatioAvg": 0,
"readMsAvg": "0",
"readRatioMax": 0,
"readMsMax": "0",
"computeRatioAvg": 0.024896265560165973,
"computeMsAvg": "6",
"computeRatioMax": 0.024896265560165973,
"computeMsMax": "6",
"writeRatioAvg": 0.066390041493775934,
"writeMsAvg": "16",
"writeRatioMax": 0.066390041493775934,
"writeMsMax": "16",
"shuffleOutputBytes": "228",
"shuffleOutputBytesSpilled": "0",
"recordsRead": "50",
"recordsWritten": "5",
"parallelInputs": "1",
"completedParallelInputs": "1",
"status": "COMPLETE",
```

あとはWebUIからクエリプラン情報を可視化したものを見ることも出来る。

### Increasing Query Speed

クエリを効率化するための方法が実際のクエリ例とともに紹介されている。クエリ例は書籍にお任せして、ここでは意識する点を箇条書きでまとめておく。

- SELECTで目的意識を持つ: カラムナファイルフォーマットなので`*`は使わず、カラムを指定する
- 読み取りデータの削減: フィルタやグループ化には`hoge_id`ではなく`hoge_name`を利用する
- 高価な計算の回数を減らす: ジオ分析の距離計算は高価なので、最初から用意しておいて`join`の処理を避ける
- 以前のクエリの結果をキャッシュする: 空白の有り無しでキャッシュされるかが変わる
- 中間結果のキャッシュ: 一時テーブルやマテビューを活用し、I/Oの増加を犠牲にして全体的なパフォーマンスを向上させる
- BI Engineによるクエリの高速化: BIで頻繁にアクセスするテーブル、ダッシュボード、集計、フィルタなどがある場合、BI Engineを利用する
- 効率的なJOINの実行: `Join`は高価なので、非正規化を積極的に利用する
- 大きなテーブルの自己結合を避ける: ウインドウ関数を優先して利用する
- 結合するデータを減らす: 結合するのであれば、予め集計して小さくしておく
- 事前に計算された値での結合: 何度も、毎回計算するのではなく、最初に計算して、それを使い回す
- 大きなソートの制限: ソートはワーカーのメモリを圧迫する。小さくしてからソートする
- データ・スキュー: グループ化する際に、各グループでレコードサイズに歪みがあるとワーカーのメモリを圧迫するので、`LIMIT`を使う方法もある
- ユーザー定義関数の最適化: JavaScriptのUDFをサポートしているが使わない。SQLでUDFを記述する。
- 近似集約関数の使用: `COUNT(DISTINCT ...)`の代わりに、小さな不確実性が許容できるのであれ`APPROX_COUNT_DISTINCT`を使う。他にも、`APPROX_TOP_COUNT`や`APPROX_TOP_SUM`を使う
- count-distinct 問題: HyperLogLog++アルゴリズムをサポートしているので、`HLL_COUNT.MERGE`を利用する
- ネットワーク・オーバーヘッドの最小化: データセットは同じリージョンに配置するしてオーバーヘッドを減らす
- 圧縮されたレスポンス: REST APIを直接呼び出す場合、HTTPヘッダでgzipを指定する
- 複数のリクエストのバッチ処理: REST APIを直接呼び出す場合、multipart/mixed content typeを使用する
- 効率的なストレージフォーマットの選択: 外部データソースではなく、ネイティブテーブルを使用する
- ステージングバケットでのライフサイクル管理の設定: Google Cloud StorageにステージングしてBigQueryにロードする場合は、データのロード後にGoogle Cloud Storageのデータを削除する
- 構造体の配列としてデータを格納する: データを配列として格納すると、テーブルの行数は劇的に減少する
- 地理タイプとしてデータを保存する: 地理データを扱うのであれば`GEOGRAPHY`型を検討する
- パーティショニングされたテーブル: テーブルの保存はパーテーションを使用する
- テーブルのクラスタリング: カーディナリティが高いものはクラスタリングを使用する

クエリ例も豊富なので、個人的にはすごく役に立ったチャプターの1つ。


## :pencil2: Chapter8　Advanced Queries

BigQueryでサポートされる標準SQLのパーサとアナライザは、ZetaSQLとしてオープンソース化されている。

### Named parameters

pythonなどからクエリを発行する際に、SQLの内容をパラメタ化できる。下記の例は名前付きパラメタの例。他にも、タイムスタンプ・パラメタ、位置パラメタ、配列と構造体のパラメタなどが利用できる。

```sql
query = """
SELECT
start_station_name
, AVG(duration) as avg_duration
FROM
`bigquery-public-data`.london_bicycles.cycle_hire
WHERE
start_station_name LIKE CONCAT('%', @STATION, '%')
AND duration BETWEEN @MIN_DURATION AND @MAX_DURATION
GROUP BY start_station_name
"""
```

### SQL User-Defined Functions

BigQueryでは、ユーザーが定義した関数をUDF関数と呼ぶ。下記の例は、曜日名を返すUDF関数。

```sql
CREATE TEMPORARY FUNCTION dayOfWeek(x TIMESTAMP) AS
(
['Sun','Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']
[ORDINAL(EXTRACT(DAYOFWEEK from x))]
);
CREATE TEMPORARY FUNCTION getDate(x TIMESTAMP) AS
(
EXTRACT(DATE FROM x)
);
```

クエリ間で再利用したい関数がある場合、データセットに関数を保存し、任意の数のクエリから参照する方法が良い。`TEMPORARY`を外し、保存するデータセットを指定する。

```sql
CREATE OR REPLACE FUNCTION ch08eu.dayOfWeek(x TIMESTAMP) AS (
['日','月','火','水','木','金','土']
[ORDINAL(EXTRACT(DAYOFWEEK from x))] 。
);
```

パブリックUDFも存在しており、中央値の例がまさにパブリックUEFの一例。

```sql
CREATE OR REPLACE FUNCTION fhoffa.x.median (arr ANY TYPE) AS ((
SELECT IF (MOD(ARRAY_LENGTH(arr), 2) = 0,
( arr[OFFSET(DIV(ARRAY_LENGTH(arr), 2) - 1)] +
arr[OFFSET(DIV(ARRAY_LENGTH(arr), 2))] ) / 2,
arr[OFFSET(DIV(ARRAY_LENGTH(arr), 2))]
)
FROM (SELECT ARRAY_AGG(x ORDER BY x) AS arr FROM UNNEST(arr) AS x)
));

```

`fhoffa`データセットは公開されているので、誰でも利用可能である。

```
with the longest median duration of trips:
SELECT
start_station_name
, COUNT(*) AS num_trips
, fhoffa.x.median(ARRAY_AGG(tripduration)) AS typical_duration
FROM `bigquery-public-data`.new_york_citibike.citibike_trips
GROUP BY start_station_name
HAVING num_trips > 1000
ORDER BY typical_duration DESC
LIMIT 5
```

### Defining constants

`with`句の中に定数を定義することで、その後のクエリで予め設定した定数を利用できる。このようなクエリにしておけは、定数の変更があった際は簡単に修正できる。

```sql
WITH params AS (
SELECT 600 AS DURATION_THRESH
)
SELECT
start_station_name
, COUNT(duration) as num_trips
FROM
`bigquery-public-data`.london_bicycles.cycle_hire
, params
WHERE duration >= DURATION_THRESH
GROUP BY start_station_name
ORDER BY num_trips DESC
LIMIT 5
```

### Working with Arrays

BigQueryにおける配列は、同じデータ型の値を含む順序付きリストとして扱われる。このテーブルを保存して、後で他のクエリやBigQuery から分析することもできるが、テーブルから読み込んだ行は順序が保証されていない。

```sql
SELECT
  bike_id,
  COUNT(*) AS num_trips
FROM
  `bigquery-public-data`.london_bicycles.cycle_hire
GROUP BY
  bike_id
ORDER BY
  num_trips DESC
LIMIT
  5
```

|bike_id|num_trips|
|:----|:----|
|12942|8197|
|11077|8063|
|10601|7970|
|12779|7941|
|12926|7716|

順序を維持する必要性がある場合、配列に押し込むことで順序を維持できる。また、マテリアライズド・ビューを使用することでも解決できる。

```sql
WITH
  numtrips AS (
  SELECT
    bike_id AS id,
    COUNT(*) AS num_trips
  FROM
    `bigquery-public-data`.london_bicycles.cycle_hire
  GROUP BY
    bike_id ) 
SELECT
  ARRAY_AGG(STRUCT(id,
      num_trips)
  ORDER BY
    num_trips DESC
  LIMIT
    5) AS bike
FROM
  numtrips
```

### Using arrays for generating data

前にも書いたが、データを生成するときに配列は便利。

```sql
WITH days AS (
SELECT
GENERATE_DATE_ARRAY('2019-06-23', '2019-08-22', INTERVAL 10 DAY) AS summer
)
SELECT summer_day
FROM days, UNNEST(summer) AS summer_day
```

|summer_day|
|:----|
|2019-06-23|
|2019-07-03|
|2019-07-13|
|2019-07-23|
|2019-08-02|
|2019-08-12|
|2019-08-22|

2カラム以上作りたいときは、配列とオフセットを利用する。

```sql
WITH
  days AS (
  SELECT
    GENERATE_DATE_ARRAY('2019-06-23', '2019-08-22', INTERVAL 10 DAY) AS summer,
    ['Lak', 'Jordan', 'Graham'] AS minions )
SELECT
  summer[ORDINAL(dayno)] AS summer_day,
  minions[
OFFSET
  (MOD(dayno, ARRAY_LENGTH(minions)))] AS minion
FROM
  days,
  UNNEST(GENERATE_ARRAY(1,ARRAY_LENGTH(summer),1)) dayno
ORDER BY
  summer_day ASC
```

|summer_day|minion|
|:----|:----|
|2019-06-23|Jordan|
|2019-07-03|Graham|
|2019-07-13|Lak|
|2019-07-23|Jordan|
|2019-08-02|Graham|
|2019-08-12|Lak|
|2019-08-22|Jordan|

### Building queries dynamically

`INFORMATION_SCHEMA`を使用することで、データセット内のすべてのテーブルに関するメタデータを含んでいる。テーブルのカラム名を取り出したいときは下記のようにすればよい。

```sql
SELECT column_name
FROM `bigquery-public-data`.irs_990.INFORMATION_SCHEMA.COLUMNS
WHERE table_name = 'irs_990_2015'
```

|column_name|
|:----|
|ein|
|elf|
|tax_pd|
|subseccd|
|s501c3or4947a1cd|
|schdbind|
|(snip)|
|othrinc509|
|totsupp509|

カラム名をすべて`SELECT`文に入れたい場合、手作業やその他プログラムから作るのも手間がかかるが、このメタデータを使って動的にSQLを生成することもできる。

```sql
WITH columns AS (
SELECT column_name
FROM `bigquery-public-data`.irs_990.INFORMATION_SCHEMA.COLUMNS
WHERE table_name = 'irs_990_2015' AND column_name != 'ein'
)
SELECT CONCAT(
'SELECT ein, ARRAY_AGG(STRUCT(',
ARRAY_TO_STRING(ARRAY(SELECT column_name FROM columns), ',\n '),
'\n) FROM `bigquery-public-data`.irs_990.irs_990_2015\n',
'GROUP BY ein')
```
実際に生成されるSQLは下記の通り。

```sql
SELECT ein, ARRAY_AGG(STRUCT(elf,
 tax_pd,
 subseccd,
 s501c3or4947a1cd,
 schdbind,
 (snip)
 othrinc509,
 totsupp509
) FROM `bigquery-public-data`.irs_990.irs_990_2015
GROUP BY ein
```
### Time travel

タイムトラベルという機能があり、7日間までテーブルの履歴を参照できる。例えば、6時間前のテーブルを照会するには、`SYSTEM_TIME`を使用すればよい。

```sql
SELECT
  *
FROM
  `bigquery-public-data`.london_bicycles.cycle_stations FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 6 HOUR)
```

### Data Definition Language and Data Manipulation Language

新しいテーブルを作成したければCTASを利用すればよいが、オプションも同時に設定でき、有効期限やタイムスタンプ、ラベルなどが設定できる。

```sql
CREATE OR REPLACE TABLE
  ch08eu.hydepark_stations OPTIONS( expiration_timestamp=TIMESTAMP "2020-01-01 00:00:00 UTC",
    description="Stations with Hyde Park in the name",
    labels=[("cost_center", "abc123")] ) AS
SELECT
  * EXCEPT(longitude,
    latitude),
  ST_GEOGPOINT(longitude, latitude) AS location
FROM
  `bigquery-public-data.london_bicycles.cycle_stations`
WHERE
  name LIKE '%Hyde%'
```

変更する場合は`ALTER TABLE`で変更できる。

```sql
ALTER TABLE ch08eu.hydepark_rides
SET OPTIONS(
expiration_timestamp=TIMESTAMP「2021-01-01 00:00:00 UTC」、require_partition_filter=True、
labels=[("cost_center", "def456")]。
)
```

もうすでに利用しているが、`INSERT`など基本的なDMLは利用できる。

```sql
INSERT ch08eu.hydepark_rides
SELECT
start_date AS start_time
  , duration
  , start_station_id
  , start_station_name
  , end_station_id
  , end_station_name
FROM
  `bigquery-public-data`.london_bicycles.cycle_hire
WHERE
  start_station_name LIKE '%Hyde%'
```

### MERGE statement

`MERGE`文は、`INSERT`、`UPDATE`、`DELETE`操作をアトミックに組み合わせたもので、1つの文として実行される。ソーステーブルからのレコードがターゲットテーブルに挿入され、各行に対して一連の操作が実行される。`MATCHED`、`NOT MATCHED BY TARGET`、`NOT MATCHED BY SOURCE`の3つのシナリオで、異なる操作を定義することが可能。読んでもわかりにくいので、見たほうがはやい。

```sql
MERGE ch08eu.hydepark_stations AS T
USING
  (SELECT *
  FROM `bigquery-public-data`.london_bicycles.cycle_stations
  WHERE name LIKE '%Hyde%') AS S
ON T.id = S.id
WHEN MATCHED THEN
  UPDATE
    SET bikes_count = S.bikes_count
WHEN NOT MATCHED BY TARGET THEN
  INSERT(id, installed, locked, name, bikes_count)
  VALUES(id, installed, locked, name, bikes_count)
WHEN NOT MATCHED BY SOURCE THEN
  DELETE
```

このSQLは、`ch08eu.hydepark_stations` テーブルを対象として指定している。そのため、このテーブルにデータが統合されるか、更新、挿入、削除されることになる。そして、サブクエリを使用して、ソーステーブル（`bigquery-public-data.london_bicycles.cycle_stations`）から条件を満たすデータを取得し、エイリアス `S` で参照します。ここでは、`name` 列に `'Hyde'` という文字列を含むデータを取得している。

ターゲットテーブル（`ch08eu.hydepark_stations`）とソーステーブル（`S`）のデータを、`id` 列をキーとして結合し、対応するデータをマッチングさせる。マッチングが成功した場合、つまりターゲットテーブルとソーステーブルの結合に成功した場合、ターゲットテーブルの該当する行のデータをソーステーブルのデータで更新。ここでは、`bikes_count` 列のデータを `S.bikes_count` の値で更新。ターゲットテーブルに存在しないソーステーブルのデータがある場合、新しい行として挿入する。つまり、「ターゲットテーブルにはないがソーステーブルにあるデータ」が対応する。そして、ソーステーブルに存在しないターゲットテーブルのデータがある場合、それを削除します。これにより、ターゲットテーブルのデータが不要である場合にそれを削除できる。

### Hash Algorithms

BigQueryは一般的なハッシュアルゴリズムもサポートしている。フィンガープリントアルゴリズムやMD5とSHAも利用可能。

```sql
WITH identifier AS (
SELECT
CONCAT(
CAST(bike_id AS STRING), '***',
CAST(start_date AS STRING), '***',
CAST(start_station_id AS STRING)
) AS rowid
FROM `bigquery-public-data.london_bicycles.cycle_hire`
LIMIT 10
)
SELECT
rowid, FARM_FINGERPRINT(rowid) AS fingerprint
FROM identifier
;

SELECT
name
, MD5(name) AS md5_
, SHA256(name) AS sha256_
, SHA512(name) AS sha512_
FROM UNNEST(['Joe Customer', 'Jane Employee']) AS name
```

## :pencil2: Chapter9 

BigQueryのBigQueryMLは一旦パス。必要なときに見返してメモする。

## :closed_book: Reference

- [Google BigQuery: The Definitive Guide](https://www.oreilly.com/library/view/google-bigquery-the/9781492044451/)
- [bigquery-oreilly-book](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book)