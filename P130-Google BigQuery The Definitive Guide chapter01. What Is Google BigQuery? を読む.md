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
|2023-08-06 14:00:30.000000 UTC|680|
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
|2023-08-06 13:04:13.000000 UTC|636|
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
|2023-08-06 13:00:00|678.72|


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
|2023-08-06 13:00:00|678.72|
|2023-08-06 14:00:00|672.0|

なんか検索するとBigQueryの時間関係はややこしいみたいなので、重点的におさらいしておく必要がありそう。


## :pencil2: Chapter5　Developing with　BigQuery

## :pencil2: Chapter6

## :pencil2: Chapter7

## :pencil2: Chapter8

## :pencil2: Chapter9

## :closed_book: Reference

- [Google BigQuery: The Definitive Guide](https://www.oreilly.com/library/view/google-bigquery-the/9781492044451/)
- [bigquery-oreilly-book](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book)