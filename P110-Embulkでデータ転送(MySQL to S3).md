## :memo: Overview

ここでは Embulk でデータ転送を行うための基本的な方法をまとめておく。構成としては、AWS 環境下の EC2 で Embulk を実行し、同じ AWS 環境下の MySQL から S3 のバケットに csv を定期的にロードすることが目的。定期実行は cron で行っているが、本来は digdag を利用するのが良いと思われる。

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

- [Embulk](https://www.embulk.org/)
- [Embulk Built-in Plugins](https://www.embulk.org/docs/index.html)
- [Embulk configuration file format](https://www.embulk.org/docs/built-in.html)

余談ですが、Embulk の文字を持ち上げている[クジラのロゴ](https://github.com/embulk/logo/blob/master/embulk-logo.svg)が可愛いですね。

![Embulk](https://github.com/SugiAki1989/sql_note/blob/main/image/p110-embulk.png)

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`Embulk`, `embulk-input-mysql`, `embulk-output-s3`

## :pencil2: AWS の道具立て

まずは Embulk を実行するための AWS 環境の構築を行う。東京リージョンを選択して、下記の手順で構築する。RDS にはイベントスケジューラーを設定して、疑似的に稼働中のシステムのデータベースとする。イベントスケジューラーの設定は 3 年前に書いた[このメモ](https://mysql.hatenablog.jp/entry/2020/01/12/113215)を参照。

- VPC を設定する
  - サブネットを設定する
  - ap-northeast-1a にパブリックサブネット、プライベートサブネットを作成
  - ap-northeast-1c にプライベートサブネットを作成(RDS 用の設定)
  - インターネットゲートウェイを作成し、VPC にアタッチしておく
  - ルーティングを設定する(0.0.0.0/24:インターネットゲートウェイ)
- EC2(Amazon-Linux-2)を作成する
  - パブリックサブネットに設置する
  - t2.micro では Embulk のプラグインがインストールできないので注意
  - セキュリティグループ(ssh-MyHomeIpAddr / http / https)を作成する
  - ElasticIP を EC2 に設定する
- RDS(MySQL8.0)を作成する
  - RDS のセキュリティグループ(MySQL/Aurora:EC2 のセキュリティグループ)を作成する
  - DB サブネットグループで ap-northeast-1a のプライベートサブネット、ap-northeast-1c のプライベートサブネットを設定する
  - パラメタグループを設定する(time_zone:Asia/Tokyo / event_scheduler:on)
  - ap-northeast-1a のプライベートサブネットに設置する
  - パブリックアクセスは不可
  - 各種、セキュリティグループやパラメタグループを作成する。
- AIM のユーザーを作成する
  - Embulk が使用する AIM でアクセスキー、シークレットアクセスキーが必要なのでダウンロードしてメモしておく
  - S3 へのアクセス権限を設定する
- AWS CLI の設定を行う
  - 使用している IAM の情報を登録する
  - AWS アクセスキー、シークレットアクセスキー、デフォルトリージョン名(ap-northeast-1)、デフォルト出力形式(json)を設定する

```
$ aws --version
aws-cli/2.11.27 Python/3.11.3 Darwin/21.6.0 exe/x86_64 prompt/off

# chmod 600 で権限付与
$ ssh -i {path_to_key.pem} ec2-user@11.111.111.111

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

# MySQLにアクセス
$ mysql -h {mysql host} -u {user} -p

mysql> show variables like '%time_zone';
+------------------+------------+
| Variable_name    | Value      |
+------------------+------------+
| system_time_zone | UTC        |
| time_zone        | Asia/Tokyo |
+------------------+------------+

mysql> show variables like 'event%';
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| event_scheduler | ON    |
+-----------------+-------+

mysql> create database event_auto_insert;
mysql> use event_auto_insert;

mysql> create table event_auto_insert.logs (
   id int auto_increment,
    datetime datetime not null,
    value1 int not null,
    value2 int not null,
    category varchar(1) not null,
    flg int not null,
    primary key (id)
);

mysql> create event logs
    on schedule every 10 second
    starts now()
    do insert into event_auto_insert.logs(datetime, value1, value2, category, flg) values (
     now(),
     case when month(now()) in (1,2,3) then floor(rand() * (100 * -1))
     when month(now()) in (4,5,6) then floor(rand() * (10 * -2))
     when month(now()) in (7,8,9) then floor(rand() * (100 * 1))
     else floor(rand() * (10 * 2)) end,
     floor(rand() * 100),
     substring(md5(rand()), 1, 1),
     floor(rand() * 2)
     );

mysql> create index embulk_incremental_loading_index on table (logs);

mysql> select * from event_auto_insert.logs limit 10;

+----+---------------------+--------+--------+----------+-----+
| id | datetime            | value1 | value2 | category | flg |
+----+---------------------+--------+--------+----------+-----+
|  1 | 2023-06-16 13:59:59 |     -2 |     36 | 3        |   0 |
|  2 | 2023-06-16 14:00:09 |    -17 |     89 | f        |   0 |
|  3 | 2023-06-16 14:00:19 |    -10 |     50 | e        |   1 |
|  4 | 2023-06-16 14:00:29 |    -11 |     89 | 9        |   1 |
|  5 | 2023-06-16 14:00:39 |     -7 |     59 | 2        |   1 |
|  6 | 2023-06-16 14:00:49 |     -4 |     94 | 4        |   1 |
|  7 | 2023-06-16 14:00:59 |     -9 |      2 | 4        |   1 |
|  8 | 2023-06-16 14:01:09 |     -4 |     36 | 2        |   0 |
|  9 | 2023-06-16 14:01:19 |     -1 |     84 | 1        |   1 |
| 10 | 2023-06-16 14:01:29 |     -6 |     19 | e        |   1 |
+----+---------------------+--------+--------+----------+-----+

mysql> select * from event_auto_insert.logs order by datetime desc limit 10;
+------+---------------------+--------+--------+----------+-----+
| id   | datetime            | value1 | value2 | category | flg |
+------+---------------------+--------+--------+----------+-----+
| 3064 | 2023-06-18 11:48:59 |    -17 |     90 | 3        |   1 |
| 3063 | 2023-06-18 11:48:49 |     -8 |     95 | c        |   1 |
| 3062 | 2023-06-18 11:48:39 |    -18 |     33 | 7        |   0 |
| 3061 | 2023-06-18 11:48:29 |    -13 |     14 | 4        |   0 |
| 3060 | 2023-06-18 11:48:19 |     -3 |     89 | b        |   0 |
| 3059 | 2023-06-18 11:48:09 |    -12 |     63 | 9        |   0 |
| 3058 | 2023-06-18 11:47:59 |     -2 |     20 | 0        |   1 |
| 3057 | 2023-06-18 11:47:49 |    -13 |     22 | a        |   1 |
| 3056 | 2023-06-18 11:47:39 |     -5 |     66 | 2        |   0 |
| 3055 | 2023-06-18 11:47:29 |     -2 |     67 | 5        |   1 |
+------+---------------------+--------+--------+----------+-----+
10 rows in set (0.01 sec)

```

## :pencil2: EC2 の道具立て

まずは必要なものをインストールしておく。Embulk は java が必要なので、ここでは java8 のバージョンを利用する。また、現時点で最新の Embulk を利用するとプラグインがインストールできないので、バージョンを落としたものを利用している。

ここでは、MySQL からデータを読み込んで、S3 にロードするデータ転送を行いたいので、 `embulk-input-mysql`, `embulk-output-s3`をインストールしている。

```
# setup
$ sudo yum update -y
$ sudo yum install tree
$ sudo yum -y install mysql

# docker は今後利用するので
# $ sudo yum -y install docker
# $ sudo systemctl start docker
# $ sudo systemctl enable docker
# $ sudo gpasswd -a ec2-user docker
# $ exit

# java
$ sudo yum install java-1.8.0-openjdk

# embulk
$ curl --create-dirs -o ~/.embulk/bin/embulk -L "https://dl.embulk.org/embulk-0.9.25.jar"
$ chmod +x ~/.embulk/bin/embulk
$ echo 'export PATH="$HOME/.embulk/bin:$PATH"' >> ~/.bashrc
$ source ~/.bashrc
$ embulk -version
embulk 0.9.25

$ embulk gem install embulk-input-mysql
$ embulk gem install embulk-output-s3
$ embulk gem list
*** LOCAL GEMS ***

bundler (1.16.0)
did_you_mean (default: 1.0.1)
embulk (0.9.25 java)
embulk-input-mysql (0.13.2 java)
embulk-output-s3 (1.7.1 java)
jar-dependencies (default: 0.3.10)
jruby-openssl (0.9.21 java)
jruby-readline (1.2.0 java)
json (1.8.3 java)
liquid (4.0.0)
minitest (default: 5.4.1)
msgpack (1.1.0 java)
net-telnet (default: 0.1.1)
power_assert (default: 0.2.3)
psych (2.2.4 java)
rake (default: 10.4.2)
rdoc (default: 4.2.0)
test-unit (default: 3.1.1)
```

## :pencil2: Embulk の道具立て

最終的な状態は下記の通りで、シェルファイルを実行することで Embulk を動かす。vim をつかいこなせないので、各 yml ファイルはローカル環境から`scp`で転送している。

```
# scp ~/Desktop/embulk_mysql_s3.yml.liquid ec2-user@11.111.111.111:~/
$ tree ./
./
├── embulk.diff.yml
├── embulk_mysql_s3.yml.liquid
├── embulk_mysql_s3_bk.yml.liquid
└── exec_embulk_mysql.sh
```

- `embulk.diff.yml`は差分み込みに必要なファイル
- `embulk_mysql_s3.yml.liquid`は Embulk のコンフィグファイル
- `embulk_mysql_s3_bk.yml.liquid`は実行時に前回の認証ファイルのバックアップ
- `exec_embulk_mysql.sh`は Embulk を実行するためのシェルファイル

`embulk_mysql_s3.yml.liquid`の中身は下記の通り。ここでは増分ロードを`id`をキーに設定している。
`exec`で指定しているのはタスクを 1 つにまとめ、ファイルが 1 つになるようにするためで、1 回の実行で、毎回 S3 になぜか毎回 2 つの csv ファイルが生成されてしまうのを防ぐため。1 つ目の csv にはデータが入っているのに、2 つ目の csv はヘッダー行だけで生成されていた。解決方法がわからなかったので、とりあえずこうしている。

```
in:
  type: mysql
  host: {{ env.mysql_host }}
  user: {{ env.mysql_user }}
  password: {{ env.mysql_password }}
  database: {{ env.mysql_database }}
  table: logs
  select: 'id, datetime, value1, category, flg'
  where: flg = 1
  # order_by: id ASC
  # query: select id, datetime, value1, category, flg from logs order by id asc
  # use_raw_query_with_incremental: true
  incremental: true
  incremental_columns:
    - id
  options:
    serverTimezone: Asia/Tokyo
  fetch_rows: 10000
  columns:
    - { name: id, type: long }
    - { name: datetime, type: timestamp, format: '%Y-%m-%d %H:%M:%S' }
    - { name: value1, type: long }
    - { name: category, type: string }
    - { name: flg, type: long }
out:
  type: s3
  endpoint: {{ env.s3_endpoint }}
  bucket: {{ env.s3_bucket }}
  access_key_id: {{ env.s3_key1 }}
  secret_access_key: {{ env.s3_key2 }}
  path_prefix: out/data　## ここが毎回タイムスタンプつきに書き換わる
  sequence_format: ''
  file_ext: .csv
  formatter:
    type: csv
    delimiter: ','
    newline: LF
    newline_in_field: LF
    charset: UTF-8
    quote_policy: ALL
    quote: '"'
    escape: '"'
    null_string: ''
    default_timezone: Asia/Tokyo

exec:
  min_output_tasks: 1
  max_threads: 1
```

`exec_embulk_mysql.sh`の中身は下記の通り。`chmod +x exec_embulk_mysql.sh`で実行権限も忘れずに。

ここでも問題があって、実行するたびに S3 に異なる名前の csv をエクスポートしたかったが、それができず、毎回上書き更新されてしまう問題を解決できていない。ドキュメントを読んでみたものの解決方法が分からなかったので、シェルファイルスクリプトで強引に解決している。

また cron で呼び出すため`/home/ec2-user/.embulk/bin/embulk run`としており、crontab 内で PATH 定義をおこなってない。

```
#!/bin/bash

# Set environment variables
export mysql_host=your_host.ap-northeast-1.rds.amazonaws.com
export mysql_user=your_user
export mysql_password=your_password
export mysql_database=event_auto_insert

export s3_endpoint=s3-ap-northeast-1.amazonaws.com
export s3_bucket=your_s3_bucket
export s3_key1=your_access_key
export s3_key2=your_secret_key

# Add timestamp to filenames loaded at runtime
timestamp=$(TZ=Asia/Tokyo date +'%Y%m%d%H%M%S')
new_path_prefix="out/data_${timestamp}"
sed -i "s|path_prefix:.*|path_prefix: ${new_path_prefix}|" embulk_mysql_s3.yml.liquid

# embulk_mysql_s3.yml.liquid
/home/ec2-user/.embulk/bin/embulk run -c embulk.diff.yml embulk_mysql_s3.yml.liquid
cp -r embulk_mysql_s3.yml.liquid embulk_mysql_s3_bk.yml.liquid
```

あとはこれを実行するだけ。とりあえず、動くかどうか確認したいので、1 分ごとに実行する。

```
$ crontab -l
* * * * * /home/ec2-user/exec_embulk_mysql.sh
```

数分間放置しておくと、下記のように S3 にデータが転送されている。

```
$ aws s3 ls s3://embulk-mysql-to-s3/out/

2023-06-18 10:03:04      68401 data_20230618100255.csv # 手動実行で転送した分
2023-06-18 10:17:49       2377 data_20230618101739.csv # 手動実行で転送した分
2023-06-18 10:20:10        540 data_20230618102001.csv # 手動実行で転送した分
2023-06-18 10:30:10       1992 data_20230618103001.csv # 以降、cronで転送した分
2023-06-18 10:31:10        208 data_20230618103101.csv
2023-06-18 10:32:11         97 data_20230618103201.csv
2023-06-18 10:33:11        320 data_20230618103301.csv
2023-06-18 10:34:10        266 data_20230618103401.csv
2023-06-18 10:35:11        209 data_20230618103501.csv
2023-06-18 10:36:11        265 data_20230618103601.csv
2023-06-18 10:37:11        154 data_20230618103701.csv
2023-06-18 10:38:11        265 data_20230618103801.csv
2023-06-18 10:39:11        263 data_20230618103901.csv
2023-06-18 10:40:10        152 data_20230618104001.csv
2023-06-18 10:41:10        152 data_20230618104101.csv
2023-06-18 10:42:11        264 data_20230618104201.csv
2023-06-18 10:43:11        318 data_20230618104301.csv
2023-06-18 10:44:11        210 data_20230618104401.csv
2023-06-18 10:45:10        209 data_20230618104501.csv
```

1 番最初に転送した分の冒頭 10 行と最後に転送したデータを見ておく。MySQL のクエリの内容と値が異なるのは、ただこのメモを作成するために MySQL を放ったらかしにしていた間、イベントスケジューラーがもりもりレコードを生成したため。

```
$ aws s3 cp s3://embulk-mysql-to-s3/out/data_20230618100255.csv -

"id","datetime","value1","category","flg"
"3","2023-06-16 14:00:19.000000 +0900","-10","e","1"
"4","2023-06-16 14:00:29.000000 +0900","-11","9","1"
"5","2023-06-16 14:00:39.000000 +0900","-7","2","1"
"6","2023-06-16 14:00:49.000000 +0900","-4","4","1"
"7","2023-06-16 14:00:59.000000 +0900","-9","4","1"
"9","2023-06-16 14:01:19.000000 +0900","-1","1","1"
"10","2023-06-16 14:01:29.000000 +0900","-6","e","1"

$ aws s3 cp s3://embulk-mysql-to-s3/out/data_20230618104501.csv -
"id","datetime","value1","category","flg"
"2677","2023-06-18 10:44:29.000000 +0900","-19","e","1"
"2679","2023-06-18 10:44:49.000000 +0900","-13","5","1"
"2680","2023-06-18 10:44:59.000000 +0900","-8","8","1"
```

## :closed_book: Reference

- [Embulk](https://www.embulk.org/)
- [Embulk Built-in Plugins](https://www.embulk.org/docs/index.html)
- [Embulk configuration file format](https://www.embulk.org/docs/built-in.html)
- [ETL 基盤を構築してみた 〜Embulk 編〜](https://tech.griphone.co.jp/2018/12/04/advent-calendar-20181204/)
- [GCE で digdag+embulk の ETL 基盤を構築してみた](https://qiita.com/k_0120/items/0313499451da240cce91)
- [Digdag MySQL Tutorial](http://www.alphasentaurii.com/programming/2020/06/07/digdag-mysql-tutorial.html)
- [Digdag PostgreSQL Tutorial](http://www.alphasentaurii.com/programming/2020/07/07/digdag-postgresql-tutorial.html)
