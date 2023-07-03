## :memo: Overview

ここでは Digdag のドキュメントを読みながら、Digdag でできることをまとめていく。個人的に気になったポイントをまとめているだけ。

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Digdag のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

- [Digdag](https://docs.digdag.io/)

## :floppy_disk: Database

None

## :bookmark: Tag

`Digdag`, ``

## :pencil2: Digdag とは？

Digdag はワークフロー管理ツールの 1 つで、他に Airflow や Luigi などがある。ワークフロー管理ツールは ETL 処理の自動化を行う際に非常に便利。

Digdag は、複数のステップから構成される処理の依存関係や、直列/並列で行うジョブの順序など yaml 形式のコンフィグファイルで設定できる作りになっている。シェルスクリプトで複雑に管理していた ETL のバッチ処理からデータ集計までワークフローを、メンテナンス性の高いワークフローに置き換えることが可能。下記の機能が揃っている。

- タスクの依存関係順の実行

  - 過去分の一括実行
  - 定期的な実行(時刻などの変数も使用可能)
  - s3 などでファイルが生成されてからの実行

- エラー処理

  - 失敗したら通知でき、失敗した場所から再実行

- ステートの監視
  - 実行時間が設定値を超えたら通知
  - タスクの実行時間の可視化
  - 実行ログの収集

## :pencil2: Digdag のワークフロー

Digdag のワークフローファイル(`sample.dig`)は下記のように記載する。

```
$ cat example.dig

timezone: Asia/Tokyo

+task1:
  sh>: echo ${session_date}

+task2:
  _parallel: true
  +subtask1:
    _export:
      subtask_id1: 1
    sh>: echo 'ID ${subtask_id1} - Subtask1'
  +subtask2:
    _export:
      subtask_id2: 2
    sh>: echo 'ID ${subtask_id2} - Subtask2'
  +subtask3:
    _export:
      subtask_id3: 'X'
    sh>: echo 'ID ${subtask_id3} - Subtask3'

+task3:
  for_each>:
    i: [1,2,3]
    j: [1,2]
  _do:
    echo>: 'i: ${i} - j: ${j}'
```

各ブロックの説明はあとで書くことにするとして、細かいことはさておき、とりあえず実行してみると、`+task1`が実行され、`+task2`の中のサブタスクが並列に実行され、`+task3`が実行されていることがわかる。

```
$ digdag run example.dig
2023-06-22 21:45:30 +0000: Digdag v0.10.5
2023-06-22 21:45:32 +0000 [WARN] (main): Using a new session time 2023-06-22T00:00:00+09:00.
2023-06-22 21:45:32 +0000 [INFO] (main): Using session /home/ec2-user/.digdag/status/20230622T000000+0900.
2023-06-22 21:45:32 +0000 [INFO] (main): Starting a new session project id=1 workflow name=example session_time=2023-06-22T00:00:00+09:00
2023-06-22 21:45:34 +0000 [INFO] (0016@[0:default:1:1]+example+task1): sh>: echo 2023-06-22
2023-06-22
2023-06-22 21:45:36 +0000 [INFO] (0018@[0:default:1:1]+example+task2+subtask2): sh>: echo 'ID 2 - Subtask2'
2023-06-22 21:45:36 +0000 [INFO] (0019@[0:default:1:1]+example+task2+subtask3): sh>: echo 'ID X - Subtask3'
2023-06-22 21:45:36 +0000 [INFO] (0016@[0:default:1:1]+example+task2+subtask1): sh>: echo 'ID 1 - Subtask1'
ID 2 - Subtask2
ID X - Subtask3
ID 1 - Subtask1
2023-06-22 21:45:37 +0000 [INFO] (0016@[0:default:1:1]+example+task3): for_each>: {i=[1, 2, 3], j=[1, 2]}
2023-06-22 21:45:39 +0000 [INFO] (0016@[0:default:1:1]+example+task3^sub+for-0=i=0=1&1=j=0=1): echo>: i: 1 - j: 1
i: 1 - j: 1
2023-06-22 21:45:40 +0000 [INFO] (0016@[0:default:1:1]+example+task3^sub+for-0=i=0=1&1=j=1=2): echo>: i: 1 - j: 2
i: 1 - j: 2
2023-06-22 21:45:42 +0000 [INFO] (0016@[0:default:1:1]+example+task3^sub+for-0=i=1=2&1=j=0=1): echo>: i: 2 - j: 1
i: 2 - j: 1
2023-06-22 21:45:43 +0000 [INFO] (0016@[0:default:1:1]+example+task3^sub+for-0=i=1=2&1=j=1=2): echo>: i: 2 - j: 2
i: 2 - j: 2
2023-06-22 21:45:45 +0000 [INFO] (0016@[0:default:1:1]+example+task3^sub+for-0=i=2=3&1=j=0=1): echo>: i: 3 - j: 1
i: 3 - j: 1
2023-06-22 21:45:46 +0000 [INFO] (0016@[0:default:1:1]+example+task3^sub+for-0=i=2=3&1=j=1=2): echo>: i: 3 - j: 2
i: 3 - j: 2
Success. Task state is saved at /home/ec2-user/.digdag/status/20230622T000000+0900 directory.
  * Use --session <daily | hourly | "yyyy-MM-dd[ HH:mm:ss]"> to not reuse the last session time.
  * Use --rerun, --start +NAME, or --goal +NAME argument to rerun skipped tasks.
```

再度同じコマンドを実行すると、ログの出力内容が異なることがわかる。すべてのタスクが`Skipped`になっており、前回の実行でタスクは実行済みということなので、処理をスキップしていると思われる。途中で失敗した際に、失敗したところから実行できるのは、この仕組みがあるからだろうか。これらの細かい点は、これ以降にまとめていく。

```
$ digdag run example.dig
2023-06-22 21:59:46 +0000: Digdag v0.10.5
2023-06-22 21:59:48 +0000 [WARN] (main): Reusing the last session time 2023-06-22T00:00:00+09:00.
2023-06-22 21:59:48 +0000 [INFO] (main): Using session /home/ec2-user/.digdag/status/20230622T000000+0900.
2023-06-22 21:59:48 +0000 [INFO] (main): Starting a new session project id=1 workflow name=example session_time=2023-06-22T00:00:00+09:00
2023-06-22 21:59:49 +0000 [WARN] (0016@[0:default:1:1]+example+task1): Skipped
2023-06-22 21:59:49 +0000 [WARN] (0016@[0:default:1:1]+example+task2+subtask1): Skipped
2023-06-22 21:59:49 +0000 [WARN] (0016@[0:default:1:1]+example+task2+subtask2): Skipped
2023-06-22 21:59:49 +0000 [WARN] (0017@[0:default:1:1]+example+task2+subtask3): Skipped
2023-06-22 21:59:51 +0000 [WARN] (0017@[0:default:1:1]+example+task3): Skipped
2023-06-22 21:59:52 +0000 [WARN] (0017@[0:default:1:1]+example+task3^sub+for-0=i=0=1&1=j=0=1): Skipped
2023-06-22 21:59:53 +0000 [WARN] (0017@[0:default:1:1]+example+task3^sub+for-0=i=0=1&1=j=1=2): Skipped
2023-06-22 21:59:54 +0000 [WARN] (0017@[0:default:1:1]+example+task3^sub+for-0=i=1=2&1=j=0=1): Skipped
2023-06-22 21:59:55 +0000 [WARN] (0017@[0:default:1:1]+example+task3^sub+for-0=i=1=2&1=j=1=2): Skipped
2023-06-22 21:59:56 +0000 [WARN] (0017@[0:default:1:1]+example+task3^sub+for-0=i=2=3&1=j=0=1): Skipped
2023-06-22 21:59:57 +0000 [WARN] (0017@[0:default:1:1]+example+task3^sub+for-0=i=2=3&1=j=1=2): Skipped
Success. Task state is saved at /home/ec2-user/.digdag/status/20230622T000000+0900 directory.
  * Use --session <daily | hourly | "yyyy-MM-dd[ HH:mm:ss]"> to not reuse the last session time.
  * Use --rerun, --start +NAME, or --goal +NAME argument to rerun skipped tasks.
```

## :pencil2: Digdag の定義

ワークフローファイルの各パーツの説明をまとめておく。

### タスク`+`

タスクとして認識させるためには`+`を利用し、このタスクが上から順に実行されることになるとのこと。タスクはサブタスクを持つことができ、これらをグループタスクとして扱うことでワークフローが管理しやすくなる。

### オペレーター`>`

`>`はオペレーターを意味する。Shell スクリプト`sh>`、Python スクリプト`py>`、Ruby スクリプト`rb>`など様々な種類の演算子が用意されている。

### 変数

`${変数}`をワークフローに変数を埋め込むことができる。組み込み変数も用意されており、ざまざまなフォーマットの日付や日時が利用できる。詳細は[公式ドキュメント](http://docs.digdag.io/workflow_definition.html?highlight=_export#using-variables)を参照。
また」、ここでは javascript が利用でき、Digdag には時間計算用の Moment.js がバンドルされている。

```
timezone: America/Los_Angeles

+format_session_time:
  # "2016-09-24 00:00:00 -0700"
  echo>: ${moment(session_time).format("YYYY-MM-DD HH:mm:ss Z")}

+format_in_utc:
  # "2016-09-24 07:00:00"
  echo>: ${moment(session_time).utc().format("YYYY-MM-DD HH:mm:ss")}

+format_tomorrow:
  # "September 24, 2016 12:00 AM"
  echo>: ${moment(session_time).add(1, 'days').format("LLL")}

+get_execution_time:
  # "2016-09-24 05:24:49 -0700"
  echo>: ${moment().format("YYYY-MM-DD HH:mm:ss Z")}
```

自分で変数を設定する場合は`_export`パラメタを使用する。これはワークフロー冒頭で記載すればグローバルな変数として利用でき、タスク内に記載すればローカルな変数として利用できる。

```
_export:
  foo: 1

+task1:
  ~~~~~

+task2:
  _export:
    bar: 2
```

### 並列実行

タスクを並列実行させることもでき、並列実行させる場合は`_parallel: true`を設定する。下記のように記載すればサブタスク`subtask1,2,3`は並列で実行される。

```
+task1:
  _parallel: true

  +subtask1:
    sh>: task1.sh

  +subtask2:
    sh>: task2.sh

  +subtask3:
    sh>: task3.sh

+task2:
  sh>: task.sh
```

`_background: true`を利用することで、このパラメタが記載されているタスクは前のタスクと並行して実行され、次のタスクは、バックグラウンドタスクが設定されているグループが完了するまで待機させることができる。

### リトライ

実行に失敗した際にリトライする機能もついている。パラメータがグループに設定されている場合、1 つ以上の小タスクが失敗すると、グループのタスクを最初からリトライする。リトライ機能を試すために、下記に簡単なワークフローを定義する。`sh>: ./hello3.sh`を実行しても、このようなファイルはないのでエラーとなる。

```
$ ls
hello.dig   hello1.sh*  hello2.sh*  error.sh*

$ cat hello.dig
timezone: Asia/Tokyo

+task1:
  sh>: echo ${timezone} ${moment().format("YYYY-MM-DD HH:mm:ss Z")}
+task2:
  _retry:
    limit: 3
    interval: 10
    interval_type: exponential

  +subtask1:
    sh>: ./hello1.sh
  +subtask2:
    sh>: ./hello2.sh
  +subtask3:
    sh>: ./hello3.sh
```

`limit`はリトライ回数、`interval`はインターバル秒数、`interval_type`を設定できる。`exponential`であれば指数的にインターバル秒数を増やすため、最初の再試行間隔は 10 秒、2 回目は 20 秒、3 回目は 40 秒となる。説明に必要なログの部分だけを残している。`-----`は説明のために入れている。

```
$ digdag run hello.dig
2023-06-23 22:44:14 +0900 [INFO] (0021@[0:default:1:1]+hello+task1): sh>: echo Asia/Tokyo 2023-06-23 22:44:14 +09:00
Asia/Tokyo 2023-06-23 22:44:14 +09:00
2023-06-23 22:44:14 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask1): sh>: ./hello1.sh
Hello1
2023-06-23 22:44:14 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask2): sh>: ./hello2.sh
Hello2
2023-06-23 22:44:15 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask3): sh>: ./hello3.sh
.digdag/tmp/digdag-sh-6-9003247520186915324/runner.sh: line 1: ./hello3.sh: No such file or directory
2023-06-23 22:44:15 +0900 [ERROR] (0021@[0:default:1:1]+hello+task2+subtask3): Task failed with unexpected error: Command failed with code 127
java.lang.RuntimeException: Command failed with code 127
	at io.digdag.standards.operator.ShOperatorFactory$ShOperator.runCode(ShOperatorFactory.java:121)
  (snip)
	at java.base/java.lang.Thread.run(Thread.java:834)
------------------------------------------------------------------------------------------------------------------------------------------------------------
2023-06-23 22:44:32 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask1): sh>: ./hello1.sh
Hello1
2023-06-23 22:44:32 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask2): sh>: ./hello2.sh
Hello2
2023-06-23 22:44:32 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask3): sh>: ./hello3.sh
.digdag/tmp/digdag-sh-9-586985705750635498/runner.sh: line 1: ./hello3.sh: No such file or directory
2023-06-23 22:44:32 +0900 [ERROR] (0021@[0:default:1:1]+hello+task2+subtask3): Task failed with unexpected error: Command failed with code 127
java.lang.RuntimeException: Command failed with code 127
	at io.digdag.standards.operator.ShOperatorFactory$ShOperator.runCode(ShOperatorFactory.java:121)
	(snip)
	at java.base/java.lang.Thread.run(Thread.java:834)
------------------------------------------------------------------------------------------------------------------------------------------------------------
2023-06-23 22:44:59 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask1): sh>: ./hello1.sh
Hello1
2023-06-23 22:44:59 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask2): sh>: ./hello2.sh
Hello2
2023-06-23 22:44:59 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask3): sh>: ./hello3.sh
.digdag/tmp/digdag-sh-12-6662559149905818705/runner.sh: line 1: ./hello3.sh: No such file or directory
2023-06-23 22:44:59 +0900 [ERROR] (0021@[0:default:1:1]+hello+task2+subtask3): Task failed with unexpected error: Command failed with code 127
java.lang.RuntimeException: Command failed with code 127
	at io.digdag.standards.operator.ShOperatorFactory$ShOperator.runCode(ShOperatorFactory.java:121)
	(snip)
	at java.base/java.lang.Thread.run(Thread.java:834)
------------------------------------------------------------------------------------------------------------------------------------------------------------
2023-06-23 22:45:45 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask1): sh>: ./hello1.sh
Hello1
2023-06-23 22:45:45 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask2): sh>: ./hello2.sh
Hello2
2023-06-23 22:45:45 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask3): sh>: ./hello3.sh
.digdag/tmp/digdag-sh-15-9702633111687091652/runner.sh: line 1: ./hello3.sh: No such file or directory
2023-06-23 22:45:45 +0900 [ERROR] (0021@[0:default:1:1]+hello+task2+subtask3): Task failed with unexpected error: Command failed with code 127
java.lang.RuntimeException: Command failed with code 127
	at io.digdag.standards.operator.ShOperatorFactory$ShOperator.runCode(ShOperatorFactory.java:121)
	(snip)
	at java.base/java.lang.Thread.run(Thread.java:834)
------------------------------------------------------------------------------------------------------------------------------------------------------------
2023-06-23 22:45:45 +0900 [INFO] (0021@[0:default:1:1]+hello^failure-alert): type: notify

error:
  * +hello+task2+subtask3:
    Command failed with code 127 (runtime)
  * +hello+task2+subtask3:
    Command failed with code 127 (runtime)
  * +hello+task2+subtask3:
    Command failed with code 127 (runtime)
  * +hello+task2+subtask3:
    Command failed with code 127 (runtime)
```

### エラー通知

`_error`パラメータを設定すれば、ワークフローが失敗したときにオペレータを実行できる。ドキュメントの例のように、エラーとなった場合、シャルスクリプトが実行できる。

```
_error:
  sh>: tasks/runs_when_workflow_failed.sh

+analyze:
  sh>: tasks/analyze_prepared_data_sets.sh
```

ここでは、サブタスクの 3 つ目でエラーをおこし、`error.sh`を実行するようにする。`error.sh`は`Some jobs were not successful`というメッセージをエコーするだけ。

```
$ cat error.sh
#!/bin/bash

echo 'Some jobs were not successful'
```

ワークフローは下記の通り。

```
$ cat hello.dig
timezone: Asia/Tokyo

+task1:
  sh>: echo ${timezone} ${moment().format("YYYY-MM-DD HH:mm:ss Z")}
+task2:
  +subtask1:
    sh>: ./hello1.sh
  +subtask2:
    sh>: ./hello2.sh
  +subtask3:
    sh>: ./hello3.sh
    _error:
      sh>: ./error.sh
```

サブタスク 3 でエラーが発生し、`error.sh`が実行されていることがわかる。

```
$ digdag run hello.dig

2023-06-23 23:10:39 +0900 [INFO] (0021@[0:default:1:1]+hello+task1): sh>: echo Asia/Tokyo 2023-06-23 23:10:39 +09:00
Asia/Tokyo 2023-06-23 23:10:39 +09:00
2023-06-23 23:10:39 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask1): sh>: ./hello1.sh
Hello1
2023-06-23 23:10:40 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask2): sh>: ./hello2.sh
Hello2
2023-06-23 23:10:40 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask3): sh>: ./hello3.sh
.digdag/tmp/digdag-sh-6-3563231010638112199/runner.sh: line 1: ./hello3.sh: No such file or directory
2023-06-23 23:10:40 +0900 [ERROR] (0021@[0:default:1:1]+hello+task2+subtask3): Task failed with unexpected error: Command failed with code 127
java.lang.RuntimeException: Command failed with code 127
	at io.digdag.standards.operator.ShOperatorFactory$ShOperator.runCode(ShOperatorFactory.java:121)
  (snip)
	at java.base/java.lang.Thread.run(Thread.java:834)
2023-06-23 23:10:40 +0900 [INFO] (0021@[0:default:1:1]+hello+task2+subtask3^error): sh>: ./error.sh
Some jobs were not successful
2023-06-23 23:10:40 +0900 [INFO] (0021@[0:default:1:1]+hello^failure-alert): type: notify
error:
  * +hello+task2+subtask3:
    Command failed with code 127 (runtime)

```

エラー通知はメールで通知することも可能で、`mail>`に必要な情報を記載する。詳細は[こちら](https://docs.digdag.io/operators/mail.html)。

## :closed_book: Reference

- [Digdag](https://docs.digdag.io/)
