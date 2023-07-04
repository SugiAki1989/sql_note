## :memo: Overview

ここでは Embulk でデータ転送を行うための基本的な方法をまとめておく。構成としては、AWS 環境下の EC2 で Embulk を実行し、同じ AWS 環境下の SFTP サーバーに置かれている CSV を MySQL にロードすることが目的。

- [SFTP file input plugin for Embulk](https://github.com/embulk/embulk-input-sftp)
- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`embulk-input-sftp`, `embulk-output-mysql`

## :pencil2: SFTP サーバーの構築

SFTP サーバーを利用したことはあっても、SFTP サーバーを構築した経験はないので、下記の記事をもとに SFTP サーバーは構築した。非常に丁寧にまとめられているので問題なく構築できた。ちなみに、AWS Transfer for SFTP などもあると思うが、実際に手を動かして SFTP サーバーを作る、そのために、どのような設定が最低限必要なのかを学べて非常によかった。

[SFTP サーバにファイルを 〜サーバ構築編〜](https://blog.serverworks.co.jp/tech/2020/05/07/post-83932/)

まずはルートでユーザー`sftp-etl-user`を作成。その後は SSH に必要なキーを作成し、Embulk を実行する EC2 に転送しておく。これがないと Embulk のインプット設定で、鍵を指定できない。また、`authorized_keys`は SFTP サーバーに接続する際に認証に使用される公開鍵のリスト、とのことなので、これをコピーしておく。

```
[EC2 SFTP root]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ sudo su -
# useradd sftp-etl-user
# passwd sftp-etl-user
# sudo su - sftp-etl-user

[EC2 SFTP User]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ mkdir .ssh
$ chmod 700 .ssh
$ cd .ssh
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/sftp-etl-user/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/sftp-etl-user/.ssh/id_rsa.
Your public key has been saved in /home/sftp-etl-user/.ssh/id_rsa.pub.
The key fingerprint is:
(snip)

$ cp id_rsa.pub authorized_keys
$ chmod 600 authorized_keys
$ ls
authorized_keys id_rsa id_rsa.pub
// id_rsa はEmbulkを動かすEC2にコピー
```

OpenSSH の設定ファイルを書き換えて、SFTP サーバーとして機能するようにする。

```
[EC2 SFTP root]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ exit
# vim /etc/ssh/sshd_config

------------------------------------------
# override default of no subsystems
Subsystem sftp  /usr/libexec/openssh/sftp-server

# set sftp info
Match user sftp-etl-user
       ChrootDirectory /home/sftp-etl-user/root-dir
       ForceCommand internal-sftp
------------------------------------------

# service sshd reload
# chown root: /home/sftp-etl-user
# chmod 755 /home/sftp-etl-user
# cd /home/sftp-etl-user
# mkdir root-dir
# chown root: root-dir
# chmod 755 root-dir

# cd root-dir
# mkdir sftp-work-dir
# chown sftp-user: sftp-work-dir
# chmod 777 sftp-work-dir
```

これで SFTP サーバーの設定は完了。

## :pencil2:　ローカル PC から SFTP サーバーに接続

ローカル端末から SFTP サーバーにアクセスして、Embulk の転送で利用する csv を保存しておく。

```
[Local PC]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# chmod 755  ~/.ssh/aws-sftp-ssh-key.pem
$ sftp -i ~/.ssh/aws-sftp-ssh-key.pem sftp-etl-user@22.222.222.222
sftp> cd sftp-work-dir
sftp> put ~/Desktop/data20230701.csv
sftp> put ~/Desktop/data20230702.csv
sftp> put ~/Desktop/data20230703.csv
sftp> ls
data20230701.csv  data20230702.csv  data20230703.csv
```

ちなみに今回使用する csv ファイルは、果物屋の注文履歴データ。5 日分のサンプルデータを用意している。

```
[Local PC]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ cat data20230701.csv
"user_id","order_date","order_id","product_name","price"
"user01","2023-07-01","order01","apple","100"
"user02","2023-07-01","order02","banana","50"
"user03","2023-07-01","order03","orange","75"
"user04","2023-07-01","order04","melon","200"
"user05","2023-07-01","order05","apple","100"

$ cat data20230702.csv
"user_id","order_date","order_id","product_name","price"
"user06","2023-07-02","order06","strawberry","150"
"user07","2023-07-02","order07","raspberry","100"
"user08","2023-07-02","order08","kiwi","75"
"user09","2023-07-02","order09","grapefruit","100"
"user10","2023-07-02","order10","coconut","250"

$ cat data20230703.csv
"user_id","order_date","order_id","product_name","price"
"user11","2023-07-03","order11","pineapple","150"
"user12","2023-07-03","order12","papaya","100"
"user13","2023-07-03","order13","cherry","75"
"user14","2023-07-03","order14","durian","100"
"user15","2023-07-03","order15","grape","150"

$ cat data20230704.csv
"user_id","order_date","order_id","product_name","price"
"user16","2023-07-04","order16","plum","150"
"user17","2023-07-04","order17","blueberry","100"
"user18","2023-07-04","order18","muscat","300"
"user19","2023-07-04","order19","muskmelon","400"
"user20","2023-07-04","order20","mango","150"

$ cat data20230705.csv
"user_id","order_date","order_id","product_name","price"
"user21","2023-07-05","order21","yuzu","100"
"user22","2023-07-05","order22","lime","100"
"user23","2023-07-05","order23","lemon","100"
"user24","2023-07-05","order24","apricot","200"
"user25","2023-07-05","order25","fig","150"
```

## :pencil2:　 Embulk(SFTP > STDOUT)

Embulk を実行する EC2 から SFTP サーバーにアクセスできるかどうか、疎通の確認をしておく。保存しておいた csv ファイルを確認できたので、問題なく SFTP サーバーにアクセスできている。

```
[EC2 ETL server]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# id_rsa: aws-sftp-ssh-key.pemとする
# SFTPサーバーにアクセスできるかどうか、疎通確認
$ sftp -i ~/.ssh/aws-sftp-ssh-key.pem sftp-etl-user@22.222.222.222
sftp> cd sftp-work-dir/
sftp> ls
data20230701.csv  data20230702.csv  data20230703.csv
```

今回はインプットが SFTP サーバーになるので、下記のプラグインをインストールしておく。

```
$ embulk gem install embulk-input-sftp
```

今回の yml ファイルは下記の通り。`secret_key_file`には SFTP サーバーで作成した秘密鍵を指定し、`path_prefix`は SFTP サーバーで csv を保存するディレクトリにした場所で、`path_match_pattern`で末尾が csv のファイルを対象にしている。`path_match_pattern`で正規表現でファイルをマッチできる。そして、ここでは csv ファイルをインプットにしているので、`parser`が必要になる。また、`incremental`を設定することで、増分ロードを有効にできる。

```
$ cat embulk_sftp_mysql.yml

in:
  type: sftp
  host: 22.222.222.222
  port: 22
  user: sftp-etl-user
  secret_key_file: /path/to/ssh-key.pem
  timeout: 600
  path_prefix: /sftp-work-dir
  path_match_pattern: .csv$
  incremental: true
  parser:
    charset: UTF-8
    newline: CRLF
    type: csv
    delimiter: ','
    quote: '"'
    escape: '"'
    null_string: ''
    skip_header_lines: 1
    columns:
      - { name: user_id, type: string }
      - { name: order_date, type: timestamp, format: '%Y-%m-%d' }
      - { name: order_id, type: string }
      - { name: product_name, type: string }
      - { name: price, type: long }
out:
  type: stdout
```

`-c`をつけて実行すると、問題なく csv の内容が標準出力されていることがわかる。

```
$ embulk run -c embulk.diff.yml embulk_sftp_mysql.yml

2023-07-03 22:26:09.677 +0900 [INFO] (0001:transaction): {done:  0 / 3, running: 0}
user01,2023-07-01,order01,apple,100
user02,2023-07-01,order02,banana,50
user03,2023-07-01,order03,orange,75
user04,2023-07-01,order04,melon,200
user05,2023-07-01,order05,apple,100
user11,2023-07-03,order11,pineapple,150
user12,2023-07-03,order12,papaya,100
user13,2023-07-03,order13,cherry,75
user14,2023-07-03,order14,durian,100
user15,2023-07-03,order15,grape,150
2023-07-03 22:26:10.271 +0900 [INFO] (0001:transaction): {done:  2 / 3, running: 1}
user06,2023-07-02,order06,strawberry,150
user07,2023-07-02,order07,raspberry,100
user08,2023-07-02,order08,kiwi,75
user09,2023-07-02,order09,grapefruit,100
user10,2023-07-02,order10,coconut,250
2023-07-03 22:26:10.303 +0900 [INFO] (0001:transaction): {done:  3 / 3, running: 0}
2023-07-03 22:26:10.304 +0900 [INFO] (0001:transaction): {done:  3 / 3, running: 0}
2023-07-03 22:26:10.313 +0900 [INFO] (main): Next config diff: {"in":{"last_path":"/sftp-work-dir/data20230703.csv"},"out":{}}
```

`incremental`が機能しているか確認しておく。前回実行時の`last_path`は`data20230703.csv`なので、次回の実行ではこれよりも前のファイルはスキップされるはずなので、SFTP サーバーに csv を追加でアップロードして、Embulk を再度実行する。`incremental`が機能しているので、`2023-07-04`の情報が記録されている csv しか出力されていないことがわかる。

```
$ cat embulk.diff.yml
in: {last_path: /sftp-work-dir/data20230703.csv}
out: {}

[EC2 SFTP User]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
sftp> put ~/Desktop/data20230704.csv

[EC2 ETL server]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ embulk run -c embulk.diff.yml embulk_sftp_mysql.yml

2023-07-03 22:28:28.161 +0900 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
user16,2023-07-04,order16,plum,150
user17,2023-07-04,order17,blueberry,100
user18,2023-07-04,order18,muscat,300
user19,2023-07-04,order19,muskmelon,400
user20,2023-07-04,order20,mango,150
2023-07-03 22:28:28.556 +0900 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2023-07-03 22:28:28.569 +0900 [INFO] (main): Next config diff: {"in":{"last_path":"/sftp-work-dir/data20230704.csv"},"out":{}}
```

準備は整ったので、MySQL にデータ転送する前に、不要なファイルは削除して、環境をもとに戻しておく。

```
# Mysqlへの転送のため1度削除
$ rm embulk.diff.yml

[EC2 SFTP User]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# Mysqlへの転送のため1度削除
sftp> rm data20230704.csv
```

## :pencil2:　 Embulk(SFTP > MySQL)

パブリックサブネットの EC2 からしかプライベートサブネットの MySQL にはアクセスできないので、EC2 から MySQL にアクセスして転送先のテーブルを作成する。

```
[MySQL]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ mysql -u your_name -h your_host -p
MySQL [(none)]> create database sftp_db;
MySQL [(none)]> use sftp_db;
MySQL [sftp_db]>
create table sftp_orders (
  user_id varchar(255),
  order_date date,
  order_id varchar(255),
  product_name varchar(255),
  price int
);
```

yml ファイルに MySQL の情報を追加しておく。

```
$ cat embulk_sftp_mysql.yml

in:
  type: sftp
  host: 22.222.222.222
  port: 22
  user: sftp-etl-user
  secret_key_file: /path/to/ssh-key.pem
  timeout: 600
  path_prefix: /sftp-work-dir
  path_match_pattern: .csv$
  incremental: true
  parser:
    charset: UTF-8
    newline: CRLF
    type: csv
    delimiter: ','
    quote: '"'
    escape: '"'
    null_string: ''
    skip_header_lines: 1
    columns:
      - { name: user_id, type: string }
      - { name: order_date, type: timestamp, format: '%Y-%m-%d' }
      - { name: order_id, type: string }
      - { name: product_name, type: string }
      - { name: price, type: long }
out:
  type: mysql
  host: your_host
  user: your_user
  password: your_password
  database: sftp_db
  table: sftp_orders
  mode: insert
```

あとはこのファイルを利用して Embulk を実行する。

```
[EC2 ETL server]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ embulk run -c embulk.diff.yml embulk_sftp_mysql.yml
(snip)
2023-07-04 13:45:41.687 +0900 [INFO] (main): Next config diff: {"in":{"last_path":"/sftp-work-dir/data20230703.csv"},"out":{}}

$ cat embulk.diff.yml;
in: {last_path: /sftp-work-dir/data20230703.csv}
out: {}
```

実行結果を確認すると、問題なく MySQL のテーブルにデータが転送されている。

```
[MySQL]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
MySQL [sftp_db]> select * from sftp_orders;
+---------+------------+----------+--------------+-------+
| user_id | order_date | order_id | product_name | price |
+---------+------------+----------+--------------+-------+
| user01  | 2023-07-01 | order01  | apple        |   100 |
| user02  | 2023-07-01 | order02  | banana       |    50 |
| user03  | 2023-07-01 | order03  | orange       |    75 |
| user04  | 2023-07-01 | order04  | melon        |   200 |
| user05  | 2023-07-01 | order05  | apple        |   100 |
| user06  | 2023-07-02 | order06  | strawberry   |   150 |
| user07  | 2023-07-02 | order07  | raspberry    |   100 |
| user08  | 2023-07-02 | order08  | kiwi         |    75 |
| user09  | 2023-07-02 | order09  | grapefruit   |   100 |
| user10  | 2023-07-02 | order10  | coconut      |   250 |
| user11  | 2023-07-03 | order11  | pineapple    |   150 |
| user12  | 2023-07-03 | order12  | papaya       |   100 |
| user13  | 2023-07-03 | order13  | cherry       |    75 |
| user14  | 2023-07-03 | order14  | durian       |   100 |
| user15  | 2023-07-03 | order15  | grape        |   150 |
+---------+------------+----------+--------------+-------+
15 rows in set (0.00 sec)

```

先程と同じように、`2023-07-04`の情報が記録されている csv を SFTP サーバーに追加して、再度実行する。

```
[EC2 SFTP User]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
sftp> put ~/Desktop/data20230704.csv

[EC2 ETL server]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ embulk run -c embulk.diff.yml embulk_sftp_mysql.yml
(snip)

$ cat embulk.diff.yml;
in: {last_path: /sftp-work-dir/data20230704.csv}
out: {}
```

重複してインサートされることもなく、思った通りに機能している。続けて`2023-07-05`の情報が記録されていること csv で試してみても、問題なくデータ転送が実行されている。

```
[MySQL]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
MySQL [sftp_db]> select * from sftp_orders;
+---------+------------+----------+--------------+-------+
| user_id | order_date | order_id | product_name | price |
+---------+------------+----------+--------------+-------+
| user01  | 2023-07-01 | order01  | apple        |   100 |
| user02  | 2023-07-01 | order02  | banana       |    50 |
| user03  | 2023-07-01 | order03  | orange       |    75 |
| user04  | 2023-07-01 | order04  | melon        |   200 |
| user05  | 2023-07-01 | order05  | apple        |   100 |
| user06  | 2023-07-02 | order06  | strawberry   |   150 |
| user07  | 2023-07-02 | order07  | raspberry    |   100 |
| user08  | 2023-07-02 | order08  | kiwi         |    75 |
| user09  | 2023-07-02 | order09  | grapefruit   |   100 |
| user10  | 2023-07-02 | order10  | coconut      |   250 |
| user11  | 2023-07-03 | order11  | pineapple    |   150 |
| user12  | 2023-07-03 | order12  | papaya       |   100 |
| user13  | 2023-07-03 | order13  | cherry       |    75 |
| user14  | 2023-07-03 | order14  | durian       |   100 |
| user15  | 2023-07-03 | order15  | grape        |   150 |
| user16  | 2023-07-04 | order16  | plum         |   150 |
| user17  | 2023-07-04 | order17  | blueberry    |   100 |
| user18  | 2023-07-04 | order18  | muscat       |   300 |
| user19  | 2023-07-04 | order19  | muskmelon    |   400 |
| user20  | 2023-07-04 | order20  | mango        |   150 |
+---------+------------+----------+--------------+-------+
20 rows in set (0.00 sec)

[EC2 SFTP User]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
sftp> put ~/Desktop/data20230705.csv

[EC2 ETL server]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ embulk run -c embulk.diff.yml embulk_sftp_mysql.yml
(snip)

$ cat embulk.diff.yml;
in: {last_path: /sftp-work-dir/data20230705.csv}
out: {}

[MySQL]~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
MySQL [sftp_db]> select * from sftp_orders;
+---------+------------+----------+--------------+-------+
| user_id | order_date | order_id | product_name | price |
+---------+------------+----------+--------------+-------+
| user01  | 2023-07-01 | order01  | apple        |   100 |
| user02  | 2023-07-01 | order02  | banana       |    50 |
| user03  | 2023-07-01 | order03  | orange       |    75 |
| user04  | 2023-07-01 | order04  | melon        |   200 |
| user05  | 2023-07-01 | order05  | apple        |   100 |
| user06  | 2023-07-02 | order06  | strawberry   |   150 |
| user07  | 2023-07-02 | order07  | raspberry    |   100 |
| user08  | 2023-07-02 | order08  | kiwi         |    75 |
| user09  | 2023-07-02 | order09  | grapefruit   |   100 |
| user10  | 2023-07-02 | order10  | coconut      |   250 |
| user11  | 2023-07-03 | order11  | pineapple    |   150 |
| user12  | 2023-07-03 | order12  | papaya       |   100 |
| user13  | 2023-07-03 | order13  | cherry       |    75 |
| user14  | 2023-07-03 | order14  | durian       |   100 |
| user15  | 2023-07-03 | order15  | grape        |   150 |
| user16  | 2023-07-04 | order16  | plum         |   150 |
| user17  | 2023-07-04 | order17  | blueberry    |   100 |
| user18  | 2023-07-04 | order18  | muscat       |   300 |
| user19  | 2023-07-04 | order19  | muskmelon    |   400 |
| user20  | 2023-07-04 | order20  | mango        |   150 |
| user21  | 2023-07-05 | order21  | yuzu         |   100 |
| user22  | 2023-07-05 | order22  | lime         |   100 |
| user23  | 2023-07-05 | order23  | lemon        |   100 |
| user24  | 2023-07-05 | order24  | apricot      |   200 |
| user25  | 2023-07-05 | order25  | fig          |   150 |
+---------+------------+----------+--------------+-------+
25 rows in set (0.00 sec)
```

今回は試していないけど`proxy`の設定をすることで、プロキシサーバー経由で SFTP サーバーのファイルを取得できるとのこと。

## :closed_book: Reference

- [SFTP file input plugin for Embulk](https://github.com/embulk/embulk-input-sftp)
- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)
