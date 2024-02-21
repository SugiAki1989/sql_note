## :memo: Overview

ここでは DBeaverでSQLを書く際の設定に関するメモをまとめておく。基本的な使い方などはネットで検索すれば出てくるので、ここではSQLを記述する際に関係する設定を中心にまとめている。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`DBeaver`

## :pencil2: SQLエディタの画面

postgreSQLを接続した例だと、下記のように表示される。左側にデータベース、スキーマ、テーブル、カラムという階層関係で表示される。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-base.png)

スキーマをダブルクリックすると、スキーマ内の各テーブルの概要が表示される。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-schema.png)

テーブルをダブルクリックすると、テーブルの詳細が表示される。カラムの定義やテーブルをプレビューできる。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-colmun.png)

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-table.png)

## :pencil2: フォーマッタの設定

設定画面までは下記の順序をたどる。

- Path: DBeaver > Preferences > エディタ > SQLエディタ > SQL書式設定

フォーマッタを利用する方法は、エディタで右クリックして「整形 > SQLの整形」から利用する。デフォルトのフォーマッタはやや使いづらいかもしれない。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-formatter.png)

## :pencil2: テンプレートの設定

設定画面までは下記の順序をたどる。

- Path: DBeaver > Preferences > エディタ > SQLエディタ > テンプレート

テンプレートを呼び出す方法は、エディタで右クリックして「テンプレート」から呼び出す。ここでは下記のテンプレートを呼び出せるようにする。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-template.png)

## :pencil2: SQLエディタのカラー設定

設定画面までは下記の順序をたどる。

- Path: DBeaver > Preferences > ユーザーインターフェイス > 外観 > 色とフォント

SQLエディタの文字色を役割に応じて変更できる。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-color.png)

ここではメインフォントをFira Codeに変更してみた。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-font.png)


## :pencil2: ツールバーのカスタマイズ設定

設定画面までは下記の順序をたどる。

- Path: DBeaver > Preferences > ユーザーインターフェイス > Toolbar Customization

各ツールバーのカスタマイズできたり、アイコンの役割を確認できる。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p134-toolbar.png)

## :closed_book: Reference

none