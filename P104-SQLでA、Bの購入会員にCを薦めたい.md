## :memo: Overview

ここでは特定の商品を購入しているが、特定の商品は購入していない会員を抽出するクエリの方法をまとめておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`in`, `not in`, `all`, `join`, `exists`

## :pencil2: Example

まずはサンプルデータ用意する。

```sql
create table recommend(orid varchar, cuid varchar, item varchar);

insert into recommend(orid, cuid, item)
values
 ('or1','cuid1','A'), -- NG
 ('or2','cuid1','B'), -- NG
 ('or3','cuid1','D'), -- NG
 ('or4','cuid1','C'), -- NG
 ('or5','cuid2','A'), -- NG
 ('or6','cuid3','A'), -- OK
 ('or7','cuid3','B'), -- OK
 ('or8','cuid3','D'), -- OK
 ('or9','cuid4','C')  -- NG
;
```

今回の抽出条件は、`A,B`を購入していて、`C`を購入していない会員のリストがほしい、というもの。

- `cuid1`は`A,B,C`を購入しているので対象外
- `cuid2`は`A`しか購入してないので対象外
- `cuid3`は`A,B`を購入しているので対象
- `cuid4`は`C`を購入しているので対象外

この抽出条件を素直に SQL で記述すれば OK。`B`を購入している会員かつ、その中で`A`を購入している会員を抽出する、そして、その会員の中から、`C`を購入していない会員を条件付ければ OK。

```sql
with t1 as(
select cuid from recommend where item = 'B' and
cuid in (select cuid from recommend where item = 'A')
)
select cuid
from t1
where cuid != all(select cuid from recommend where item = 'C');

 cuid
-------
 cuid3
(1 row)
```

今回のデータに限れば`not in`でも OK。

```sql
with t1 as(
select cuid from recommend where item = 'B' and
cuid in (select cuid from recommend where item = 'A')
)
select cuid
from t1
where cuid not in(select cuid from recommend where item = 'C');

 cuid
-------
 cuid3
(1 row)
```

上のSQLをまとめたものがこちら。こっちのほうがスマートかも。

```sql
select distinct cuid
from recommend
where 
      cuid in (select cuid from recommend where item = 'A')
  and cuid in (select cuid from recommend where item = 'B')
  and cuid not in (select cuid from recommend where item in ('C'));

 cuid
-------
 cuid3
(1 row)
```

`join`で愚直に行く方法でもわかりやすくていいかもしれない。下記はすごく冗長だけど。

```sql
with list as (
select distinct cuid from recommend
-- cuid
----------
-- cuid1
-- cuid2
-- cuid3
-- cuid4
),
itema as (
select distinct cuid as cuid_a from recommend where item = 'A'
-- cuid_a
----------
-- cuid1
-- cuid2
-- cuid3
),
itemb as (
select distinct cuid as cuid_b from recommend where item = 'B'
-- cuid_b
----------
-- cuid1
-- cuid3
),
itemc as (
select distinct cuid as cuid_c from recommend where item = 'C'
-- cuid_c
-- --------
--  cuid1
--  cuid4
),
ab as (
select cuid from list as r
inner join itema as a on r.cuid = a.cuid_a
inner join itemb as b on r.cuid = b.cuid_b
-- cuid
---------
-- cuid1
-- cuid3
)
select cuid from ab as ab
left join itemc as c on ab.cuid = c.cuid_c
where c.cuid_c is null
;

 cuid
-------
 cuid3
```

`case`で対処する方法だとこうなる。A,Bを購入している場合、次の`case`に進む。購入していないと、0が確定する。購入していてほしい商品の判別が終わったら、購入してない商品かどうかの判別を行う。`case`のイメージは下記の通り。

```
cuid1 F   F   T>0
cuid2 F   T>0
cuid3 F   F   F>1
cuid4 T>0 
```

`where`に`case`を使用する。

```sql
-- A,Bを購入していて、かつCを購入していない顧客
select distinct cuid
from recommend
where 1 = (
  -- 購入している場合、次のcaseに進む。購入していないと、0が確定する。
    case 
    when cuid not in (select cuid from recommend where item = 'A') then 0
    when cuid not in (select cuid from recommend where item = 'B') then 0
    when cuid in (select cuid from recommend where item = 'C') then 0
    else 1 end
);

 cuid
-------
 cuid3
(1 row)
```

## :pencil2: おまけ

今回のケースであれば`all`でも`not in`でも可能と書いたが、下記のように`null`が入るとだめになる。前にも解説した通り、3 値論理の真理表を見ればわかる。`null`なんか入らないでしょうと？と思うかもしれないが、エンジニアががっちがっちに管理しているのであれば別だが、そうではない DB やデータの連携をたくさんしてると伝言ゲームのようになって`null`がこんにちわしていることはよくある。

```sql
create table recommend2(orid varchar, cuid varchar, item varchar);

insert into recommend2(orid, cuid, item)
values
 ('or1','cuid1','A'), -- NG
 ('or2','cuid1','B'), -- NG
 ('or3','cuid1','D'), -- NG
 ('or4','cuid1','C'), -- NG
 ('or5','cuid2','A'), -- NG
 ('or6','cuid3','A'), -- OK
 ('or7','cuid3','B'), -- OK
 ('or8','cuid3','D'), -- OK
 ('or9','cuid4','C'),  -- NG
 ('or10',null,'C')  -- null
;
```

`all`も`not in`も対象者がいなくなる。

```sql
with t1 as(
select cuid from recommend2 where item = 'B' and
cuid in (select cuid from recommend2 where item = 'A')
)
select cuid
from t1
where cuid != all(select cuid from recommend2 where item = 'C');

 cuid
------
(0 rows)

with t1 as(
select cuid from recommend2 where item = 'B' and
cuid in (select cuid from recommend2 where item = 'A')
)
select cuid
from t1
where cuid not in (select cuid from recommend2 where item = 'C');

 cuid
------
(0 rows)
```

`not exists`を使うのがベター。

```sql
with tmp as(
select cuid from recommend2 where item = 'B' and
cuid in (select cuid from recommend2 where item = 'A')
)
select cuid
from tmp as t1
where not exists (select cuid from recommend2 as t2 where item = 'C' and t1.cuid = t2.cuid);

 cuid
-------
 cuid3
(1 row)
```

もしくはやはり黙って愚直に`join`でせめる。

```sql
with list as (select distinct cuid from recommend),
itema as (select distinct cuid as cuid_a from recommend where item = 'A'),
itemb as (select distinct cuid as cuid_b from recommend where item = 'B'),
itemc as (select distinct cuid as cuid_c from recommend where item = 'C'),
ab as (
select cuid from list as r
inner join itema as a on r.cuid = a.cuid_a
inner join itemb as b on r.cuid = b.cuid_b
)
select cuid from ab as ab
left join itemc as c on ab.cuid = c.cuid_c
where c.cuid_c is null
;

 cuid
-------
 cuid3
(1 row)
```

## :closed_book: Reference

None
