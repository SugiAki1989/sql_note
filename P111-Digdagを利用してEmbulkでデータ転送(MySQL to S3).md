## :memo: Overview

ここでは Digdag と Embulk でデータ転送を行うための基本的な方法をまとめておく。構成は前回作成した AWS 環境で、EC2 に Digdag を追加でインストールし、 cron で実行していたジョブスケジューリングを Digdag で行う。

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Digdag のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

- [What’s Digdag?](https://docs.digdag.io/index.html)

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`Embulk`, `Digdag`

## :pencil2: EC2 の道具立て

まずは必要なものをインストールしておく。Digdag は java が必要で、ここでは java8 のバージョンを利用しており、Embulk のバージョンは下記の通り。

```
# java
$ java -version
openjdk version "1.8.0_372"
OpenJDK Runtime Environment (build 1.8.0_372-b07)
OpenJDK 64-Bit Server VM (build 25.372-b07, mixed mode)

# embulk
$ embulk -version
embulk 0.9.25
```

ドキュメントに従って、Digdag をインストールしていく。

```
$ curl -o ~/bin/digdag --create-dirs -L "https://dl.digdag.io/digdag-latest"
$ chmod +x ~/bin/digdag
$ echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
$ source ~/.bashrc
$ digdag --version
0.10.5
```

ドキュメントを見るとサンプルフローが実行できるようなので、動くかどうかを試しておく。

```
$ digdag init mydag
2023-06-19 10:54:17 +0000: Digdag v0.10.5
  Creating mydag/mydag.dig
  Creating mydag/.gitignore
Done. Type `cd mydag` and then `digdag run mydag.dig` to run the workflow. Enjoy!
```

`mydag.dig`というファイルが生成されたようなので、中身をみておく。

```
$ cd mydag
$ cat mydag.dig
timezone: UTC

+setup:
  echo>: start ${session_time}

+disp_current_date:
  echo>: ${moment(session_time).utc().format('YYYY-MM-DD HH:mm:ss Z')}

+repeat:
  for_each>:
    order: [first, second, third]
    animal: [dog, cat]
  _do:
    echo>: ${order} ${animal}
  _parallel: true

+teardown:
  echo>: finish ${session_time}
```

問題なく、文字列を繰り返し実行してくれている。

```
$ digdag run mydag.dig
2023-06-19 10:54:39 +0000: Digdag v0.10.5
2023-06-19 10:54:40 +0000 [WARN] (main): Using a new session time 2023-06-19T00:00:00+00:00.
2023-06-19 10:54:40 +0000 [INFO] (main): Using session /home/ec2-user/mydag/.digdag/status/20230619T000000+0000.
2023-06-19 10:54:40 +0000 [INFO] (main): Starting a new session project id=1 workflow name=mydag session_time=2023-06-19T00:00:00+00:00
2023-06-19 10:54:41 +0000 [INFO] (0016@[0:default:1:1]+mydag+setup): echo>: start 2023-06-19T00:00:00+00:00
start 2023-06-19T00:00:00+00:00
2023-06-19 10:54:42 +0000 [INFO] (0016@[0:default:1:1]+mydag+disp_current_date): echo>: 2023-06-19 00:00:00 +00:00
2023-06-19 00:00:00 +00:00
2023-06-19 10:54:43 +0000 [INFO] (0016@[0:default:1:1]+mydag+repeat): for_each>: {order=[first, second, third], animal=[dog, cat]}
2023-06-19 10:54:44 +0000 [INFO] (0016@[0:default:1:1]+mydag+repeat^sub+for-0=order=0=first&1=animal=0=dog): echo>: first dog
first dog
2023-06-19 10:54:44 +0000 [INFO] (0021@[0:default:1:1]+mydag+repeat^sub+for-0=order=2=third&1=animal=1=cat): echo>: third cat
third cat
2023-06-19 10:54:44 +0000 [INFO] (0017@[0:default:1:1]+mydag+repeat^sub+for-0=order=0=first&1=animal=1=cat): echo>: first cat
first cat
2023-06-19 10:54:44 +0000 [INFO] (0019@[0:default:1:1]+mydag+repeat^sub+for-0=order=1=second&1=animal=1=cat): echo>: second cat
second cat
2023-06-19 10:54:44 +0000 [INFO] (0018@[0:default:1:1]+mydag+repeat^sub+for-0=order=1=second&1=animal=0=dog): echo>: second dog
second dog
2023-06-19 10:54:44 +0000 [INFO] (0020@[0:default:1:1]+mydag+repeat^sub+for-0=order=2=third&1=animal=0=dog): echo>: third dog
third dog
2023-06-19 10:54:45 +0000 [INFO] (0018@[0:default:1:1]+mydag+teardown): echo>: finish 2023-06-19T00:00:00+00:00
finish 2023-06-19T00:00:00+00:00
Success. Task state is saved at /home/ec2-user/mydag/.digdag/status/20230619T000000+0000 directory.
```

## :pencil2: Digdag の道具立て

最終的な状態は下記の通りで、シェルファイルを Digdag から実行することで Embulk を動かす。vim をつかいこなせないので、各 dig ファイルはローカル環境から`scp`で転送する。

```
# scp ~/Desktop/embulk_etl.dig ec2-user@11.111.111.111:~/
$ tree ./
./
├── bin
│   └── digdag
├── embulk_etl.dig
├── embulk.diff.yml
├── embulk_mysql_s3.yml.liquid
├── embulk_mysql_s3_bk.yml.liquid
└── exec_embulk_mysql.sh

```

- `embulk.diff.yml`は差分み込みに必要なファイル
- `embulk_mysql_s3.yml.liquid`は Embulk のコンフィグファイル
- `embulk_mysql_s3_bk.yml.liquid`は実行時に前回の認証ファイルのバックアップ
- `exec_embulk_mysql.sh`は Embulk を実行するためのシェルファイル
- `embulk_etl.dig`は Embulk を定期的に実行するための dig ファイル

`embulk_etl.dig`の中身は下記の通り。非常に簡単なもので、1 分ごとに`exec_embulk_mysql.sh`を実行し、MySQL のレコードを csv にして S3 に転送する。

```
timezone: "Asia/Tokyo"

schedule:
 cron>: '* * * * *'

+task1:
  sh>: /home/ec2-user/exec_embulk_mysql.sh
```

あとはこれを実行するだけ。ちなみに今回は 2680 行目から行う。

```
$ cat embulk.diff.yml
in:
  last_record: [2680]
out: {}
```

実行して、数分間放置しておく。ログの時間が UTC になっているが、修正の仕方はまた今後調べる。

```
$ digdag scheduler
2023-06-19 11:01:20 +0000: Digdag v0.10.5
2023-06-19 11:01:22 +0000 [INFO] (main): secret encryption engine: disabled
2023-06-19 11:01:22 +0000 [INFO] (main): Added new revision 1
2023-06-19 11:01:22 +0000 [INFO] (main): XNIO version 3.3.8.Final
2023-06-19 11:01:22 +0000 [INFO] (main): XNIO NIO Implementation Version 3.3.8.Final
2023-06-19 11:01:22 +0000 [INFO] (main): Starting server on 127.0.0.1:65432
2023-06-19 11:01:22 +0000 [INFO] (main): Bound on 127.0.0.1:65432 (api)
2023-06-19 11:02:00 +0000 [INFO] (scheduler-0): Starting a new session project id=1 workflow name=embulk_etl session_time=2023-06-19T20:02:00+09:00
2023-06-19 11:02:00 +0000 [INFO] (scheduler-0): Updating next schedule time: sched=StoredSchedule{id=1, projectId=1, createdAt=2023-06-19T11:01:22.593Z, updatedAt=2023-06-19T11:01:22.593Z, workflowName=embulk_etl, workflowDefinitionId=1, nextRunTime=2023-06-19T11:02:00Z, nextScheduleTime=2023-06-19T11:02:00Z}, next=ScheduleTime{runTime=2023-06-19T11:03:00Z, time=2023-06-19T11:03:00Z}, lastSessionTime=2023-06-19T11:02:00Z
2023-06-19 11:02:01 +0000 [INFO] (0024@[0:default:1:1]+embulk_etl+task1): sh>: /home/ec2-user/exec_embulk_mysql.sh
================================== [ NOTICE ] ==================================
2023-06-19 11:02:02.229 +0000: Embulk v0.9.25

Embulkの実行ログが出力される

2023-06-19 11:02:12.296 +0000 [INFO] (main): Next config diff: {"in":{"last_record":[3207]},"out":{}}
================================================================================
2023-06-19 11:03:00 +0000 [INFO] (scheduler-0): Starting a new session project id=1 workflow name=embulk_etl session_time=2023-06-19T20:03:00+09:00
2023-06-19 11:03:00 +0000 [INFO] (scheduler-0): Updating next schedule time: sched=StoredSchedule{id=1, projectId=1, createdAt=2023-06-19T11:01:22.593Z, updatedAt=2023-06-19T11:02:00.505Z, lastSessionTime=2023-06-19T11:02:00Z, workflowName=embulk_etl, workflowDefinitionId=1, nextRunTime=2023-06-19T11:03:00Z, nextScheduleTime=2023-06-19T11:03:00Z}, next=ScheduleTime{runTime=2023-06-19T11:04:00Z, time=2023-06-19T11:04:00Z}, lastSessionTime=2023-06-19T11:03:00Z
2023-06-19 11:03:01 +0000 [INFO] (0024@[0:default:2:2]+embulk_etl+task1): sh>: /home/ec2-user/exec_embulk_mysql.sh

2023-06-19 11:01:20 +0000: Digdag v0.10.5

以降繰り返し・・・
```

数分間放置しておくと、下記のように S3 にデータが転送されている。前回の cron 分と合わせて表示している。

```
$ aaws s3 ls s3://embulk-mysql-to-s3/out/
2023-06-18 10:03:04      68401 data_20230618100255.csv
2023-06-18 10:17:49       2377 data_20230618101739.csv
2023-06-18 10:20:10        540 data_20230618102001.csv
2023-06-18 10:30:10       1992 data_20230618103001.csv
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
-----------------------------------------------------------下記がDigdagで実行した分
2023-06-19 20:02:13      14711 data_20230619200201.csv
2023-06-19 20:03:12         98 data_20230619200301.csv
2023-06-19 20:04:11        210 data_20230619200400.csv
2023-06-19 20:05:11        376 data_20230619200500.csv
2023-06-19 20:06:11        208 data_20230619200600.csv
2023-06-19 20:07:11        209 data_20230619200700.csv
2023-06-19 20:08:11        154 data_20230619200800.csv
2023-06-19 20:09:11        209 data_20230619200900.csv
2023-06-19 20:10:11        209 data_20230619201000.csv
2023-06-19 20:11:11        320 data_20230619201101.csv
2023-06-19 20:12:11        264 data_20230619201201.csv
2023-06-19 20:13:10        210 data_20230619201300.csv
2023-06-19 20:14:11        209 data_20230619201400.csv
2023-06-19 20:15:10         97 data_20230619201500.csv
2023-06-19 20:16:10        209 data_20230619201600.csv
```

スケジューラーを終了し、1 番最後に転送されたデータを見ておく。問題なくデータが転送されている。`id`が飛んでいるのは`where flg = 1`によるもの。

```
$ aws s3 cp s3://embulk-mysql-to-s3/out/data_20230619201600.csv -
"id","datetime","value1","category","flg"
"3287","2023-06-19 20:15:29.000000 +0900","-16","4","1"
"3289","2023-06-19 20:15:49.000000 +0900","-8","3","1"
"3290","2023-06-19 20:15:59.000000 +0900","-10","5","1"

$ cat embulk.diff.yml
in:
  last_record: [3290]
out: {}
```

これで定期的に実行できることがわかったので、現在夜の 20 時くらいのなので、明日の 20 時 55 分(この時間に特に意味はない)に書き直して、スケジューラーを実行しておく。

```
$ cat embulk_etl.dig
timezone: "Asia/Tokyo"

schedule:
 cron>: '55 20 * * *'

+task1:
  sh>: /home/ec2-user/exec_embulk_mysql.sh

$ digdag scheduler
```

以降、追記。

次の日に確認したところ、問題なく S3 に転送されていた。

```
# S3への保存時間は2023/06/20 08:55:13 PM JSTとのこと
$ aws s3 cp s3://embulk-mysql-to-s3/out/data_20230620205502.csv -
"id","datetime","value1","category","flg"
"3292","2023-06-19 20:16:19.000000 +0900","-9","8","1"
"3293","2023-06-19 20:16:29.000000 +0900","-13","e","1"
"3294","2023-06-19 20:16:39.000000 +0900","-5","0","1"
(snip)
"3537","2023-06-20 20:54:29.000000 +0900","-11","d","1"
"3539","2023-06-20 20:54:59.000000 +0900","-16","1","1"
"3540","2023-06-20 20:55:09.000000 +0900","-20","4","1"

$ cat embulk.diff.yml
in:
  last_record: [3540]
out: {}
```

ログの時間表記は EC2 インスタンスのタイムゾーンを JST に変更すれば直せるっぽいので、下記の方法でタイムゾーンを修正する。

```
$ date
2023年  6月 20日 火曜日 12:06:18 UTC

$ sudo cp -p /usr/share/zoneinfo/Japan /etc/localtime
$ cat /etc/sysconfig/clock
ZONE="UTC"
UTC=true

$ sudo vim /etc/sysconfig/clock
$ cat /etc/sysconfig/clock
ZONE="Asia/Tokyo"
UTC=false

$ date
2023年  6月 20日 火曜日 21:08:53 JST
```

今、21 時 10 分 なので dig ファイル内の cron を`11 21 * * *`に急ぎ書き直し、ログの状態を確認する。ログの表記を確認するだけなら、時間設定してスケジューラーで動かす必要はない。

```
$ cat embulk_etl.dig
timezone: "Asia/Tokyo"

schedule:
 cron>: '11 21 * * *'

+task1:
  sh>: /home/ec2-user/exec_embulk_mysql.sh
```

スケジューラを起動すると、時刻が JST で修正されて表示されている。

```
$ digdag scheduler
2023-06-20 21:10:55 +0900: Digdag v0.10.5
2023-06-20 21:10:56 +0900 [INFO] (main): secret encryption engine: disabled
2023-06-20 21:10:57 +0900 [INFO] (main): Added new revision 1
2023-06-20 21:10:57 +0900 [INFO] (main): XNIO version 3.3.8.Final
2023-06-20 21:10:57 +0900 [INFO] (main): XNIO NIO Implementation Version 3.3.8.Final
2023-06-20 21:10:57 +0900 [INFO] (main): Starting server on 127.0.0.1:65432
2023-06-20 21:10:57 +0900 [INFO] (main): Bound on 127.0.0.1:65432 (api)
2023-06-20 21:11:00 +0900 [INFO] (scheduler-0): Starting a new session project id=1 workflow name=embulk_etl session_time=2023-06-20T21:11:00+09:00
2023-06-20 21:11:01 +0900 [INFO] (scheduler-0): Updating next schedule time: sched=StoredSchedule{id=1, projectId=1, createdAt=2023-06-20T12:10:57.136Z, updatedAt=2023-06-20T12:10:57.136Z, workflowName=embulk_etl, workflowDefinitionId=1, nextRunTime=2023-06-20T12:11:00Z, nextScheduleTime=2023-06-20T12:11:00Z}, next=ScheduleTime{runTime=2023-06-21T12:11:00Z, time=2023-06-21T12:11:00Z}, lastSessionTime=2023-06-20T12:11:00Z
2023-06-20 21:11:02 +0900 [INFO] (0024@[0:default:1:1]+embulk_etl+task1): sh>: /home/ec2-user/exec_embulk_mysql.sh

2023-06-20 21:11:02.662 +0900: Embulk v0.9.25
2023-06-20 21:11:03.754 +0900 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2023-06-20 21:11:06.924 +0900 [INFO] (main): Gem's home and path are set by default: "/home/ec2-user/.embulk/lib/gems"
2023-06-20 21:11:09.583 +0900 [INFO] (main): Started Embulk v0.9.25
2023-06-20 21:11:09.724 +0900 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.13.2)
2023-06-20 21:11:09.813 +0900 [INFO] (0001:transaction): Loaded plugin embulk-output-s3 (1.7.1)

(snip)

2023-06-20 21:11:14 +0900 [INFO] (shutdown): Started shutdown process
2023-06-20 21:11:14 +0900 [INFO] (shutdown): Shutting down workflow executor loop
2023-06-20 21:11:14 +0900 [INFO] (shutdown): Closing HTTP listening sockets
2023-06-20 21:11:14 +0900 [INFO] (shutdown): Waiting for completion of running HTTP requests...
2023-06-20 21:11:14 +0900 [INFO] (shutdown): Shutting down HTTP worker threads
2023-06-20 21:11:14 +0900 [INFO] (shutdown): Shutting down system
2023-06-20 21:11:14 +0900 [INFO] (shutdown): Shutdown completed
```

データ転送は差分読み込みなので、前回の 3540 番からちゃんと始まっている。

```
$ aws s3 cp s3://embulk-mysql-to-s3/out/data_20230620211102.csv -
"id","datetime","value1","category","flg"
"3542","2023-06-20 20:55:29.000000 +0900","-18","9","1"
"3543","2023-06-20 20:55:39.000000 +0900","-10","9","1"
"3544","2023-06-20 20:55:49.000000 +0900","-15","c","1"
(snip)
"3632","2023-06-20 21:10:29.000000 +0900","-18","0","1"
"3635","2023-06-20 21:10:59.000000 +0900","-6","c","1"
"3636","2023-06-20 21:11:09.000000 +0900","-20","3","1"

$ cat embulk.diff.yml
in:
  last_record: [3636]
out: {}
```

## :closed_book: Reference

- [What’s Digdag?](https://docs.digdag.io/index.html)
- [Embulk](https://www.embulk.org/)
- [Embulk Built-in Plugins](https://www.embulk.org/docs/index.html)
- [Embulk configuration file format](https://www.embulk.org/docs/built-in.html)
- [Digdag MySQL Tutorial](http://www.alphasentaurii.com/programming/2020/06/07/digdag-mysql-tutorial.html)
- [Digdag PostgreSQL Tutorial](http://www.alphasentaurii.com/programming/2020/07/07/digdag-postgresql-tutorial.html)
