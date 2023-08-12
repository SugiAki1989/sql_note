## :memo: Overview

ここでは BigQuery 用のコマンドラインツール(特に`bq query`コマンド)の使い方をまとめておく。コマンドラインからクエリを実行できると何かと便利なことが多いのと、これまでドキュメントを読んだことがなく、雰囲気で利用していたので、遅ればせながらまとめておく。そして、BigQueryはほとんど仕事では縁がない。

- [bq コマンドラインツールの使用](https://cloud.google.com/bigquery/docs/bq-command-line-tool?hl=ja)

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`bq`

## :pencil2: gcloud CLIのセットアップ

コマンドラインからGCPを利用するためには、セットアップが必要になるので、下記の公式の手順に従ってセットアップを行う。

- [gcloud CLI をインストールする](https://cloud.google.com/sdk/docs/install?hl=ja)

```sql
$ gcloud version

Google Cloud SDK 442.0.0
bq 2.0.96
core 2023.08.04
gcloud-crc32c 1.0.0
gsutil 5.25
```

インストールが終わったら、コンフィグ情報を設定するために `gcloud init` コマンドを実行する。認証情報の名前、アカウント、プロジェクトなどを指示に従って入力する。

```sql
$ gcloud init
Welcome! This command will take you through the configuration of gcloud.

(snip)
```

`gcloud config list`コマンドで現在のコンフィグ情報を確認できる。

```sql
$ gcloud config list
[core]
account = your.email@gmail.com
disable_usage_reporting = True
project = YOUR-PROJECT

Your active configuration is: [your-config-name]
```

## :pencil2: bqコマンドの基礎

`bq show`コマンドでプロジェクトを確認でき、`bq ls`コマンドでデータセットを確認できる。

```sql
$ bq show
Project sqlsandbox12345

  friendlyName
 --------------
  sqlsandbox

$ bq ls
  datasetId
 -----------
  ch04

$ bq ls ch04
         tableId            Type     Labels   Time Partitioning   Clustered Fields
 ----------------------- ---------- -------- ------------------- ------------------
  co2                     EXTERNAL
  college_scorecard_etl   TABLE
```

`--dataset`オプションをつけることで、データセットのテーブルの情報を確認できる。プロジェクトの名前ではなく、IDでないとだめ。

```sql
$ bq show --dataset {projectj-id}:ch04
Dataset {projectj-id}:ch04

   Last modified               ACLs               Labels    Type
 ----------------- ----------------------------- -------- ---------
  06 Aug 03:21:33   Owners:                                DEFAULT
                      projectOwners,
                      your.email@gmail.com
                    Writers:
                      projectWriters
                    Readers:
                      projectReaders
```

`schema`オプションでデータセット内のテーブルを指定すれば、スキーマの状態を確認できる。

```sql
-- $ bq show --schema ch04.college_scorecard_etl でもよい
$ bq show --schema {projectj-id}:ch04.college_scorecard_etl
[{"name":"INSTNM","type":"STRING"},{"name":"ADM_RATE_ALL","type":"FLOAT"},{"name":"FIRST_GEN","type":"FLOAT"},{"name":"MD_FAMINC","type":"FLOAT"},{"name":"SAT_AVG","type":"FLOAT"},{"name":"MD_EARN_WNE_P10","type":"STRING"}]
```

`format`オプションで、出力方法を制御できる。`pretty`、`csv`、`sparse`、`prettyjson`、`josn`などを指定できる。


```sql
$ bq show --schema --format prettyjson ch04.college_scorecard_etl
[
  {
    "name": "INSTNM",
    "type": "STRING"
  },
  {
    "name": "ADM_RATE_ALL",
    "type": "FLOAT"
  },
  {
    "name": "FIRST_GEN",
    "type": "FLOAT"
  },
  {
    "name": "MD_FAMINC",
    "type": "FLOAT"
  },
  {
    "name": "SAT_AVG",
    "type": "FLOAT"
  },
  {
    "name": "MD_EARN_WNE_P10",
    "type": "STRING"
  }
]
```

データセットやテーブルを確認するコマンドをさらっとおさらいしたので、ここからはクエリ発行するコマンドを見ていく。`bq query`コマンドの構文は下記の通り。

```sql
bq query [FLAGS] 'QUERY'
```
`bq query`コマンドで、BigQueryにクエリを発行できるのだが、ここで使用している`ch04.co2`はGoogleDriveのSpreadSheetであり、外部テーブルなので、認証エラーが出る。

```sql
$ bq query --use_legacy_sql=false "SELECT time, co2_ppm FROM ch04.co2 LIMIT 5"
BigQuery: Permission denied while getting Drive credentials.
```

`gcloud auth login --enable-gdrive-access`コマンドで、GoogleDriveへの認証を確立できる。

```sql
$ gcloud auth login --enable-gdrive-access
```

これで実行すればOK。

```sql
$ bq query --use_legacy_sql=false "SELECT time, co2_ppm FROM ch04.co2 LIMIT 5"
Waiting on bqjob_r290c4abff974d384_00000189e94b5615_1 ... (0s) Current status: DONE
+---------------------+---------+
|        time         | co2_ppm |
+---------------------+---------+
| 2023-06-24 10:44:53 |     813 |
| 2023-06-24 10:50:00 |     849 |
| 2023-06-24 10:55:04 |     809 |
| 2023-06-24 11:00:12 |     739 |
| 2023-06-24 11:05:20 |     672 |
+---------------------+---------+
```

`-dry_run`オプションで、クエリは実行せず、評価だけを行って、スキャン量を確認できる。他にもたくさんのオプションがあるので、必要なときは公式ドキュメントを参照する。

- [bqコマンドライン ツール リファレンス](https://cloud.google.com/bigquery/docs/reference/bq-cli-reference?hl=ja)

```sql
$ bq query --dry_run --use_legacy_sql=false \
  'SELECT * FROM `sqlsandbox12345.ch04.college_scorecard_etl` '
Query successfully validated. Assuming the tables are not modified, running this query will process 401597 bytes of data.
```

分析の際に使用できそうなものをピックアップしておく。実際に使えるかは知らない…仕事でBigQueryは縁がないので。

|オプション|内容|
|:---|:---|
|`--append_table={true|false}`|宛先テーブルにデータを追加するときは`true`|
|`--destination_table=TABLE`|クエリ結果が TABLE に保存される。TABLE は `PROJECT:DATASET.TABLE` の形式で指定。|
|`--dry_run={true|false}`|クエリは検証されるが、実行されない。|
|`--label=KEY:VALUE`|クエリジョブのラベル。|
|`--max_rows=MAX_ROWS`|クエリ結果で返す行数。|
|`--maximum_bytes_billed=MAX_BYTES`|クエリに対して課金されるバイト数を制限する。単位はバイトなので注意。30MBは30000000と記載。|
|`--parameter={PATH_TO_FILE|PARAMETER}`|パラメタリストを含む JSON ファイルか、`NAME:TYPE:VALUE` 形式のパラメータを指定|
|`--replace={true|false}`|クエリ結果で宛先テーブルを上書きするかどうか。|
|`--require_cache={true|false}`|キャッシュから結果を取得できる場合にのみ、クエリが実行される。|
|`--schedule="SCHEDULE"`|定期的にスケジュールされたものにする。`--schedule="every 24 hours"`など。|

`bq query`コマンドのオプションを複数使う場合はバックスラッシュで改行できる。これは新たにテーブルを作成する際のコマンド例。

```sql
bq query \
--use_legacy_sql=false \
--label dummy_key1:value1 \
--label dummy_key2:value2 \
--batch=false \
--maximum_bytes_billed=30000000 \
--require_cache=false \
--destination_table={projectid}.{dataset}.{table} \
--destination_schema a:string,b:string,c:integer,e:integer,f:integer \
--time_partitioning_field \
--clustering_fields=name \
--time_partitioning_expiration=90000 \
SELECT a,b,c,d,e,f FROM `{projectid}.{dataset}.{table}`
```

簡略化してレコードを追加してみる。まずはテーブルを`-destination_table`、`--destination_schema`を利用して作成して、初期データを追加する。

```sql
bq query \
--use_legacy_sql=false \
--destination_table=sqlsandbox12345:ch04.co2_cli \
--destination_schema time:TIMESTAMP,co2_ppm:integer \
SELECT time, co2_ppm FROM sqlsandbox12345.ch04.co2 ORDER BY time ASC LIMIT 5
```

この段階の中身はこんな感じ。

```sql
$ bq query --use_legacy_sql=false 'SELECT * FROM sqlsandbox12345.ch04.co2_cli'
+---------------------+---------+
|        time         | co2_ppm |
+---------------------+---------+
| 2023-06-24 10:44:53 |     813 |
| 2023-06-24 10:50:00 |     849 |
| 2023-06-24 10:55:04 |     809 |
| 2023-06-24 11:00:12 |     739 |
| 2023-06-24 11:05:20 |     672 |
+---------------------+---------+
```

このテーブルに対して、`append_table`を使って、さらにレコードを追加する。

```sql
bq query \
--use_legacy_sql=false \
--destination_table=sqlsandbox12345:ch04.co2_cli \
--destination_schema time:TIMESTAMP,co2_ppm:integer \
--append_table=TRUE \
SELECT time, co2_ppm FROM sqlsandbox12345.ch04.co2 ORDER BY time DESC LIMIT 5
```

この段階の中身はこんな感じになっており、レコードが追加されている。

```sql
$ bq query --use_legacy_sql=false 'SELECT * FROM sqlsandbox12345.ch04.co2_cli'
+---------------------+---------+
|        time         | co2_ppm |
+---------------------+---------+
| 2023-06-24 10:44:53 |     813 |
| 2023-06-24 10:50:00 |     849 |
| 2023-06-24 10:55:04 |     809 |
| 2023-06-24 11:00:12 |     739 |
| 2023-06-24 11:05:20 |     672 |
| 2023-08-12 21:51:46 |    1132 |
| 2023-08-12 21:46:42 |    1133 |
| 2023-08-12 21:41:38 |    1105 |
| 2023-08-12 21:36:33 |    1123 |
| 2023-08-12 21:31:29 |    1081 |
+---------------------+---------+
```

実際のところ、アドホックで分析する際に要件に応じたテーブルを作る(SQLでforloopみたいなことをするのは面倒なので、シェルスクリプトと組み合わせてデータを作るみたいな)必要があれば、使い道はあるのかもしれないが、定期的にデータを追加するとかであればUIから設定するか、embulk、digdag、dbtなどを利用するほうがよさそう。

話は変わって、ここまで見てきたようなシンプルなSQLであれば良いのだが、分析用の長い長い長いSQLになると直書きは扱いづらい。このようなケースではシェルスクリプトに書いてしまえば、楽に扱える。

```sql
$cat bq_query 

#!/bin/bash

read -d '' QUERY_TEXT << EOF
SELECT time, co2_ppm 
FROM ch04.co2 
LIMIT 5
EOF

bq query \
  --use_legacy_sql=false  \
  -maximum_bytes_billed=30000000 \
  $QUERY_TEXT
```

実行権限を渡して、

```sql
$ chmod 755 ./bq_query.sh
```

実行すればOK。

```sql
$ ./bq_query.sh
Waiting on bqjob_r799cbfb2470b30ac_00000189e9a19855_1 ... (0s) Current status: DONE
+---------------------+---------+
|        time         | co2_ppm |
+---------------------+---------+
| 2023-06-24 10:44:53 |     813 |
| 2023-06-24 10:50:00 |     849 |
| 2023-06-24 10:55:04 |     809 |
| 2023-06-24 11:00:12 |     739 |
| 2023-06-24 11:05:20 |     672 |
+---------------------+---------+
```

この方法以外でも、`.sql`ファイルにSQLを書いて、リダイレクトで`bq query`コマンドに渡す方法でも実行できる。必要はないかもしれないが、`.sql`ファイルを表示させて実行することも可能。

```sql
$ bq query --use_legacy_sql=false < bq.sql
-----------------------------
$ QUERY_FILE="bq.sql"
$ bq query --use_legacy_sql=false "$(cat $QUERY_FILE)"
```

個人的には、オプションも含めてファイルで管理できるのでシェルスクリプトで実行するのが楽。最後におまけ程度に`bq load`コマンドの使い方をみておく。このコマンドはテーブルにデータをロードできるコマンド。

- [bq load](https://cloud.google.com/bigquery/docs/reference/bq-cli-reference?hl=ja#bq_load)

構文はこちら。

```sql
bq load [FLAGS] DESTINATION_TABLE SOURCE_DATA [SCHEMA]
```

下記のサンプルcsvをロードしてみる。

```sql
$ cat /Users/aki/Desktop/sample.csv
id,name,age
1,'tanaka',20
2,'suzuki',34
3,'sato',23
4,'saito',45
5,'bob',50

$ bq load --source_format=CSV --replace=true --skip_leading_rows=1 ch04.name_mst /Users/aki/Desktop/sample.csv id:integer,name:string,age:integer 

Upload complete.
Waiting on bqjob_r11ac5fa5d3fedcb8_00000189e98d13b5_1 ... (0s) Current status: DONE
```

問題なくロードできた模様。

```sql
$ bq query --use_legacy_sql=false 'SELECT * FROM `sqlsandbox12345.ch04.name_mst`'

+----+----------+-----+
| id |   name   | age |
+----+----------+-----+
|  1 | 'tanaka' |  20 |
|  2 | 'suzuki' |  34 |
|  3 | 'sato'   |  23 |
|  4 | 'saito'  |  45 |
|  5 | 'bob'    |  50 |
+----+----------+-----+
```

## :closed_book: Reference

- [bq コマンドラインツールの使用](https://cloud.google.com/bigquery/docs/bq-command-line-tool?hl=ja)
- [bqコマンドライン ツール リファレンス](https://cloud.google.com/bigquery/docs/reference/bq-cli-reference?hl=ja)
