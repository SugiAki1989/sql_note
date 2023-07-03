## :memo: Overview

ここでは Embulk の実行に必要なコンフィグファイルの設定方法について、ドキュメントを読みながらまとめていく。個人的に気になったポイントをまとめているだけ。

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

- [Embulk Docs](https://www.embulk.org/docs/built-in.html)

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`Embulk`, `guess`, `preview`

## :pencil2: Embulk のコンフィグファイル

Embulk のコンフィグファイルは yaml 形式で記載する。基本的には`in`と`out`の 2 つをブロックにわけて、設定を記載する。`in`にはインプットプラグイン、`out`にはアウトプットプラグインが対応する。その他にも`filters`、`exec`ブロックも記述できる。

```
in:
  type: file
  path_prefix: ./mydata/csv/
  decoders:
  - {type: gzip}
  parser:
    charset: UTF-8
    newline: CRLF
    type: csv
    delimiter: ','
    quote: '"'
    escape: '"'
    null_string: 'NULL'
    skip_header_lines: 1
    columns:
    - {name: id, type: long}
    - {name: account, type: long}
    - {name: time, type: timestamp, format: '%Y-%m-%d %H:%M:%S'}
    - {name: purchase, type: timestamp, format: '%Y%m%d'}
    - {name: comment, type: string}
filters:
  - type: speedometer
    speed_limit: 250000
out:
  type: stdout
```

## :pencil2: インプット

`in`には、レコードベース(MySQL、DynamoDB など)やファイルベース(S3、HTTP など)などが利用でき、`parser`や`decoder`を利用して、csv、json、gzip、zip、tar などを解析、デコードする設定を記述できる。`out`には、レコードベースやファイルベースが利用でき、`formatter`や`encoder`を利用して、csv、jsonl、gzip、などをフォーマット、エンコードする設定を記述できる。環境変数を使うこともでき、その場合は`.yml.liquid`形式でコンフィグファイルを用意する。

ローカルファイルシステムの場合、複数のファイルを読み込む場合、下記のようなファイル構成であれば、`path_prefix: /path/to/files/sample_`と書けばよいとのこと。これで`sample_`のファイルが読みこみ対象になる。`last_path: /path/to/files/sample_02.csv`と設定すれば、丸かっこ内の挙動になる。

```
.
`-- path
    `-- to
        `-- files
            |-- sample_01.csv   -> read (skip)
            |-- sample_02.csv   -> read (skip)
            |-- sample_03.csv   -> read (read)
            |-- sample_04.csv   -> read (read)

```

csv パーサープラグインでは、他の csv パーサーと同じく様々な設定を記述できる。詳細は[こちら](https://www.embulk.org/docs/built-in.html#:~:text=CSV%20parser-,plugin,-The%20csv%20parser)。

UNIX 秒(1470148959)は`format: '%s'`でサポートされているが、ミリ秒まではサポートしてないので、`long`型で解析してから、`timestamp_format`プラグインを適用して、`long`型をタイムスタンプに変換することで解析できるとのこと。

```
in:
  type: file
  path_prefix: /my_csv_files
  parser:
    ...
    columns:
    - {name: timestamp_in_seconds, type: timestamp, format: '%s'}
    - {name: timestamp_in_millis, type: long}
filters:
  - type: timestamp_format
    columns:
      - {name: timestamp_in_millis, from_unit: ms}
```

他にも、カラムの設定を自動的に生成してくれる`guess`コマンドは便利。

```
$ mkdir test_guess
$ cd test_guess/
$ embulk example ./try1
```

`example`コマンドで生成されたファイルを利用して、手元にあったサンプル csv で実行してみた。

```
# $ scp ~/Desktop/rhc.csv ec2-user@11.111.111.1:~/test_guess/try1/csv/
$ tree .
.
├── csv
│   └── rhc.csv
└── seed.yml

$ cat seed.yml
in:
  type: file
  path_prefix: '/home/ec2-user/test_guess/try1/csv/rhc.csv'
out:
  type: stdout
```

大量にカラムがある場合はとりあえず、自動でカラムを生成してくれるので便利。これをたたきにコマコマ修正する。

```
$ embulk guess ./seed.yml -o config.yml
in:
  type: file
  path_prefix: /home/ec2-user/test_guess/try1/csv/rhc.csv
  parser:
    charset: UTF-8
    newline: CRLF
    type: csv
    delimiter: ','
    quote: '"'
    escape: '"'
    trim_if_not_quoted: false
    skip_header_lines: 1
    allow_extra_columns: false
    allow_optional_columns: false
    columns:
    - {name: ptid, type: long}
    - {name: age, type: double}
    - {name: sex, type: string}
    - {name: aps1, type: long}
    - {name: chfhx, type: long}
    - {name: cat1, type: string}
    - {name: cat2, type: string}
    - {name: ca, type: string}
    - {name: death.num, type: long}
    - {name: death, type: boolean}
    - {name: cardiohx, type: long}
    - {name: dementhx, type: long}
    - {name: psychhx, type: long}
    - {name: chrpulhx, type: long}
    - {name: renalhx, type: long}
    - {name: liverhx, type: long}
    - {name: gibledhx, type: long}
    - {name: malighx, type: long}
    - {name: immunhx, type: long}
    - {name: transhx, type: long}
    - {name: amihx, type: long}
    - {name: edu, type: double}
    - {name: surv2md1, type: double}
    - {name: das2d3pc, type: double}
    - {name: t3d30, type: long}
    - {name: dth30, type: boolean}
    - {name: dth30.num, type: long}
    - {name: scoma1, type: long}
    - {name: meanbp1, type: long}
    - {name: wblc1, type: double}
    - {name: hrt1, type: long}
    - {name: resp1, type: long}
    - {name: temp1, type: double}
    - {name: pafi1, type: double}
    - {name: alb1, type: double}
    - {name: hema1, type: double}
    - {name: bili1, type: double}
    - {name: crea1, type: double}
    - {name: sod1, type: long}
    - {name: pot1, type: double}
    - {name: paco21, type: long}
    - {name: ph1, type: double}
    - {name: swang1.cat, type: string}
    - {name: swang1, type: long}
    - {name: rhc, type: long}
    - {name: wtkilo1, type: double}
    - {name: dnr1, type: boolean}
    - {name: ninsclas, type: string}
    - {name: resp, type: boolean}
    - {name: card, type: boolean}
    - {name: neuro, type: boolean}
    - {name: gastr, type: boolean}
    - {name: renal, type: boolean}
    - {name: meta, type: boolean}
    - {name: hema, type: boolean}
    - {name: seps, type: boolean}
    - {name: trauma, type: boolean}
    - {name: ortho, type: boolean}
    - {name: adld3p, type: string}
    - {name: urin1, type: string}
    - {name: race, type: string}
    - {name: income, type: string}
out: {type: stdout}

Created 'config.yml' file.
```

公式ドキュメントに記載されている通り、csv には含まれてない日時関係であればフォーマット含め推測してくれる。

```
    columns:
    - {name: id, type: long}
    - {name: account, type: long}
    - {name: time, type: timestamp, format: '%Y-%m-%d %H:%M:%S'}
    - {name: purchase, type: timestamp, format: '%Y%m%d'}
    - {name: comment, type: string}
```

`guess`コマンドのために作ったディレクトリは不要なので削除しておく。

```
$ rm -r test_guess
```

## :pencil2: アウトプット

インプットと同様、アウトプットも様々なオプションを設定できる。例えば、csv を転送する際は、下記のように転送先のディレクトリを指定して、ファイル名のルールを設定できる。`sequence_format`で、ファイル名のナンバリングをインクリメントできる。その他の csv のアウトプットに関する設定は[こちら](https://www.embulk.org/docs/built-in.html#:~:text=csv%0A%20%20formatter%3A%0A%20%20%20%20...-,CSV%E3%83%95%E3%82%A9%E3%83%BC%E3%83%9E%E3%83%83%E3%82%BF%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3,-%E3%83%95%E3%82%A9%E3%83%BC%E3%83%9E%E3%83%83%E3%82%BFcsv%E3%83%97%E3%83%A9%E3%82%B0)。

```
out:
  type: file
  path_prefix: /path/to/output/sample_
  sequence_format: "%03d.%02d."
  file_ext: csv
  formatter:

.
`-- path
    `-- to
        `-- output
            |-- sample_01.000.csv
            |-- sample_02.000.csv
            |-- sample_03.000.csv
            |-- sample_04.000.csv

```

## :pencil2: フィルター

Embulk にはフィルターでデータクレンジングできる機能がついている。この機能をいくつか試しておく。フィルター機能を使う前に、`preview`コマンドでドライランが実行できることを確認しておく。

```
$ mkdir test_preview
$ cd test_preview/
$ embulk example .
$ embulk guess ./seed.yml -o config.yml
$ embulk preview config.yml

2023-06-21 20:14:34.354 +0900: Embulk v0.9.25
2023-06-21 20:14:35.183 +0900 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2023-06-21 20:14:37.176 +0900 [INFO] (main): Gem's home and path are set by default: "/home/ec2-user/.embulk/lib/gems"
2023-06-21 20:14:37.795 +0900 [INFO] (main): Started Embulk v0.9.25
2023-06-21 20:14:37.890 +0900 [INFO] (0001:preview): Listing local files at directory '/home/ec2-user/test_preview/./csv' filtering filename by prefix 'sample_'
2023-06-21 20:14:37.891 +0900 [INFO] (0001:preview): "follow_symlinks" is set false. Note that symbolic links to directories are skipped.
2023-06-21 20:14:37.892 +0900 [INFO] (0001:preview): Loading files [/home/ec2-user/test_preview/./csv/sample_01.csv.gz]
2023-06-21 20:14:37.901 +0900 [INFO] (0001:preview): Try to read 32,768 bytes from input source
+---------+--------------+-------------------------+-------------------------+----------------------------+
| id:long | account:long |          time:timestamp |      purchase:timestamp |             comment:string |
+---------+--------------+-------------------------+-------------------------+----------------------------+
|       1 |       32,864 | 2015-01-27 19:23:49 UTC | 2015-01-27 00:00:00 UTC |                     embulk |
|       2 |       14,824 | 2015-01-27 19:01:23 UTC | 2015-01-27 00:00:00 UTC |               embulk jruby |
|       3 |       27,559 | 2015-01-28 02:20:02 UTC | 2015-01-28 00:00:00 UTC | Embulk "csv" parser plugin |
|       4 |       11,270 | 2015-01-29 11:54:36 UTC | 2015-01-29 00:00:00 UTC |                            |
+---------+--------------+-------------------------+-------------------------+----------------------------+
```

まずは、`remove_columns`フィルタ。これは指定したカラムを落としてくれる。ここでは、`purchase, comment`を削除してみる。

```
$ cat config.yml
in:
  type: file
  path_prefix: /home/ec2-user/test_preview/./csv/sample_
  decoders:
  - {type: gzip}
  parser:
    charset: UTF-8
    newline: LF
    type: csv
    delimiter: ','
    quote: '"'
    escape: '"'
    null_string: 'NULL'
    trim_if_not_quoted: false
    skip_header_lines: 1
    allow_extra_columns: false
    allow_optional_columns: false
    columns:
    - {name: id, type: long}
    - {name: account, type: long}
    - {name: time, type: timestamp, format: '%Y-%m-%d %H:%M:%S'}
    - {name: purchase, type: timestamp, format: '%Y%m%d'}
    - {name: comment, type: string}
out: {type: stdout}
filters:
  - type: remove_columns
    remove: ["purchase", "comment"]

$ embulk preview config.yml

+---------+--------------+-------------------------+
| id:long | account:long |          time:timestamp |
+---------+--------------+-------------------------+
|       1 |       32,864 | 2015-01-27 19:23:49 UTC |
|       2 |       14,824 | 2015-01-27 19:01:23 UTC |
|       3 |       27,559 | 2015-01-28 02:20:02 UTC |
|       4 |       11,270 | 2015-01-29 11:54:36 UTC |
+---------+--------------+-------------------------+
```

フィルターは複数機能させることができるようで、下記のようにカラム名を小文字から大文字に変換することも可能。

```
filters:
    - type: remove_columns
      remove: ["purchase", "comment"]
    - type: rename
      rules:
      - rule: lower_to_upper

$ embulk preview config.yml
+---------+--------------+-------------------------+
| ID:long | ACCOUNT:long |          TIME:timestamp |
+---------+--------------+-------------------------+
|       1 |       32,864 | 2015-01-27 19:23:49 UTC |
|       2 |       14,824 | 2015-01-27 19:01:23 UTC |
|       3 |       27,559 | 2015-01-28 02:20:02 UTC |
|       4 |       11,270 | 2015-01-29 11:54:36 UTC |
+---------+--------------+-------------------------+
```

他にも追加のプラグインを利用すれば、ハッシュ化することができる。この場合、フィルターの順序に注意が必要で、`account`を`ACCOUNT`に変換しているので、ハッシュ化する対象は大文字でなければ、カラム名が一致せずエラーとなってしまう。

```
$ embulk gem install embulk-filter-hash

filters:
    - type: remove_columns
      remove: ["purchase", "comment"]
    - type: rename
      rules:
      - rule: lower_to_upper
    - type: hash
      columns:
      - { name: ACCOUNT }

$ embulk preview config.yml
+---------+------------------------------------------------------------------+-------------------------+
| ID:long |                                                   ACCOUNT:string |          TIME:timestamp |
+---------+------------------------------------------------------------------+-------------------------+
|       1 | 5001005fce342d61388dbfdd08106571334f950e64bfc909206a5eb9d5bf9792 | 2015-01-27 19:23:49 UTC |
|       2 | 715d9526f896b1d61038e6563101c6caed617c79061201657660a9ea9e545bee | 2015-01-27 19:01:23 UTC |
|       3 | 78900179607eef6f475fd5887fcf9f238ebb9265967e3ff72f175902656e9676 | 2015-01-28 02:20:02 UTC |
|       4 | 0ca09f3815a1a11d4c18f8f468291f391d2ee0716a1f96083eb0bde233afa43c | 2015-01-29 11:54:36 UTC |
+---------+------------------------------------------------------------------+-------------------------+
```

他にもカラム名を一意にするための`unique_number_suffix`フィルタなどもある模様。さがせば色々と出てきそう。最後に後片付けしておく。

```
$ rm -r test_preview/
```

## :closed_book: Reference

- [Embulk](https://www.embulk.org/)
