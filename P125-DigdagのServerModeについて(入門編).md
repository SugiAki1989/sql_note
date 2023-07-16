## :memo: Overview

ここでは Digdag の Server モードについて基本的な方法をまとめておく。前回の環境構築編の続き。

- [Digdag - Server-mode commands](https://docs.digdag.io/command_reference.html#server-mode-commands)

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。また、下記の記事を参考にさせていただいた。非常にわかりやすくまとまっていて、たいへん助かった。

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`digdag`

## :pencil2:　 Digdag Server

まずは digdag server が機能するかを確認するために、`digdag init`コマンドで、digdag のワークフローを作成し、試運転しておく。

```
$ digdag init mydag
2023-07-16 10:19:20 +0900: Digdag v0.10.5
  Creating mydag/mydag.dig
  Creating mydag/.gitignore
Done. Type `cd mydag` and then `digdag run mydag.dig` to run the workflow. Enjoy!
```

`Asia/Tokyo`を`UTC`から変更しておく。

```
$ vim mydag/mydag.dig

timezone: "Asia/Tokyo"

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

`digdag init`コマンドの指示に従い、`mydag`ワークフローディレクトリに移動して、`mydag.dig`を実行する。出力内容を見る限り、問題なく動いているように見える。

```
$ cd mydag
$ digdag run mydag.dig
2023-07-16 10:22:19 +0900: Digdag v0.10.5
2023-07-16 10:22:21 +0900 [WARN] (main): Using a new session time 2023-07-16T00:00:00+09:00.
2023-07-16 10:22:21 +0900 [INFO] (main): Using session /opt/digdag/mydag/.digdag/status/20230716T000000+0900.
2023-07-16 10:22:21 +0900 [INFO] (main): Starting a new session project id=1 workflow name=mydag session_time=2023-07-16T00:00:00+09:00
2023-07-16 10:22:24 +0900 [INFO] (0016@[0:default:1:1]+mydag+setup): echo>: start 2023-07-16T00:00:00+09:00
start 2023-07-16T00:00:00+09:00
2023-07-16 10:22:26 +0900 [INFO] (0016@[0:default:1:1]+mydag+disp_current_date): echo>: 2023-07-15 15:00:00 +00:00
2023-07-15 15:00:00 +00:00
2023-07-16 10:22:27 +0900 [INFO] (0016@[0:default:1:1]+mydag+repeat): for_each>: {order=[first, second, third], animal=[dog, cat]}
2023-07-16 10:22:30 +0900 [INFO] (0020@[0:default:1:1]+mydag+repeat^sub+for-0=order=2=third&1=animal=0=dog): echo>: third dog
third dog
2023-07-16 10:22:30 +0900 [INFO] (0019@[0:default:1:1]+mydag+repeat^sub+for-0=order=1=second&1=animal=1=cat): echo>: second cat
2023-07-16 10:22:30 +0900 [INFO] (0018@[0:default:1:1]+mydag+repeat^sub+for-0=order=1=second&1=animal=0=dog): echo>: second dog
2023-07-16 10:22:30 +0900 [INFO] (0021@[0:default:1:1]+mydag+repeat^sub+for-0=order=2=third&1=animal=1=cat): echo>: third cat
2023-07-16 10:22:30 +0900 [INFO] (0016@[0:default:1:1]+mydag+repeat^sub+for-0=order=0=first&1=animal=0=dog): echo>: first dog
2023-07-16 10:22:30 +0900 [INFO] (0017@[0:default:1:1]+mydag+repeat^sub+for-0=order=0=first&1=animal=1=cat): echo>: first cat
second cat
first cat
first dog
third cat
second dog
2023-07-16 10:22:31 +0900 [INFO] (0017@[0:default:1:1]+mydag+teardown): echo>: finish 2023-07-16T00:00:00+09:00
finish 2023-07-16T00:00:00+09:00
Success. Task state is saved at /opt/digdag/mydag/.digdag/status/20230716T000000+0900 directory.
  * Use --session <daily | hourly | "yyyy-MM-dd[ HH:mm:ss]"> to not reuse the last session time.
  * Use --rerun, --start +NAME, or --goal +NAME argument to rerun skipped tasks.
```

`mydag`ワークフローディレクトリを作成し`.dig`ファイルを実行したが、WebUI にワークフローが登録されて、UI から何かを出来るようになるわけではない。`digdag push`コマンドでワークフローを登録する必要がある。

カレントディレクトリに`.dig`ファイルが存在していないとエラーになるらしいので、意図的にエラーを起こしてみる。内容を見ると、そのディレクトリの中にあるファイルたちもアーカイブされている模様。

```
$ cd ..
$ pwd
/opt/digdag

$ digdag push mydag
2023-07-16 10:36:03 +0900: Digdag v0.10.5
Creating /opt/digdag/.digdag/tmp/archive-6638116193383808036.tar.gz...
  Archiving logs/digdag_server.log
  Archiving server.properties
  Archiving bin/digdag
  Archiving accesslogs/access.2023-07-16.log
  Archiving accesslogs/access.log
  Archiving mydag/mydag.dig
Workflows:
  WARNING: This project doesn't include workflows. Usually, this is a mistake.
           Please make sure that all *.dig files are on the top directory.
           *.dig files in subdirectories are not recognized as workflows.

error: Status code 400: {"message":"Size of the uploaded archive file exceeds limit (2097152 bytes)","status":400}
```

`mydag`ワークフローディレクトに移動して、ワークフローを登録する。`Uploaded`に記載されている内容が、WebUI からも確認できるようになる。

```
$ cd mydag
$ digdag push mydag
2023-07-16 10:37:05 +0900: Digdag v0.10.5
Creating .digdag/tmp/archive-1290147963694497.tar.gz...
  Archiving mydag.dig
Workflows:
  mydag.dig
Uploaded:
  id: 1
  name: mydag
  revision: f6f2f9c2-b16f-4084-a0e7-081c000264ee
  archive type: db
  project created at: 2023-07-16T01:37:07Z
  revision updated at: 2023-07-16T01:37:07Z

Use `digdag workflows` to show all workflows.
```

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-1.png)

「Workflows > mydag」と進むと、ワークフローの詳細が確認でき、ここから手動実行することも可能。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-2.png)

「RUN」から手動で実行してみると、「Sessions」に情報が追加される。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-3.png)

「Sessions」の「Status: Success」をクリックすると、実行の詳細が確認できる。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-4.png)

登録されているワークフローは`digdag workflows`コマンドから確認できる。

```
$ digdag workflows mydag
2023-07-16 10:51:51 +0900: Digdag v0.10.5
  mydag
    mydag

Use `digdag workflows <project-name> <name>` to show details.
```

ワークフローの詳細は`digdag workflows <project-name> <name>`で見れるようなので、素直に従って実行する。

```
$ digdag workflows mydag mydag
2023-07-16 10:52:09 +0900: Digdag v0.10.5
"+setup":
  echo>: "start ${session_time}"
"+disp_current_date":
  echo>: "${moment(session_time).utc().format('YYYY-MM-DD HH:mm:ss Z')}"
"+repeat":
  for_each>:
    order:
    - "first"
    - "second"
    - "third"
    animal:
    - "dog"
    - "cat"
  _do:
    echo>: "${order} ${animal}"
  _parallel: true
"+teardown":
  echo>: "finish ${session_time}"
```

コマンドから実行する場合は、`digdag start <project-name> <name> -- session <hourly | daily | now | yyyy-MM-dd | "yyyy-MM-dd HH:mm:ss">`で実行する。ここではすぐに実行したいので、`--session now`で実行する。

```
$ digdag start mydag mydag --session now
2023-07-16 10:55:35 +0900: Digdag v0.10.5
Started a session attempt:
  session id: 2
  attempt id: 2
  uuid: 02eb149a-ed13-43f5-882e-5683231e6644
  project: mydag
  workflow: mydag
  session time: 2023-07-16 10:55:36 +0900
  retry attempt name:
  params: {}
  created at: 2023-07-16 10:55:37 +0900

* Use `digdag session 2` to show session status.
* Use `digdag task 2` and `digdag log 2` to show task status and logs.
```

詳細は`digdag session <id>`コマンド、`digdag task <id>`コマンド、`digdag log <id>`コマンドで確認できる。

```
$ digdag session 2
2023-07-16 11:09:06 +0900: Digdag v0.10.5
  session id: 2
  attempt id: 2
  uuid: 02eb149a-ed13-43f5-882e-5683231e6644
  project: mydag
  workflow: mydag
  session time: 2023-07-16 10:55:36 +0900
  retry attempt name:
  params: {}
  created at: 2023-07-16 10:55:37 +0900
  kill requested: false
  status: success

--------------------------------------------------------------------------------------------------------------------------------------------------------
$ digdag task 2
2023-07-16 11:09:18 +0900: Digdag v0.10.5
   id: 13
   name: +mydag
   state: success
   started:
   updated: 2023-07-16 10:55:45 +0900
   config: {}
   parent: null
   upstreams: []
   export params: {}
   store params: {}
   state params: {}

   (snip)

   id: 24
   name: +mydag+repeat^sub+for-0=order=2=third&1=animal=1=cat
   state: success
   started: 2023-07-16 10:55:42 +0900
   updated: 2023-07-16 10:55:43 +0900
   config: {"echo>":"${order} ${animal}","_export":{"order":"third","animal":"cat"}}
   parent: 18
   upstreams: []
   export params: {"order":"third","animal":"cat"}
   store params: {}
   state params: {}

12 entries.
--------------------------------------------------------------------------------------------------------------------------------------------------------
$ digdag log 2
2023-07-16 11:09:26 +0900: Digdag v0.10.5
2023-07-16 01:55:39.268 +0000 [INFO] (0119@[0:mydag:2:2]+mydag+setup) io.digdag.core.agent.OperatorManager: echo>: start 2023-07-16T10:55:36+09:00
start 2023-07-16T10:55:36+09:00
2023-07-16 01:55:40.782 +0000 [INFO] (0119@[0:mydag:2:2]+mydag+disp_current_date) io.digdag.core.agent.OperatorManager: echo>: 2023-07-16 01:55:36 +00:00
2023-07-16 01:55:36 +00:00
2023-07-16 01:55:41.906 +0000 [INFO] (0119@[0:mydag:2:2]+mydag+repeat) io.digdag.core.agent.OperatorManager: for_each>: {order=[first, second, third], animal=[dog, cat]}
2023-07-16 01:55:43.757 +0000 [INFO] (0123@[0:mydag:2:2]+mydag+repeat^sub+for-0=order=2=third&1=animal=0=dog) io.digdag.core.agent.OperatorManager: echo>: third dog
third dog
2023-07-16 01:55:43.755 +0000 [INFO] (0122@[0:mydag:2:2]+mydag+repeat^sub+for-0=order=1=second&1=animal=1=cat) io.digdag.core.agent.OperatorManager: echo>: second cat
second cat
2023-07-16 01:55:43.770 +0000 [INFO] (0121@[0:mydag:2:2]+mydag+repeat^sub+for-0=order=1=second&1=animal=0=dog) io.digdag.core.agent.OperatorManager: echo>: second dog
second dog
2023-07-16 01:55:43.809 +0000 [INFO] (0120@[0:mydag:2:2]+mydag+repeat^sub+for-0=order=0=first&1=animal=1=cat) io.digdag.core.agent.OperatorManager: echo>: first cat
first cat
2023-07-16 01:55:43.763 +0000 [INFO] (0119@[0:mydag:2:2]+mydag+repeat^sub+for-0=order=0=first&1=animal=0=dog) io.digdag.core.agent.OperatorManager: echo>: first dog
first dog
2023-07-16 01:55:43.777 +0000 [INFO] (0124@[0:mydag:2:2]+mydag+repeat^sub+for-0=order=2=third&1=animal=1=cat) io.digdag.core.agent.OperatorManager: echo>: third cat
third cat
2023-07-16 01:55:45.093 +0000 [INFO] (0120@[0:mydag:2:2]+mydag+teardown) io.digdag.core.agent.OperatorManager: echo>: finish 2023-07-16T10:55:36+09:00
finish 2023-07-16T10:55:36+09:00
```

「Sessions」に実行内容が追加されている。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-5.png)

プロジェクトを削除する場合は、`digdag delete <project-name>`コマンドを実行する。

```
$ digdag delete new-project
2023-07-16 11:21:43 +0900: Digdag v0.10.5
Project:
  id: 2
  name: new-project
  latest revision: a7ace850-0fb0-410d-81d0-8ce98515ae9d
  created at: 2023-07-16 11:17:37 +0900
  last updated at: 2023-07-16 11:20:10 +0900
Are you sure you want to delete this project? [y/N]: yes
Project 'new-project' is deleted.
```

続いてはスケジュール実行の設定を行う。`.dig`ファイルに下記の通り、`schedule`の項目を追加する。

```
schedule:
 cron>: '* * * * *'
```

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-6.png)

「save」して保存してから数分放ったらかしにしておくと、1 分ごとに定期的に実行されていることがわかる。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-7.png)

`.dig`ファイルを修正する、削除するなどしてスケジューラーを停止させても良いが、ここでは WebUI の「PAUSE」ボタンから停止しておく。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-8.png)

## Server モードから Embulk でデータ転送(MySQL to MySQL)

前回 MySQL to MySQL で利用した Embulk の転送スクリプトを利用して、データ転送をスケジュール実行する。WebUI から各種ファイルを作成してもよいが、必要なファイルをローカル環境から移動させておく。

```
[local]$ scp -r ~/Desktop/MySQLtoMySQL ec2-user@11.11.111.111:/home/ec2-user

[ec2-user]$ sudo scp -r MySQLtoMySQL /opt/digdag/
[ec2-user]$ sudo chown -R digdag:digdag /opt/digdag/MySQLtoMySQL
[ec2-user]$ sudo chmod -R 755 /opt/digdag/MySQLtoMySQL
[ec2-user]$ sudo su digdag
```

`digdag`ユーザーに変更して、

```
[digdag]$ cd ~/MySQLtoMySQL/
[digdag]$ ls
embulk_etl.dig  embulk_mysql_mysql.yml.liquid  exec_embulk_mysql.sh
```

必要なファイルをワークフローを登録すると、処理が開始される。

```
$ digdag push MySQLtoMySQL
2023-07-16 13:06:13 +0900: Digdag v0.10.5
Creating /opt/digdag/MySQLtoMySQL/.digdag/tmp/archive-8972810629314144001.tar.gz...
  Archiving exec_embulk_mysql.sh
  Archiving embulk_etl.dig
  Archiving embulk_mysql_mysql.yml.liquid
Workflows:
  embulk_etl.dig
(snip)
```

WebUI で確認すると、エラーが出ているころがわかる。エラーログを確認するためには、画像の「REFRESH LOGS」を押す。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-9.png)

原因は MySQL に Embulk がアクセスするためのプラグインをインストールしてなかったこと。

```
[digdag]$ embulk gem install embulk-input-mysql
[digdag]$ embulk gem install embulk-output-mysql
```

問題なく実行できるか、`digdag run`コマンドで手動実行する。

```
[digdag]$ digdag run embulk_etl.dig
```

エラーはでないので、MySQL のテーブルを確認すると、データが転送できていることがわかる。

```
MySQL [(none)]> use event_auto_insert;
MySQL [event_auto_insert]> select * from summary_logs;
+----+---------------------+--------+--------+---------------------+---------------------+-------+
| id | dt_minute           | min_id | max_id | start               | end                 | count |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
|  1 | 2023-06-30 21:39:00 |    386 |    391 | 2023-06-30 21:39:00 | 2023-06-30 21:40:00 |     6 |
(snip)
| 15 | 2023-07-16 14:55:00 |  12912 |  12917 | 2023-07-16 14:55:00 | 2023-07-16 14:56:00 |     6 | -- 手動実行
+----+---------------------+--------+--------+---------------------+---------------------+-------+
15 rows in set (0.00 sec)
```

WebUI から下記を追加し、5 分ごとのスケジュール実行を設定する。

```
schedule:
 cron>: '*/5 * * * *'
```

数分間放ったらかしにしておくと、「Sessions」に定期実行の履歴が溜まっていく。どれも「Success」なので問題なくデータが転送できている。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p125−digdag-10.png)

MySQL のテーブルを確認すると、特に問題は起こっておらず、意図した通りに機能してくれている。

```
MySQL [event_auto_insert]> select * from summary_logs;
+----+---------------------+--------+--------+---------------------+---------------------+-------+
| id | dt_minute           | min_id | max_id | start               | end                 | count |
+----+---------------------+--------+--------+---------------------+---------------------+-------+
|  1 | 2023-06-30 21:39:00 |    386 |    391 | 2023-06-30 21:39:00 | 2023-06-30 21:40:00 |     6 |
(snip)
| 15 | 2023-07-16 14:55:00 |  12912 |  12917 | 2023-07-16 14:55:00 | 2023-07-16 14:56:00 |     6 | -- 手動実行
| 16 | 2023-07-16 15:09:00 |  12996 |  13001 | 2023-07-16 15:09:00 | 2023-07-16 15:10:00 |     6 | -- Digdag Server の schedule
| 17 | 2023-07-16 15:14:00 |  13026 |  13031 | 2023-07-16 15:14:00 | 2023-07-16 15:15:00 |     6 | -- Digdag Server の schedule
| 18 | 2023-07-16 15:19:00 |  13056 |  13061 | 2023-07-16 15:19:00 | 2023-07-16 15:20:00 |     6 | -- Digdag Server の schedule
+----+---------------------+--------+--------+---------------------+---------------------+-------+
18 rows in set (0.01 sec)
```

digdag を server モードで実行する際に役立ちそうなコマンドのオプションを[ドキュメント](https://docs.digdag.io/command_reference.html#server-mode-commands)を参考に下記をまとめておく。

- `--disable-scheduler`: サーバー上のスケジュール実行プログラムを無効化する。ワークフローファイルを変更せずにすべてのスケジュールを無効にできる。
- `--params-file PATH`: YAML ファイルからパラメタを読む。ネストされたパラメータは`.`を使ってアクセスする。

## 余談

digdag sever は、停止期間中の実行予定分は、サーバー起動時に遡って実行される模様。それを知らなかったので、MySQL to MySQL への転送を行う際に、1 分間ごとに実行するスケジュール設定にしていたため、プラグインのエラーが出て、その間に次のエラーが出て、その間に次のエラーが出て…を繰り返し、EC2 に SSH できなくなってしまうトラブルに遭遇した。

そのため、AWS の管理画面から直接 EC2 を再起動してアクセスを試みたが、停止期間中の実行予定分は、サーバー起動時に遡って実行されるため、また、プラグインのエラーが出て、その間に次のエラーが出て、その間に次のエラーが出て…を繰り返した…泣。

本来、1 分ごとにデータ転送を行うことは個人的にはないので、学習として気軽に 1 分ごとにデータ転送を設定してしまったため、このようなことになった。手を動かしてみないと、(自分の未熟さゆえ)こんなトラブルも起こせないので、よい経験だった。

## :closed_book: Reference

- [Digdag サーバの設定メモ](https://qiita.com/chocomintkusoyaro/items/3dbf4141e098d8dde9da)
- [Digdag 入門](https://recruit.gmo.jp/engineer/jisedai/blog/introduction-to-digdag/)
- [EC2 上の digdag server に外部からアクセスする](https://www.capybara-engineer.com/entry/2020/11/11/205119)
