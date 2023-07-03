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

SFTP サーバーを利用したことはあっても、構築した経験はないので、下記の記事をもとに SFTP サーバーは構築した。

[SFTP サーバにファイルを 〜サーバ構築編〜](https://blog.serverworks.co.jp/tech/2020/05/07/post-83932/)

構築手順まとめる

```
a
```

## :pencil2:　ローカル PC から SFTP サーバーに接続

転送する csv を作成して、SFTP サーバーに転送する内容をまとめる。

```
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

aaaa

```

## :pencil2:　 Embulk(SFTP > STDOUT)

EC2 から SFTP サーバーに疎通、保存 csv ファイルを確認。
下記の`embulk run`実行までの手順をまとめる

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

yml の`incremental: true`が機能しているか確認する内容をまとめる。

```
aaa
```

## :pencil2:　 Embulk(SFTP > MySQL)

yml を書き直して、MySQL に転送す内容を書く。

```

```

## :closed_book: Reference

- [SFTP file input plugin for Embulk](https://github.com/embulk/embulk-input-sftp)
- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)
