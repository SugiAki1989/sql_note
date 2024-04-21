## :memo: Overview

以前、ブログでAthenaに関してまとめていたものをそのままこっちに引っ越したした。数年前に書いたブログの内容になるので、今では少し古いかもしれない。


## :floppy_disk: Database

Athena

## :bookmark: Tag

`athena`

## :pencil2: 01. Athenaで1stクエリを発行するまでの設定方法
 
この記事はAmazon Athenaの自分用の備忘録です。下記の料金アラート関連は忘れずに。

- [https://qiita.com/kono-hiroki/items/18eabffd3edd325d4924:title]

### Athenaとは
[Amazon Athena](https://docs.aws.amazon.com/ja_jp/athena/latest/ug/what-is.html)の特徴は下記の通り。

- Amazon S3のデータに直接クエリを発行できるサービス
- ストレージとコンピューティングリソースが分離され、データをAthenaは保存しない
- 標準SQLが使用可能
- サーバーレスであり、インフラ設定や管理は不要
- 実行したクエリにのみ課金[(公式ドキュメント)](https://aws.amazon.com/jp/athena/pricing/?nc=sn&loc=3)
- エンジンにPrestoが利用され、自動的にスケールして並列実行
- 大規模データや複雑なクエリでも素早く分析可能

### 課金体系
一番気になる料金。「スキャンされたデータ1TBあたり5ドル」という説明があるが、これだけではよくわからないの公式ドキュメントの内容をおさらい。

### Athenaから発生する料金

- バイト数はメガバイト単位で切り上げられ、10MB未満のクエリは10MBとなる
- `CREATE TABLE`、`ALTER TABLE`、`DROP TABLE` などのDDLステートメント、パーティション管理用ステートメントに対しては課金されない
- 正常に実行されなかったクエリに対しては課金されないが、「キャンセルされたクエリ」は、スキャン量に基づいて課金される
- データを圧縮すると、スキャンされるデータ量を減るのでコストカットできる。例えばParquet形式を利用する。
- すべてのデータソースにわたって集計した場合(横串検索)、スキャン量に対して課金。

### その他のサービスから発生する料金

- AthenaはS3から直接データをクエリ処理するが、それはS3から発生する費用という扱い
- ストレージ、リクエスト、データ転送に対してS3の標準料金が発生
- デフォルトでは、クエリ結果は指定したS3バケットに保存され、S3の標準料金が発生
- Glueデータカタログを使用する場合は、標準のGlue データカタログのレートで課金。
- 他にも、Lambda、SageMakerなど、Athenaで使用するAWSのサービスの料金が発生

### 料金イメージ
[公式ドキュメント](https://aws.amazon.com/jp/athena/pricing/?nc=sn&loc=3#:~:text=%E5%8F%82%E7%85%A7%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82-,%E6%96%99%E9%87%91%E3%81%AE%E4%BE%8B,-%E3%82%B5%E3%82%A4%E3%82%BA%E3%81%8C%E5%90%8C%E3%81%98)に料金の例が記載されていたのでおさらいします。ここではサイズが同じ3つの列があるデータに対して、形式の違いによる料金の変化が書かれています。

はじめはテキストデータの例。S3での合計サイズが3TBになるテキストテーブルがある1列からデータを取得するクエリを実行する場合、テキスト形式は分割できないため、ファイルへのフルスキャンが発生し、15ドル(3TB*5ドル/TB)かかる。

次に、GZIPファイルの例。圧縮ファイルが1TBになったとしても、フルスキャンが発生するため、5ドル(1TB*5ドル/TB)かかることになる。

最後に、Parquet形式の例。ファイルが1TBになったとし、列指向ストレージ形式であるため、1列のみのスキャンになります。そのため、1.67ドル(0.33TB*5ドル/TB)かかることになる。

|形式|サイズ|スキャン|課金料金|
|:---|:---|:---|:---|
|テキスト|3TB|フルスキャン|15ドル|
|GZIP|1TB|フルスキャン|5ドル|
|Parquet|1TB|関連カラムのみスキャン|1.67ドル|

このようにデータの形式によっては、スキャン量が異なるので、無駄な費用を発生させないためにも、Parquet形式を利用することが必要。あとは知らぬ間にAthena以外のサービスでも課金が発生するので注意。下記の公式のドキュメントがわかりやすい。

- [Amazon Athena のパフォーマンスチューニング Tips トップ 10](https://aws.amazon.com/jp/blogs/news/top-10-performance-tuning-tips-for-amazon-athena/)

### Athenaの分析準備
### サンプルデータの準備
Athenaでクエリを発行するために準備していく。ここでは、ニューヨークの黄色タクシーの乗降データを利用。このデータには、乗車と降車の日時、乗降場所、移動距離、運賃の明細、料金の種類、支払いの種類、運転手から報告された乗客数などの情報が含まれる。とりあえず2021年1月の`yellow_tripdata_2021-01.csv`をダウンロードしてデスクトップに保存。Athenaで利用する際は、このCSVをGZIP形式に変換してから利用する。

- [TLC Trip Record Data](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

```
# 下記でも可
# curl -OL https://s3.amazonaws.com/nyc-tlc/trip+data/yellow_tripdata_2021-01.csv
➜ wget https://s3.amazonaws.com/nyc-tlc/trip+data/yellow_tripdata_2021-01.csv
➜ gzip yellow_tripdata_2021-01.csv 
➜ ls
yellow_tripdata_2021-01.csv.gz
```

|Name|Description|
|:---|:---|
|VendorID|レコードを提供したLPEPプロバイダーコード。1=CMT、2=VF|
|tpep_pickup_datetime|メーターが作動し始めた日時|
|tpep_dropoff_datetime|メーターが解除された日時|
|passenger_count|乗車人数。 これは運転手が入力した値。|
|trip_distance|タクシーメーターによって報告された走行距離 (マイル単位)|
|RatecodeID|乗車終了時に適用される最終的な料金コード。1=標準料金、2=JFK、3=Newark、4=NassauまたはWestchester、5=ネゴシエート料金、6=グループ乗車|
|store_and_fwd_flag|車両がサーバーに接続されていないため、乗車記録がベンダーに送信される前に車両のメモリに保持されていたどうかのフラグ。Y=ストアアンドフォワード乗車、N=ストアアンドフォワード乗車ではない|
|PULocationID|タクシーメーターが作動し始めたTLCタクシーゾーン|
|DOLocationID|タクシーメーターが解除されたTLCタクシーゾーン|
|payment_type|乗客が乗車料金を支払ったかを示すコード。1=クレジットカード、2=現金、3=無料、4=争議、5=不明、6=無効な乗車|
|fare_amount|メーターによって計算された時間距離併用運賃|
|extra|その他の割増料金と追加料金。 現在、これには 0.50 ドルおよび 1 ドルのラッシュアワー料金と夜間料金のみが含まれます|
|mta_tax|使用中のメーター制料金に基づいて自動的にトリガーされる0.50ドルのMTA税|
|tip_amount|チップの金額。クレジットカードのチップの場合に自動的に入力されます。現金のチップは含まれません|
|tolls_amount|乗車中に支払われたすべての通行料金の合計金額|
|improvement_surcharge|初乗り運賃での乗車に課される0.30ドルの改善追加料金。改善追加料金は2015年に徴収が開始されました|
|total_amount|乗客に請求される合計金額。 現金のチップは含まれません|
|congestion_surcharge|市が課した時間、交通料金に関連する課徴金|

```
➜ cat yellow_tripdata_2021-01.csv | head -5
VendorID,tpep_pickup_datetime,tpep_dropoff_datetime,passenger_count,trip_distance,RatecodeID,store_and_fwd_flag,PULocationID,DOLocationID,payment_type,fare_amount,extra,mta_tax,tip_amount,tolls_amount,improvement_surcharge,total_amount,congestion_surcharge
1,2021-01-01 00:30:10,2021-01-01 00:36:12,1,2.10,1,N,142,43,2,8,3,0.5,0,0,0.3,11.8,2.5
1,2021-01-01 00:51:20,2021-01-01 00:52:19,1,.20,1,N,238,151,2,3,0.5,0.5,0,0,0.3,4.3,0
1,2021-01-01 00:43:30,2021-01-01 01:11:06,1,14.70,1,N,132,165,1,42,0.5,0.5,8.65,0,0.3,51.95,0
1,2021-01-01 00:15:48,2021-01-01 00:31:01,0,10.60,1,N,138,132,1,29,0.5,0.5,6.05,0,0.3,36.35,0
```

### CLIの準備
AWS CLIを利用するので、コンフィグ情報はAWSのコンソールにアクセスし、IAMのユーザーの「認証情報」から取得。

```
➜ curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
➜ sudo installer -pkg AWSCLIV2.pkg -target /

➜ aws --version
aws-cli/2.4.2 Python/3.8.8 Darwin/20.2.0 exe/x86_64 prompt/off

➜ aws configure
AWS Access Key ID [None]: ***************************
AWS Secret Access Key [None]: ***************************************************
Default region name [None]: ap-northeast-1
Default output format [None]: (空のままクリック)
```

### S3バケットの準備
S3にバケットを作成。バケット名は`<YOUR S3 BUCKET NAME>`。S3には、`tables/nyc_taxi`というディレクトリにアップロード。Athenaのクエリのエクスポート先は`s3://<YOUR S3 BUCKET NAME>/results/`に設定。

```
# S3バケットを作成
➜ aws s3api create-bucket \
--bucket <YOUR S3 BUCKET NAME> \
--region ap-northeast-1 \
--create-bucket-configuration LocationConstraint=ap-northeast-1

{
    "Location": "http://<YOUR S3 BUCKET NAME>.s3.amazonaws.com/"
}

# S3バケットの確認
➜ aws s3 ls
2021-11-27 17:23:10 <YOUR S3 BUCKET NAME>

# S3にアップロード
➜ aws s3 cp ~/Desktop/yellow_tripdata_2021-01.csv.gz \
s3://<YOUR S3 BUCKET NAME>/tables/nyc_taxi/yellow_tripdata_2021-01.csv.gz
upload: ./yellow_tripdata_2021-01.csv.gz to s3://<YOUR S3 BUCKET NAME>/tables/nyc_taxi/yellow_tripdata_2021-01.csv.gz

# 確認
➜ aws s3 ls s3://<YOUR S3 BUCKET NAME> --recursive
2021-11-27 17:26:58   25045045 tables/nyc_taxi/yellow_tripdata_2021-01.csv.gz
```

### Glueにメタデータ登録
AthenaからSQLを書いてクエリを発行する、データはS3にあるから、S3の設定が必要なのはわかるけど…Glueとは何か。色々と情報を探していると、Glueはデータを管理するマネージドサービスで、その中のGlueデータカタログは、データのロケーション、スキーマ情報などのメタ情報を保存してくれるサービスとのこと。下記の画像のようにクローラーを走らせて、S3やRDSのメタ情報を取得するとのこと。

![meta](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-001.png)


メタ情報は1つのテーブル単位で保存されていくようで、`s3://<YOUR S3 BUCKET NAME>/tables/nyc_taxi`という単位で作成される。AthenaはS3にSQLを発行する際に、Glueデータカタログを参照し、スキーマーなどを利用。

```
# メタデータを作成
➜ aws glue create-database \
--database-input "{\"Name\":\"<YOUR S3 BUCKET NAME>\"}" \
--region ap-northeast-1

# メタデータの確認
➜ aws glue get-database --name <YOUR S3 BUCKET NAME>
```

### Athenaでテーブル作成
Athenaの画面に移動し、`CREATE EXTERNAL TABLE`でテーブルを作成。

```
CREATE EXTERNAL TABLE `<YOUR S3 BUCKET NAME>`.`nyc_taxi`(
  `VendorID` bigint, 
  `tpep_pickup_datetime` string, 
  `tpep_dropoff_datetime` string, 
  `passnger_count` bigint, 
  `trip_distance` double, 
  `RatecodeID` bigint, 
  `store_and_fwd_flag` string, 
  `PULocationID` bigint, 
  `DOLocationID` bigint, 
  `payment_type` bigint, 
  `fare_amount` double, 
  `extra` double, 
  `mta_tax` double, 
  `tip_amount` double, 
  `tolls_amount` double, 
  `improvement_surcharge` double, 
  `total_amount` double, 
  `congestion_surcharge` double)
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://<YOUR S3 BUCKET NAME>/tables/nyc_taxi/'
TBLPROPERTIES (
  'areColumnsQuoted'='false', 
  'columnsOrdered'='true', 
  'compressionType'='gzip', 
  'delimiter'=',',
  'skip.header.line.count'='1', 
  'typeOfData'='file')
```

これでAthenaでSQLでクエリを発行してデータを分析する準備が完了。

### Athenaの分析準備
試しにデータをセレクトしてみると、問題なく実行できていそう。

```
select * from nyc_taxi limit 10;
```

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-002.png)


Athenaの初期設定で指定したS3のロケーションに、Athenaで発行したSQLの結果がCSVで保存されていることも確認できる。

```
➜ aws s3 ls <YOUR S3 BUCKET NAME>/results/ --recursive
2021-11-29 20:14:55       1613 results/Unsaved/2021/11/29/6f28f9ea-cd84-429e-b2cb-aea938073431.csv
2021-11-29 20:14:55       1024 results/Unsaved/2021/11/29/6f28f9ea-cd84-429e-b2cb-aea938073431.csv.metadata
2021-12-04 15:00:11       1613 results/Unsaved/2021/12/04/ded08b5c-832e-4d5c-9807-b569d26f6b2b.csv
2021-12-04 15:00:11       1024 results/Unsaved/2021/12/04/ded08b5c-832e-4d5c-9807-b569d26f6b2b.csv.metadata
2021-12-04 15:01:31       1613 results/Unsaved/2021/12/04/f8024d48-bc50-4604-bcfa-452ea1b1cf1b.csv
2021-12-04 15:01:31       1024 results/Unsaved/2021/12/04/f8024d48-bc50-4604-bcfa-452ea1b1cf1b.csv.metadata
```

SQLの実行結果の情報も取得できる。

```
➜ aws athena get-query-execution --query-execution-id f8024d48-bc50-4604-bcfa-452ea1b1cf1b

{
    "QueryExecution": {
        "QueryExecutionId": "f8024d48-bc50-4604-bcfa-452ea1b1cf1b",
        "Query": "select * from nyc_taxi limit 10",
        "StatementType": "DML",
        "ResultConfiguration": {
            "OutputLocation": "s3://<YOUR BUCKET NAME>/results/Unsaved/2021/12/04/f8024d48-bc50-4604-bcfa-452ea1b1cf1b.csv"
        },
        "QueryExecutionContext": {
            "Database": "<YOUR BUCKET NAME>",
            "Catalog": "awsdatacatalog"
        },
        "Status": {
            "State": "SUCCEEDED",
            "SubmissionDateTime": "2021-12-04T15:01:29.246000+09:00",
            "CompletionDateTime": "2021-12-04T15:01:30.589000+09:00"
        },
        "Statistics": {
            "EngineExecutionTimeInMillis": 996,
            "DataScannedInBytes": 119391,
            "TotalExecutionTimeInMillis": 1343,
            "QueryQueueTimeInMillis": 244,
            "QueryPlanningTimeInMillis": 211,
            "ServiceProcessingTimeInMillis": 103
        },
        "WorkGroup": "primary",
        "EngineVersion": {
            "SelectedEngineVersion": "AUTO",
            "EffectiveEngineVersion": "Athena engine version 2"
        }
    }
}
```

ということで、とりあえずAthenaからSQLを発行できる状態まで作業が完了したので、今回はここまで。


## :pencil2: 02. Athenaの特徴をまとめる

この記事はAmazon Athenaの自分用の備忘録です。

## Athenaの特徴
今回はAthenaの特徴をまとめておく。

### DDL・DML
一般的にはDDL・DMLは下記のことを指す。

- DDL(Data Definition Language)
    - `CREATE`、`DROP`、`ALTER`などデータベースオブジェクトの生成、削除、変更を行うコマンド。
- DML(Data Manipulation Language)
    - `SELECT`、`INSERT`、`UPDATE`、`DELETE`などテーブルにデータの取得、追加、更新、削除を行うコマンド。

Athenaではテーブルを定義して、スキーマーを更新したり、テーブルのパーティションを定義したりすることで、メタデータを更新することができる。もちろん、DMLステートメントを利用してデータを抽出できる。下記の通り、サポートされないDDLもあるので注意。

- [サポートされないDDL](https://docs.aws.amazon.com/ja_jp/athena/latest/ug/unsupported-ddl.html)

例えば、一般的なトランザクションはサポートされないと聞いていたが、最近(2021/12)の発表によると、サポートされるようになったらしい。

- [Amazon Athena が新しい Lake Formation の詳細設定可能なセキュリティと信頼性の高いテーブル機能のサポートを開始](https://aws.amazon.com/jp/about-aws/whats-new/2021/11/amazon-athena-lake-formation-security-table-features/)

ほかには、クエリ結果からテーブル作成を行う`CTAS(CREATE TABLE AS SELECT)`クエリも利用可能。CTASは、別のクエリからの`SELECT`ステートメントの結果を、Athenaで新しいテーブルとして作成する機能。Athenaは、`CTAS`ステートメントによって作成されたデータをS3の指定されたロケーションに保存する。使用場面としては、クエリ結果をParquet形式に変換することで、Athenaでのクエリパフォーマンスが向上し、クエリコストを削減したり、必要なデータのみが含まれる既存のテーブルのコピーを作成するなど。`CTAS`、`INSERTINTO`ステートメントで自動的にパーティションを自動更新できる。`CTAS`ステートメントについては後日調べるとことにする。

### ANSI SQL準拠
AthenaはANSI SQLに準拠したPrestoが使われている。Prestoとは、ビッグデータ用の高性能な分散SQLクエリエンジンで、Facebookで設計、開発された経緯がある。
大きなテーブル結合、ウィンドウ関数を使用した複雑な分析、`MAPS`、`STRUCTS`、`LISTS`などのデータタイプも対応可能。

### サポートファイルフォーマット
CSV、TSV、AVRO、JSON、GZIP、ApacheORC、ApacheParquetなどの列指向データフォーマットなど幅広いファイルフォーマットに対応。

### 追加のコスト
前回も料金については記載したが、ここではもう少し細かく調べて記載しておく。基本的にAWSのサービスは、サービス単体のコストだけを考えてればよいわけではなく、何かと何かが相互にやりとりをしているため、色んな所でお金が発生する。ざっくりとAthenaの挙動をまとめておく。

Athenaが最初に行うことは、Glueデータカタログにクエリで使用されるテーブルのメタデータを確認すること(GlueへのAPIコール)。次に、Glueからメタデータを収集したあとは、S3にアクセスしてデータを読み始めること。読み取りが必要なデータを確認するために各パーティションのオブジェクトをリストする必要がある(S3へのAPIコール)。そして、Athenaがデータを読んだあとは、S3のデータが暗号化されているのであれば、KMSから複合キーを探して、復号化が行われる(KMSへのAPIコール)。このような流れとなる。AthenaはスキャンしたTB単位で課金されるので、Parquetのような形式で保存することが望まれる。

```
# CTASでCSVからParquetに変換できる
CREATE TABLE nyc_taxi_parquet WITH (
	format = 'Parquet',
	parquet_compression = 'SNAPPY',
	external_location = 's3://<YOUR BUCKET NAME>/tables/nyc_taxi_parquet/'
) AS
SELECT * FROM nyc_taxi;
```

変換後のS3の中身は下記のようになっている。

```
➜ aws s3 ls s3://<YOUR BUCKET NAME>/tables/nyc_taxi/ --recursive --sum --human
2021-11-27 17:26:58   23.9 MiB tables/nyc_taxi/yellow_tripdata_2021-01.csv.gz
Total Objects: 1
   Total Size: 23.9 MiB

➜ aws s3 ls s3://<YOUR BUCKET NAME>/tables/nyc_taxi_parquet/ --recursive --sum --human
2021-12-05 11:57:45  921.1 KiB tables/nyc_taxi_parquet/20211205_025736_00134_bu7gr_05b26478-2b75-413e-a7b3-395715c4c082
[略]
2021-12-05 11:57:45  948.7 KiB tables/nyc_taxi_parquet/20211205_025736_00134_bu7gr_f0c07ae2-8ee2-4097-a9e6-c2b81792d666

Total Objects: 30
   Total Size: 28.4 MiB
```

これらのデータを使って下記のようクエリを発行すると、データサイズが小さいので、実行時間の恩恵は受けれていないが、スキャンしたデータ量が小さくなっている。

```
# parquet 
# 実行時間:2.339 秒 スキャンしたデータ:188.84 KB
SELECT vendorid FROM nyc_taxi_parquet;SELECT vendorid FROM nyc_taxi_parquet;

# csv
# 実行時間:2.568 秒 スキャンしたデータ:23.88 MB
SELECT vendorid FROM nyc_taxi_parquet;SELECT vendorid FROM nyc_taxi;

```

クエリを分析したいときは、RDS同様、`EXPLAIN`が利用できる。

- [Using EXPLAIN and EXPLAIN ANALYZE in Athena](https://docs.aws.amazon.com/athena/latest/ug/athena-explain-statement.html)

```
Query Plan
- Output[vendorid, tpep_pickup_datetime, tpep_dropoff_datetime] => [[vendorid, tpep_pickup_datetime, tpep_dropoff_datetime]]
    - Limit[10] => [[vendorid, tpep_pickup_datetime, tpep_dropoff_datetime]]
        - LocalExchange[SINGLE] () => [[vendorid, tpep_pickup_datetime, tpep_dropoff_datetime]]
            - RemoteExchange[GATHER] => [[vendorid, tpep_pickup_datetime, tpep_dropoff_datetime]]
                - LimitPartial[10] => [[vendorid, tpep_pickup_datetime, tpep_dropoff_datetime]]
                    - TableScan[awsdatacatalog:HiveTableHandle{schemaName=<YOUR BUCKET NAME>, tableName=nyc_taxi, analyzePartitionValues=Optional.empty}] => [[vendorid, tpep_pickup_datetime, tpep_dropoff_datetime]]
                            LAYOUT: <YOUR BUCKET NAME>.nyc_taxi
                            tpep_dropoff_datetime := tpep_dropoff_datetime:string:2:REGULAR
                            vendorid := vendorid:bigint:0:REGULAR
                            tpep_pickup_datetime := tpep_pickup_datetime:string:1:REGULAR

```

簡易見積もりツールが用意されているので、遊んでみた。東京リージョンで、1日100クエリ、クエリのスキャン量は5GBという設定
だと、だいたい8000円前後くらいとなる。

- [Configure Amazon Athena](https://calculator.aws/#/createCalculator/Athena)

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-003.png)


### ワークグループ管理
Athenaではワークグループが管理でき、グループごとに設定を管理できる。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-004.png)

設定項目は下記のとおり。

- クエリの結果の場所と暗号化(S3エクスポート先を指定できる)
- クエリエンジンのバージョン
- 設定(CloudWatchにメトリクスを送るかなどを設定できる)
- クエリデータ使用状況コントロールごとに管理(クエリがスキャンできる最大量を設定。制限を超えた場合はキャンセル扱い。)
- ワークグループデータの使用状況アラート(SNSトピックを作成)
- タグ

### アドホック分析
Athenaはアドホック分析を行うのに適している。前回の記事でみたように、S3にデータが保存されていって、Glueデータカタログにメタデータが登録されていれば、クエリを発行して分析する準備は整っているので、分析環境の構築が必要なく、非常に早く分析に着手できる。また、シンプルなETLパイプラインを作成することにも適している。
また、アクセス権の管理も容易におこなうことができる。IAMポリシーからユーザーに対して、データのS3パスへのアクセスを拒否、許可すればよい。加え、カラムや行ごとの制限が必要になるのであれば、LakeFormationを組み合わせることで管理することになる。

ここらへんで今回はおわり。

## :pencil2: 03. AthenaでETL(Extract-Transform-Load)

この記事はAmazon Athenaの自分用の備忘録です。

### データ変換(CTAS)
今回行うことの準備として、以前から利用しているNYの黄色タクシーのトリップデータを201801~201806(5千万レコードくらい)まで使用し、S3にアップロードしておく。

### 備考：データの準備
今回行うことの前準備として、以前から利用しているNYの黄色タクシーのトリップデータを複数S3にアップロードしておく。そのために、まずはS3にアップロード先となるバケット`<YOUR S3 BUCKET NAME2>`を作成。下記の`data_prep.sh`を作成し、実行権限を付与しておく。`<YOUR S3 BUCKET NAME2>`を引数として渡し、シェルスクリプトを実行。個別にダウンロードして、手動でS3にアップロードしても良い。

```
#/bin/bash

BUCKET=$1

# 渡された引数の数を表示
# 引数が0であれば、バケット名を指定するようにエラーを返す
if [ $# -eq 0 ]
  then
    echo "[error] The argument must be S3 Bucket Name for upload"
    exit 0
fi

echo "[info] using bucket: $BUCKET"

# ダウンロードするCSV名を配列で保存
array=( yellow_tripdata_2018-01.csv 
	yellow_tripdata_2018-02.csv
	yellow_tripdata_2018-03.csv
	yellow_tripdata_2018-04.csv
	yellow_tripdata_2018-05.csv
	yellow_tripdata_2018-06.csv
	)

# URLと要素名を結合してリクエスト
# wgetでダウンロードしたあとは、gzipに変換
# gzipをS3にアップロードして、gzipを削除
# これを配列の要素数ループを回す
for i in "${array[@]}"
do
	FILE=$i
	ZIP_FILE="${FILE}.gz"
	echo "[info] Downloading ${FILE} from nyc-tlc"
	wget https://s3.amazonaws.com/nyc-tlc/trip+data/${FILE}
	echo "[info] Performing gzip on ${FILE}"
	gzip ${FILE}
	echo "[info] Uploading ${ZIP_FILE} to S3 bucket ${BUCKET}"
	aws s3 cp ./${ZIP_FILE} s3://$BUCKET/tables/nyc_taxi_csv/
	echo "[info] Cleaning up file ${ZIP_FILE}"
	rm $ZIP_FILE
done
```

S3に問題なくアップロードできているかを確かめておく。

```
# アップロードできているか確認
➜ aws s3 ls s3://<YOUR BUCKET NAME2>/tables/nyc_taxi_csv/
2021-12-07 23:34:07  151931420 yellow_tripdata_2018-01.csv.gz
2021-12-07 23:40:38   94355275 yellow_tripdata_2018-02.csv.gz
2021-12-07 23:46:58  164436244 yellow_tripdata_2018-03.csv.gz
2021-12-08 00:18:02  162733006 yellow_tripdata_2018-04.csv.gz
2021-12-08 00:13:34  161916903 yellow_tripdata_2018-05.csv.gz
2021-12-08 00:13:34  152990344 yellow_tripdata_2018-06.csv.gz
```

Glueにメタデータを登録しておく。

```
# メタデータを作成
aws glue create-database \
--database-input "{\"Name\":\"<YOUR BUCKET NAME2>\"}" \
--region ap-northeast-1

# メタデータの確認
➜ aws glue get-database --name <YOUR BUCKET NAME2>
{
    "Database": {
        "Name": "<YOUR BUCKET NAME2>",
        "CreateTime": "2021-12-08T00:19:00+09:00",
        "CreateTableDefaultPermissions": [
            {
                "Principal": {
                    "DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"
                },
                "Permissions": [
                    "ALL"
                ]
            }
        ],
        "CatalogId": "185260556888"
    }
}
```

Atehna上で今回使用するテーブル定義を実行しておく。

```
# テーブル定義
CREATE EXTERNAL TABLE `<YOUR BUCKET NAME2>`.`nyc_taxies`(
  `vendorid` bigint, 
  `tpep_pickup_datetime` string, 
  `tpep_dropoff_datetime` string, 
  `passenger_count` bigint, 
  `trip_distance` double, 
  `ratecodeid` bigint, 
  `store_and_fwd_flag` string, 
  `pulocationid` bigint, 
  `dolocationid` bigint, 
  `payment_type` bigint, 
  `fare_amount` double, 
  `extra` double, 
  `mta_tax` double, 
  `tip_amount` double, 
  `tolls_amount` double, 
  `improvement_surcharge` double, 
  `total_amount` double, 
  `congestion_surcharge` double)
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://<YOUR BUCKET NAME2>/tables/nyc_taxi_csv/'
TBLPROPERTIES (
  'areColumnsQuoted'='false', 
  'columnsOrdered'='true', 
  'compressionType'='gzip', 
  'delimiter'=',',
  'skip.header.line.count'='1', 
  'typeOfData'='file')
```

想定通り5千万レコードが読み込まれている。

```
SELECT COUNT(1) FROM nyc_taxies;
# 50873169
```

データを準備した段階では、ローデータがデータレイクに溜まっているような状態。つまり、Gzipファイルがそのままアップロードされているだけなので、コスト効率が良くない。まずはCTASでParquet形式に変換する。`tpep_pickup_datetime`のフォーマットが日時になっていないデータや201801~201806までのデータなのに、過去や未来のログも含まれるので、それは除外してCTASを実行する。確認したデータは記事末尾を参照。また、一度S3からParquetデータを削除しても、データカタログにはテーブルのメタデータが残っているので、それを削除しないと、再実行できない。

CTAS(CREATE TABLE AS SELECT)を実行すると、5千万レコードくらいでも数分かかる。あとでパーティションされたS3の中身を紹介しているが、年と月でデータを分けるということは、データに対してソートやグループ化が行われる必要があるため、単純にフルスキャンして実行結果を得る時よりも計算コストがかかるため。また、パーティション数が多くなりすぎるとクエリ実行時間が長くなるので、パーティションの数が多ければ多いほど良いわけではないとのこと。 

```
CREATE TABLE nyc_taxies_parquet
WITH (
	  external_location = 's3://<YOUR BUCKET NAME2>/tables/nyc_taxi_parquet/',
      format = 'Parquet',
      parquet_compression = 'SNAPPY',
      partitioned_by = ARRAY['year', 'month']
      )
AS SELECT 
	vendorid,
	tpep_pickup_datetime,
	tpep_dropoff_datetime,
	passenger_count,
	trip_distance,
	ratecodeid,
	store_and_fwd_flag,
	pulocationid,
	dolocationid,
	payment_type,
	fare_amount,
	extra,
	mta_tax,
	tip_amount,
	tolls_amount,
	improvement_surcharge,
	total_amount,
	congestion_surcharge,
	year(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) as year,
 	month(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) as month
FROM 
    nyc_taxies
WHERE 
    # tpep_pickup_datetimeのフォーマット(YYYY-MM-DD HH:MM:SS)が19桁で統一されていないレコードがあるので除外。
    # 除外しないと下記のエラーが表示されて、CTASが成功しない
    # [Error] INVALID_FUNCTION_ARGUMENT: Invalid format: "20" is too short.
    length(tpep_pickup_datetime) = 19 AND
    year(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) = 2018 AND 
    month(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) >= 1 AND
    month(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) <= 6
;

SELECT COUNT(1) FROM nyc_taxies_parquet;
# 50872197
```

CTASはSELECT文を既存のテーブルに適用して、新しいテーブルを作成する機能。Viewの作成と似ているが、CTASは実行結果をテーブル化するので、PostgresSQLのマテリアライズドビューに似ている。Viewも作ることはもちろんできるが、Viewの場合はRDSと同じように、実行されるたびにSQLが実行される。パーティションとして利用するカラムにここでは、`year, month`を利用しているので、S3のオブジェクト構成は、年と月ごとに分解されて、下記のように保存される。

```
aws s3 ls s3://<YOUR BUCKET NAME2>/tables/nyc_taxi_parquet/year=2018/
                           PRE month=1/
                           PRE month=2/
                           PRE month=3/
                           PRE month=4/
                           PRE month=5/
                           PRE month=6/
```

### データ追加(INSERT)
ここまで実行すると、201801~201806までのParquetデータが作成できているが、ここからさらに過去の期間のデータを追加する場合は`INSERT-INTO`を行っていく。まずはNYの黄色タクシーのトリップデータを201707~201712(5千万レコードくらい)まで先ほどのシェルスクリプトを調整して、S3にアップロードしておく。準備方法は、記事の末尾を参照。

CTASのクエリと違って、WITH句内のS3のロケーション、フォーマット形式、パーティションの設定に関する記述は必要がない。実行完了まで2-3分かかる。

```
INSERT INTO nyc_taxies_parquet
SELECT 
	vendorid,
	tpep_pickup_datetime,
	tpep_dropoff_datetime,
	passenger_count,
	trip_distance,
	ratecodeid,
	store_and_fwd_flag,
	pulocationid,
	dolocationid,
	payment_type,
	fare_amount,
	extra,
	mta_tax,
	tip_amount,
	tolls_amount,
	improvement_surcharge,
	total_amount,
	congestion_surcharge,
	year(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) as year,
 	month(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) as month
FROM 
    nyc_taxies
WHERE 
    length(tpep_pickup_datetime) = 19 AND
    year(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) = 2017 AND 
    month(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) >= 7 AND
    month(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) <= 12
;
```

S3も問題なくアップロードされているか確認しておく。

```
➜ aws s3 ls s3://<YOUR BUCKET NAME2>/tables/nyc_taxi_parquet/year=2017/
                           PRE month=10/
                           PRE month=11/
                           PRE month=12/
                           PRE month=7/
                           PRE month=8/
                           PRE month=9/

```

Athena側からもデータが正しくインサートできているか、簡単に確認しておく。`2017-07-01 00:00:00`から`2018-06-30 23:59:59`までのデータが保存されていることがわかる。年月ごとにレコード数をカウントし、CSVをダウンロードして中身を確かめておく。

```
SELECT 
    year,
    month
    count(1) as cnt
FROM
    nyc_taxies_parquet
GROUP BY
    year,
    month
ORDER BY
    year ASC,
    month ASC
;

➜ column -s, -t b1f995d5-e127-4e06-817a-5f504fb3bf5f.csv 
"year"  "month"  "cnt"
"2017"  "7"      "8588486"
"2017"  "8"      "8422197"
"2017"  "9"      "8945574"
"2017"  "10"     "9768740"
"2017"  "11"     "9284777"
"2017"  "12"     "9508274"
"2018"  "1"      "8760088"
"2018"  "2"      "5439940"
"2018"  "3"      "9429402"
"2018"  "4"      "9305286"
"2018"  "5"      "9224091"
"2018"  "6"      "8713390"

# SELECT 
#     min(tpep_pickup_datetime) AS min_date,
#     max(tpep_pickup_datetime) AS max_date
# FROM
#     nyc_taxies_parquet
# ;
# min_date: 2017-07-01 00:00:00
# max_date: 2018-06-30 23:59:59
```

### Parquetの特徴
Parquetの特徴を探るために下記の2つのクエリを実行してみる。内容としては2017年の`tpep_pickup_datetime`のユニーク数を集計するもの。実際に実行すると、1/10の速さで処理が完了する。

```
######################
# Part1 - csv
######################
SELECT 
    count(distinct tpep_pickup_datetime) as date_uu
FROM
    nyc_taxies
WHERE
    length(tpep_pickup_datetime) = 19 AND
    year(date_parse(tpep_pickup_datetime,'%Y-%m-%d %H:%i:%s')) = 2017
;

# 19.932秒/1.71GB
# date_uu
# 14049152

######################
# Part2 - Parquet
######################
SELECT 
    count(distinct tpep_pickup_datetime) as date_uu
FROM
    nyc_taxies_parquet
WHERE
    year = 2017
;

# 2.659秒/278.43MB
# date_uu
# ユニークの件数が異なるのはCSVには想定期間外のデータも含まれているため。
# 14049150
```

このような違いが出る理由は、`date_parse()`があるなしも関係しているが、フォーマット形式や設定に由来する部分が大きい。csvに対するクエリはスキャン量が1.71GBとなっていることから、フルスキャンしていることがわかる。これはS3に保存しているサイズとも一致している。パーティションもないので、2017年にフィルタしようとしても、全部データを見る必要がある。

```
➜ aws s3 ls s3://<YOUR BUCKET NAME2>/tables/nyc_taxi_csv/ --human --sum
2021-12-09 09:27:01  142.6 MiB yellow_tripdata_2017-07.csv.gz
2021-12-09 09:27:01  139.9 MiB yellow_tripdata_2017-08.csv.gz
2021-12-09 09:27:01  149.2 MiB yellow_tripdata_2017-09.csv.gz
2021-12-09 09:27:01  162.7 MiB yellow_tripdata_2017-10.csv.gz
2021-12-09 09:27:01  154.5 MiB yellow_tripdata_2017-11.csv.gz
2021-12-09 09:27:01  157.8 MiB yellow_tripdata_2017-12.csv.gz
2021-12-07 23:34:07  144.9 MiB yellow_tripdata_2018-01.csv.gz
2021-12-07 23:40:38   90.0 MiB yellow_tripdata_2018-02.csv.gz
2021-12-07 23:46:58  156.8 MiB yellow_tripdata_2018-03.csv.gz
2021-12-08 00:18:02  155.2 MiB yellow_tripdata_2018-04.csv.gz
2021-12-08 00:13:34  154.4 MiB yellow_tripdata_2018-05.csv.gz
2021-12-08 00:13:34  145.9 MiB yellow_tripdata_2018-06.csv.gz

Total Objects: 12
   Total Size: 1.7 GiB
```

また、Parquetの方は、テーブルの統計情報を保持しているので、`count(distinct)`など集計関数の結果

parquetはデータのブロック(row group, column chunk, page)ごとに、`min`、`max`、`count`<YOUR BUCKET NAME2>一方で、統計情報からは返せないようなクエリであっても、パーティションが利用できるので、スキャン量を圧縮でき、結果も速く得ることができる。

パフォーマンスチューニングについては、下記の公式記事が詳しい。

- [Amazon Athena のパフォーマンスチューニング Tips トップ 10](https://aws.amazon.com/jp/blogs/news/top-10-performance-tuning-tips-for-amazon-athena/)

### データのサンプリング
Athenaを使うということは、それなりに大きなデータを扱うことが前提になっている場合が多いと思われる。このようなケースで実務上問題となるのが、個人的には、初見のデータでテーブル構造がどうなっているのかわからないようなケース。SQLが高速に処理されるのと、テーブル構造に対して、適切なSQLの設計を考え、データを取得することは別問題。簡単な集計や前処理であればこんなこと必要ないが、分析用のSQLというのは、いささか面倒が多く、複雑になりやすい。そんなときに、テーブル構造へのおおよその理解を助けるためのサンプリング関数のことをまとめておく。

- [TABLESAMPLE](https://docs.aws.amazon.com/athena/latest/ug/select.html#:~:text=TABLESAMPLE%20%5B%20BERNOULLI%20%7C%20SYSTEM,independent%20sampling%20probabilities.)

この関数は、指定した割合でテーブルをサンプリングしてくれる関数。6ヶ月で5000万件くらいになるので、`TABLESAMPLE BERNOULLI　(10)`とすると500万件くらいになっていることがわかる。れる。一方で`TABLESAMPLE SYSTEM　(10)`も10％くらいのデータをサンプリングしてくれている。細かい違いは下記の通り。

`BERNOULLI`は、テーブル・サンプルに含まれる各行を`percentage`の確率で選択。テーブルのすべての物理ブロックがスキャンされ、サンプルのパーセンテージと実行時に計算されるランダムな値との比較に基づいて、特定の行がスキップされる。つまりスキャン量が多くなる。

`SYSTEM`では、テーブルをデータの論理的なセグメントに分割し、その粒度でテーブルをサンプリング。特定のセグメントのすべての行が選択されるか、またはサンプルの割合と実行時に計算されたランダムな値との比較に基づいてセグメントがスキップ。つまり、こちらのほうがコスト的には良いが、サンプルのランダムサンプリング度合いは小さくなる。特にもとのデータがひどく偏りがある場合、精度は悪くなる。

```
#################################
# Part1 - No Sampling
#################################
SELECT 
    year, count(1) as cnt
FROM
    nyc_taxies_parquet
GROUP BY
    year
;

# 2018 50872197
# 2017 54518048

#################################
# Part2 - TABLESAMPLE BERNOULLI
#################################
SELECT 
    year, count(1) as cnt
FROM
    nyc_taxies_parquet
TABLESAMPLE BERNOULLI (10)
GROUP BY
    year
;

# 2017 5450682
# 2018 5087594

#################################
# Part3 - TABLESAMPLE SYSTEM
#################################
SELECT 
    year, count(1) as cnt
FROM
    nyc_taxies_parquet
TABLESAMPLE SYSTEM (10)
GROUP BY
    year
;
# 2018 6859351
# 2017 6547754
```

### 備考：データの不備

CTASを実行する際に、`tpep_pickup_datetime`のフォーマットが日時になっていないデータや201801~201806までのデータなのに、過去や未来のログも含まれている例。

```
# 不要なデータを含めてCTASした際のデータ内容
$ column -s, -t ee4209d2-f4ac-4325-9f36-3ad66648efb7.csv 
"year"  "month"  "cnt"
"2001"  "1"      "7"
"2002"  "12"     "15"
"2003"  "1"      "11"
"2008"  "12"     "151"
"2009"  "1"      "272"
"2017"  "1"      "2"
"2017"  "12"     "224"
"2018"  "1"      "8760088"
"2018"  "2"      "5439940"
"2018"  "3"      "9429402"
"2018"  "4"      "9305286"
"2018"  "5"      "9224091"
"2018"  "6"      "8713390"
"2018"  "7"      "154"
"2018"  "8"      "22"
"2018"  "9"      "21"
"2018"  "10"     "10"
"2018"  "11"     "15"
"2018"  "12"     "9"
"2019"  "1"      "7"
"2019"  "2"      "8"
"2019"  "3"      "4"
"2019"  "4"      "5"
"2019"  "9"      "5"
"2020"  "3"      "6"
"2026"  "2"      "2"
"2029"  "5"      "2"
"2031"  "2"      "2"
"2037"  "11"     "1"
"2041"  "6"      "1"
"2042"  "12"     "1"
"2084"  "11"     "8"
```


## :pencil2: 04. Athenaのメタストア

この記事はAmazon Athenaの自分用の備忘録です。Glueクローラーでデータカタログはおしりの方に記載。

### メタストア
メタストアはデータソースを構成する要素のうちの1つらしいがよくわからない。公式ドキュメントなどを参照すると、Athenaがクエリするために必要な情報を提供してくれる機能のことらしい。言い換えると、SQL文をAthenaから発行すると、Athenaはクエリの解析を行い、必要なテーブルとカラムを識別する。そこから、メタストアに、「どこにデータが有るのか」「どのように保存されているのか」「フォーマットはどのような形式なのか」などの必要なデータにまつわる情報を取得する。このときに利用されているのがメタストアとのこと。
このような仕組みになっているのはビックデータの世界では、分析エンジン(Athena)とメタデータ(Glueデータカタログ)、データストア(S3)を別々に保存しておくのが一般的な構成になるため。


### データベース
Athenaでは、テーブルの集合をデータベースと呼ぶ。Athenaでデータベースを作成する場合は下記のDDLクエリを発行すれば良い。ここでは、`train-database`という名前のデータベースを、S3の`train-database`バケットに作成する場合のDDLクエリの例をあげておく。

```
# Athena
# `train-database`データベースを`train-database`バケットに作成する
CREATE DATABASE `train-database` LOCATION 's3://train-database/tables/'
```

Athenaではデータ自体を保存してないので、FROM句での記法は下記を意識しておけばよい。
これはなんてことないが、データベース「A」を選択して、Aの中の「Aa」テーブルを参照したい場合、FROM句では下記のように書けば良い。

```
# 選択しているデータベースは「A」
FROM "A"."Aa"
FROM "Aa"
```

このようなデータベース「B」を選択して、データベース「A」の中の「Aa」テーブルを参照したい場合、FROM句では下記のように書けば良い。

```
# 選択しているデータベースは「B」
FROM "A"."Aa"
```

### テーブル(データセット)

データベースという階層があり、その下の階層が「テーブル」という階層がある。さきほどは、まず「データベース」をS3に作成した。ここでは、その「データベース」の中に「テーブル(`nyc_taxi_partitioned`)」を作成している。テーブルを作成する際には、上から順に「テーブルの場所」、「スキーマー」、「保存ファイル形式」、「保存先」など。

```
CREATE EXTERNAL TABLE `train-database`.`nyc_taxi_partitioned`(
  `VendorID` bigint, 
   /////
  `congestion_surcharge` double)
PARTITIONED BY (`year` INT, `month` INT)
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://train-database/tables/nyc_taxi_partitioned/'
TBLPROPERTIES (
  'areColumnsQuoted'='false', 
  'columnsOrdered'='true', 
  'compressionType'='gzip', 
  'delimiter'=',',
  'skip.header.line.count'='1'
)
```
[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-005.png)

### スキーマー
スキーマーは、クエリ可能なカラムのリストとデータ型を示す。テーブルスキーマーで指定されたデータ型と保存されているデータのタイプと一致している必要がある。ファイル形式によっては、カラムの順序とスキーマーの設定が異なる場合、誤ったクエリを取得する可能性がある。パーティションは仮想的なカラムに基づいて、データを分割して保存する機能のこと。必要なデータだけをAthenaが読み込むことができるため、効率がよくなる。

### パーティション
パーティションを設定していない場合、下記のDDL文でパーティションを作成できるがあまり推奨されない。`ALTER TABLE`が推奨される。

```
MSCK REPAIR TABLE nyc_taxi_partitioned;
 
or

ALTER TABLE nyc_taxi_partitioned
    ADD PARTITON (year='2020', month='1')
    LOCATION 's3://train-database-bucket/tables/nyc_taxi_partitioned/year=2020/month=1'
```
### データ形式
下記の部分ではデータ形式などを判断し、アウトプットフォーマットなどを決定している。

```
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
```

### テーブルプロパティ
テーブルプロパティでは、様々な用途に使用されるデータに関する情報が記載される。キーと値の形式で記述する。

- [CREATE TABLE](https://docs.aws.amazon.com/ja_jp/athena/latest/ug/create-table.html)

```
# Ref | https://dev.classmethod.jp/articles/20200627-amazon-athena-partition-projection/
# このような感じで色々追加できるとのこと
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.orderdate.type' = 'date',
  'projection.orderdate.range' = '1992/01/01,NOW',
  'projection.orderdate.format' = 'yyyy/MM/dd',
  'projection.orderdate.interval' = '1',
  'projection.orderdate.interval.unit' = 'DAYS',
  'storage.location.template' = 's3://<bucket_name>/ssbgz/lineorder-partitioned-daily/${orderdate}',
  'classification'='csv', 
  'compressionType'='gzip', 
  'delimiter'='|', 
  'typeOfData'='file')
```

### データソース
AthenaのデータソースはS3が最も一般的。もちろん他のAWSのデータベースのサービスとも連携できる。S3でデータをクエリするには、GlueデータカタログやHiveストアを利用することになるが、基本的には「Glueデータカタログ」が推奨される。Glueデータカタログはサーバレスなサービスで、最初の100万個オブジェクトは無料、10万個ごとに追加で1ドルがかかってくる。そして、最初の100万リクエストまでは無料で、100万リクエストごとに追加で1ドルかかる。Athenaでこれを使い切るのは困難かと思うが、GlueデータカタログはGlueETL,LakeFormation,Redshifts,EMRなどと共有できるので、なんだかんだでリクエストが飛びまくることになるのかも。Glueデータカタログはバージョン管理もサポートされており、テーブル変更があると保存され、破壊した場合もロールバックできる。

### Glueクローラーを使ってメタストアを作成
ここではメタストアへ登録をGlueクローラーを使って行い、Athenaでクエリを発行する方法をまとめておく。GlueクローラーはS3のデータをディレクトリを横断してスキャンしてスキーマを確認する。その過程で、パーティション化されたと水族できる場合、パーティションを行う。スキーマーが異なるのであれば、各ディレクトリは、個別のテーブルと判断される。情報を収集したあと、それらをデータカタログに登録・更新する。

### Glueクローラーを使ったメタストアへの登録
S3に`train-athena-gluecrawlers-yellowtaxies`というバケットを作成し、下記のようにディレクトリを作成する。`day=1`、`day=2`にはこれまで使用しているタクシーデータのフォルダ名の日付に対応するフィルタリングした小さなデータをアップロードしておく。

```
train-athena-gluecrawlers-yellowtaxies/ # database
    tables/
        train-athena-gluecrawlers/ # table
            year=2021/
                month=1/
                    day=1
                        log_2021_01_01.csv
                    day=2
                        log_2021_01_02.csv
```
AthenaのUIから「AWS Glueクローラー」を選択する。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-006.png)

Glueクローラーのセットアップウィザードに従って、情報を入力していく。まずはクローラーを識別する名前を入力する。説明や暗号化などの設定もこの段階で行う。ここでは`train-athena-gluecrawlers`という名前のクローラーを作成している。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-007.png)

この画面ではデータストアの選択を行う。S3ロケーションを指定するか、既存のカタログテーブルを指定するかを選択できる。またS3へのクローラーの繰り返し設定を選択する。クロールの際に全部のフォルダを再度クロールするのか、追加分だけクロールするのか、S3のイベントによって行うのかなどを選択する。ここでは「S3のデータに対して追加分だけ」クローラーが動くように設定している。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-008.png)

この画面ではS3への接続設定を行う。`train-athena-gluecrawlers-yellowtaxies/tables/train-athena-gluecrawlers`というパス以下のファイルをクロールするように設定している。Glueクローラーは、パーティションを意識したようなフォルダ構成にしておけば、パーティションが設定される。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-009.png)

今回の例はあてはまらないが、例えば、`tables/`まででインクルードパスを設定し、下記のようなS3のフォルダ構成であれば、Athena上のテーブル名は`train-athena-gluecrawlers`、`cuid_master`、`products_master`という感じによしなにやってくれる。

```
train-athena-gluecrawlers-yellowtaxies/ # database
    tables/
        - train-athena-gluecrawlers/ # table
            year=2021/
                month=1/
                    day=1
                        log_2021_01_01.csv
        - cuid_master/
            mst_cuid
        - products_master/
            mst_products

```

1回のクロールで複数のデータソースをクローリングできるので、必要であれば、その設定をここで行っておく。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-010.png)

この画面では、クローラーのIAMロールを設定する。作成していないのであれば、この画面から作成できる。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-011.png)

クローラーのクローリング頻度を設定する。ここでは、カスタムで最小の5分ごとの実行(`*/5 * * * ? *`)を設定している。他にも、オンデマンド実行、毎月、毎日、毎時などで設定できる。クローラーを定期的に動かす1つの理由は、勉強不足なので、おそらくになるが、パーティションで区切られている場合、月毎や日毎でフォルダが作成される。今回で言えばクローラーは、2021年1月1日、2021年1月2日までのフォルダのデータの存在は知っているので、Athenaのクエリに反映させることができるが、2021年1月3日分のフォルダとデータが保存されていても、クローラーをその存在をしらず、このようなタイミングでクエリを発行しても対象となるのは2日分だけ。新しく作成されてデータを認識するために、クローラーが定期的に実行され、メタデータとして収集することが必要になる。と思われる。一回こっきりのアドホックな分析であれば「オンデマンド」で一度実行すれば良い。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-012.png)

この画面では作成するデータベース名を設定する。ここでは、S3バケットと合わせて、`train-athena-gluecrawlers-yellowtaxies`と設定。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-013.png)

クローラーの設定が完了すると、クローラーの一覧画面に戻ってくるので、この画面からとりあえず初回実行をボタンから行っておく。クローラーが動き出すと、「Starting」にステータスが変更される。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-014.png)

クローラーが実行しているかどうかは、CloudWatchのログをみるとわかる。ログを見てみると、データの保存構造を解析して、年、月、日でパーティションを作成していることがわかる。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-015.png)

クローラーの実行が完了すると、Athenaからクエリが発行できるようになっているので、試しにS3にある2021年1月1日、2021年1月2日の2日分のデータがあるかどうか集計しておく。結果を見る限り問題なさそうである。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-016.png)

クローラーがメタデータを自動で更新する挙動を確認するため、S3に2021年1月3日のデータをアップロードしておく。

```
train-athena-gluecrawlers-yellowtaxies/ # database
    tables/
        train-athena-gluecrawlers/ # table
            year=2021/
                month=1/
                    day=1
                        log_2021_01_01.csv
                    day=2
                        log_2021_01_02.csv
                    day=3
                        log_2021_01_03.csv
```

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-017.png)

CloudWatchのログをみると、5分ごとに実行としているので、2021年1月3日のデータを更新していることがわかる。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-018.png)

Athenaからクエリを発行して、試しにS3にある2021年1月1日、2021年1月2日、2021年1月3日の3日分のデータに更新されているかどうか集計して確認しておく。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-019.png)


## :pencil2: 05. Athenaですぐにアドホック分析を開始する方法

この記事はAmazon Athenaの自分用の備忘録です。

### S3にデータを保存
とりあえず、ここまで使用しているNYの黄色タクシーのトリップデータをここでも利用する。S3に`train-athena-yellowtaxi-splited`というバケットを作成して、csvデータをアップロード。

```
$ aws s3api create-bucket \
--bucket train-athena-yellowtaxi-splited \
--region ap-northeast-1 \
--create-bucket-configuration LocationConstraint=ap-northeast-1

$ aws s3 cp ~/Desktop/yellow_trip_separated_data_202101/log_2021_01_01.csv \
s3://train-athena-yellowtaxi-splited/log_2021_01_01.csv

$ aws s3 ls s3://train-athena-yellowtaxi-splited --recursive   
2021-12-18 02:22:37    2296393 log_2021_01_01.csv
```

### テーブル定義の作成

次はテーブル定義を用意する。カラム名が少なければよいが、多い場合はCSVのヘッダーを取り出して、少しだけ楽する。また、テーブルが少なければ、このような方法でテーブルを定義していけばよいかもしれないが、テーブルが大量になるとGlueクローラーでメタデータを作成するほうが楽。


```
# cat log_2021_01_01.csv | head -n 1 | sed 's/,/\n/g'
# 上記の結果をコピーすればカラム名を得られるのでデータ型を付与する

VendorID bigint, 
tpep_pickup_datetime string, 
tpep_dropoff_datetime string,   
passnger_count bigint, 
trip_distance double, 
RatecodeID bigint, 
store_and_fwd_flag string, 
PULocationID bigint, 
DOLocationID bigint, 
payment_type bigint, 
fare_amount double, 
extra double, 
mta_tax double, 
tip_amount double, 
tolls_amount double, 
improvement_surcharge double, 
total_amount double, 
congestion_surcharge double
```

### Athena上で設定
AthenaのUIから「S3バケットデータ」を選択。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-020.png)


設定画面では、テーブル名やデータベース設定、データセットの参照元、データ形式、パーティションなどが設定できる。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-021.png)

「列の詳細」でカラムを1つずつ設定していくこともできるが、ここでは先程用意したテーブル定義文を「列の一括追加」から利用する。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-022.png)

設定に問題なければ「テーブルを作成」をクリックして、テーブルの作成は完了。

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-023.png)

今回の設定だと下記のような`CREATE`文が作成される。

```
CREATE EXTERNAL TABLE IF NOT EXISTS `train-athena-yellowtaxi-splited`.`nyc_splited` (
  `vendorid` bigint,
  `tpep_pickup_datetime` string,
  `tpep_dropoff_datetime` string,
  `passnger_count` bigint,
  `trip_distance` double,
  `ratecodeid` bigint,
  `store_and_fwd_flag` string,
  `pulocationid` bigint,
  `dolocationid` bigint,
  `payment_type` bigint,
  `fare_amount` double,
  `extra` double,
  `mta_tax` double,
  `tip_amount` double,
  `tolls_amount` double,
  `improvement_surcharge` double,
  `total_amount` double,
  `congestion_surcharge` double
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe' 
WITH SERDEPROPERTIES (
  'serialization.format' = ',',
  'field.delim' = ','
) LOCATION 's3://train-athena-yellowtaxi-splited/'
TBLPROPERTIES ('has_encrypted_data'='true');
```

### クエリを発行する

準備は完了しているので、データの取り出しができることが確認できる。

```
SELECT * FROM "train-athena-yellowtaxi-splited"."nyc_splited" limit 10;
## SELECT count(1) FROM "train-athena-yellowtaxi-splited"."nyc_splited";
## 24828
```

[](https://github.com/SugiAki1989/sql_note/blob/main/image/p146-024.png)


Athenaから作成することでGlueにデータカタログが自動で作成されている。

```
$ aws glue get-database --name train-athena-yellowtaxi-splited
{
    "Database": {
        "Name": "train-athena-yellowtaxi-splited",
        "CreateTime": "2021-12-18T02:43:13+09:00",
        "CreateTableDefaultPermissions": [
            {
                "Principal": {
                    "DataLakePrincipalIdentifier": "IAM_ALLOWED_PRINCIPALS"
                },
                "Permissions": [
                    "ALL"
                ]
            }
        ],
        "CatalogId": "185260556888"
    }
}
```

データをS3に追加してAthenaからクエリを発行すると、追加したデータも集計対象内に入っていることがわかる。

```
aws s3 cp ~/Desktop/yellow_trip_separated_data_202101/log_2021_01_02.csv \
s3://train-athena-yellowtaxi-splited/log_2021_01_02.csv

$ aws s3 ls s3://train-athena-yellowtaxi-splited --recursive
2021-12-18 02:31:21    2296393 log_2021_01_01.csv
2021-12-18 02:52:19    3181024 log_2021_01_02.csv

SELECT count(1) FROM "train-athena-yellowtaxi-splited"."nyc_splited";
59138
```

片付けとしてS3のバケットなどは削除しておく。

## :closed_book: Reference

none
