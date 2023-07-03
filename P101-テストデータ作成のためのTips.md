## :memo: Overview

ここでは下記を参考にテストデータを作成するためのデータ生成関連のコマンドをまとめておく。ここでは扱わないが、`delete`や`update`などの方法をのっていて、役立つ Tips が満載。

- [Postgres Tips And Tricks](https://pgdash.io/blog/postgres-tips-and-tricks.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`serial`, `DEFAULT`, `returning`

## :pencil2: Example

連番や更新日時を利用するには、`serial`や`default now()`を利用する。

```sql
create table dummy1 (
    serial_no serial,
    created_at timestamptz default now(),
    val int
);

insert into dummy1(val) values (1) returning serial_no, created_at, val;
 serial_no |          created_at          | val
-----------+------------------------------+-----
         1 | 2022-09-18 10:35:04.72813+09 |   1
(1 row)

insert into dummy1(val) values (2) returning serial_no, created_at, val;
 serial_no |          created_at          | val
-----------+------------------------------+-----
         2 | 2022-09-18 10:35:35.75618+09 |   2
(1 row)


insert into dummy1(val) values (3) returning serial_no, created_at, val;
 serial_no |          created_at           | val
-----------+-------------------------------+-----
         3 | 2022-09-18 10:35:50.379427+09 |   3
(1 row)

select * from dummy1;
 serial_no |          created_at           | val
-----------+-------------------------------+-----
         1 | 2022-09-18 10:35:04.72813+09  |   1
         2 | 2022-09-18 10:35:35.75618+09  |   2
         3 | 2022-09-18 10:35:50.379427+09 |   3
(3 rows)
```

`uuid-ossp`モジュールの`uuid default uuid_generate_v4()`は、汎用一意識別子を生成する。型とデフォルトを設定するとユニークな ID を生成できる。

```sql
create extention if not exists "uuid-ossp";

create table dummy2 (
    cuid uuid DEFAULT uuid_generate_v4(),
    serial_no serial,
    created_at timestamptz default now(),
    val int
);

insert into dummy2(val) values (1);
insert into dummy2(val) values (2);
insert into dummy2(val) values (3);
insert into dummy2(val) values (4);
insert into dummy2(val) values (5);

select * from dummy2;
                 cuid                 | serial_no |          created_at           | val
--------------------------------------+-----------+-------------------------------+-----
 32f24026-9f2e-4c98-9ad7-daaf90300bc0 |         1 | 2022-09-18 10:40:43.895681+09 |   1
 8a3692bd-1714-42ae-b179-2066cfd19e1d |         2 | 2022-09-18 10:40:43.90019+09  |   2
 c74ddc4f-9f76-444b-8901-5be10bf2966c |         3 | 2022-09-18 10:40:43.901238+09 |   3
 f8d7a25a-6e9c-4a1e-87b1-a0723dee2693 |         4 | 2022-09-18 10:40:43.901511+09 |   4
 320f625e-8666-488f-a476-cd3315e654d4 |         5 | 2022-09-18 10:40:43.901741+09 |   5
(5 rows)
```

サイコロを振りたいときは、下記の通り`unnest(array[1,2,3,4,5,6]))`としなくても`random()`に上限値+1 をかけて、整数にキャストすればよい。

```sql
select value
from unnest(array[1,2,3,4,5,6,
1,2,3,4,5,6,
1,2,3,4,5,6,
1,2,3,4,5,6,
1,2,3,4,5,6,]) as value;

select setseed(0.1989);
select 1 + (5 * random())::int as dice
from generate_series(1, 20);

 dice
------
    4
    4
    4
    4
    2
    4
    5
    6
    2
    5
    1
    2
    3
    2
    3
    2
    2
    2
    2
    5
(20 rows)
```

28 ビット固定長のハッシュ値は、`md5()`で生成できる。

```sql
select setseed(0.1989);
select md5(random()::text) FROM generate_series(1, 10);

               md5
----------------------------------
 3302c03c99c1a88e898bb4fc9eb60768
 c110909b4bcabb3251747063638ad544
 a9f4436f38e58fb8e4ffb422b8da3160
 6e78dd6c3cc43af05f1924a351a94174
 231cd7b73230a82b85c7c0b1f1ee35c5
 0c831cb6869410e150057c59928e163b
 b2e564ec386da7bdaac031106d2866be
 599b3b9fb7ef8b78c6c7516cd09f2599
 bc24a73db269253a180b7d315f5070bc
 f4eaaed8adeb7139cbbc9f3ffed07df7
(10 rows)
```

日付をランダムに生成したければ`interval`の日の数字を乱数で生成すればよく、年間日数 365 に年数 1 をかければ、起点日から およそ 1 年の範囲で生成できる。

```sql
select date(date '2022-01-01' + ((random()*365*1)::int || ' days')::interval)
from generate_series(1, 10);

    date
------------
 2022-01-19
 2022-04-08
 2022-06-05
 2022-03-08
 2022-05-09
 2022-04-06
 2022-03-04
 2022-04-20
 2022-04-04
 2022-09-25
(10 rows)
```

増加傾向を持つテストデータは`row_number()`を利用すればよい。

```sql
-- 増加傾向
select
    to_char(day, 'YYYY-MM-DD') as day,
    log(row_number() over()) as eta,
    (100 + 10 * random()) * log(row_number() over()) as value
from generate_series('2022-01-01'::date, '2022-01-31'::date, '1 day'::interval) as day;

    day     |         eta         |       value
------------+---------------------+--------------------
 2022-01-01 |                   0 |                  0
 2022-01-02 |  0.3010299956639812 | 31.886926071118246
 2022-01-03 | 0.47712125471966244 |  49.93444250361544
 2022-01-04 |  0.6020599913279624 |  65.40718472560201
 2022-01-05 |  0.6989700043360189 |  71.87494296492974
 2022-01-06 |  0.7781512503836436 |  83.36861309353894
 2022-01-07 |  0.8450980400142568 |  90.33624140436272
 2022-01-08 |  0.9030899869919435 |  96.53294913287016
 2022-01-09 |  0.9542425094393249 |  102.8137387391158
 2022-01-10 |                   1 | 109.57368774908599
 2022-01-11 |  1.0413926851582251 | 108.89123736943945
 2022-01-12 |  1.0791812460476249 | 109.25612180080017
 2022-01-13 |  1.1139433523068367 | 113.72476530846104
 2022-01-14 |   1.146128035678238 | 118.63983500898425
 2022-01-15 |  1.1760912590556813 |  120.5444397813621
 2022-01-16 |  1.2041199826559248 | 123.75818406413893
 2022-01-17 |  1.2304489213782739 | 125.04478386980361
 2022-01-18 |   1.255272505103306 | 127.79769398382999
 2022-01-19 |  1.2787536009528289 | 132.30160910956474
 2022-01-20 |  1.3010299956639813 |  131.0261318391964
 2022-01-21 |  1.3222192947339193 |  144.8737593943207
 2022-01-22 |  1.3424226808222062 | 136.75217757201764
 2022-01-23 |  1.3617278360175928 | 143.86919770172673
 2022-01-24 |   1.380211241711606 | 140.75749732404995
 2022-01-25 |  1.3979400086720377 | 151.30895469210054
 2022-01-26 |   1.414973347970818 | 144.90008700497174
 2022-01-27 |  1.4313637641589874 | 156.37159470060462
 2022-01-28 |  1.4471580313422192 | 150.46333119337837
 2022-01-29 |   1.462397997898956 | 152.09111194102326
 2022-01-30 |  1.4771212547196624 |  159.7813050050977
 2022-01-31 |  1.4913616938342726 | 162.91095651876213
(31 rows)
```

ここまでの内容を組み合わせると、目的はさておきテストデータを作成できる。

```sql
select
unnest(array(select ('2022-01-01'::date + ((random()*365*1)::int || ' days')::interval)::date from generate_series(1,10))) as randay,
unnest(array(select random() from generate_series(1,10))) as rannum,
unnest(array(select md5(random()::text) from generate_series(1,10))) as ranchar
;
       rannum        |             ranchar              |   randay
---------------------+----------------------------------+------------
   0.431220202844937 | 1b1bbbbaa2a6d4c71e0f28e0cb8e8dbe | 2022-10-30
  0.8043366599402404 | 0116b38c7a3a3059d180c4094b1c4a80 | 2022-07-05
 0.31386451971571105 | ff45edc4393ae4b429f7724515484d9c | 2022-01-26
  0.5870063935301175 | 1cbf89e6fdf28b9fbd677abdf40b29c6 | 2023-01-01
  0.5266037286682028 | 72276037db7f5de563e75625ac3cb83a | 2022-04-05
  0.7026724683443675 | e2236ccf97ee36189717ba6a70dcca0b | 2022-09-21
 0.42444991223404216 | 0d9e24a00fcb328a1e608e7175705e2d | 2022-01-13
  0.5604558632273751 | 3c24920c84fadd3bf3508ccf8fc59e15 | 2022-12-09
  0.9975557073313119 | b1b7dd4905e124d6260bf25db6d92a0c | 2022-08-16
 0.21400295328643182 | 3e763302e0e83644d5672390367eec93 | 2022-10-26
(10 rows)
```

既存のテーブルからサンプリングしたいときは、`tablesample bernoulli(ratio)`を利用する。

```sql
select count(1) from event;
  count
----------
 16995997
(1 row)

select count(1) from event tablesample bernoulli(1);
 count
--------
 169410
(1 row)

select count(1) from event tablesample bernoulli(10);
  count
---------
 1699502
(1 row)

select count(1) from event tablesample bernoulli(50);
  count
---------
 8498206
(1 row)
```

サンプリングして別のテーブルにインサートする際は下記の通り。

```sql
create table mini_event as
select * from event
tablesample bernoulli(10);
SELECT 1698169

select count(1) from mini_event;
  count
---------
 1698169
(1 row)
```

作ったデータをダンプする際は`copy`コマンドを利用する。

```sql
-- \copy mini_event to '/Users/aki/Desktop/mini_event.csv' (format csv);
copy mini_event
to '/Users/aki/Desktop/mini_event.csv' (format csv, header);
COPY 1698169

$ head -n 10  ~/Desktop/mini_event.csv
account_id,event_time,event_type_id
1,2020-01-20 23:59:07,4
1,2020-01-27 08:51:19,2
1,2020-02-08 23:08:07,4
1,2020-02-09 23:51:41,1
1,2020-02-28 20:42:51,4
1,2020-03-03 13:02:37,2
1,2020-03-12 08:04:59,4
1,2020-03-14 11:05:53,4
1,2020-03-20 19:35:54,3
```

ダンプする際に SQL でデータを処理してダンプできる。

```sql
copy (select account_id, event_time from mini_event)
to '/Users/aki/Desktop/mini_event.csv' (format csv, header);
COPY 1698169

$ head -n 10  ~/Desktop/mini_event.csv
account_id,event_time
1,2020-01-20 23:59:07
1,2020-01-27 08:51:19
1,2020-02-08 23:08:07
1,2020-02-09 23:51:41
1,2020-02-28 20:42:51
1,2020-03-03 13:02:37
1,2020-03-12 08:04:59
1,2020-03-14 11:05:53
1,2020-03-20 19:35:54
```

## :closed_book: Reference

- [Postgres Tips And Tricks](https://pgdash.io/blog/postgres-tips-and-tricks.html)
