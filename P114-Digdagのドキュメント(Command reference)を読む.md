## :memo: Overview

ここでは Digdag のドキュメントを読みながら、Digdag でできることをまとめていく。個人的に気になったポイントをまとめているだけ。

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Digdag のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

- [Digdag](https://docs.digdag.io/)

## :floppy_disk: Database

None

## :bookmark: Tag

`Digdag`

## :pencil2: Digdag のコマンド

ここでは主に Local-mode commands を中心に digdag を実行しながらまとめていく。Server-mode commands、Client-mode commands はここではまとめてないので、下記のドキュメントを参照。

- [Local-mode commands](https://docs.digdag.io/command_reference.html#local-mode-commands)
- [Server-mode commands](https://docs.digdag.io/command_reference.html#server-mode-commands)
- [Client-mode commands](https://docs.digdag.io/command_reference.html#client-mode-commands)

共通オプションについては先にまとめておく。

| Options                 | Description                                                                                                                                         |
| :---------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------- |
| `-L, --log PATH`        | ログをファイルに出力する。デフォルトは STDOUT。ログファイルは 10MB ごとにローテーションされ、gzip で圧縮され、最大 5 つの古いファイルが保持される。 |
| `-l, --log-level LEVEL` | ログレベルを変更する。trace, debug, info, warn, or error。                                                                                          |

### Local-mode commands

#### `init`

`init`コマンドは新しいワークフローのプロジェクトディレクトリを作成する。`-t sh`とすれば指定したタイプに応じたプロジェクトファイルを作成してくれる。

```
$ digdag init mydag
$ tree
.
└── mydag
    └── mydag.dig

$ cat mydag/mydag.dig
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

#### `run`

ワークフローを実行するときは、`run`コマンドを利用する。オプションを色々選択できる。

```
$ digdag run <workflow.dig> [+task] [options...]
```

| Options                 | Description                                                                                                                                                                                                                                                                                                                  |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--project DIR`         | 指定したディレクトリをプロジェクトディレクトリとして使用する。デフォルトはカレントディレクトリ                                                                                                                                                                                                                               |
| `-o, --save DIR`        | 指定したディレクトリをセッションステータスの読み取りと書き込みに利用する。デフォルトは`.digdag/status`。タスクが正常終了した場合、Digdag はこのディレクトリにファイルを作成する。digdag を再実行すると、このディレクトリにファイルが存在する場合、タスクがスキップされる。失敗したワークフローを途中から再開する場合に便利。 |
| `-a, --rerun`           | 以前にタスクが正常に終了した場合でも、`status`ディレクトリ内のファイルを無視し、すべてのタスクを再実行する。                                                                                                                                                                                                                 |
| `-s、--start +TASKNAME` | 以前にタスクが正常に終了した場合でも、このタスクと後続のタスクを実行 s うる。他のタスクは、状態ファイルがディレクトリに保存されている場合はスキップされる                                                                                                                                                                    |
| `-g, --goal +TASKNAME`  | タスクが以前に正常に終了した場合でも、このタスクとその子タスクを実行する。他のタスクは、状態ファイルがディレクトリに保存されている場合はスキップされる。                                                                                                                                                                     |
| `-e、--end +TASKNAME`   | このタスクの直前でワークフローを停止します。このタスクと後続のタスクはスキップされます。                                                                                                                                                                                                                                     |

`-o, --save DIR`の詳細を確認しておく。前回利用した下記のワークフローファイルはシェルスクリプトが存在してないので、サブタスクの 3 つ目でエラーが起こってしまう。この場合のステータスの状態を確認してみると、`+hello+task2+subtask3^error.yml`となっておりサブタスクの 3 つ目でエラーになって、`./error.sh`が実行されていることがわかる。

```
$ digdag run hello.dig
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

$ ls .digdag/status/20230624T000000+0900
+hello+task1.yml
+hello+task2+subtask1.yml
+hello+task2+subtask2.yml
+hello+task2+subtask3^error.yml
+hello^failure-alert.yml
```

| Options                  | Description                                                                                                |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `--session EXPR`         | session_time を設定する。`daily`、`hourly`、`schedule`、`last`、`timestamp` が利用可能。デフォルトは`last` |
| `--no-save`              | セッションの状態ファイルを無効にする                                                                       |
| `--max-task-threads N`   | タスク実行スレッドの最大数を制限する                                                                       |
| `-O, --task-log DIR`     | タスク ログをこのディレクトリに保存する                                                                    |
| `-p, --param KEY=VALUE`  | セッション パラメーターを追加する                                                                          |
| `-P、--params-file PATH` | YAML/JSON ファイルからパラメータを読み取る。ネストされたパラメータは`.`で繋げてアクセスする。              |
| `-d, --dry-run`          | ドライランを実行する                                                                                       |
| `-E, --show-params`      | タスクを実行する前に、タスクに与えられた計算パラメータを表示する。                                         |

#### `check`

`check`コマンドは、ワークフローの定義とスケジュールを表示する。`run`コマンドと同じ`--project`、`-p`、`-P`が利用できる。

```
$ digdag check [workflow.dig] [options...]
```

`check`コマンドを実行するとタイムゾーン、定義、タスク数、パラメタ、スケジュールが表示される。

```
$ digdag check hello.dig
2023-06-24 14:13:08 +0900: Digdag v0.10.5
  System default timezone: Asia/Tokyo

  Definitions (1 workflows):
    hello (6 tasks)

  Parameters:
    {}

  Schedules (0 entries):
```

#### `scheduler`

`scheduler`コマンドでスケジュールを定期的に実行するワークフロースケジューラを実行する。カレントディレクトリにある`.dig`サフィックスが付いた名前のすべてのワークフロー定義ファイルが取得される。`scheduler`コマンドでスケジュールを実行すると、`http://127.0.0.1:65432`にアクセスして UI でワークフローやジョブを管理できる。

```
$ digdag scheduler
```

`--project`、Web インターフェイスと API クライアントをリッスンするポート番号`-p`、HTTP クライアントをリッスンするための IP アドレス`-b`などが利用できる。他にも、ステータスをデータベースに保存する`-o, --database DIR`、タスクログを保存する`-O, --task-log DIR`、`--max-task-threads N`、`-p`、`-P`が利用できる。設定ファイルを読み込む場合は`-c, --config PATH`で読み込む。

```
$ cat hello.dig
timezone: Asia/Tokyo

schedule:
  minutes_interval>: 1

+task1:
  sh>: echo ${timezone} ${moment().format("YYYY-MM-DD HH:mm:ss Z")}
```

スケジューラーを実行するときは`scheduler`コマンドを実行するだけでよく、`minutes_interval`は`1`に設定しているので、1 分ごとに繰り返されることになる。時間間隔は下記の通り設定できる。

| Options                 | Description                     |
| :---------------------- | :------------------------------ |
| `daily>: HH:MM:SS`      | 毎日 HH:MM:SS に実行            |
| `hourly>: MM:SS`        | 毎時 MM:SS に実行               |
| `weekly>: DDD,HH:MM:SS` | 毎週 DDD 曜日の HH:MM:SS に実行 |
| `monthly>: D,HH:MM:SS`  | 毎月 D 日の HH:MM:SS に実行     |
| `cron>: * * * * *`      | cron 形式で実行                 |

```
$ digdag scheduler
(snip)
2023-06-24 14:44:01 +0900 [INFO] (0035@[0:default:1:1]+hello+task1): sh>: echo Asia/Tokyo 2023-06-24 14:44:01 +09:00
Asia/Tokyo 2023-06-24 14:44:01 +09:00
(snip)
2023-06-24 14:45:00 +0900 [INFO] (0035@[0:default:2:2]+hello+task1): sh>: echo Asia/Tokyo 2023-06-24 14:45:00 +09:00
Asia/Tokyo 2023-06-24 14:45:00 +09:00
(snip)
2023-06-24 14:46:00 +0900 [INFO] (0037@[0:default:3:3]+hello+task1): sh>: echo Asia/Tokyo 2023-06-24 14:46:00 +09:00
Asia/Tokyo 2023-06-24 14:46:00 +09:00
```

スケジュールされたタスクが想定時間内に終了しているかどうかを知るためのアラートを`sla`で設定できる。他にも、次のワークフローセッションをスキップしたり、開始終了を設定できたりする。詳細は下記の公式ドキュメントを参照。

- [ワークフローのスケジュール設定](https://docs.digdag.io/scheduling_workflow.html#)

```
timezone: UTC

schedule:
  daily>: 07:00:00

sla:
  # triggers this task at 02:00
  time: 02:00
  +notice:
    sh>: notice.sh

+long_running_job:
  sh>: long_running_job.sh
```

`scheduler`コマンドは指定したワークフローディレクトリの`.dig`ファイルをすべて読み込みとあったので、下記のような挨拶をするワークフローファイルを 2 つ用意して実行してみる。

```
$ cat hello.dig
timezone: Asia/Tokyo

schedule:
  minutes_interval>: 1

+task1:
  sh>: echo 'Hello!' ${moment().format("YYYY-MM-DD HH:mm:ss Z")}

$ cat goodbye.dig
timezone: Asia/Tokyo

schedule:
  minutes_interval>: 2

+task1:
  sh>: echo 'GoodBye!' ${moment().format("YYYY-MM-DD HH:mm:ss Z")}
```

わかりやすくするために転機する際にはタブを入れている点は注意。

```
$ digdag scheduler
(snip)
2023-06-24 14:50:01 +0900 [INFO] (0035@[0:default:1:1]+hello+task1): sh>: echo 'Hello!' 2023-06-24 14:50:01 +09:00
2023-06-24 14:50:01 +0900 [INFO] (0036@[0:default:2:2]+goodbye+task1): sh>: echo 'GoodBye!' 2023-06-24 14:50:01 +09:00
  Hello! 2023-06-24 14:50:01 +09:00
  GoodBye! 2023-06-24 14:50:01 +09:00
(snip)
2023-06-24 14:51:01 +0900 [INFO] (0035@[0:default:3:3]+hello+task1): sh>: echo 'Hello!' 2023-06-24 14:51:01 +09:00
  Hello! 2023-06-24 14:51:01 +09:00
(snip)
2023-06-24 14:52:00 +0900 [INFO] (0035@[0:default:4:4]+hello+task1): sh>: echo 'Hello!' 2023-06-24 14:52:00 +09:00
  Hello! 2023-06-24 14:52:00 +09:00
2023-06-24 14:52:00 +0900 [INFO] (0039@[0:default:5:5]+goodbye+task1): sh>: echo 'GoodBye!' 2023-06-24 14:52:00 +09:00
  GoodBye! 2023-06-24 14:52:00 +09:00
(snip)
2023-06-24 14:53:00 +0900 [INFO] (0040@[0:default:6:6]+hello+task1): sh>: echo 'Hello!' 2023-06-24 14:53:00 +09:00
  Hello! 2023-06-24 14:53:00 +09:00
(snip)
2023-06-24 14:54:01 +0900 [INFO] (0043@[0:default:8:8]+goodbye+task1): sh>: echo 'GoodBye!' 2023-06-24 14:54:01 +09:00
2023-06-24 14:54:01 +0900 [INFO] (0042@[0:default:7:7]+hello+task1): sh>: echo 'Hello!' 2023-06-24 14:54:01 +09:00
  GoodBye! 2023-06-24 14:54:01 +09:00
  Hello! 2023-06-24 14:54:01 +09:00
(snip)
2023-06-24 14:55:00 +0900 [INFO] (0042@[0:default:9:9]+hello+task1): sh>: echo 'Hello!' 2023-06-24 14:55:00 +09:00
  Hello! 2023-06-24 14:55:00 +09:00
```

`check`コマンドを利用すれば、いつから最初のスケジュールが実行されるかを確認できる。

```
$ digdag check
  System default timezone: Asia/Tokyo

  Definitions (2 workflows):
    hello (2 tasks)
    goodbye (2 tasks)

  Parameters:
    {}

  Schedules (2 entries):
    hello:
      minutes_interval>: 1
      first session time: 2023-06-24 15:44:00 +0900
      first scheduled to run at: 2023-06-24 15:44:00 +0900 (in 25s)
    goodbye:
      minutes_interval>: 2
      first session time: 2023-06-24 15:44:00 +0900
      first scheduled to run at: 2023-06-24 15:44:00 +0900 (in 25s)
```

#### `selfupdate`

`selfupdate`コマンドは実行可能なバイナリファイルを最新バージョンまたは指定されたバージョンに更新する。

```
$ digdag selfupdate [version]
```

## :closed_book: Reference

- [Digdag](https://docs.digdag.io/)
