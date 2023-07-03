## :memo: Overview

今更ながら `insert`、`update`、`delete` などの使い方をまとめておく。Embulk でデータ転送を行う際に出力先テーブルの更新方法をいくつか選べるが、分析作業では DML(Data Manipulation Language)の `select` 以外は使う機会があまりなかったので、今更ながらまとめておく。

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`update`, `insert`, `delete`

## :pencil2: テーブルの道具立て

まずはテーブルを用意する。

```
mysql> create database dml;
Query OK, 1 row affected (0.01 sec)

mysql> use dml;
Database changed

mysql> create table f (
  id int auto_increment,
  name varchar(255),
  quality float,
  createdat timestamp default current_timestamp,
  updatedat timestamp default current_timestamp,
  primary key (id)
);
Query OK, 0 rows affected (0.02 sec)

mysql> desc f;
+-----------+--------------+------+-----+-------------------+-------------------+
| Field     | Type         | Null | Key | Default           | Extra             |
+-----------+--------------+------+-----+-------------------+-------------------+
| id        | int          | NO   | PRI | NULL              | auto_increment    |
| name      | varchar(255) | YES  |     | NULL              |                   |
| quality   | float        | YES  |     | NULL              |                   |
| createdat | timestamp    | YES  |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
| updatedat | timestamp    | YES  |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
+-----------+--------------+------+-----+-------------------+-------------------+
5 rows in set (0.01 sec)
```

## :pencil2: insert コマンド

まずはデータをテーブルに追加する `insert` から始める。

```
mysql> insert into f (name, quality) values ('apple', 1.0);
Query OK, 1 row affected (0.01 sec)

mysql> select * from f;
+----+-------+---------+---------------------+---------------------+
| id | name  | quality | createdat           | updatedat           |
+----+-------+---------+---------------------+---------------------+
|  1 | apple |       1 | 2023-07-01 10:30:44 | 2023-07-01 10:30:44 |
+----+-------+---------+---------------------+---------------------+
1 row in set (0.00 sec)
```

`insert` はカラム名を省略した記法でも問題なく、データも複数行追加できるが、その場合、列名は省略することができない。

```
mysql> insert into f values ('apple', 1.1), ('apple', 1.2);
ERROR 1136 (21S01): Column count doesn't match value count at row 1
```

そのため、`id`の番号も指定しなければならず、非常に厄介ではある。

```
mysql> insert into f values
(2, 'apple', 1.1, current_timestamp, current_timestamp),
(3, 'apple', 1.2, current_timestamp, current_timestamp)
;

mysql> select * from f;
+----+-------+---------+---------------------+---------------------+
| id | name  | quality | createdat           | updatedat           |
+----+-------+---------+---------------------+---------------------+
|  1 | apple |       1 | 2023-07-01 10:30:44 | 2023-07-01 10:30:44 |
|  2 | apple |     1.1 | 2023-07-01 10:34:22 | 2023-07-01 10:34:22 |
|  3 | apple |     1.2 | 2023-07-01 10:34:22 | 2023-07-01 10:34:22 |
+----+-------+---------+---------------------+---------------------+
3 rows in set (0.00 sec)
```

これを回避する方法として、`null`を使えば番号を直打ちする必要がない。`id`は`auto_increment`しているのでこれでよいが、日付関係のカラムは`null`になるので注意。

```
mysql> insert into f values
(null, 'apple', 1.3, current_timestamp, current_timestamp),
(null, 'apple', 1.4, null, null)
;

mysql> select * from f;
+----+-------+---------+---------------------+---------------------+
| id | name  | quality | createdat           | updatedat           |
+----+-------+---------+---------------------+---------------------+
|  1 | apple |       1 | 2023-07-01 10:30:44 | 2023-07-01 10:30:44 |
|  2 | apple |     1.1 | 2023-07-01 10:34:22 | 2023-07-01 10:34:22 |
|  3 | apple |     1.2 | 2023-07-01 10:34:22 | 2023-07-01 10:34:22 |
|  4 | apple |     1.3 | 2023-07-01 10:36:17 | 2023-07-01 10:36:17 |
|  5 | apple |     1.4 | 2023-07-01 10:36:17 | 2023-07-01 10:36:17 |
+----+-------+---------+---------------------+---------------------+
5 rows in set (0.00 sec)
```

他のテーブルを`insert`することもできる。そのために違うテーブルを用意しておく。

```
mysql> create table f2 (
  id int auto_increment,
  name varchar(255),
  quality float,
  createdat timestamp default current_timestamp,
  updatedat timestamp default current_timestamp,
  primary key (id)
);

mysql> insert into f2 values
(null, 'orange', 1.0, current_timestamp, current_timestamp),
(null, 'orange', 1.1, current_timestamp, current_timestamp),
(null, 'orange', 1.2, current_timestamp, current_timestamp);

mysql> select * from f2;
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | orange |       1 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  2 | orange |     1.1 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  3 | orange |     1.2 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
+----+--------+---------+---------------------+---------------------+
3 rows in set (0.00 sec)
```

`id`は`auto_increment`しているので、`id`を追加するとエラーになる。

```
mysql>
insert into f (id, name, quality, createdat, updatedat)
select id, name, quality, createdat, updatedat from f2;
ERROR 1062 (23000): Duplicate entry '1' for key 'f.PRIMARY'
```

これを回避するためには、`id`を指定しなければよく、`auto_increment`の番号の調整もいらない。例えば、`insert into f select (select max(id) from f) + id from f2`のような形で修正する必要がない。

```
mysql>
insert into f (name, quality, createdat, updatedat)
select name, quality, createdat, updatedat from f2;
Query OK, 3 rows affected (0.00 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |       1 | 2023-07-01 10:39:25 | 2023-07-01 10:39:25 |
|  2 | apple  |     1.1 | 2023-07-01 10:39:31 | 2023-07-01 10:39:31 |
|  3 | apple  |     1.2 | 2023-07-01 10:39:31 | 2023-07-01 10:39:31 |
|  4 | apple  |     1.3 | 2023-07-01 10:40:42 | 2023-07-01 10:40:42 |
|  5 | apple  |     1.4 | NULL                | NULL                |
|  6 | orange |       1 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  7 | orange |     1.1 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  8 | orange |     1.2 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
+----+--------+---------+---------------------+---------------------+
8 rows in set (0.00 sec)
```

## :pencil2: replace コマンド

`insert` は一旦ここまでにして、ここからは他のコマンドをみていく。存在する場合は元にデータを置き換え、存在しない場合は追加したい、こういうケースはよくありそうで、`insert` と `update` を別々に行えば実現できる。MySQL には `replace` が実装されているので、データが既存の場合は変更して、存在しない場合は追加することが `replace` コマンド 1 つで実現できる。他のデータベースだと `merge` コマンドとして機能するものと同じ。

まずは、9 番目に存在しないレコードをインサートする。

```
mysql> replace into f values (null, 'orange', 1.3, current_timestamp, current_timestamp);
Query OK, 1 row affected (0.00 sec)

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |       1 | 2023-07-01 10:39:25 | 2023-07-01 10:39:25 |
|  2 | apple  |     1.1 | 2023-07-01 10:39:31 | 2023-07-01 10:39:31 |
|  3 | apple  |     1.2 | 2023-07-01 10:39:31 | 2023-07-01 10:39:31 |
|  4 | apple  |     1.3 | 2023-07-01 10:40:42 | 2023-07-01 10:40:42 |
|  5 | apple  |     1.4 | NULL                | NULL                |
|  6 | orange |       1 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  7 | orange |     1.1 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  8 | orange |     1.2 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  9 | orange |     1.3 | 2023-07-01 11:14:52 | 2023-07-01 11:14:52 |
+----+--------+---------+---------------------+---------------------+
9 rows in set (0.00 sec)
```

1 行目の存在するレコードを更新する。`2 rows affected`と表示されるのは、内部ではレコードを削除して追加しているため。

```
mysql> replace into f values (1, 'apple', 0.9, '2023-07-01 10:39:25', current_timestamp);
Query OK, 2 rows affected (0.00 sec)

mysql> select * from f;
# 変更後
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |     0.9 | 2023-07-01 10:39:25 | 2023-07-01 11:19:12 |

# 変更前
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |       1 | 2023-07-01 10:39:25 | 2023-07-01 10:39:25 |
```

## :pencil2: update コマンド

次は `update` コマンドでデータを変更してみる。MySQL では `replace` `があるので、replace` でもよいが一般的には `update` のほうが基本なので、`update` でデータを更新しておく。先程変更した 1 行目のデータの`quality`をもとに戻す。`where`をつけ忘れると、全レコードが対象になるので注意。

```
mysql> update f set quality = 1 where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |       1 | 2023-07-01 10:39:25 | 2023-07-01 11:19:12 |
|  2 | apple  |     1.1 | 2023-07-01 10:39:31 | 2023-07-01 10:39:31 |
|  3 | apple  |     1.2 | 2023-07-01 10:39:31 | 2023-07-01 10:39:31 |
|  4 | apple  |     1.3 | 2023-07-01 10:40:42 | 2023-07-01 10:40:42 |
|  5 | apple  |     1.4 | NULL                | NULL                |
|  6 | orange |       1 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  7 | orange |     1.1 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  8 | orange |     1.2 | 2023-07-01 10:44:37 | 2023-07-01 10:44:37 |
|  9 | orange |     1.3 | 2023-07-01 11:14:52 | 2023-07-01 11:14:52 |
+----+--------+---------+---------------------+---------------------+
9 rows in set (0.00 sec)
```

複数列の値を更新するときは、`set cola = 'a', colb = b`と記述する。`and`ではないので注意。

```
mysql> update f set name = 'banana', updatedat = current_timestamp where name = 'orange';
Query OK, 4 rows affected (0.01 sec)
Rows matched: 4  Changed: 4  Warnings: 0

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |       1 | 2023-07-01 10:39:25 | 2023-07-01 11:19:12 |
|  2 | apple  |     1.1 | 2023-07-01 10:39:31 | 2023-07-01 10:39:31 |
|  3 | apple  |     1.2 | 2023-07-01 10:39:31 | 2023-07-01 10:39:31 |
|  4 | apple  |     1.3 | 2023-07-01 10:40:42 | 2023-07-01 10:40:42 |
|  5 | apple  |     1.4 | NULL                | NULL                |
|  6 | banana |       1 | 2023-07-01 10:44:37 | 2023-07-01 11:29:40 |
|  7 | banana |     1.1 | 2023-07-01 10:44:37 | 2023-07-01 11:29:40 |
|  8 | banana |     1.2 | 2023-07-01 10:44:37 | 2023-07-01 11:29:40 |
|  9 | banana |     1.3 | 2023-07-01 11:14:52 | 2023-07-01 11:29:40 |
+----+--------+---------+---------------------+---------------------+
9 rows in set (0.00 sec)
```

`set`は計算式も使用可能なので、`quality`の数字を下げてみる。

```
mysql> update f set quality = quality - 1, updatedat = current_timestamp;
Query OK, 9 rows affected (0.01 sec)
Rows matched: 9  Changed: 9  Warnings: 0

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |       0 | 2023-07-01 10:39:25 | 2023-07-01 11:41:24 |
|  2 | apple  |     0.1 | 2023-07-01 10:39:31 | 2023-07-01 11:41:24 |
|  3 | apple  |     0.2 | 2023-07-01 10:39:31 | 2023-07-01 11:41:24 |
|  4 | apple  |     0.3 | 2023-07-01 10:40:42 | 2023-07-01 11:41:24 |
|  5 | apple  |     0.4 | NULL                | 2023-07-01 11:41:24 |
|  6 | banana |       0 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  7 | banana |     0.1 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  8 | banana |     0.2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  9 | banana |     0.3 | 2023-07-01 11:14:52 | 2023-07-01 11:41:24 |
+----+--------+---------+---------------------+---------------------+
9 rows in set (0.00 sec)
```

`update`は`case`文で条件分けして値を更新することもできる。

```
mysql> update f set quality = case when name = 'apple' then quality + 1 else quality + 2 end;
Query OK, 9 rows affected (0.00 sec)
Rows matched: 9  Changed: 9  Warnings: 0

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |       1 | 2023-07-01 10:39:25 | 2023-07-01 11:41:24 |
|  2 | apple  |     1.1 | 2023-07-01 10:39:31 | 2023-07-01 11:41:24 |
|  3 | apple  |     1.2 | 2023-07-01 10:39:31 | 2023-07-01 11:41:24 |
|  4 | apple  |     1.3 | 2023-07-01 10:40:42 | 2023-07-01 11:41:24 |
|  5 | apple  |     1.4 | NULL                | 2023-07-01 11:41:24 |
|  6 | banana |       2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  7 | banana |     2.1 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  8 | banana |     2.2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  9 | banana |     2.3 | 2023-07-01 11:14:52 | 2023-07-01 11:41:24 |
+----+--------+---------+---------------------+---------------------+
9 rows in set (0.00 sec)
```

少し高度な更新方法として別のテーブルの存在確認をしてから更新するかどうかを実行する方法もある。例えば、`fm`テーブルには`apple`は存在するが、`banana`は存在していない。`in`を使う方法もあるが、`not in`の場合は注意が必要。

```
mysql> create table fm (
  id int auto_increment,
  name2 varchar(255),
  primary key (id)
);
Query OK, 0 rows affected (0.02 sec)

mysql> insert into fm (name2) values ('apple');
Query OK, 1 row affected (0.01 sec)

mysql> select * from fm;
+----+-------+
| id | name2 |
+----+-------+
|  1 | apple |
+----+-------+
1 row in set (0.00 sec)
```

このテーブルで`apple, banana`の存在確認をしてから更新する。`exists`は`select`のリスト結果で更新を決めるわけではなく、`where`で決まるので、`select null`は`1`でもなんでもよい。

```
mysql>
update f set quality = quality * 10, updatedat = current_timestamp
where exists (select null from fm where f.name = fm.name2);

Query OK, 5 rows affected (0.01 sec)
Rows matched: 5  Changed: 5  Warnings: 0

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+
| id | name   | quality | createdat           | updatedat           |
+----+--------+---------+---------------------+---------------------+
|  1 | apple  |      10 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 |
|  2 | apple  |      11 | 2023-07-01 10:39:31 | 2023-07-01 12:02:11 |
|  3 | apple  |      12 | 2023-07-01 10:39:31 | 2023-07-01 12:02:11 |
|  4 | apple  |      13 | 2023-07-01 10:40:42 | 2023-07-01 12:02:11 |
|  5 | apple  |      14 | NULL                | 2023-07-01 12:02:11 |
|  6 | banana |       2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  7 | banana |     2.1 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  8 | banana |     2.2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 |
|  9 | banana |     2.3 | 2023-07-01 11:14:52 | 2023-07-01 11:41:24 |
+----+--------+---------+---------------------+---------------------+
9 rows in set (0.00 sec)
```

新しいカラムを作成して、そのカラムの値を更新したい場合は`alter`でカラムを作成してから、更新する。

```
mysql> alter table f add column color varchar(255);
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> update f set color = case when name = 'apple' then 'red' else 'yellow' end;
Query OK, 9 rows affected (0.00 sec)
Rows matched: 9  Changed: 9  Warnings: 0

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+--------+
| id | name   | quality | createdat           | updatedat           | color  |
+----+--------+---------+---------------------+---------------------+--------+
|  1 | apple  |      10 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
|  2 | apple  |      11 | 2023-07-01 10:39:31 | 2023-07-01 12:02:11 | red    |
|  3 | apple  |      12 | 2023-07-01 10:39:31 | 2023-07-01 12:02:11 | red    |
|  4 | apple  |      13 | 2023-07-01 10:40:42 | 2023-07-01 12:02:11 | red    |
|  5 | apple  |      14 | NULL                | 2023-07-01 12:02:11 | red    |
|  6 | banana |       2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  7 | banana |     2.1 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  8 | banana |     2.2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  9 | banana |     2.3 | 2023-07-01 11:14:52 | 2023-07-01 11:41:24 | yellow |
+----+--------+---------+---------------------+---------------------+--------+
9 rows in set (0.00 sec)
```

## :pencil2: delete コマンド

最後は`delete`について見ていく。基本的には`update`と同じで、削除する行を特定して、削除することになる。

```
mysql> delete from f where quality = 14;
Query OK, 1 row affected (0.00 sec)

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+--------+
| id | name   | quality | createdat           | updatedat           | color  |
+----+--------+---------+---------------------+---------------------+--------+
|  1 | apple  |      10 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
|  2 | apple  |      11 | 2023-07-01 10:39:31 | 2023-07-01 12:02:11 | red    |
|  3 | apple  |      12 | 2023-07-01 10:39:31 | 2023-07-01 12:02:11 | red    |
|  4 | apple  |      13 | 2023-07-01 10:40:42 | 2023-07-01 12:02:11 | red    |
|  6 | banana |       2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  7 | banana |     2.1 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  8 | banana |     2.2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  9 | banana |     2.3 | 2023-07-01 11:14:52 | 2023-07-01 11:41:24 | yellow |
+----+--------+---------+---------------------+---------------------+--------+
8 rows in set (0.00 sec)
```

`where`はもちろん不等号を使って検索することもできる。

```
mysql> delete from f where quality > 10;
Query OK, 3 rows affected (0.00 sec)

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+--------+
| id | name   | quality | createdat           | updatedat           | color  |
+----+--------+---------+---------------------+---------------------+--------+
|  1 | apple  |      10 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
|  6 | banana |       2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  7 | banana |     2.1 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  8 | banana |     2.2 | 2023-07-01 10:44:37 | 2023-07-01 11:41:24 | yellow |
|  9 | banana |     2.3 | 2023-07-01 11:14:52 | 2023-07-01 11:41:24 | yellow |
+----+--------+---------+---------------------+---------------------+--------+
5 rows in set (0.00 sec)
```

他のテーブルを利用して参照整合性が満たされないレコードを削除する。例えば、何らかのテーブルをもとに、そのテーブルに存在しないレコードがあれば削除するという感じ。先程の`apple`しか入ってないテーブル`fm`を使うと、`banana`をすべて削除できる。

```
mysql> select * from fm;
+----+-------+
| id | name2 |
+----+-------+
|  1 | apple |
+----+-------+
1 row in set (0.00 sec)
```

実行した結果は下記の通り、`banana`は存在していないので削除される。

```
mysql> delete from f where not exists (select * from fm where f.name = fm.name2);
Query OK, 4 rows affected (0.00 sec)

mysql> select * from f;
+----+-------+---------+---------------------+---------------------+-------+
| id | name  | quality | createdat           | updatedat           | color |
+----+-------+---------+---------------------+---------------------+-------+
|  1 | apple |      10 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red   |
+----+-------+---------+---------------------+---------------------+-------+
1 row in set (0.00 sec)
```

仕事でデータベースのデータを扱っていると下記のように重複したレコードに遭遇することは割ある。本来は重複することはないはずのものだが・・・。

```
insert into f (id, name, quality, createdat, updatedat, color) values
(null, 'apple', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'red'),
(null, 'apple', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'red'),
(null, 'mikan', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'orange'),
(null, 'mikan', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'orange'),
(null, 'mikan', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'orange'),
(null, 'banana', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'yellow'),
(null, 'grape', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'violet'),
(null, 'grape', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'violet'),
(null, 'grape', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'violet'),
(null, 'grape', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'violet');
Query OK, 10 rows affected (0.00 sec)
Records: 10  Duplicates: 0  Warnings: 0

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+--------+
| id | name   | quality | createdat           | updatedat           | color  |
+----+--------+---------+---------------------+---------------------+--------+
|  1 | apple  |      10 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
| 10 | apple  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
| 11 | apple  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
| 12 | mikan  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | orange |
| 13 | mikan  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | orange |
| 14 | mikan  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | orange |
| 15 | banana |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | yellow |
| 16 | grape  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
| 17 | grape  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
| 18 | grape  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
| 19 | grape  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
+----+--------+---------+---------------------+---------------------+--------+
11 rows in set (0.00 sec)
```

例えば、必要なのは重複しているブロックの最小の`id`でよければ下記の SQL を`delete`に組み合わせる。この SQL で不要なレコードの`id`を取り出せる。

```
mysql> select * from f where id not in (select min(id) as delid from f group by name);
+----+-------+---------+---------------------+---------------------+--------+
| id | name  | quality | createdat           | updatedat           | color  |
+----+-------+---------+---------------------+---------------------+--------+
| 10 | apple |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
| 11 | apple |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
| 13 | mikan |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | orange |
| 14 | mikan |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | orange |
| 17 | grape |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
| 18 | grape |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
| 19 | grape |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
+----+-------+---------+---------------------+---------------------+--------+
7 rows in set (0.00 sec)
```

実行すると MySQL だとエラーになる。これは`delete`では同じテーブルを 2 回続けて参照できないため。

```
mysql> delete from f where id not in (select min(id) from f group by name);
ERROR 1093 (HY000): You can't specify target table 'f' for update in FROM clause
```

なので、少し SQL を書き換える必要がある。

```
delete from f
where id not in
  (select min(id)
   from (select * from f) as tmp
   group by name
   );
Query OK, 7 rows affected (0.01 sec)

mysql> select * from f;
+----+--------+---------+---------------------+---------------------+--------+
| id | name   | quality | createdat           | updatedat           | color  |
+----+--------+---------+---------------------+---------------------+--------+
|  1 | apple  |      10 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
| 12 | mikan  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | orange |
| 15 | banana |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | yellow |
| 16 | grape  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
+----+--------+---------+---------------------+---------------------+--------+
4 rows in set (0.00 sec)
```

## :pencil2: おまけ

先程のテーブルをみるとわかるが、`auto_increment`が連番になっていない。連番に戻す一番簡単な方法はテーブルを作り直せばよいが、そう簡単にはテーブルは削除できない。そのため、`auto_increment`の番号を管理している`information_schema.tables`の値を調整する。今は 20 番から採番される模様。

```
select auto_increment from information_schema.tables where table_schema = 'dml' and table_name = 'f';
mysql> select auto_increment from information_schema.tables where table_schema = 'dml' and table_name = 'f';
+----------------+
| AUTO_INCREMENT |
+----------------+
|             20 |
+----------------+
1 row in set (0.01 sec)
```

ということで、更新しようとするとエラーが返される。これは、`information_schema`の情報はこのようなクエリでは基本的に更新できないようになっているため。

```
mysql> update information_schema.tables set auto_increment = 100 where table_schema = 'dml' and table_name = 'f';
ERROR 1044 (42000): Access denied for user 'root'@'localhost' to database 'information_schema'
```

そのため、`alter`を利用して更新してからインサートすれば、設定した値から採番できる。

```
mysql> alter table dml.f auto_increment = 100;
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> insert into f (id, name, quality, createdat, updatedat, color) values (null, 'melon', 1.0, '2023-07-01 10:39:25', '2023-07-01 12:02:11', 'green');
Query OK, 1 row affected (0.00 sec)

mysql> select * from f;
+-----+--------+---------+---------------------+---------------------+--------+
| id  | name   | quality | createdat           | updatedat           | color  |
+-----+--------+---------+---------------------+---------------------+--------+
|   1 | apple  |      10 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | red    |
|  12 | mikan  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | orange |
|  15 | banana |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | yellow |
|  16 | grape  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | violet |
| 100 | melon  |       1 | 2023-07-01 10:39:25 | 2023-07-01 12:02:11 | green  |
+-----+--------+---------+---------------------+---------------------+--------+
5 rows in set (0.00 sec)
```

## :closed_book: Reference

- [MySQL](https://dev.mysql.com/doc/refman/8.0/ja/sql-data-manipulation-statements.html)
