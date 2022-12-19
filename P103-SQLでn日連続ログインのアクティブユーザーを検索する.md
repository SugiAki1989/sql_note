## :memo: Overview

ここでは、n 日連続でログインしたユーザーをアクティブユーザーと定義した場合に、このようなアクティブユーザーを検索するための SQL をまとめる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`lead`、`datediff`

## :pencil2: Example

ここでは、ログイン履歴が管理されているサンプルデータを用意する。このテーブルはログインするたびに履歴が蓄積されるテーブルである。

```sql
create table act5_login(
    id int,
    login_date date
);

insert into act5_login
(id, login_date)
values
('1', '2022-05-30'),
('1', '2022-06-07'),
('2', '2022-05-30'),
('2', '2022-05-31'),
('2', '2022-06-01'),
('2', '2022-06-02'),
('2', '2022-06-02'),
('2', '2022-06-03'),
('2', '2022-06-10')
;
```

ここでは 5 日連続でログインしているユーザーを「アクティブユーザー」と定義する。5 日連続を判定するために、まずはログイン履歴を必要であれば日時から日付に丸めてから、重複を削除する。この際に期間指定も必要であれば行っておく。そのテーブルを利用して、4 日ずらすことで、毎日連続でログインしていれば差分が 4 日になる。毎日連続でログインしていることで蓄積される履歴テーブルの特徴を利用する。

```sql
with tmp as (
select
    id,
    login_date,
    lead(login_date, 4) over(partition by id order by login_date asc) as date_5,
    lead(login_date, 4) over(partition by id order by login_date asc) - login_date as diff_days
from
    (select distinct * from act5_login) as t
) select * from tmp;

 id | login_date |   date_5   | diff_days
----+------------+------------+-----------
  1 | 2022-05-30 |            |
  1 | 2022-06-07 |            |
  2 | 2022-05-30 | 2022-06-03 |         4
  2 | 2022-05-31 | 2022-06-10 |        10
  2 | 2022-06-01 |            |
  2 | 2022-06-02 |            |
  2 | 2022-06-03 |            |
  2 | 2022-06-10 |            |
(8 rows)
```

あとはこの差分が 4 日のレコードを残して、`id`の一覧が必要であればこのまま重複を削除する。アクティブユーザー数を集計したければ、グループ化して集計する。

```sql
with tmp as (
select
    id,
    login_date,
    lead(login_date, 4) over(partition by id order by login_date asc) as date_5,
    lead(login_date, 4) over(partition by id order by login_date asc) - login_date as diff_days
from
    (select distinct * from act5_login) as t
)
select distinct
    id
from
    tmp
where
    diff_days = 4
;

 id
----
  2
(1 row)
```

アクティブユーザーの定義に合わせて日数を変更する。他にも似たような例として、初日登録の次の日ログインしたユーザーの割合などは下記のようにかける。

```sql
with tmp as
(select id,
login_date,
 min(login_date) over(partition by id) as login_date,
 case when login_date- min(login_date) over(partition by id) = 1 then 1
 else 0
 end as sign
 from act5_login)
select
    sum(tmp.sign) as denominator,
    count(distinct tmp.id) as numerator,
    sum(tmp.sign)::real/count(distinct tmp.id) as ratio
from tmp
;

 denominator | numerator | ratio
-------------+-----------+-------
           1 |         2 |   0.5
(1 row)
```

## :closed_book: Reference

None
