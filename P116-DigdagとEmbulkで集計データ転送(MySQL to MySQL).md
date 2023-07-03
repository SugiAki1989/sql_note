## :memo: Overview

ここでは Embulk でデータ転送を行うための基本的な方法をまとめておく。構成としては、AWS 環境下の EC2 で Embulk を実行し、同じ AWS 環境下の MySQL から MySQL に 集計データ を digdag で定期的にロードする構成。作業を日を分けて実行したので MySQL の値の日時が 1 日経過してたりするのはそのせい。

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`Embulk`, `Digdag`, `embulk-output-mysql`

## :pencil2: Docker ファイルの準備

これまでのノートで使用しているファイル類をローカルで編集し、scp で EC2 に転送するところから始める。最終的なファイル構成は下記の通り。

```
$ ls
bin embulk_etl.dig embulk_mysql_mysql.yml.liquid exec_embulk_mysql.sh
```

`embulk_mysql_mysql.yml.liquid`では、SQL や定期的に動くかを検証したいので、`stdout`を出力に設定しておく。

```
$ cat embulk_mysql_mysql.yml.liquid

in:
  type: mysql
  host: {{ env.mysql_host }}
  user: {{ env.mysql_user }}
  password: {{ env.mysql_password }}
  database: {{ env.mysql_database }}
  query: |
    select
      date_format(datetime, '%y-%m-%d %H:%i:00') as dt_minute,
      min(id) as min_id,
      max(id) as max_id,
      min(date_format(now() - interval 1 minute, '%y-%m-%d %H:%i:00')) as start,
      max(date_format(now(), '%y-%m-%d %H:%i:00')) as end,
      count(1) as count
    from
      logs
    where
      datetime >= date_format(now() - interval 1 minute, '%y-%m-%d %H:%i:00') and
      datetime < date_format(now(), '%y-%m-%d %H:%i:00')
    group by
      dt_minute
  options:
    serverTimezone: Asia/Tokyo
  columns:
    - { name: id, type: long }
    - { name: datetime, type: timestamp, format: '%Y-%m-%d %H:%M:%S' }
    - { name: value1, type: long }
    - { name: category, type: string }
    - { name: flg, type: long }
out:
  type: stdout
```

SQL の部分は、タスクが実行された時間の 1 分前のデータを集計するという内容の SQL。目的としていることが実行できれば良いので、このような内容にとりあえずしている。本来 1 分間隔でデータを転送するのであれば、ツール含めストリームデータ処理を検討するべきだし、このような方法だとバッチ処理の冪等性の問題もある。ただ、ここでは学習のために集計したデータを定期的にロードすることができればよいので、そのあたりは気にしない。

```
select
  date_format(datetime, '%y-%m-%d %H:%i:00') as dt_minute,
  min(id) as min_id,
  max(id) as max_id,
  min(date_format(now() - interval 1 minute, '%y-%m-%d %H:%i:00')) as start,
  max(date_format(now(), '%y-%m-%d %H:%i:00')) as end,
  count(1) as count
from
  logs
where
  datetime >= date_format(now() - interval 1 minute, '%y-%m-%d %H:%i:00') and
  datetime < date_format(now(), '%y-%m-%d %H:%i:00')
group by
  dt_minute

+----+---------------------+--------+--------+---------------------+---------------------+-------+
| id | dt_minute           | min_id | max_id | start               | end                 | count |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
|  1 | 2023-06-29 21:07:00 |    133 |    138 | 2023-06-29 21:07:00 | 2023-06-29 21:08:00 |     6 |
+----+---------------------+--------+--------+---------------------+---------------------+-------+

```

`exec_embulk_mysql.sh`については、前回からあまり変更はなく、S3 の情報を削除しただけ。S3 に転送するファイル名を、実行のたびに変更するシェルスクリプトを削除したので、`embulk_etl.dig`から直接`embulk>`オペレーターで`yml.liquid`を呼び出す形でも問題はなさそう。

```
$ cat exec_embulk_mysql.sh

#!/bin/bash

# Set environment variables
export mysql_host=your_host.ap-northeast-1.rds.amazonaws.com
export mysql_user=your_user
export mysql_password=your_password
export mysql_database=event_auto_insert

# embulk_mysql_mysql.yml.liquid
/home/ec2-user/.embulk/bin/embulk run embulk_mysql_mysql.yml.liquid
```

`embulk_etl.dig`は、SQL や定期的に動くかを検証したいので、`stdout`の出力に開始と終了の時間を表示させる。

```
$ cat embulk_etl.dig

timezone: 'Asia/Tokyo'

schedule:
  cron>: '* * * * *'

+start:
  echo>: "Start Data Transfer: ${moment(session_time).format('YYYY-MM-DD HH:mm:ss Z')}"

+task1:
  sh>: /home/ec2-user/exec_embulk_mysql.sh

+end:
  echo>: "End Data Transfer: ${moment(session_time).format('YYYY-MM-DD HH:mm:ss Z')}"
```

最後に実行権限をつけて、スケジューラーを起動する。

```
$ chmod 755 exec_embulk_mysql.sh
```

出力結果を見る限り、特に問題は起こってなさそう。ログの必要な部分だけ取り出している。

```
$ digdag scheduler

(snip)
Start Data Transfer: 2023-06-29 20:45:00 +09:00
23-06-29 20:44:00,5589,5593,23-06-29 20:44:00,23-06-29 20:45:00,5
End Data Transfer: 2023-06-29 20:45:00 +09:00
(snip)
Start Data Transfer: 2023-06-29 20:46:00 +09:00
23-06-29 20:45:00,5594,5600,23-06-29 20:45:00,23-06-29 20:46:00,7
End Data Transfer: 2023-06-29 20:46:00 +09:00
(snip)
Start Data Transfer: 2023-06-29 20:47:00 +09:00
23-06-29 20:46:00,5601,5606,23-06-29 20:46:00,23-06-29 20:47:00,6
End Data Transfer: 2023-06-29 20:47:00 +09:00
(snip)
Start Data Transfer: 2023-06-29 20:48:00 +09:00
23-06-29 20:47:00,5607,5612,23-06-29 20:47:00,23-06-29 20:48:00,6
End Data Transfer: 2023-06-29 20:48:00 +09:00
(snip)
Start Data Transfer: 2023-06-29 20:49:00 +09:00
23-06-29 20:48:00,5613,5618,23-06-29 20:48:00,23-06-29 20:49:00,6
End Data Transfer: 2023-06-29 20:49:00 +09:00
(snip)
Start Data Transfer: 2023-06-29 20:50:00 +09:00
23-06-29 20:49:00,5619,5624,23-06-29 20:49:00,23-06-29 20:50:00,6
End Data Transfer: 2023-06-29 20:50:00 +09:00
(snip)
Start Data Transfer: 2023-06-29 20:51:00 +09:00
23-06-29 20:50:00,5625,5630,23-06-29 20:50:00,23-06-29 20:51:00,6
End Data Transfer: 2023-06-29 20:51:00 +09:00
```

スケジューラーを実行する前に、MySQL にロードするために下記のプラグインが必要になるので、インストールして使用可能な状態にしておく。

```
$ embulk gem install embulk-output-mysql
```

`embulk_mysql_mysql.yml.liquid`の`out`の部分を書き直して、定期的に MySQL の`summary_logs`テーブルにロードされるようにする。インサート先のテーブルは予め作成しておいた。

```
out:
  type: mysql
  host: {{ env.mysql_host }}
  user: {{ env.mysql_user }}
  password: {{ env.mysql_password }}
  database: {{ env.mysql_database }}
  table: summary_logs
  mode: insert
```

これでスケジューラを起動して、実行すると問題が発生する。インサートすると日時関係のカラムが`NULL`になってしまう。

```
MySQL [event_auto_insert]> select * from summary_logs;
+----+-----------+--------+--------+-------+------+-------+
| id | dt_minute | min_id | max_id | start | end  | count |
+----+-----------+--------+--------+-------+------+-------+
|  1 | NULL      |     19 |     24 | NULL  | NULL |     6 |
|  2 | NULL      |     25 |     30 | NULL  | NULL |     6 |
|  3 | NULL      |     31 |     36 | NULL  | NULL |     6 |
|  4 | NULL      |     37 |     42 | NULL  | NULL |     6 |
|  5 | NULL      |     43 |     48 | NULL  | NULL |     6 |
|  6 | NULL      |     49 |     54 | NULL  | NULL |     6 |
|  7 | NULL      |     55 |     60 | NULL  | NULL |     6 |
|  8 | NULL      |     61 |     66 | NULL  | NULL |     6 |
|  9 | NULL      |     67 |     72 | NULL  | NULL |     6 |
| 10 | NULL      |     73 |     78 | NULL  | NULL |     6 |
| 11 | NULL      |     79 |     84 | NULL  | NULL |     6 |
+----+-----------+--------+--------+-------+------+-------+
```

下記のプラグインのドキュメントやサイトの情報を読んでいるといくつか考慮が抜けていたことがわかった。

- [MySQL input plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)
- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-output-jdbc/blob/master/embulk-output-mysql/README.md)
- [Embulk を使った分析基盤のデータ型について – データ分析入門シリーズ](https://pzgleaner.com/archives/embulk-type)
- [embulk で MySQL にコピーする時にハマったこと](https://qiita.com/s5601026/items/61c3a2f24d78ea296680)

`embulk-input-mysql`は下記の点で注意する必要がある。MySQL の値は MySQL スキーマのデータ型である`long`、`double`、`float`、`decimal`、`boolean`、`string`、`json`、`date`、`time`、`timestamp` として読み込むことができて、Embulk のスキーマのデータ型`boolean`、`long`、`double`、`string`、`json`、`timestamp`のいずれかに変換される。つまり、MySQL から`value_type`で指定した型で取得し、`type`で指定した embulk の型に変換される。

```
column_options: advanced: key-value pairs where key is a column name and value is options for the column.

  value_type: embulk get values from database as this value_type. Typically, the value_type determines getXXX method of java.sql.PreparedStatement. (string, default: depends on the sql type of the column. Available values options are: long, double, float, decimal, boolean, string, json, date, time, timestamp)

  type: Column values are converted to this embulk type. Available values options are: boolean, long, double, string, json, timestamp). By default, the embulk type is determined according to the sql type of the column (or value_type if specified).

  timestamp_format: If the sql type of the column is date/time/datetime and the embulk type is string, column values are formatted by this timestamp_format. And if the embulk type is timestamp, this timestamp_format may be used in the output plugin. For example, stdout plugin use the timestamp_format, but csv formatter plugin doesn't use. (string, default : %Y-%m-%d for date, %H:%M:%S for time, %Y-%m-%d %H:%M:%S for timestamp)

  timezone: If the sql type of the column is date/time/datetime and the embulk type is string, column values are formatted in this timezone. (string, value of default_timezone option is used by default)
```

同じく`embulk-output-mysql`は下記の点で注意する必要がある。`value_type`の通り、入力カラムの型（embulk 型）をデータベースの型に変換し、INSERT 文を作成する。その際、`value_type`オプションで INSERT 文の値の型を制御できる。利用可能な値オプションは`byte`、`short`、`int`、`long`、`double`、`float`、`boolean`、`string`、`nstring`、`date`、`time`、`timestamp`、`decimal`、`json`、`null`、`pass`とのこと。

```
column_options: advanced: a key-value pairs where key is a column name and value is options for the column.

  type: type of a column when this plugin creates new tables (e.g. VARCHAR(255), INTEGER NOT NULL UNIQUE). This used when this plugin creates intermediate tables (insert, insert_truncate and merge modes), when it creates the target table (insert_direct, merge_direct and replace modes), and when it creates nonexistent target table automatically. (string, default: depends on input column type. BIGINT if input column type is long, BOOLEAN if boolean, DOUBLE PRECISION if double, CLOB if string, TIMESTAMP if timestamp)

  value_type: This plugin converts input column type (embulk type) into a database type to build a INSERT statement. This value_type option controls the type of the value in a INSERT statement. (string, default: depends on the sql type of the column. Available values options are: byte, short, int, long, double, float, boolean, string, nstring, date, time, timestamp, decimal, json, null, pass)

  timestamp_format: If input column type (embulk type) is timestamp and value_type is string or nstring, this plugin needs to format the timestamp value into a string. This timestamp_format option is used to control the format of the timestamp. (string, default: %Y-%m-%d %H:%M:%S.%6N)

  timezone: If input column type (embulk type) is timestamp, this plugin needs to format the timestamp value into a SQL string. In this cases, this timezone option is used to control the timezone. (string, value of default_timezone option is used by default)
```

まず、MySQL のテーブルは`datetime`型だったので、`timestamp`型に修正する。

```
MySQL [event_auto_insert]>
CREATE TABLE event_auto_insert.logs (
   id int AUTO_INCREMENT,
    datetime timestamp NOT NULL,
    value1 int NOT NULL,
    value2 int NOT NULL,
    category varchar(1) NOT NULL,
    flg int NOT NULL,
    PRIMARY KEY (id)
);

MySQL [event_auto_insert]>
create table summary_logs (
  id int auto_increment,
  dt_minute timestamp,
  min_id int,
  max_id int,
  start timestamp,
  end timestamp,
  count int,
  primary key (id)
);
Query OK, 0 rows affected (0.05 sec)
```

素直にすべて`timestamp`に設定してみると、どうやらうまくいっていない。

```
in:
  (snip)
  column_options:
    datetime: {type: timestamp, value_type: timestamp}
out:
  (snip)
  column_options:
    dt_minute: {type: timestamp, value_type: timestamp}
    start: {type: timestamp, value_type: timestamp}
    end: {type: timestamp, value_type: timestamp}

+----+---------------------+--------+--------+---------------------+---------------------+-------+
| id | dt_minute           | min_id | max_id | start               | end                 | count |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
|  2 | NULL                |    284 |    289 | NULL                | NULL                |     6 |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
```

MySQL から`value_type`で指定した`timestamp`型で取得し、`type`で指定した embulk の`string`型に変換してみる。これでは駄目。

```
in:
  (snip)
  column_options:
    datetime: {type: string, value_type: timestamp}
out:
  (snip)
  column_options:
    dt_minute: {type: timestamp, value_type: timestamp}
    start: {type: timestamp, value_type: timestamp}
    end: {type: timestamp, value_type: timestamp}

+----+---------------------+--------+--------+---------------------+---------------------+-------+
| id | dt_minute           | min_id | max_id | start               | end                 | count |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
|  4 | NULL                |    338 |    343 | NULL                | NULL                |     6 |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
```

MySQL から`in`の`value_type`で`timestamp`型で取得し、`type`で embulk の`string`型に変換、そして、`out`の`value_type`で Embulk の`string`型で取得し、`type`で`timestamp`型に変換してインサートする。これであれば問題なく転送できる。イメージは`in > timestamp > string > out > string > timestamp`というような感じ(あってるのだろうか？)。

```
in:
  (snip)
  column_options:
    datetime: {type: string, value_type: timestamp}
out:
  (snip)
  column_options:
    dt_minute: {type: timestamp, value_type: string}
    start: {type: timestamp, value_type: string}
    end: {type: timestamp, value_type: string}

+----+---------------------+--------+--------+---------------------+---------------------+-------+
| id | dt_minute           | min_id | max_id | start               | end                 | count |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
|  5 | 2023-06-30 21:34:00 |    356 |    361 | 2023-06-30 21:34:00 | 2023-06-30 21:35:00 |     6 |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
```

最終的な`embulk_mysql_mysql.yml.liquid`は下記の通り。

```
$ cat embulk_mysql_mysql.yml.liquid
in:
  type: mysql
  host: {{ env.mysql_host }}
  user: {{ env.mysql_user }}
  password: {{ env.mysql_password }}
  database: {{ env.mysql_database }}
  query: |
    select
      date_format(datetime, '%y-%m-%d %H:%i:00') as dt_minute,
      min(id) as min_id,
      max(id) as max_id,
      min(date_format(now() - interval 1 minute, '%y-%m-%d %H:%i:00')) as start,
      max(date_format(now(), '%y-%m-%d %H:%i:00')) as end,
      count(1) as count
    from
      logs
    where
      datetime >= date_format(now() - interval 1 minute, '%y-%m-%d %H:%i:00') and
      datetime < date_format(now(), '%y-%m-%d %H:%i:00')
    group by
      dt_minute
  options:
    serverTimezone: Asia/Tokyo
  columns:
    - { name: id, type: long }
    - { name: datetime, type: timestamp, format: '%Y-%m-%d %H:%M:%S' }
    - { name: value1, type: long }
    - { name: category, type: string }
    - { name: flg, type: long }
  column_options:
    datetime: {type: string, value_type: timestamp}
out:
  type: mysql
  host: {{ env.mysql_host }}
  user: {{ env.mysql_user }}
  password: {{ env.mysql_password }}
  database: {{ env.mysql_database }}
  table: summary_logs
  mode: insert
  column_options:
    dt_minute: {type: timestamp, value_type: string}
    start: {type: timestamp, value_type: string}
    end: {type: timestamp, value_type: string}
```

これで問題なく転送できそうなので、MySQL の転送先のテーブルを掃除しておく。

```
MySQL [event_auto_insert]> truncate table summary_logs;
```

実行した結果はこちら。目的にしていたログを抽出して、集計してから転送することができている。

```
[EC2]
$ digdag scheduler

MySQL [event_auto_insert]> select * from summary_logs;
+----+---------------------+--------+--------+---------------------+---------------------+-------+
| id | dt_minute           | min_id | max_id | start               | end                 | count |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
|  1 | 2023-06-30 21:39:00 |    386 |    391 | 2023-06-30 21:39:00 | 2023-06-30 21:40:00 |     6 |
|  2 | 2023-06-30 21:40:00 |    392 |    397 | 2023-06-30 21:40:00 | 2023-06-30 21:41:00 |     6 |
|  3 | 2023-06-30 21:41:00 |    398 |    403 | 2023-06-30 21:41:00 | 2023-06-30 21:42:00 |     6 |
|  4 | 2023-06-30 21:42:00 |    404 |    409 | 2023-06-30 21:42:00 | 2023-06-30 21:43:00 |     6 |
|  5 | 2023-06-30 21:43:00 |    410 |    415 | 2023-06-30 21:43:00 | 2023-06-30 21:44:00 |     6 |
|  6 | 2023-06-30 21:44:00 |    416 |    421 | 2023-06-30 21:44:00 | 2023-06-30 21:45:00 |     6 |
|  7 | 2023-06-30 21:45:00 |    422 |    427 | 2023-06-30 21:45:00 | 2023-06-30 21:46:00 |     6 |
|  8 | 2023-06-30 21:46:00 |    428 |    433 | 2023-06-30 21:46:00 | 2023-06-30 21:47:00 |     6 |
|  9 | 2023-06-30 21:47:00 |    434 |    439 | 2023-06-30 21:47:00 | 2023-06-30 21:48:00 |     6 |
| 10 | 2023-06-30 21:48:00 |    440 |    445 | 2023-06-30 21:48:00 | 2023-06-30 21:49:00 |     6 |
| 11 | 2023-06-30 21:49:00 |    446 |    451 | 2023-06-30 21:49:00 | 2023-06-30 21:50:00 |     6 |
| 12 | 2023-06-30 21:50:00 |    452 |    457 | 2023-06-30 21:50:00 | 2023-06-30 21:51:00 |     6 |
| 13 | 2023-06-30 21:51:00 |    458 |    463 | 2023-06-30 21:51:00 | 2023-06-30 21:52:00 |     6 |
| 14 | 2023-06-30 21:52:00 |    464 |    469 | 2023-06-30 21:52:00 | 2023-06-30 21:53:00 |     6 |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
14 rows in set (0.00 sec)
```

## :closed_book: Reference

- [MySQL input plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)
- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-output-jdbc/blob/master/embulk-output-mysql/README.md)
- [Embulk を使った分析基盤のデータ型について – データ分析入門シリーズ](https://pzgleaner.com/archives/embulk-type)
- [embulk で MySQL にコピーする時にハマったこと](https://qiita.com/s5601026/items/61c3a2f24d78ea296680)
