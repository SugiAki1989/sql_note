## :memo: Overview

今回は再帰 SQL(Recursive SQL) を実践的な利用方法をまとめる。recursive CTE と呼ぶほうが良いかもしれないが、そこらへんは一旦おいておく。

ここで扱うケースは、下記の teratail に記載されていたもので、実践でありえそうなケースだったので参考にさせていただいた。

- [SQL:再帰的に最新の会員番号をもとめて、一つの会員として売上を集計したい。](https://teratail.com/questions/cae1gknhvu5uiq)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`recursive`, `with`

## :pencil2: Example

まずはサイトに記載されているサンプルデータを用意する。

```sql
create table cardmembers (
    card_no integer,
    prev_card_no integer,
    is_expired integer);

insert into cardmembers values (5,4,null);
insert into cardmembers values (4,3,1);
insert into cardmembers values (3,2,1);
insert into cardmembers values (2,1,1);
insert into cardmembers values (1,null,1);
insert into cardmembers values (12,11,null);
insert into cardmembers values (11,null,1);

create table sales(
    card_no integer not null,
    sales_amount integer not null,
    sales_date varchar not null,
    primary key (card_no,sales_amount,sales_date));

insert into sales values (5,300,'20220103');
insert into sales values (4,500,'20220102');
insert into sales values (3,200,'20220101');
insert into sales values (2,100,'20211231');
insert into sales values (1,1000,'20221230');
insert into sales values (12,300,'20211229');
insert into sales values (11,400,'20211228');
```

`cardmembers`テーブルには 2 人の会員がいて、会員カードが切り替わるたびに、前の会員カード番号`prev_card_no`を紐付けて、最新の会員番号`card_no`が記録されていく。また、切り替え前の会員番号は、`is_expired`で有効期限切れとして管理される。

`1`は、`1`から`2`に変更し、`2`から`3`に変更し、`3`から`4`に変更し、`4`から`5`に変更したという変更履歴を持つ。

```sql
select * from cardmembers;

 card_no | prev_card_no | is_expired
---------+--------------+------------
       5 |            4 |
       4 |            3 |          1
       3 |            2 |          1
       2 |            1 |          1
       1 |              |          1
      12 |           11 |
      11 |              |          1
```

次に先程の会員カードに紐づく形で、売上履歴`sales`テーブルに記録されている。`1`に該当する人物の売上合計を計算しようとすると、`1`から`5`までで合算される必要があるが、ここで問題が発生し、このテーブルは取引発生時点の会員番号が記録されるので、`1`しか売上を計算できない。そのため、`1`から`5`まででを最新の会員番号か、最小の会員番号でグループ化のための単位を作る必要があるが、これを実現しようとすると、非常に多くの`join`が必要となり、また、もとのテーブルが更新されるたびに必要な`join`の数も変化するため、この方法では解決できても現実的ではない。アドホックな一回こっきりなケースであれば、力技でも問題ないが…。また、本来はこのような対応関係であってもサンプルテーブルのように規則的な値の対応関係は持たない。

```sql
select * from sales;

 card_no | sales_amount | sales_date
---------+--------------+------------
       5 |          300 | 20220103
       4 |          500 | 20220102
       3 |          200 | 20220101
       2 |          100 | 20211231
       1 |         1000 | 20221230
      12 |          300 | 20211229
      11 |          400 | 20211228
```

このようケースでは、再帰 SQL を使えば、現在の状態から過去の状態までを遡ることでき、多数の`join`も必要がなくなる。

```sql
with recursive recur (newest_card_no, card_no, prev_card_no) as (
    -- 非再帰クエリ
    select
        card_no,
        card_no,
        prev_card_no
    from
        cardmembers
    where
        is_expired is null

    union all
    -- 再帰クエリ
    select
        recur.newest_card_no,
        cm.card_no,
        cm.prev_card_no
    from
        recur
    join
        cardmembers as cm
    on
        recur.prev_card_no = cm.card_no
)
-- メインクエリ
select
    newest_card_no,
    sum(sales_amount) as sum_sales
from
    recur
join
    sales
on
    recur.card_no = sales.card_no
group by
    newest_card_no
order by
    newest_card_no asc
;

 newest_card_no | sum
----------------+------
              5 | 2100
             12 |  700
```

あくまでもイメージなので、実際の動きとは異なるが、この再帰 SQL の動作イメージを図にすると下記のようになる。

![RecursiveSQL](https://user-images.githubusercontent.com/65038325/182152759-29dbbcc7-69ba-402e-a10a-33184be8b7f8.png)

まず、AnchorPart と呼ばれる非再帰クエリが実行されて`recur`テーブルが作成される。この時点でこの`recur`テーブルが利用可能になるので、再帰クエリがそのテーブルを使って、`cardmembers`テーブルを`recur.prev_card_no = cm.card_no`で`join`する。この`join`の結果がワークキングテーブルとして保持され、`recur`テーブルにユニオンされて更新される。そのユニオンされて更新した`recur`テーブルを使って、再度`cardmembers`テーブルを`recur.prev_card_no = cm.card_no`で`join`する。そして、この`join`の結果がワークキングテーブルとして保持され、`recur`テーブルにユニオンされて更新される。

これを繰り返すことで、再帰的にクエリを実行していく。この例であれば、最初のカード番号の`previous_card_no`は`null`なので、これを`join`すると検索結果がなくなり、再帰が終了する。これが再帰の終了条件にあたる。そして、更新した`recur`テーブルがメインクエリの`from`から呼び出されてメインクエリの処理が実行される。

このような形で、再帰的にクエリを呼び出すことができるのが、再帰 SQL である。

## :closed_book: Reference

- [再帰的共通テーブル式の概要](https://runebook.dev/ja/docs/mariadb/recursive-common-table-expressions-overview/index)
