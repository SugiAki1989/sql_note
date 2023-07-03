## :memo: Overview

ここでは Embulk の MySQL プラグインの転送モードについてドキュメントを読みながらまとめていく。個人的に気になったポイントをまとめているだけ。特に`merge`の転送では、`INSERT INTO <target*table> ...ON DUPLICATE KEY UPDATE ...`クエリを利用するので、このクエリがまずどのように機能するのかを確認した後で、Embulk の`merge`転送の動きを確認する。

- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`insert`, `insert_direct`, `truncate_insert`, `merge`, `merge_direct`, `replace`

## :pencil2: 転送モードの説明

Embulk の[MySQL output plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)の転送モードには、下記の通り説明がある。

`mode` は`insert`、`insert_direct`、`truncate_insert`、`merge`、`merge_direct`、`replace`が利用できる。

- `insert`:

  - Behavior: 初めにいくつかの中間テーブルにレコードを書き込む。これらのタスクがすべて正しく実行された場合、`INSERT INTO <target*table> SELECT * FROM <intermediate*table_1> UNION ALL SELECT * FROM <intermediate_table_2> UNION ALL ...`クエリを実行し、ターゲットテーブルが存在しない場合は、自動的に作成してインサートする。
  - Transactional: Yes。このモードでは、全ての行の書き込みに成功するか、0 行の書き込みに失敗します。
  - Resumable: No

- `insert_direct`:

  - Behavior: ターゲットテーブルに直接レコードをインサートする。ターゲットテーブルが存在しない場合は自動的に作成する。
  - Transactional: 失敗した場合、ターゲットテーブルにいくつかのレコードがインサートされる可能性がある。
  - Resumable: No

- `truncate_insert`:

  - Behavior: `insert` モードと同じだが、最後の `INSERT ...`クエリの直前にターゲットテーブルを`truncate`する。
  - Transactional: Yes
  - Resumable: No

- `replace`:

  - Behavior: まず中間テーブルにレコードを書き込む。これらのタスクがすべて正しく実行された場合、ターゲットテーブルを削除し、中間テーブルの名前をターゲットテーブルの名前に変更する。
  - Transactional: 失敗した場合、ターゲットテーブルは削除される可能性がある（MySQL はロールバック DDL ができないため）。
  - Resumable: No

- `merge`:

  - Behavior: まずいくつかの中間テーブルにレコードを書き込む。これらのタスクが全て正しく実行された場合、`INSERT INTO <target*table> SELECT * FROM <intermediate*table_1> UNION ALL SELECT * FROM <intermediate_table_2> UNION ALL ...ON DUPLICATE KEY UPDATE ...`クエリを実行。つまり、中間テーブルのレコードの主キーがターゲットテーブルに既に存在する場合、ターゲットレコードは中間レコードによって更新され、主キーがターゲットテーブルに存在しない場合は中間テーブルのレコードがインサートされる。ターゲットテーブルが存在しない場合は、自動的に作成してインサートする。
  - Transactional: Yes
  - Resumable: No

- `merge_direct`:
  - Behavior: `INSERT INTO ...ON DUPLICATE KEY UPDATE ...`クエリを使用してターゲットテーブルに直接レコードをインサートする。クエリを使用してターゲット・テーブルに直接行を挿入します。ターゲットテーブルが存在しない場合は、自動的に作成してインサートする。
  - Transactional：No
  - Resumable: No

今回ここで扱うのは、`merge`の転送について。このモードでは、`INSERT INTO <target*table> ...ON DUPLICATE KEY UPDATE ...`クエリを利用する。このクエリが、どのように機能するのか確認しておく。

## :pencil2: MySQL の ON DUPLICATE KEY UPDATE

まずはテストのデータベースにテーブルを作成し、テーブルをインサートする。

```
mysql> create database mode_test;

mysql> create table modes_from (
  id int auto_increment,
  big varchar(5),
  small varchar(5),
  createdat timestamp default current_timestamp,
  updatedat timestamp default current_timestamp,
  primary key (id)
);

mysql> desc modes_from;
+-----------+------------+------+-----+-------------------+-------------------+
| Field     | Type       | Null | Key | Default           | Extra             |
+-----------+------------+------+-----+-------------------+-------------------+
| id        | int        | NO   | PRI | NULL              | auto_increment    |
| big       | varchar(5) | YES  |     | NULL              |                   |
| small     | varchar(5) | YES  |     | NULL              |                   |
| createdat | timestamp  | YES  |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
| updatedat | timestamp  | YES  |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
+-----------+------------+------+-----+-------------------+-------------------+

mysql>
insert into modes_from (big, small) values
  ('A', 'a'),
  ('B', 'b')
;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | A    | a     | 2023-07-02 11:50:49 | 2023-07-02 11:50:49 |
|  2 | B    | b     | 2023-07-02 11:50:49 | 2023-07-02 11:50:49 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

この構文に関しては、検索すればたくさん使い方がまとめられているのでここでは扱わない。基本的に、下記のようなクエリを記述すれば、`big = 'AA'`と`updatedat`を更新できる。

```
mysql>
insert into modes_from (id, big, small, createdat, updatedat)
values (1, 'AA', 'a', current_timestamp, current_timestamp)
on duplicate key update
big = values(big),
updatedat = values(updatedat)
;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 11:50:49 | 2023-07-02 11:51:01 |
|  2 | B    | b     | 2023-07-02 11:50:49 | 2023-07-02 11:50:49 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.01 sec)
```

`createdat`,`updatedat`は、デフォルト設定をしているので記載しなくて問題はない。

```
mysql>
insert into modes_from (id, big, small)
values (1, 'AAA', 'a')
on duplicate key update
big = values(big),
updatedat = values(updatedat)
;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AAA  | a     | 2023-07-02 11:50:49 | 2023-07-02 11:51:19 |
|  2 | B    | b     | 2023-07-02 11:50:49 | 2023-07-02 11:50:49 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

ちなみに`small`も更新に関係ないので記載しなくて問題ない。

```
mysql>
insert into modes_from (id, big)
values (1, 'AAAA')
on duplicate key update
big = values(big),
updatedat = values(updatedat)
;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AAAA | a     | 2023-07-02 11:50:49 | 2023-07-02 11:51:33 |
|  2 | B    | b     | 2023-07-02 11:50:49 | 2023-07-02 11:50:49 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

ちなみに、レコードを識別できる `id` があれば更新はできる。`updatedat`をみれば`2023-07-02 11:51:33`から`2023-07-02 11:51:59`に変更されているのがわかる。

```
mysql>
insert into modes_from (id)
values (1)
on duplicate key update
updatedat = values(updatedat)
;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AAAA | a     | 2023-07-02 11:50:49 | 2023-07-02 11:51:59 |
|  2 | B    | b     | 2023-07-02 11:50:49 | 2023-07-02 11:50:49 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

ちなみにレコードが識別できる`id`がないので、新規レコードとして追加される。

```
mysql>
insert into modes_from (id)
values (null)
on duplicate key update
id = values(id)
;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AAAA | a     | 2023-07-02 11:50:49 | 2023-07-02 11:51:59 |
|  2 | B    | b     | 2023-07-02 11:50:49 | 2023-07-02 11:50:49 |
|  3 | NULL | NULL  | 2023-07-02 11:52:29 | 2023-07-02 11:52:29 |
+----+------+-------+---------------------+---------------------+
3 rows in set (0.00 sec)
```

このレコードの`null`を更新したければ、`update`でもよいが、このクエリでも更新できる。

```
mysql>
insert into modes_from (id, big, small)
values (3, 'C', 'c')
on duplicate key update
big = values(big),
small = values(small),
updatedat = values(updatedat)
;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AAAA | a     | 2023-07-02 11:50:49 | 2023-07-02 11:51:59 |
|  2 | B    | b     | 2023-07-02 11:50:49 | 2023-07-02 11:50:49 |
|  3 | C    | c     | 2023-07-02 11:52:29 | 2023-07-02 11:53:27 |
+----+------+-------+---------------------+---------------------+
3 rows in set (0.00 sec)
```

MySQL の ON DUPLICATE KEY UPDATE がどのように機能するかを書くにできたので、Embulk の MySQL アウトプットプラグインの転送モード`merge`を利用してみる。

## :pencil2: Embulk と merge

Embulk を実行する際のコンフィグファイルは下記の通り。

```
in:
  type: mysql
  host: YOUR_HOST
  user: YOUR_USER
  password: YOUR_PASSWORD
  database: mode_test
  table: modes_from
  select: 'id, big, small, createdat, updatedat'
  options:
    serverTimezone: Asia/Tokyo
out:
  type: mysql
  host: YOUR_HOST
  user: YOUR_USER
  password: YOUR_PASSWORD
  database: mode_test
  table: modes_to
  mode: merge
  create_table_constraint: 'primary key(id)'
  column_options:
    id: { type: 'int auto_increment' }
```

Embulk を動かす前に MySQL の掃除をしておく。以降は紛らわしいので`[転送元From]`、`[転送元To]`のラベルをつけている。

```
mysql>
drop table modes_from;

mysql>
create table modes_from (
  id int auto_increment,
  big varchar(5),
  small varchar(5),
  createdat timestamp default current_timestamp,
  updatedat timestamp default current_timestamp,
  primary key (id)
);

mysql>
insert into modes_from (big, small) values
  ('A', 'a'),
  ('B', 'b')
;

[転送元From]
mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | A    | a     | 2023-07-02 12:24:57 | 2023-07-02 12:24:57 |
|  2 | B    | b     | 2023-07-02 12:24:57 | 2023-07-02 12:24:57 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

Embulk を実行すると、まずは転送先のテーブルが作成され、転送元のテーブルの値が転送されていることがわかる。

```
$ embulk run embuBlk_mysql_mysql.yml

[転送先To]
mysql> select * from modes_to;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | A    | a     | 2023-07-02 12:24:57 | 2023-07-02 12:24:57 |
|  2 | B    | b     | 2023-07-02 12:24:57 | 2023-07-02 12:24:57 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)

mysql> desc modes_to;
+-----------+-----------+------+-----+---------+----------------+
| Field     | Type      | Null | Key | Default | Extra          |
+-----------+-----------+------+-----+---------+----------------+
| id        | int       | NO   | PRI | NULL    | auto_increment |
| big       | text      | YES  |     | NULL    |                |
| small     | text      | YES  |     | NULL    |                |
| createdat | timestamp | YES  |     | NULL    |                |
| updatedat | timestamp | YES  |     | NULL    |                |
+-----------+-----------+------+-----+---------+----------------+
5 rows in set (0.00 sec)
```

まずは転送元のテーブルのレコードを更新する。`id=1`の`big=AA`、`updatedat`、`id=2`の`small=bb`、`updatedat`が更新されていることがわかる。

```
[転送元From]
mysql>
insert into modes_from (id, big, small)
values (1, 'AA', 'a'), (2, 'B', 'bb')
on duplicate key update
big = values(big),
small = values(small),
updatedat = values(updatedat)
;

[転送元From]
mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

これを転送すると、転送元と同じテーブルが同じテーブルが作成される。

```
$ embulk run embulk_mysql_mysql.yml

[転送先To]
mysql> select * from modes_to;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

次はレコードが追加されたとき、どのように機能するかを確認する。転送元でレコードを追加する。

```
[転送元From]
mysql>
insert into modes_from (big, small) values('C', 'c');

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
+----+------+-------+---------------------+---------------------+
3 rows in set (0.01 sec)
```

MySQL の ON DUPLICATE KEY UPDATE は、対応するレコードが存在しない場合、レコードを追加するので、1 レコード追加される。

```
$ embulk run embulk_mysql_mysql.yml

[転送先To]
mysql> select * from modes_to;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
+----+------+-------+---------------------+---------------------+
3 rows in set (0.00 sec)
```

転送元のテーブルに`null`があっても、

```
[転送元From]
mysql>
insert into modes_from (big, small) values('D', null);

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | NULL  | 2023-07-02 12:42:53 | 2023-07-02 12:42:53 |
+----+------+-------+---------------------+---------------------+
4 rows in set (0.01 sec)
```

当たり前ではあるが`null`のまま転送される。

```
$ embulk run embulk_mysql_mysql.yml

[転送先To]
mysql> select * from modes_to;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | NULL  | 2023-07-02 12:42:53 | 2023-07-02 12:42:53 |
+----+------+-------+---------------------+---------------------+
4 rows in set (0.00 sec)
```

そのため、転送元が修正されれば、

```
[転送元From]
mysql>
update modes_from set small = 'ddd', updatedat = current_timestamp where id = 4;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | ddd   | 2023-07-02 12:42:53 | 2023-07-02 12:46:35 |
+----+------+-------+---------------------+---------------------+
4 rows in set (0.00 sec)
```

転送先も値が更新される。

```
$ embulk run embulk_mysql_mysql.yml

[転送先To]
mysql> select * from modes_to;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | ddd   | 2023-07-02 12:42:53 | 2023-07-02 12:46:35 |
+----+------+-------+---------------------+---------------------+
4 rows in set (0.00 sec)
```

当たり前といえば当たり前ではあるが、レコードが削除されたときも、そのテーブルを転送すれば転送先でもレコードが削除されるとうっかりと勘違いしてしまう。例えば、`id=1`を削除して転送する。

```
[転送元From]
mysql> delete from modes_from where id = 1;

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | ddd   | 2023-07-02 12:42:53 | 2023-07-02 12:46:35 |
+----+------+-------+---------------------+---------------------+
3 rows in set (0.00 sec)
```

転送先のテーブルでは`id=1`は削除されない。MySQL の ON DUPLICATE KEY UPDATE ではレコードは、追加、更新はするが、削除するわけではない。

```
$ embulk run embulk_mysql_mysql.yml

[転送先To]
mysql> select * from modes_to;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | ddd   | 2023-07-02 12:42:53 | 2023-07-02 12:46:35 |
+----+------+-------+---------------------+---------------------+
4 rows in set (0.00 sec)
```

これに気づかず、新しくレコードが追加されたり、更新されたりすると、

```
[転送元From]
mysql> insert into modes_from (big, small) values('E', 'e');

mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | ddd   | 2023-07-02 12:42:53 | 2023-07-02 12:46:35 |
|  5 | E    | e     | 2023-07-02 12:50:34 | 2023-07-02 12:50:34 |
+----+------+-------+---------------------+---------------------+
4 rows in set (0.00 sec)
```

削除されないまま、転送先のテーブルの情報がマージされて更新される。

```
$ embulk run embulk_mysql_mysql.yml

[転送先To]
mysql> select * from modes_to;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | ddd   | 2023-07-02 12:42:53 | 2023-07-02 12:46:35 |
|  5 | E    | e     | 2023-07-02 12:50:34 | 2023-07-02 12:50:34 |
+----+------+-------+---------------------+---------------------+
5 rows in set (0.00 sec)
```

レコードを削除したいのであれば、転送モードを`replace`にする必要がある。

```
in:
  type: mysql
  host: YOUR_HOST
  user: YOUR_USER
  password: YOUR_PASSWORD
  database: mode_test
  table: modes_from
  select: 'id, big, small, createdat, updatedat'
  options:
    serverTimezone: Asia/Tokyo
out:
  type: mysql
  host: YOUR_HOST
  user: YOUR_USER
  password: YOUR_PASSWORD
  database: mode_test
  table: modes_to
  mode: replace
  create_table_constraint: 'primary key(id)'
  column_options:
    id: { type: 'int auto_increment' }
-------------------------------------------------------------------------------------
$ embulk run embulk_mysql_mysql.yml

[転送先To]
mysql> select * from modes_to;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  2 | B    | bb    | 2023-07-02 12:24:57 | 2023-07-02 12:33:58 |
|  3 | C    | c     | 2023-07-02 12:40:59 | 2023-07-02 12:40:59 |
|  4 | D    | ddd   | 2023-07-02 12:42:53 | 2023-07-02 12:46:35 |
|  5 | E    | e     | 2023-07-02 12:50:34 | 2023-07-02 12:50:34 |
+----+------+-------+---------------------+---------------------+
4 rows in set (0.01 sec)
```

## :pencil2: テーブル設定の失敗

オプションを転送先のテーブルにつけず、テーブル作成を Embulk 側にまかせると、一意のキー制約に違反した場合には既存の行を更新する機能が機能せず、単純に追加されてしまった。予め自分で転送先のテーブルを作成するか、オプションを設定する必要がありそう(たぶんこれが原因だと思われる…)。

```
[失敗例]
in:
  type: mysql
  host: YOUR_HOST
  user: YOUR_USER
  password: YOUR_PASSWORD
  database: mode_test
  table: modes_from
  select: 'id, big, small, createdat, updatedat'
  options:
    serverTimezone: Asia/Tokyo
out:
  type: mysql
  host: YOUR_HOST
  user: YOUR_USER
  password: YOUR_PASSWORD
  database: mode_test
  table: modes_to
  mode: merge
-------------------------------------------------------------------------------------

[転送先From]
mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | A    | a     | 2023-07-02 13:13:30 | 2023-07-02 13:13:30 |
|  2 | B    | b     | 2023-07-02 13:13:30 | 2023-07-02 13:13:30 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)

[転送先To]
$ embulk run embulk_mysql_mysql.yml
mysql> desc modes_to;
+-----------+-----------+------+-----+---------+-------+
| Field     | Type      | Null | Key | Default | Extra |
+-----------+-----------+------+-----+---------+-------+
| id        | bigint    | YES  |     | NULL    |       |
| big       | text      | YES  |     | NULL    |       |
| small     | text      | YES  |     | NULL    |       |
| createdat | timestamp | YES  |     | NULL    |       |
| updatedat | timestamp | YES  |     | NULL    |       |
+-----------+-----------+------+-----+---------+-------+
5 rows in set (0.01 sec)

[転送先To]
mysql> select * from modes_to;
+------+------+-------+---------------------+---------------------+
| id   | big  | small | createdat           | updatedat           |
+------+------+-------+---------------------+---------------------+
|    1 | A    | a     | 2023-07-02 13:13:30 | 2023-07-02 13:13:30 |
|    2 | B    | b     | 2023-07-02 13:13:30 | 2023-07-02 13:13:30 |
+------+------+-------+---------------------+---------------------+
2 rows in set (0.01 sec)

[転送先From]
mysql>
insert into modes_from (id, big, small)
values (1, 'AA', 'a'), (2, 'B', 'bb')
on duplicate key update
big = values(big),
small = values(small),
updatedat = values(updatedat)
;
mysql> select * from modes_from;
+----+------+-------+---------------------+---------------------+
| id | big  | small | createdat           | updatedat           |
+----+------+-------+---------------------+---------------------+
|  1 | AA   | a     | 2023-07-02 13:13:30 | 2023-07-02 13:14:50 |
|  2 | B    | bb    | 2023-07-02 13:13:30 | 2023-07-02 13:14:50 |
+----+------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)

[転送先To]
$ embulk run embulk_mysql_mysql.yml
mysql> select * from modes_to;
+------+------+-------+---------------------+---------------------+
| id   | big  | small | createdat           | updatedat           |
+------+------+-------+---------------------+---------------------+
|    1 | A    | a     | 2023-07-02 13:13:30 | 2023-07-02 13:13:30 |
|    2 | B    | b     | 2023-07-02 13:13:30 | 2023-07-02 13:13:30 |
|    1 | AA   | a     | 2023-07-02 13:13:30 | 2023-07-02 13:14:50 |
|    2 | B    | bb    | 2023-07-02 13:13:30 | 2023-07-02 13:14:50 |
+------+------+-------+---------------------+---------------------+
4 rows in set (0.01 sec)
```

## :closed_book: Reference

- [MySQL input plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)
- [MySQL output plugin for Embulk](https://github.com/embulk/embulk-input-jdbc/blob/master/embulk-input-mysql/README.md)
