## :memo: Overview

カンマ区切りで 1 つのカラムに値を格納するいわゆる jaywalk のデータを作ったり、分解するための方法をまとめる。PostgreSQL では、`string_agg`と`split_part`が利用でき、MySQL では`group_concat`と`substring_index`が利用できる。

## :floppy_disk: Database

PostgreSQL / MySQL

## :bookmark: Tag

`string_agg`, `split_part`, `regexp_split_to_table`, `group_concat`, `substring_index`

## :pencil2: Example

まずは jaywalk のデータを作るためのサンプルデータを用意する。`id=0`は`null`しか持たないレコードで処理の過程で削除されるため、最終的に必要なのであれば、必要に応じて`join`して紐付けるなどして回復する。jaywalk というのは信号無視という意味らしく、カンマ区切りで値を保存するのはデーターベースではルール違反ということからきているのでしょうか。

```sql
create table delimiter (id varchar(10), fruit varchar(255));

insert into delimiter
  (id, fruit)
values
  (0, null),
  (1, 'apple'),
  (1, 'cherry'),
  (1, 'kiwi fruit'),
  (2, 'pomegranate'),
  (3, 'banana'),
  (3, 'loquat'),
  (3, 'cherry'),
  (3, 'apple'),
  (3, 'grape'),
  (4, 'peach'),
  (4, 'cherry'),
  (5, 'peach'),
  (5, 'cherry'),
  (5, 'mandarin'),
  (5, 'tangerine');

```

まずは PostgreSQL の例を記載する。PostgreSQL では`string_agg`を利用すれば、カンマ区切りで集約できる。区切り文字は指定できる。 この後、Wide 型を Long 型に変換する方法も記載するので、都合のためビューにしておく。

```sql
create view wide as (
select
	id,
	string_agg(fruit, ',') as fruit_group
from
	delimiter
group by
	id
order by
	id asc
);

select * from wide;

 id |           fruit_group
----+----------------------------------
 0  |
 1  | apple,cherry,kiwi fruit
 2  | pomegranate
 3  | banana,loquat,cherry,apple,grape
 4  | peach,cherry
 5  | peach,cherry,mandarin,tangerine
```

Wide 型を Long 型に変換する方法として愚直なやり方としては、下記方法もあるが、カンマの数が増えると更新が面倒だったり、漏れが発生して集計誤りに繋がりやすい。

```sql
with tmp as (
select id, split_part(fruit_group, ',', 1) as fruit from wide
union all
select id, split_part(fruit_group, ',', 2) as fruit from wide
union all
select id, split_part(fruit_group, ',', 3) as fruit from wide
union all
select id, split_part(fruit_group, ',', 4) as fruit from wide
union all
select id, split_part(fruit_group, ',', 5) as fruit from wide
)
select * from tmp
where fruit is not null and fruit != ''
;
 id |    fruit
----+-------------
 1  | apple
 2  | pomegranate
 3  | banana
 4  | peach
 5  | peach
 1  | cherry
 3  | loquat
 4  | cherry
 5  | cherry
 1  | kiwi fruit
 3  | cherry
 5  | mandarin
 3  | apple
 5  | tangerine
 3  | grape
```

回避策としては大きめな連番をもつテーブルと直積をとることで必要な数だけ処理する方法がある。

`where`のところの処理は、例えば`apple,cherry,kiwi fruit`を例にすると、カンマありだと 23 文字で、カンマなしだと 21 文字となる。つまりカンマは 2 個含まれており、フルーツは 3 個含まれていることがわかる。`split_part(fruit_group, ',', irer.i)`で必要なインデックスはカンマ 2 個にプラス 1 をした 3 以下のインデックスを取得すれば、必要な数だけ取り出すことができる。

```sql
select
	w.id,
	w.fruit_group,
	split_part(w.fruit_group, ',', iter.id) as fruit,
	iter.id as iternum,
	length(fruit_group) as len,
	length(replace(fruit_group, ',','')) as lenwithout,
	length(fruit_group) - length(replace(fruit_group, ',','')) + 1 as lendiff
from
	wide as w
cross join
	t10 as iter
where
	iter.id <= (length(fruit_group) - length(replace(fruit_group, ',',''))) + 1
order by
	id asc
;

 id |           fruit_group            |    fruit    | iternum | len | lenwithout | lendiff
----+----------------------------------+-------------+---------+-----+------------+---------
 1  | apple,cherry,kiwi fruit          | apple       |       1 |  23 |         21 |       3
 1  | apple,cherry,kiwi fruit          | cherry      |       2 |  23 |         21 |       3
 1  | apple,cherry,kiwi fruit          | kiwi fruit  |       3 |  23 |         21 |       3
 2  | pomegranate                      | pomegranate |       1 |  11 |         11 |       1
 3  | banana,loquat,cherry,apple,grape | banana      |       1 |  32 |         28 |       5
 3  | banana,loquat,cherry,apple,grape | loquat      |       2 |  32 |         28 |       5
 3  | banana,loquat,cherry,apple,grape | cherry      |       3 |  32 |         28 |       5
 3  | banana,loquat,cherry,apple,grape | apple       |       4 |  32 |         28 |       5
 3  | banana,loquat,cherry,apple,grape | grape       |       5 |  32 |         28 |       5
 4  | peach,cherry                     | peach       |       1 |  12 |         11 |       2
 4  | peach,cherry                     | cherry      |       2 |  12 |         11 |       2
 5  | peach,cherry,mandarin,tangerine  | peach       |       1 |  31 |         28 |       4
 5  | peach,cherry,mandarin,tangerine  | cherry      |       2 |  31 |         28 |       4
 5  | peach,cherry,mandarin,tangerine  | mandarin    |       3 |  31 |         28 |       4
 5  | peach,cherry,mandarin,tangerine  | tangerine   |       4 |  31 |         28 |       4
```

`regexp_split_to_table`を使えばもっと楽にテーブルを変換できる。

```sql
select
	id,
	fruit_group,
	regexp_split_to_table(fruit_group, ',') as fruit
from
	wide
;

 id |           fruit_group            |  fruit
----+----------------------------------+-------------
 1  | apple,cherry,kiwi fruit          | apple
 1  | apple,cherry,kiwi fruit          | cherry
 1  | apple,cherry,kiwi fruit          | kiwi fruit
 2  | pomegranate                      | pomegranate
 3  | banana,loquat,cherry,apple,grape | banana
 3  | banana,loquat,cherry,apple,grape | loquat
 3  | banana,loquat,cherry,apple,grape | cherry
 3  | banana,loquat,cherry,apple,grape | apple
 3  | banana,loquat,cherry,apple,grape | grape
 4  | peach,cherry                     | peach
 4  | peach,cherry                     | cherry
 5  | peach,cherry,mandarin,tangerine  | peach
 5  | peach,cherry,mandarin,tangerine  | cherry
 5  | peach,cherry,mandarin,tangerine  | mandarin
 5  | peach,cherry,mandarin,tangerine  | tangerine
(15 rows)
```

ここからは MySQL バージョン。`group_concat`でカンマ区切りでフルーツを集約する。区切り文字は指定できる。 この後、Wide 型を Long 型に変換する方法も記載するので、都合のためビューにしておく。

```sql
create view wide as (
select
	id,
  -- カンマ区切り後の並び替えを行いたい場合
	-- group_concat(fruit order by fruit separator ',') as fruit_group
  group_concat(fruit separator ',') as fruit_group
from
	delimiter
group by
	id
);

select * from wide;
+------+----------------------------------+
| id   | fruit_group                      |
+------+----------------------------------+
| 0    | NULL                             |
| 1    | apple,cherry,kiwi fruit          |
| 2    | pomegranate                      |
| 3    | banana,loquat,cherry,apple,grape |
| 4    | peach,cherry                     |
| 5    | peach,cherry,mandarin,tangerine  |
+------+----------------------------------+
```

Wide 型を Long 型に変換するが、`substring_index`の動きを先に確認しておく。`substring_index`は指定された区切り文字が位置する番号よりも前の文字列を取得する。そのため、指定した番号よりも前の文字列を取得して、その返却された文字列に対して、後ろから数えて 1 つ目の区切り文字以上の文字列を取得することで、カンマ区切りの文字列を取得する。

```sql
select
	fruit_group,
	substring_index(fruit_group, ',', 1) as ind1,
	substring_index(substring_index(fruit_group, ',', 1), ',', -1) as ind1,
	substring_index(fruit_group, ',', 2) as ind2,
	substring_index(substring_index(fruit_group, ',', 2), ',', -1) as ind2,
	substring_index(fruit_group, ',', 3) as ind3 ,
	substring_index(substring_index(fruit_group, ',', 3), ',', -1) as ind3
from
	wide
;

+----------------------------------+-------------+-------------+---------------+-------------+-------------------------+-------------+
| fruit_group                      | ind1        | ind1        | ind2          | ind2        | ind3                    | ind3        |
+----------------------------------+-------------+-------------+---------------+-------------+-------------------------+-------------+
| NULL                             | NULL        | NULL        | NULL          | NULL        | NULL                    | NULL        |
| apple,cherry,kiwi fruit          | apple       | apple       | apple,cherry  | cherry      | apple,cherry,kiwi fruit | kiwi fruit  |
| pomegranate                      | pomegranate | pomegranate | pomegranate   | pomegranate | pomegranate             | pomegranate |
| banana,loquat,cherry,apple,grape | banana      | banana      | banana,loquat | loquat      | banana,loquat,cherry    | cherry      |
| peach,cherry                     | peach       | peach       | peach,cherry  | cherry      | peach,cherry            | cherry      |
| peach,cherry,mandarin,tangerine  | peach       | peach       | peach,cherry  | cherry      | peach,cherry,mandarin   | mandarin    |
+----------------------------------+-------------+-------------+---------------+-------------+-------------------------+-------------+
```

あとは PostgreSQL と同じようにクエリを記述すれば OK。

```sql
select
	w.id,
	w.fruit_group,
	substring_index(substring_index(w.fruit_group, ',', iter.id), ',', -1) as fruit,
	iter.id as iternum,
	length(fruit_group) as len,
	length(replace(fruit_group, ',','')) as lenwithout,
	length(fruit_group) - length(replace(fruit_group, ',',''))+1 as lendiff
from
	wide as w
cross join
	t10 as iter
where
	iter.id <= (length(fruit_group) - length(replace(fruit_group, ',','')))+1
order by
	id asc,
    iternum asc
;

+------+----------------------------------+-------------+---------+------+------------+---------+
| id   | fruit_group                      | fruit       | iternum | len  | lenwithout | lendiff |
+------+----------------------------------+-------------+---------+------+------------+---------+
| 1    | apple,cherry,kiwi fruit          | apple       |       1 |   23 |         21 |       3 |
| 1    | apple,cherry,kiwi fruit          | cherry      |       2 |   23 |         21 |       3 |
| 1    | apple,cherry,kiwi fruit          | kiwi fruit  |       3 |   23 |         21 |       3 |
| 2    | pomegranate                      | pomegranate |       1 |   11 |         11 |       1 |
| 3    | banana,loquat,cherry,apple,grape | banana      |       1 |   32 |         28 |       5 |
| 3    | banana,loquat,cherry,apple,grape | loquat      |       2 |   32 |         28 |       5 |
| 3    | banana,loquat,cherry,apple,grape | cherry      |       3 |   32 |         28 |       5 |
| 3    | banana,loquat,cherry,apple,grape | apple       |       4 |   32 |         28 |       5 |
| 3    | banana,loquat,cherry,apple,grape | grape       |       5 |   32 |         28 |       5 |
| 4    | peach,cherry                     | peach       |       1 |   12 |         11 |       2 |
| 4    | peach,cherry                     | cherry      |       2 |   12 |         11 |       2 |
| 5    | peach,cherry,mandarin,tangerine  | peach       |       1 |   31 |         28 |       4 |
| 5    | peach,cherry,mandarin,tangerine  | cherry      |       2 |   31 |         28 |       4 |
| 5    | peach,cherry,mandarin,tangerine  | mandarin    |       3 |   31 |         28 |       4 |
| 5    | peach,cherry,mandarin,tangerine  | tangerine   |       4 |   31 |         28 |       4 |
+------+----------------------------------+-------------+---------+------+------------+---------+
```

おまけ。隣接都道府県マスタのようなものを作ることもできる。

```sql
create table neighbor(
    no int,
    pref varchar(255),
    neighbor_num int,
    neighbor_pref varchar(255)
);

insert into neighbor values('1','北海道','1','青森県');
insert into neighbor values('2','青森県','3','北海道,岩手県,秋田県');
insert into neighbor values('3','岩手県','3','青森県,宮城県,秋田県');
insert into neighbor values('4','宮城県','4','岩手県,秋田県,山形県,福島県');
insert into neighbor values('5','秋田県','4','青森県,岩手県,宮城県,山形県');
insert into neighbor values('6','山形県','4','宮城県,秋田県,福島県,新潟県');
insert into neighbor values('7','福島県','6','宮城県,山形県,茨城県,栃木県,群馬県,新潟県');
insert into neighbor values('8','茨城県','4','福島県,栃木県,埼玉県,千葉県');
insert into neighbor values('9','栃木県','4','福島県,茨城県,群馬県,埼玉県');
insert into neighbor values('10','群馬県','5','福島県,栃木県,埼玉県,新潟県,長野県');
insert into neighbor values('11','埼玉県','7','茨城県,栃木県,群馬県,千葉県,東京都,長野県,山梨県');
insert into neighbor values('12','千葉県','4','茨城県,埼玉県,東京都,神奈川県');
insert into neighbor values('13','東京都','4','埼玉県,千葉県,神奈川県,山梨県');
insert into neighbor values('14','神奈川県','4','千葉県,東京都,山梨県,静岡県');
insert into neighbor values('15','新潟県','5','山形県,福島県,群馬県,富山県,長野県');
insert into neighbor values('16','富山県','4','新潟県,石川県,長野県,岐阜県');
insert into neighbor values('17','石川県','3','富山県,福井県,岐阜県');
insert into neighbor values('18','福井県','4','石川県,岐阜県,滋賀県,京都府');
insert into neighbor values('19','山梨県','5','埼玉県,東京都,神奈川県,長野県,静岡県');
insert into neighbor values('20','長野県','8','群馬県,埼玉県,新潟県,富山県,山梨県,岐阜県,静岡県,愛知県');
insert into neighbor values('21','岐阜県','7','富山県,石川県,福井県,長野県,愛知県,三重県,滋賀県');
insert into neighbor values('22','静岡県','4','神奈川県,山梨県,長野県,愛知県');
insert into neighbor values('23','愛知県','4','長野県,岐阜県,静岡県,三重県');
insert into neighbor values('24','三重県','6','岐阜県,愛知県,滋賀県,京都府,奈良県,和歌山県');
insert into neighbor values('25','滋賀県','4','福井県,岐阜県,三重県,京都府');
insert into neighbor values('26','京都府','6','福井県,三重県,滋賀県,大阪府,兵庫県,奈良県');
insert into neighbor values('27','大阪府','4','京都府,兵庫県,奈良県,和歌山県');
insert into neighbor values('28','兵庫県','5','京都府,大阪府,鳥取県,岡山県,徳島県');
insert into neighbor values('29','奈良県','4','三重県,京都府,大阪府,和歌山県');
insert into neighbor values('30','和歌山県','3','三重県,大阪府,奈良県');
insert into neighbor values('31','鳥取県','4','兵庫県,島根県,岡山県,広島県');
insert into neighbor values('32','島根県','3','鳥取県,広島県,山口県');
insert into neighbor values('33','岡山県','4','兵庫県,鳥取県,広島県,香川県');
insert into neighbor values('34','広島県','5','鳥取県,島根県,岡山県,山口県,愛媛県');
insert into neighbor values('35','山口県','3','島根県,広島県,福岡県');
insert into neighbor values('36','徳島県','4','兵庫県,香川県,愛媛県,高知県');
insert into neighbor values('37','香川県','3','岡山県,徳島県,愛媛県');
insert into neighbor values('38','愛媛県','4','広島県,徳島県,香川県,高知県');
insert into neighbor values('39','高知県','2','徳島県,愛媛県');
insert into neighbor values('40','福岡県','4','山口県,佐賀県,熊本県,大分県');
insert into neighbor values('41','佐賀県','2','福岡県,長崎県');
insert into neighbor values('42','長崎県','1','佐賀県');
insert into neighbor values('43','熊本県','4','福岡県,大分県,宮崎県,鹿児島県');
insert into neighbor values('44','大分県','3','福岡県,熊本県,宮崎県');
insert into neighbor values('45','宮崎県','3','熊本県,大分県,鹿児島県');
insert into neighbor values('46','鹿児島県','2','熊本県,宮崎県');
insert into neighbor values('47','沖縄県','0',null);


select
	n.no,
    n.pref,
	split_part(n.neighbor_pref, ',', iter.id) as neighbor_pref,
    neighbor_num,
	iter.id as iternum,
	length(neighbor_pref) as len,
	length(replace(neighbor_pref, ',','')) as lenwithout,
	length(neighbor_pref) - length(replace(neighbor_pref, ',','')) + 1 as lendiff
from
	neighbor as n
cross join
	t10 as iter
where
	iter.id <= (length(neighbor_pref) - length(replace(neighbor_pref, ',',''))) + 1
order by
	no asc
;

 no |   pref   | neighbor_pref | neighbor_num | iternum | len | lenwithout | lendiff
----+----------+---------------+--------------+---------+-----+------------+---------
  1 | 北海道   | 青森県        |            1 |       1 |   3 |          3 |       1
  2 | 青森県   | 北海道        |            3 |       1 |  11 |          9 |       3
  2 | 青森県   | 岩手県        |            3 |       2 |  11 |          9 |       3
  2 | 青森県   | 秋田県        |            3 |       3 |  11 |          9 |       3
  3 | 岩手県   | 宮城県        |            3 |       2 |  11 |          9 |       3
  3 | 岩手県   | 秋田県        |            3 |       3 |  11 |          9 |       3
  3 | 岩手県   | 青森県        |            3 |       1 |  11 |          9 |       3
  4 | 宮城県   | 秋田県        |            4 |       2 |  15 |         12 |       4
  4 | 宮城県   | 山形県        |            4 |       3 |  15 |         12 |       4
  4 | 宮城県   | 福島県        |            4 |       4 |  15 |         12 |       4
  4 | 宮城県   | 岩手県        |            4 |       1 |  15 |         12 |       4
  5 | 秋田県   | 青森県        |            4 |       1 |  15 |         12 |       4
  5 | 秋田県   | 宮城県        |            4 |       3 |  15 |         12 |       4
  5 | 秋田県   | 岩手県        |            4 |       2 |  15 |         12 |       4
  5 | 秋田県   | 山形県        |            4 |       4 |  15 |         12 |       4
  6 | 山形県   | 福島県        |            4 |       3 |  15 |         12 |       4
  6 | 山形県   | 宮城県        |            4 |       1 |  15 |         12 |       4
  6 | 山形県   | 秋田県        |            4 |       2 |  15 |         12 |       4
  6 | 山形県   | 新潟県        |            4 |       4 |  15 |         12 |       4
```

## :closed_book: Reference

- [SQL Cookbook](https://www.oreilly.com/library/view/sql-cookbook/0596009763/)
- [SQL クックブック 第 2 版](https://www.oreilly.co.jp/books/9784873119779/)
