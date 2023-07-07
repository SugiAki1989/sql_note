## :memo: Overview

ここでは Embulk でデータ転送を行うための基本的な方法をまとめておく。構成としては、AWS 環境下の EC2 で Embulk を実行し、GoogleSpreadSheet のシートデータを AWS 環境下の MySQL にロードすることが目的。データ分析基盤の構築では、すべてがシステムから連携できるのが理想ではあるが、システム化されていないデータは GoogleSpreadSheet で管理されているケースも多く、なんだかんだで分析にも必要だったりするので、このプラグインは非常にありがたい。

- [Google Spreadsheets input plugin for Embulk](https://github.com/medjed/embulk-input-google_spreadsheets)
- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`embulk-input-google_spreadsheets`, `embulk-output-mysql`

## :pencil2: GoogleSpreadSheet to STDOUT

まずは EC2 に SSH でアクセスして、必要なプラグインをインストールする。GoogleSpreadSheet のプラグインをインストールするとエラーが表示される。Embulk をインストールした際に、最新版だとインストールしても使えないプラグインがあった影響で、`embulk (0.9.25 java)`を利用している。そのため、下記のプラグインに必要なパッケージ?ライブラリ?(Ruby での呼び方がわからない)のバージョンに問題がある模様。

```
$ embulk gem install embulk-input-google_spreadsheets
ERROR:  Error installing embulk-input-google_spreadsheets:
	jwt requires Ruby version >= 2.5.
```

`jwt`は JSON Web Token の略で、OAuth に必要な情報をセキュアにやリとりするために必要なもの。GoogleSpreadSheet のサービスアカウントのキーファイルをコンフィグで設定する必要があるので、その際に利用されているものだと思われる。

その他でもエラーが頻発するが、下記のサイトに従って、必要なバージョンでインストールすれば利用できるようになる。

- [Embulk の embulk-output-bigquery のインストールエラー回避方法](https://qiita.com/kaaaaaaaaaaai/items/6c2a459236ff2f714dc8)

```
$ embulk gem install jwt:2.3.0
$ embulk gem install public_suffix -v 4.0.7
$ embulk gem install representable -v 3.0.4
$ embulk gem install multipart-post -v 2.1.1
# これは不要だった
# $ embulk gem install multipart-post -v 2.1.1
```

無事にインストールされている模様。

```
$ embulk gem install embulk-input-google_spreadsheets
$ embulk gem list | grep embulk

embulk (0.9.25 java)
embulk-filter-hash (0.5.0)
embulk-input-google_spreadsheets (1.1.1)
embulk-input-mysql (0.13.2 java)
embulk-input-sftp (0.4.0 java)
embulk-output-mysql (0.10.2 java)
embulk-output-s3 (1.7.1 java)
```

あまり関係ないが、ちなみに Ruby を AmazonLinux2 にインストールする方法は下記とのこと。

```
$ amazon-linux-extras list | grep ruby
 57  ruby3.0                  available    [ =stable ]
$ sudo amazon-linux-extras install ruby3.0
$ ruby -v
ruby 3.0.6p216 (2023-03-30 revision 23a532679b) [x86_64-linux]
```

`embulk-input-google_spreadsheets`のコンフィグでは、`json_keyfile`を指定する必要があるので、そのファイルを EC2 に転送しておく。作成方法は下記の通り。

- GCP にアクセスして、プロジェクトを作成
- 作成したプロジェクトを選択し、画面左側の「API とサービス」から画面中央上部の「+API とサービスを有効化」を選択
- GoogleSpreadSheet を検索し、「有効にする」を選択
- GCP のホーム画面に戻り、画面左側の「API とサービス」から「認証情報」を選択
- 画面中央上部の「+認証情報を作成」から「サービスアカウント」を選択
- サービスアカウントの詳細を入力し作成する
- サービスアカウントにはメールアドレスがあるので、GoogleSpreadSheet で権限を設定しておく
- メールアドレスをクリックし、「鍵を追加」から「新しい鍵を作成」を選択し、JSON を選択してダウンロード

```
$ scp ~/Desktop/service_account_key.json ec2-user@11.111.1.111:/home/ec2-user
```

GoogleSpreadSheet の値はこのようになっている。自宅の部屋の CO2 を RaspberryPi に取り付けているセンサーから取得し、Python で GoogleSpreadSheet に 5 分間隔で記録しているものをここでは使用する。

```
Time	CO2(ppm)
2023-07-06 21:09:11	781
2023-07-06 21:14:14	839
2023-07-06 21:19:18	882
2023-07-06 21:24:22	901
2023-07-06 21:29:27	917
2023-07-06 21:34:30	927
2023-07-06 21:39:34	925
2023-07-06 21:44:38	930
2023-07-06 21:49:42	942
2023-07-06 21:54:45	957
2023-07-06 21:59:49	962
2023-07-06 22:04:54	953
2023-07-06 22:09:58	966
2023-07-06 22:15:01	1004
2023-07-06 22:20:11	1021
2023-07-06 22:25:15	1027
2023-07-06 22:30:18	1059
2023-07-06 22:36:03	995
```

まずは標準出力を行い、データが問題なく転送できるかを確認する。今回使用するコンフィグファイルは下記のような感じ。

```
$ cat embulk_gss_stdout.yml

in:
  type: google_spreadsheets
  auth_method: service_account
  json_keyfile: service_account_key.json
  spreadsheets_url: https://docs.google.com/spreadsheets/d/*****/edit#gid=0
  worksheet_title: minutely
  start_row: 2
  default_timezone: 'Asia/Tokyo'
  null_string: ''
  default_typecast: strict
  columns:
    - { name: Time, type: timestamp, format: '%Y-%m-%d %H:%M:%S' }
    - { name: CO2(ppm), type: long }
out:
  type: stdout

```

標準出力した結果を見ると、`default_timezone: 'Asia/Tokyo'`だとなぜか時間がずれていることがわかる(私の設定ミス)。本来出力されるのは 21,22 時台の情報のはず。

```
$ embulk run embulk_gss_stdout.yml

2023-07-06 22:38:59.724 +0900 [INFO] (0013:task-0000): `embulk-input-google_spreadsheets`: fetched 18 rows in A2:B10001 (tatal: 18 rows)
2023-07-06 12:09:11,781
2023-07-06 12:14:14,839
2023-07-06 12:19:18,882
2023-07-06 12:24:22,901
2023-07-06 12:29:27,917
2023-07-06 12:34:30,927
2023-07-06 12:39:34,925
2023-07-06 12:44:38,930
2023-07-06 12:49:42,942
2023-07-06 12:54:45,957
2023-07-06 12:59:49,962
2023-07-06 13:04:54,953
2023-07-06 13:09:58,966
2023-07-06 13:15:01,1004
2023-07-06 13:20:11,1021
2023-07-06 13:25:15,1027
2023-07-06 13:30:18,1059
2023-07-06 13:36:03,995
2023-07-06 22:38:59.740 +0900 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
```

`default_timezone`は、シート上にあるタイムスタンプの値に関して、どのタイムゾーンで値を解釈するかを指定するものと思われる。そのため、`Asia/Tokyo`とすると、21 時は 9 時間巻き戻って 12 時として Embulk に解釈される。そのため、今回は`UTC`のデフォルト設定で問題ない。

```
in:
  type: google_spreadsheets
  auth_method: service_account
  json_keyfile: service_account_key.json
  spreadsheets_url: https://docs.google.com/spreadsheets/d/*****/edit#gid=0
  worksheet_title: minutely
  start_row: 2
  default_timezone: 'UTC'
  null_string: ''
  default_typecast: strict
  columns:
    - { name: Time, type: timestamp, format: '%Y-%m-%d %H:%M:%S' }
    - { name: CO2(ppm), type: long }
out:
  type: stdout
```

意図したとおりに標準出力されている。

```
$ embulk run embulk_gss_stdout.yml

2023-07-06 22:52:28.459 +0900 [INFO] (0013:task-0000): `embulk-input-google_spreadsheets`: fetched 18 rows in A2:B10001 (tatal: 18 rows)
2023-07-06 21:09:11,781
2023-07-06 21:14:14,839
2023-07-06 21:19:18,882
2023-07-06 21:24:22,901
2023-07-06 21:29:27,917
2023-07-06 21:34:30,927
2023-07-06 21:39:34,925
2023-07-06 21:44:38,930
2023-07-06 21:49:42,942
2023-07-06 21:54:45,957
2023-07-06 21:59:49,962
2023-07-06 22:04:54,953
2023-07-06 22:09:58,966
2023-07-06 22:15:01,1004
2023-07-06 22:20:11,1021
2023-07-06 22:25:15,1027
2023-07-06 22:30:18,1059
2023-07-06 22:36:03,995
2023-07-06 22:52:28.464 +0900 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
```

## :pencil2: GoogleSpreadSheet to MySQL

標準出力で問題がなかったので、MySQL への転送を実行する。ここではデータベース`googlespreadsheet`、テーブル`co2`を作成している。テーブル定義を雑に行っているため、あとで痛い目に遭う。

```
$ mysql -u your_name -h your_host -p
MySQL [(none)]> create database googlespreadsheet;
MySQL [(none)]> use googlespreadsheet;

MySQL [(none)]>
CREATE TABLE co2 (
   id int AUTO_INCREMENT,
   time timestamp,
   co2 int,
  primary key (id)
);
```

さきほどのコンフィグを書き換えて実行する。

```
$ cat embulk_gss_mysql.yml

in:
  type: google_spreadsheets
  auth_method: service_account
  json_keyfile: service_account_key.json
  spreadsheets_url: https://docs.google.com/spreadsheets/d/*****/edit#gid=0
  worksheet_title: minutely
  start_row: 2
  default_timezone: 'UTC'
  null_string: ''
  default_typecast: strict
  columns:
    - { name: Time, type: timestamp, format: '%Y-%m-%d %H:%M:%S' }
    - { name: CO2(ppm), type: long }
out:
  type: mysql
  host: your_host
  user: your_user
  password: your_password
  database: googlespreadsheet
  table: co2
  mode: insert
```

実行結果を見ると、`co2`のカラムが`NULL`になっている。

```
$ embulk run embulk_gss_mysql.yml
(snip)

MySQL [googlespreadsheet]> select * from co2;
+----+---------------------+------+
| id | time                | co2  |
+----+---------------------+------+
|  1 | 2023-07-07 06:09:11 | NULL |
|  2 | 2023-07-07 06:14:14 | NULL |
|  3 | 2023-07-07 06:19:18 | NULL |
|  4 | 2023-07-07 06:24:22 | NULL |
|  5 | 2023-07-07 06:29:27 | NULL |
|  6 | 2023-07-07 06:34:30 | NULL |
|  7 | 2023-07-07 06:39:34 | NULL |
|  8 | 2023-07-07 06:44:38 | NULL |
|  9 | 2023-07-07 06:49:42 | NULL |
| 10 | 2023-07-07 06:54:45 | NULL |
| 11 | 2023-07-07 06:59:49 | NULL |
| 12 | 2023-07-07 07:04:54 | NULL |
| 13 | 2023-07-07 07:09:58 | NULL |
| 14 | 2023-07-07 07:15:01 | NULL |
| 15 | 2023-07-07 07:20:11 | NULL |
| 16 | 2023-07-07 07:25:15 | NULL |
| 17 | 2023-07-07 07:30:18 | NULL |
| 18 | 2023-07-07 07:36:03 | NULL |
+----+---------------------+------+
```

これは GoogleSpreadSheet のカラム名と MySQL のテーブルのカラム名が一致していないためエラーが発生している。MySQL はそもそもカラム名に丸括弧を使用できない。大文字小文字は考慮されないので、`Time`が`time`でも転送ができている。

このあたりの問題を解決するために、スプレッドシートのカラムを`Time > time`、`CO2(ppm) > co2_ppm`に変更する。

```
time	co2_ppm
2023-07-06 21:09:11	781
2023-07-06 21:14:14	839
(snip)
```

MySQL のカラムも変更しておく。

```
MySQL [googlespreadsheet]> alter table co2 change column co2 `co2_ppm` int;
MySQL [googlespreadsheet]> truncate table co2;
```

あわせて`embulk_gss_mysql.yml`の`in`の設定も変更する。

```
$ cat embulk_gss_mysql.yml

in:
  type: google_spreadsheets
  auth_method: service_account
  json_keyfile: service_account_key.json
  spreadsheets_url: https://docs.google.com/spreadsheets/d/*****/edit#gid=0
  worksheet_title: minutely
  start_row: 2
  default_timezone: 'UTC'
  null_string: ''
  default_typecast: strict
  columns:
    - { name: time, type: timestamp, format: '%Y-%m-%d %H:%M:%S' }
    - { name: co2_ppm, type: long }
out:
  type: mysql
  host: your_host
  user: your_user
  password: your_password
  database: googlespreadsheet
  table: co2
  mode: insert

```

再度実行すると、問題なく転送できているようである。

```
$ embulk run embulk_gss_mysql.yml
(snip)

MySQL [googlespreadsheet]> select * from co2;
+----+---------------------+---------+
| id | time                | co2_ppm |
+----+---------------------+---------+
|  1 | 2023-07-07 06:09:11 |     781 |
|  2 | 2023-07-07 06:14:14 |     839 |
|  3 | 2023-07-07 06:19:18 |     882 |
|  4 | 2023-07-07 06:24:22 |     901 |
|  5 | 2023-07-07 06:29:27 |     917 |
|  6 | 2023-07-07 06:34:30 |     927 |
|  7 | 2023-07-07 06:39:34 |     925 |
|  8 | 2023-07-07 06:44:38 |     930 |
|  9 | 2023-07-07 06:49:42 |     942 |
| 10 | 2023-07-07 06:54:45 |     957 |
| 11 | 2023-07-07 06:59:49 |     962 |
| 12 | 2023-07-07 07:04:54 |     953 |
| 13 | 2023-07-07 07:09:58 |     966 |
| 14 | 2023-07-07 07:15:01 |    1004 |
| 15 | 2023-07-07 07:20:11 |    1021 |
| 16 | 2023-07-07 07:25:15 |    1027 |
| 17 | 2023-07-07 07:30:18 |    1059 |
| 18 | 2023-07-07 07:36:03 |     995 |
+----+---------------------+---------+
18 rows in set (0.00 sec)
```

## :closed_book: Reference

- [Google Spreadsheets input plugin for Embulk](https://github.com/medjed/embulk-input-google_spreadsheets)
- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)
- [Embulk の embulk-output-bigquery のインストールエラー回避方法](https://qiita.com/kaaaaaaaaaaai/items/6c2a459236ff2f714dc8)
