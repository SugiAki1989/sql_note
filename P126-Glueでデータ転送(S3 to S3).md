## :memo: Overview

ここでは AWS の glue を利用して S3 にあるデータを Python で変換して S3 に転送する方法をまとめておく。AWS の様々なプロダクトを組み合わせれば、いろんな処理を実行できるが、ここでは簡単な ETL の方法をまとめている。

- [AWS Glue](https://aws.amazon.com/jp/glue/)

ちなみに書いている本人は、データエンジニアではないので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

## :floppy_disk: Database

S3

## :bookmark: Tag

`glue`, `python`

## :pencil2:　 Glue でデータ転送 (S3 to S3)

今回は AWS の glue を利用して S3 にあるデータを Python で変換して、 S3 にデータ転送する方法をまとめる。AWS CLI を利用するので、下記のページを参考に設定を行っておく。

- [AWS CLI の設定](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-chap-configure.html)

```
$ curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
$ sudo installer -pkg AWSCLIV2.pkg -target /

$ which aws
/usr/local/bin/aws

$ aws --version
aws-cli/2.11.27 Python/3.11.3 Darwin/21.6.0 exe/x86_64 prompt/off

$ aws configure
AWS Access Key ID [None]: ****************IAMで調べる
AWS Secret Access Key [None]: ****************IAMで調べる
Default region name [None]: ap-northeast-1
Default output format [None]: json
```

Embulk の転送で利用した S3 バケットが確認できるので、問題なく CLI の設定が完了している。

```
$ aws s3 ls
2023-06-17 14:21:19 embulk-mysql-to-s3
```

お次は転送元となる S3 バケット`data-from-123456789` と転送先となる S3 バケット`data-to-123456789`の 2 つを作成する。S3 バケットの作り方は調べればすぐ見つかるので、ここでは省略。

```
$ aws s3 ls
2023-07-12 21:25:57 data-from-123456789
2023-07-12 21:26:15 data-to-123456789
2023-06-17 14:21:19 embulk-mysql-to-s3
```

転送元にサンプルデータを保存しておく。手元にあった自室の CO2 センサーのログをアップロードしておく。

```
$ aws s3 cp ~/Desktop/myroom_co2.csv s3://data-from-123456789/
upload: ../../Desktop/myroom_co2.csv to s3://data-from-123456789/myroom_co2.csv

$ aws s3 ls data-from-123456789/
2023-07-12 21:31:43       1037 myroom_co2.csv

$ aws s3 cp s3://data-from-123456789//myroom_co2.csv - | head -n 10
time,co2_ppm
2023-07-12 18:00:33,870
2023-07-12 18:05:37,839
2023-07-12 18:10:41,806
2023-07-12 18:15:44,802
2023-07-12 18:20:48,795
2023-07-12 18:25:52,808
2023-07-12 18:30:56,792
2023-07-12 18:36:00,764
2023-07-12 18:41:04,651
```

データの転送元、転送先の準備が完了したので、次に Glue を使って ETL を構築していく。Glue の画面左のナビから「ETL jobs」を選択し、「Python Shell script editor」を選択。これで Python スクリプトを書いてデータ変換ができる。

「job details」から最低限必要なものを設定しておく。IAM ロールは`AWSGlueServiceRole`をアタッチした IAM を使用する(本来は最低限の権限を設定する)。スクリプトパスには glue で実行するための ptyhon スクリプトが保存される。
また、`Libraries - Python library path`にライブラリを保存しておけば、実行時に利用できる。

```
[Basic properties]
Name: python-etl
Description: Convert data in python script to transfer data from s3 to s3
IAM Role: `AWSGlueServiceRole`をアタッチしたIAM
Type: Python Shell
Python version: Python3.9

[Advanced properties]
Script filename: python-etl.py
Script path: s3://aws-glue-assets-111111111111-ap-northeast-1/scripts/
```

「Script」に下記の簡単な処理を記述する。ここでは、`co2_ppm`が 800 以上だと 1 となるフラグを新たに追加する。変換後は、転送先の S3 に転送する。

```
# Load library
import pandas
import s3fs

# In-Out Setting
data_s3_from = 's3://data-from-123456789/myroom_co2.csv'
data_s3_to = 's3://data-to-123456789/myroom_co2_transformed.csv'

# Extract
co2_df = pandas.read_csv(data_s3_from)

# Transform
co2_df['is800over'] = co2_df['co2_ppm'].apply(lambda x: 1 if x >= 800 else 0)

# Load
co2_df.to_csv(data_s3_to)
```

「Run」でスクリプトを実行すると、変換されたデータが S3 に保存される。転送されたデータには`is800over`というカラムが追加されている。

```
$ aws s3 cp s3://data-to-123456789/myroom_co2_transformed.csv - | head -n 10
,time,co2_ppm,is800over
0,2023-07-12 18:00:33,870,1
1,2023-07-12 18:05:37,839,1
2,2023-07-12 18:10:41,806,1
3,2023-07-12 18:15:44,802,1
4,2023-07-12 18:20:48,795,0
5,2023-07-12 18:25:52,808,1
6,2023-07-12 18:30:56,792,0
7,2023-07-12 18:36:00,764,0
8,2023-07-12 18:41:04,651,0
```

スケジュール実行も可能で、「Schedules」からスケジュールを設定する。スケジュールの時間は UTC で設定する必要があるのと、cron の書き方が少し異なるので注意。例えば現在の時間は、

```
$ date
2023年 7月12日 水曜日 22時11分55秒 JST
```

なので、JST で 22:15:00 に実行したければ、9 時間前の`15 13 * * ? *`と設定する必要がある。

```
Name: python-etl-schedule
Frequency: Custom
CRON expression: 15 13 * * ? *
Description: Schedule JST 22:15:00
```

スケジュールが実行される前に、転送先の csv を削除しておく。

```
$ aws s3 rm s3://data-to-123456789/myroom_co2_transformed.csv
delete: s3://data-to-123456789/myroom_co2_transformed.csv

$ aws s3 ls data-to-123456789/
何もない
```

時間になったので、スケジュールが実行されているかログから確認すると、ステータスも完了となっているので、問題なく転送された模様。

```
$ aws glue get-job-runs --job-name 'python-etl'
{
    "JobRuns": [
        {
            "Id": "jr_64433d15e86e4cf3fcf01a2c90f263589a872092fce61449",
            "Attempt": 0,
            "TriggerName": "python-etl-schedule",
            "JobName": "python-etl",
            "StartedOn": "2023-07-12T22:15:39.208000+09:00",
            "LastModifiedOn": "2023-07-12T22:16:00.706000+09:00",
            "CompletedOn": "2023-07-12T22:16:00.706000+09:00",
            "JobRunState": "SUCCEEDED",
            "PredecessorRuns": [],
            "AllocatedCapacity": 0,
            "ExecutionTime": 14,
            "Timeout": 2880,
            "MaxCapacity": 0.0625,
            "LogGroupName": "/aws-glue/python-jobs",
            "GlueVersion": "3.0",
            "ExecutionClass": "STANDARD"
        },
(snip)
```

転送先の S3 バケットのデータを見ても、さきほどと同じく変換されたデータが転送されている。

```
$ aws s3 ls data-to-123456789/
2023-07-12 22:15:51       1203 myroom_co2_transformed.csv

$ aws s3 cp s3://data-to-123456789/myroom_co2_transformed.csv - | head -n 10
,time,co2_ppm,is800over
0,2023-07-12 18:00:33,870,1
1,2023-07-12 18:05:37,839,1
2,2023-07-12 18:10:41,806,1
3,2023-07-12 18:15:44,802,1
4,2023-07-12 18:20:48,795,0
5,2023-07-12 18:25:52,808,1
6,2023-07-12 18:30:56,792,0
7,2023-07-12 18:36:00,764,0
8,2023-07-12 18:41:04,651,0
```

## :closed_book: Reference

- [AWS Glue](https://aws.amazon.com/jp/glue/)
- [AWS ではじめるデータレイク: クラウドによる統合型データリポジトリ構築入門](https://www.amazon.co.jp/AWS%E3%81%A7%E3%81%AF%E3%81%98%E3%82%81%E3%82%8B%E3%83%87%E3%83%BC%E3%82%BF%E3%83%AC%E3%82%A4%E3%82%AF-%E3%82%AF%E3%83%A9%E3%82%A6%E3%83%89%E3%81%AB%E3%82%88%E3%82%8B%E7%B5%B1%E5%90%88%E5%9E%8B%E3%83%87%E3%83%BC%E3%82%BF%E3%83%AA%E3%83%9D%E3%82%B8%E3%83%88%E3%83%AA%E6%A7%8B%E7%AF%89%E5%85%A5%E9%96%80-%E4%B8%8A%E5%8E%9F-%E8%AA%A0/dp/491031301X/ref=sr_1_1?__mk_ja_JP=%E3%82%AB%E3%82%BF%E3%82%AB%E3%83%8A&crid=C0SKFEEDHROK&keywords=aws+glue&qid=1689168313&sprefix=aws+gl%2Caps%2C349&sr=8-1)
