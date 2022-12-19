## :memo: Overview

不等号を用いた`join`を利用することで、同一テーブルの比較が実行できる。1 つのテーブルがあった場合に、部分的に一致しないレコードを探す方法についてまとめる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`inequality join`

## :pencil2: Example

とある多種多様な`tanaka`さんマスタがあったとする。大家族の tanaka さん家は`ane`、`ani`は独り立ちしているので`addr`が親たちとは異なり、訳ありの tanaka さんは、分け合って別の`addr`に住んでいる。

```sql
create table tanaka(tanakaid integer, name varchar(20), addr varchar(10));
insert into
	tanaka(tanakaid, name, addr)
values
	-- 大家族のtanaka
	('1','tanaka titi','XXXXX'),
	('1','tanaka haha','XXXXX'),
	('1','tanaka ane','YYYYY'),
	('1','tanaka ani','ZZZZZ'),
	('1','tanaka imouto1','XXXXX'),
	('1','tanaka ototo1','XXXXX'),
	('1','tanaka imouto2','XXXXX'),
	('1','tanaka ototo2','XXXXX'),

	-- 訳ありのtanaka
	('2','tanaka wake','aaaaa'),
	('2','tanaka ari','AAAAA'),

	-- 新婚のtanaka
	('3','tanaka dana','BBBBB'),
	('3','tanaka yome','BBBBB'),

	-- 独身のtanaka
	('4','tanaka hitori','CCCCC')
;
```

このようデータの中から、同じ家族なんだけれど、`addr`が異なっているペアを見つけたい。テーブルのレコード間比較がしたいので、ループできれば解決できそうだけれど、SQL にはループがないので、自己結合して、直積を作って比較する方法が手っ取り早い。テーブルのサイズが大きすぎると、正直`cross join`は大変なことになるので、注意が必要。

同じ`tanakaid`で自己結合すれば、その家族内で直積が作成できる。

```sql
select
	t1.tanakaid as t1_tanakaid,
	t1.name as t1_name,
	t1.addr as t1_addr,
	t2.tanakaid as t2_tanakaid,
	t2.name as t2_name,
	t2.addr as t2_addr
from
	tanaka as t1
cross join
	tanaka as t2
where
	t1.tanakaid = t2.tanakaid
;


 t1_tanakaid |    t1_name     | t1_addr | t2_tanakaid |    t2_name     | t2_addr
-------------+----------------+---------+-------------+----------------+---------
           1 | tanaka titi    | XXXXX   |           1 | tanaka ototo2  | XXXXX
           1 | tanaka titi    | XXXXX   |           1 | tanaka imouto2 | XXXXX
           1 | tanaka titi    | XXXXX   |           1 | tanaka ototo1  | XXXXX
           1 | tanaka titi    | XXXXX   |           1 | tanaka imouto1 | XXXXX
           1 | tanaka titi    | XXXXX   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka titi    | XXXXX   |           1 | tanaka ane     | YYYYY
           1 | tanaka titi    | XXXXX   |           1 | tanaka haha    | XXXXX
           1 | tanaka titi    | XXXXX   |           1 | tanaka titi    | XXXXX
略
```

この結果をみると、さらに条件として、`t1_addr != t2_addr`を追加すれば、同じ家族なんだけれど、違う`addr`に住んでいるという条件でデータを抽出できそうである。実際に実行してみると、間違いではないが、大家族 tanaka さんのレコードが大量に生成される。`tanaka ane`と`tanaka ani`は他の家族全員と異なる住所に住んでいるためである。

```sql
select
	t1.tanakaid as t1_tanakaid,
	t1.name as t1_name,
	t1.addr as t1_addr,
	t2.tanakaid as t2_tanakaid,
	t2.name as t2_name,
	t2.addr as t2_addr
from
	tanaka as t1
cross join
	tanaka as t2
where
	t1.tanakaid = t2.tanakaid and
	t1.addr != t2.addr
;
 t1_tanakaid |    t1_name     | t1_addr | t2_tanakaid |    t2_name     | t2_addr
-------------+----------------+---------+-------------+----------------+---------
           1 | tanaka titi    | XXXXX   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka titi    | XXXXX   |           1 | tanaka ane     | YYYYY
           1 | tanaka haha    | XXXXX   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka haha    | XXXXX   |           1 | tanaka ane     | YYYYY
           1 | tanaka ane     | YYYYY   |           1 | tanaka ototo2  | XXXXX
           1 | tanaka ane     | YYYYY   |           1 | tanaka imouto2 | XXXXX
           1 | tanaka ane     | YYYYY   |           1 | tanaka ototo1  | XXXXX
           1 | tanaka ane     | YYYYY   |           1 | tanaka imouto1 | XXXXX
           1 | tanaka ane     | YYYYY   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka ane     | YYYYY   |           1 | tanaka haha    | XXXXX
           1 | tanaka ane     | YYYYY   |           1 | tanaka titi    | XXXXX
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka ototo2  | XXXXX
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka imouto2 | XXXXX
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka ototo1  | XXXXX
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka imouto1 | XXXXX
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka ane     | YYYYY
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka haha    | XXXXX
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka titi    | XXXXX
           1 | tanaka imouto1 | XXXXX   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka imouto1 | XXXXX   |           1 | tanaka ane     | YYYYY
           1 | tanaka ototo1  | XXXXX   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka ototo1  | XXXXX   |           1 | tanaka ane     | YYYYY
           1 | tanaka imouto2 | XXXXX   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka imouto2 | XXXXX   |           1 | tanaka ane     | YYYYY
           1 | tanaka ototo2  | XXXXX   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka ototo2  | XXXXX   |           1 | tanaka ane     | YYYYY
           2 | tanaka wake    | XXXXX   |           2 | tanaka ari     | AAAAA
           2 | tanaka ari     | AAAAA   |           2 | tanaka wake    | XXXXX
(28 rows)
```

例えば、下記のような組み合わせは順番が違うだけで、1 行あれば十分なので、これも除外しておく。

```
 t1_tanakaid |    t1_name     | t1_addr | t2_tanakaid |    t2_name     | t2_addr
-------------+----------------+---------+-------------+----------------+---------
           1 | tanaka titi    | XXXXX   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka titi    | XXXXX

           1 | tanaka titi    | XXXXX   |           1 | tanaka ane     | YYYYY
           1 | tanaka ane     | YYYYY   |           1 | tanaka titi    | XXXXX

           1 | tanaka ane     | YYYYY   |           1 | tanaka ani     | ZZZZZ
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka ane     | YYYYY
略
```

SQL はこちら。

```sql
select
	t1.tanakaid as t1_tanakaid,
	t1.name as t1_name,
	t1.addr as t1_addr,
	t2.tanakaid as t2_tanakaid,
	t2.name as t2_name,
	t2.addr as t2_addr
from
	tanaka as t1
cross join
	tanaka as t2
where
	t1.tanakaid = t2.tanakaid and
	t1.addr != t2.addr and
	t1.name > t2.name
;

 t1_tanakaid |    t1_name     | t1_addr | t2_tanakaid |  t2_name   | t2_addr
-------------+----------------+---------+-------------+------------+---------
           1 | tanaka titi    | XXXXX   |           1 | tanaka ani | ZZZZZ
           1 | tanaka titi    | XXXXX   |           1 | tanaka ane | YYYYY
           1 | tanaka haha    | XXXXX   |           1 | tanaka ani | ZZZZZ
           1 | tanaka haha    | XXXXX   |           1 | tanaka ane | YYYYY
           1 | tanaka ani     | ZZZZZ   |           1 | tanaka ane | YYYYY
           1 | tanaka imouto1 | XXXXX   |           1 | tanaka ani | ZZZZZ
           1 | tanaka imouto1 | XXXXX   |           1 | tanaka ane | YYYYY
           1 | tanaka ototo1  | XXXXX   |           1 | tanaka ani | ZZZZZ
           1 | tanaka ototo1  | XXXXX   |           1 | tanaka ane | YYYYY
           1 | tanaka imouto2 | XXXXX   |           1 | tanaka ani | ZZZZZ
           1 | tanaka imouto2 | XXXXX   |           1 | tanaka ane | YYYYY
           1 | tanaka ototo2  | XXXXX   |           1 | tanaka ani | ZZZZZ
           1 | tanaka ototo2  | XXXXX   |           1 | tanaka ane | YYYYY
           2 | tanaka wake    | XXXXX   |           2 | tanaka ari | AAAAA
(14 rows)
```

## :closed_book: Reference

None
