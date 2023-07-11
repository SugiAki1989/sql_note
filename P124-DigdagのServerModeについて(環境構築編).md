## :memo: Overview

ここでは Digdag の Server モードについて基本的な方法をまとめておく。ただ、少し長くなったので今回は環境構築までしかまとまっていない。これまでも何回か Digdag についてまとめていたが、これまでは Local モードを利用しており、ここでは Local モードではなく、Server モードでの使い方をまとめている。

- [Digdag - Server-mode commands](https://docs.digdag.io/command_reference.html#server-mode-commands)

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。また、下記の記事を参考にさせていただいた。非常にわかりやすくまとまっていて、たいへん助かった。

- [Digdag サーバの設定メモ](https://qiita.com/chocomintkusoyaro/items/3dbf4141e098d8dde9da)

一方で、私の知識がゆるふわなため、役割に応じてユーザーを分けて、実行させるうんぬんくんぬんが適切にできていない場合がある。これは参考にした記事のせいではなく、私の力量によるもの。

## :floppy_disk: Database

None

## :bookmark: Tag

`digdag`

## :pencil2:　 Digdag Server

新しい EC2 を利用して Digdag Server を構築していく。Digdag を Server モードで利用する場合、情報の保存先として PostgreSQL を使うことが多いみたいなので、PostgreSQL の設定も合わせて行っていく。ワークフロー情報や履歴、スケジュール、実行ログ、タスクキューの情報を記録できるとのこと。

まずは、EC2 に java をインストールしておく。頻繁にユーザーを切り替えるので、どのユーザーでコマンドを実行しているのかをわかりやすくするために、ユーザー名もメモしている(後で見返すときに絶対忘れているので…)。

```
# set up
[ec2-user]$ sudo yum update -y
# 必要あれば
[ec2-user]$ sudo yum install -y tree　

# java8
[ec2-user]$ sudo yum install -y java-1.8.0-openjdk
```

次は EC2 で digdag 用のユーザー(ホームディレクトリは`/opt/digdag`とする)を作成する。また、各種ログを格納する保存先を作成しておく。

```
# digdagグループを作って、digdagグループにユーザーdigdagを追加する
[ec2-user]$ sudo groupadd digdag
[ec2-user]$ sudo useradd -g digdag -d /opt/digdag digdag
[ec2-user]$ sudo passwd digdag
[ec2-user]$ sudo mkdir /opt/digdag/logs            # digdag serverログ
[ec2-user]$ sudo mkdir /opt/digdag/logs/accesslogs # digdag WebUIのログ
[ec2-user]$ sudo mkdir /opt/digdag/logs/tasklogs   # digdag ワークフローのログ
[ec2-user]$ sudo chown -R digdag:digdag /opt/digdag
```

もろもろ作成したので確認しておく。

```
[ec2-user]$ sudo su digdag
[digdag]$ cd ~
[digdag]$ pwd
/opt/digdag

[digdag]$ ls -l ./logs
合計 0
drwxr-xr-x 2 digdag digdag 6  7月 11 06:59 accesslogs
drwxr-xr-x 2 digdag digdag 6  7月 11 06:59 tasklogs
```

ここからは PostgreSQL のインストールを行っていく。`postgresql-contrib`を忘れると Digdag Server の起動時にエラーが出るので忘れずに。[こちらの issue](https://github.com/treasure-data/digdag/issues/1619)を参考に解決できた。

```
# postgresql
# amazon-linux-extras list | grep postgresql14
# 63 postgresql14 available [ =stable ]

[ec2-user]$ sudo amazon-linux-extras install -y postgresql14
[ec2-user]$ sudo yum install -y postgresql-server postgresql-devel postgresql-contrib
[ec2-user]$ psql --version
psql (PostgreSQL) 14.8

# postgresユーザーがインストールと同時に作られる
[ec2-user]$ cat /etc/passwd | grep postgres
postgres:x:26:26:PostgreSQL Server:/var/lib/pgsql:/bin/bash

[ec2-user]$ ls -la /var/lib/ | grep pgsql
drwx------  4 postgres postgres   83  7月 11 05:54 pgsql
```

インストールができたので、DB クラスタを初期化する。そして、`systemctl start`コマンドで PostgreSQL を起動して、自動起動を有効にしておく。

```
[ec2-user]$ sudo postgresql-setup initdb
WARNING: using obsoleted argument syntax, try --help
WARNING: arguments transformed to: postgresql-setup --initdb --unit postgresql
 * Initializing database in '/var/lib/pgsql/data'
 * Initialized, logs are in /var/lib/pgsql/initdb_postgresql.log

[ec2-user]$ sudo systemctl start postgresql
[ec2-user]$ sudo systemctl enable postgresql
Created symlink from /etc/systemd/system/multi-user.target.wants/postgresql.service to /usr/lib/systemd/system/postgresql.service.
```

お次は`postgres`ユーザーに変わって、まっさらな PostgreSQL に接続。`postgres`のパスワードを設定し、`digdag`ロールを作成し、パスワードを設定する。ログの保存先 DB は`digdag_db`とする。

また、`uuid-ossp`の拡張を有効にしておく。`uuid-ossp`モジュールは UUID を生成する関数を提供してくれ、これは最初にインストールした`postgresql-contrib`から提供される。

```
[ec2-user]$ sudo su postgres
[postgres]$ cd ~
[postgres]$ ls
backups  data  initdb_postgresql.log

[postgres]$ psql -U postgres -d postgres
=# ALTER USER postgres PASSWORD 'postgresのpassword';
=# CREATE ROLE digdag WITH PASSWORD 'digdagロールのパスワード' NOSUPERUSER NOCREATEDB NOCREATEROLE LOGIN;
=# CREATE DATABASE digdag_db WITH OWNER digdag;
=# CREATE EXTENSION IF NOT EXISTS "uuid-ossp"
=# \q
```

次は、PostgreSQL の`pg_hba.conf`、`postgresql.conf`を修正して、パスワード認証やリッスンアドレスを有効にする。修正後は再起動して設定を反映させる。

- [21.1. pg_hba.conf ファイル](https://www.postgresql.jp/document/14/html/auth-pg-hba-conf.html)
- [20.1.2. 設定ファイルによるパラメータ操作](https://www.postgresql.jp/document/14/html/config-setting.html#CONFIG-SETTING-CONFIGURATION-FILE)

```
[postgres]$ vim data/pg_hba.conf

# "local" is for Unix domain socket connections only
local   all             all                                     peer
↓
# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             digdag             127.0.0.1/32            md5
```

コメントアウトで無効化されているので、コメントアウトを外しておく。

```
[postgres]$ vim data/postgresql.conf

# listen_addresses = 'localhost' # what IP address(es) to listen on;
↓
listen_addresses = 'localhost'   # what IP address(es) to listen on;

[postgres]$ exit
[ec2-user]$ sudo systemctl restart postgresql
```

PostgreSQL のパスワード認証の設定次第では、接続の際にエラーが発生するが、おそらく下記のサイトをみれば解決できるはず。インストールしたままだと、peer 認証がデフォルト。peer 認証は、クライアント上のシステムユーザ名と PostgreSQL データベースユーザが同じであれば接続ができてしまう認証方法のこと。

- [PostgreSQL “対向(peer)認証に失敗しました” エラーが出るときの対処法](https://cpoint-lab.co.jp/article/201807/4217/)

ユーザー`digdag`で PostgreSQL に接続できるかどうかテストしておく。

```
[ec2-user]$ psql -U digdag -d digdag_db
ユーザー digdag のパスワード:*****
digdag_db=> \l
                                         データベース一覧
   名前    |  所有者  | エンコーディング |  照合順序   | Ctype(変換演算子) |     アクセス権限
-----------+----------+------------------+-------------+-------------------+-----------------------
 digdag_db | digdag   | UTF8             | en_US.UTF-8 | en_US.UTF-8       |
 postgres  | postgres | UTF8             | en_US.UTF-8 | en_US.UTF-8       |
 template0 | postgres | UTF8             | en_US.UTF-8 | en_US.UTF-8       | =c/postgres          +
           |          |                  |             |                   | postgres=CTc/postgres
 template1 | postgres | UTF8             | en_US.UTF-8 | en_US.UTF-8       | =c/postgres          +
           |          |                  |             |                   | postgres=CTc/postgres
```

Digdag Server の設定ファイル`server.properties`を用意する。

```
[ec2-user]$ sudo su digdag
[digdag]$ cd ~
$ vim server.properties

database.type = postgresql
database.user = digdag
database.password = digdagロールのパスワード
database.host = 127.0.0.1
database.port = 5432
database.database = digdag_db
database.maximumPoolSize = 32
server.bind = 0.0.0.0
```

これで Digdag Server を立ち上げられると思ったが、そもそも Digdag も検証で使う Embulk もインストールしてないことに気がついたので、公式ドキュメントの内容に沿ってインストールしておく。

```
# embulk
[digdag]$ curl --create-dirs -o ~/.embulk/bin/embulk -L "https://dl.embulk.org/embulk-0.9.25.jar"
[digdag]$ chmod +x ~/.embulk/bin/embulk
[digdag]$ echo 'export PATH="$HOME/.embulk/bin:$PATH"' >> ~/.bashrc
[digdag]$ source ~/.bashrc
[digdag]$ embulk -version
embulk 0.9.25

# digdag
[digdag]$ curl -o ~/bin/digdag --create-dirs -L "https://dl.digdag.io/digdag-latest"
[digdag]$ chmod +x ~/bin/digdag
[digdag]$ echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
[digdag]$ source ~/.bashrc
[digdag]$ digdag --version
0.10.5
```

一通り設定が完了したので、`digdag server`コマンドでサーバーを立ち上げる。

```
[digdag]$ digdag server --config /opt/digdag/server.properties
```

起動すると、`http://{EC2 の IPV4 アドレス}:65432`にアクセスすることで、WebUI を利用できるようになる。ただ、EC2 に設定されているセキュリティグループのインバウンドルールに、65432 ポート(digdag server のデフォルトポート)へのアクセスを許可しておく必要がある。この設定が抜けていたので数十分ほど悩んでいた…。問題なくアクセスできれば下記の画面が表示される。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p124−Digdag.png)

これで終了と行きたいところだが、最後の仕上げとしてサービスとして起動出来るようにしておく。

```
[ec2-user]$ cd /etc/systemd/system
[ec2-user]$ sudo vim digdag.service
```

`digdag.service`の内容は下記の通り。

```
[Unit]
Description=digdag

[Service]
Type=simple
User=digdag
Restart=always
RestartSec=5
TimeoutStartSec=30s
WorkingDirectory=/opt/digdag
KillMode=process

ExecStart=/bin/bash -c '/opt/digdag/bin/digdag server --config /opt/digdag/server.properties --log /opt/digdag/logs/digdag_server.log --task-log /opt/digdag/tasklogs --access-log /opt/digdag/accesslogs --bind 0.0.0.0'

[Install]
WantedBy=multi-user.target
```

これで、デーモン化できたので EC2 を再起動すれば勝手にサービスが起動する。

```
[ec2-user]$ sudo systemctl enable digdag
[ec2-user]$ sudo systemctl start digdag
[ec2-user]$ sudo systemctl status digdag
● digdag.service - digdag
   Loaded: loaded (/etc/systemd/system/digdag.service; enabled; vendor preset: disabled)
   Active: active (running) since 火 2023-07-11 10:07:32 UTC; 8s ago

# エラーが出るのであれば修正して
# $ sudo systemctl daemon-reload
# $ sudo systemctl restart digdag
```

サービスが起動しているので、URL にアクセスすれば 先程と同じく WebUI から確認できる。ちなみに、ElasticIP アドレスはお金がかかる、かつ学習用なので設定していない。そのため、URL に必要な IP アドレスが起動のたびに変わるので注意。

![Digdag WebUI](https://github.com/SugiAki1989/sql_note/blob/main/image/p124−Digdag.png)

下記はおまけ。サービスの基本的な項目に関する設定ファイルの書き方のおさらい。

```
[Unit]
Description: Unitの説明文を記載
Requires: このUnitが必要なほかのUnitを記載
Before: このUnitよりも後に起動するUnitを記載

[Service]
User: 実行ユーザー
WorkingDirectory: サービスが実行されるときのカレントディレクトリ
ExecStart: サービスの起動コマンド

[Install]
WantedBy: 動作モード
```

## :closed_book: Reference

- [Digdag サーバの設定メモ](https://qiita.com/chocomintkusoyaro/items/3dbf4141e098d8dde9da)
- [Digdag 入門](https://recruit.gmo.jp/engineer/jisedai/blog/introduction-to-digdag/)
- [EC2 上の digdag server に外部からアクセスする](https://www.capybara-engineer.com/entry/2020/11/11/205119)
