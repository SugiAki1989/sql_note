## :memo: Overview

こでは Embulk でデータ転送を行うための基本的な方法をまとめておく。構成としては、AWS 環境下の EC2 で docker ファイルからコンテナを作成し、コンテナ内の Embulk を実行し、同じ AWS 環境下の MySQL から S3 のバケットに csv を digdag で定期的にロードする構成。

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`Embulk`, `Digdag`, `Docker`

## :pencil2: Docker ファイルの準備

まずは Docker ファイルを作成する。Web サイトで検索すれば、いろんな方が Embulk と Digdag を使える Docker ファイルを書いてくれているのの、どれもイメージの作成で失敗したので使えなかった。理由はおそらくサイトの情報に誤りがあるのではなく、私のローカルの PC が Mac の M1 シリコンだからで、M1 シリコンと Docker の関係によるものと思われる。そのため、`java8`や`openjdk:8`でイメージを作り始めるとエラーになる。`arm64v8/openjdk:8`とかを使ってみるものの、次は Embulk のプラグインをインストールする際に JRuby のエラーが出てしまい、Java も Ruby、JRuby も詳しくないので、原因の切り分けができず、素直に自分で Docker ファイルを作り直した。

Docker ファイルの基本的な構成は、過去のメモでも AmazonLinux2 で作業していたので、`amazonlinux:2`イメージから、`java8`、`Embulk`、`Digdag`をインストールしている。また、`dig`ファイルや`yml.liqid`ファイルは、`tar`ファイルにして`ADD`でコンテナ内の`etl`ディレクトリに取り込んでいる。`ADD`で転送したファイルの権限周りも適切な変更方法、管理方法がよく分かっておらず、とりあえず今回は動けばよいので、root に変更している。相変わらず私はこのサーバー管理もろもろの知識、技術がない…。

```
$ mkdir docker_context
$ cd docker_context
$ vim dockerfile
$ cat dockerfile
FROM amazonlinux:2

# install amazon-linux-extras
RUN yum install amazon-linux-extras -y

# yum update & install
RUN yum update -y \
    && yum install java-1.8.0-openjdk -y \
    && yum install mysql -y

# Set Timezone
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime \
    && echo "Asia/Tokyo" > /etc/timezone

# Embulk
RUN curl --create-dirs -o ~/.embulk/bin/embulk -L "https://dl.embulk.org/embulk-0.9.25.jar" &&\
    chmod +x ~/.embulk/bin/embulk && \
    echo 'export PATH="$HOME/.embulk/bin:$PATH"' >> ~/.bashrc && \
    source ~/.bashrc && \
    embulk gem install embulk-input-mysql embulk-output-s3

# Digdag
RUN curl -o ~/bin/digdag --create-dirs -L "https://dl.digdag.io/digdag-latest" && \
    chmod +x ~/bin/digdag && \
    echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc && \
    source ~/.bashrc

ADD compressed.tar /root/
RUN chown -R root:root /root/etl && chmod -R 755 /root/etl
WORKDIR /root/etl

# Exec shell
CMD ["/bin/bash"]
```

Docker ファイルの準備が終わったので、下記の通りビルドコンテキストを作成して、EC2 に転送する。

```
[local]
// ビルドコンテキストにファイルを移動してtarファイルに圧縮する。
$ tree etl/
etl/
├── embulk_etl.dig
├── embulk_mysql_s3.yml.liquid
└── exec_embulk_mysql.sh

$ tar -cvf compressed.tar etl
$ rm -r etl
$ ls
compressed.tar dockerfile

$ ssh -i ~/.ssh/{your key}.pem ec2-user@11.111.11.111
$ scp -r ~/Desktop/docker_context ec2-user@511.111.11.111:~
```

これで準備は完了なので、以降は EC2 で作業を行う。ちなみに`dig`、`yml.liqid`、`sh`ファイルは前回利用したものを使っている。

```
$ cat embulk_etl.dig
timezone: 'Asia/Tokyo'

schedule:
  cron>: '* * * * *'

+task1:
  sh>: ~/etl/exec_embulk_mysql.sh

-------------------------------------------------------------------------------------------
$ cat exec_embulk_mysql.sh
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
sed -i "s|path_prefix:.*|path_prefix: ${new_path_prefix}|" ~/etl/embulk_mysql_s3.yml.liquid

# embulk_mysql_s3.yml.liquid
~/.embulk/bin/embulk run -c ~/etl/embulk.diff.yml ~/etl/embulk_mysql_s3.yml.liquid
cp -r ~/etl/embulk_mysql_s3.yml.liquid ~/etl/embulk_mysql_s3_bk.yml.liquid

-------------------------------------------------------------------------------------------
$ cat embulk_mysql_s3.yml.liquid
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
  path_prefix: out/data_20230625114600
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

## :pencil2: データ転送

EC2 では、Docker をインストールして、転送されたビルドコンテキストを利用して、イメージを作成する。これには 5 分ほどかかった。

```
# dockerをインストールする
$ sudo yum -y install docker
$ sudo systemctl start docker
$ sudo systemctl enable docker
$ sudo gpasswd -a ec2-user docker
$ exit

$ cd docker_context/
$ docker build -t docker-etl .
```

作成されたイメージ`docker-etl`を利用してコンテナにアクセスする。

```
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
docker-etl    latest    a3b2d3850b9a   51 seconds ago   1.39GB
amazonlinux   2         e1c55520046f   4 days ago       165MB

$ docker run -it docker-etl
```

デフォルトで`etl`ディレクトリにアクセスできるようにしているので、あとは Digdag を実行すれば OK。

```
[Docker Container]
# digdag scheduler
2023-06-25 11:33:55 +0900: Digdag v0.10.5
2023-06-25 11:33:57 +0900 [INFO] (main): secret encryption engine: disabled
(snip)
2023-06-25 11:46:11 +0900 [INFO] (shutdown): Shutdown completed
```

一旦、スケジューラを強制終了し、ディレクトリの状態を確認すると、これまでの通り EC2 で動かした後と同じファイル構成となっている。

```
# ls
embulk.diff.yml  embulk_etl.dig  embulk_mysql_s3.yml.liquid  embulk_mysql_s3_bk.yml.liquid  exec_embulk_mysql.sh
```

S3 の転送されたデータを確認しておく。設定した通り、1 分ごとにデータを転送できている。`id`が飛んでいるのは転送条件で`flg=1`を設定しているため。

```
$ aws s3 ls s3://embulk-mysql-to-s3/out/
2023-06-25 11:34:11     115895 data_20230625113400.csv
2023-06-25 11:35:11        154 data_20230625113501.csv
2023-06-25 11:36:11        154 data_20230625113600.csv
2023-06-25 11:37:10        265 data_20230625113700.csv
2023-06-25 11:38:11        263 data_20230625113800.csv
2023-06-25 11:39:11        154 data_20230625113900.csv
2023-06-25 11:40:11        207 data_20230625114000.csv
2023-06-25 11:41:11        153 data_20230625114100.csv
2023-06-25 11:42:11        265 data_20230625114200.csv
2023-06-25 11:43:11        208 data_20230625114301.csv
2023-06-25 11:44:10         42 data_20230625114400.csv
2023-06-25 11:45:10        265 data_20230625114500.csv
2023-06-25 11:46:10         98 data_20230625114600.csv

$ aws s3 cp s3://embulk-mysql-to-s3/out/data_20230625113400.csv - | head -10
"id","datetime","value1","category","flg"
"3","2023-06-16 14:00:19.000000 +0900","-10","e","1"
"4","2023-06-16 14:00:29.000000 +0900","-11","9","1"
"5","2023-06-16 14:00:39.000000 +0900","-7","2","1"
"6","2023-06-16 14:00:49.000000 +0900","-4","4","1"
"7","2023-06-16 14:00:59.000000 +0900","-9","4","1"
"9","2023-06-16 14:01:19.000000 +0900","-1","1","1"
"10","2023-06-16 14:01:29.000000 +0900","-6","e","1"
"12","2023-06-16 14:01:49.000000 +0900","-18","8","1"
"13","2023-06-16 14:01:59.000000 +0900","-3","3","1"

$ aws s3 cp s3://embulk-mysql-to-s3/out/data_20230625114600.csv - | head -10
"id","datetime","value1","category","flg"
"4168","2023-06-25 11:45:09.000000 +0900","-20","4","1"
```

## :closed_book: Reference

- [Amazon Linux 2 の Docker イメージから開発環境を作り Visual Studio Code で接続してみる](https://dev.classmethod.jp/articles/amazon-linux-2-docker-aws-cli-visual-studio-code/)
- [Embulk × Digdag on Docker](https://qiita.com/dich1/items/17bf2ea4cf818bc66d1a)
- [GCE で digdag+embulk の ETL 基盤を構築してみた](https://qiita.com/k_0120/items/0313499451da240cce91#etldig-1)
