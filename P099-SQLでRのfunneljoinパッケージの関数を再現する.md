## :memo: Overview

ここでは、R の`funneljoin`パッケージの一部の関数を SQL で再現する。パッケージの目的は、行動ファネル分析を容易にすること。たとえば、ページにアクセスしてから登録する人を見つけたいなど。

- [funneljoin](https://github.com/robinsones/funneljoin)
- [Introducing the funneljoin package](https://hookedondata.org/introducing-the-funneljoin-package/)

`join`の不等号結合を工夫すれば再現できる。以前、[R でまとめた記事](https://rlang.hatenablog.jp/entry/2020/05/25/224324)を SQL で再現したものになる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`funnel join`, `inequality join`

## :pencil2: Example

まずはパッケージに含まれるサンプルデータを用意する。

```sql
create table landed(user_id_x int, landed_at date);
insert into landed(user_id_x, landed_at)
values
    ('1','2018-07-01'),
    ('2','2018-07-01'),
    ('3','2018-07-02'),
    ('4','2018-07-01'),
    ('4','2018-07-04'),
    ('5','2018-07-10'),
    ('5','2018-07-12'),
    ('6','2018-07-07'),
    ('6','2018-07-08');

create table registered(user_id_y int, registered_at date);
insert into registered(user_id_y, registered_at)
values
    ('1','2018-07-02'),
    ('3','2018-07-02'),
    ('4','2018-06-10'),
    ('4','2018-07-02'),
    ('5','2018-07-11'),
    ('6','2018-07-10'),
    ('6','2018-07-11'),
    ('7','2018-07-07');

# Rの準備
library(tidyverse)
library(funneljoin)

landed <- landed %>% rename(landed_at = timestamp, user_id_x = user_id)
registered <- registered %>% rename(registered_at = timestamp, user_id_y = user_id)
```

基本的には、`after_left_join()`の`type`を変更しながら再現する。SQL で実行した後は、R での結果を併記する。

`first-first`は、参加前の各ユーザーの最も古い`x`と`y`を紐付ける。ただ、最初の`x`の前に`y`がある場合は除く。例えば、実験があったとして、最初の「参加」以降に、最初に「登録」した時を取得したい場合はこのタイプを利用する。この場合、登録して、実験に参加し、再び登録した場合は紐付かない、というもの。

![ff](https://user-images.githubusercontent.com/65038325/190848721-beace0f9-4894-41ce-a0e7-275cc94413b2.png)

```sql
with landed_min as (
select user_id_x, min(landed_at) as landed_at_min
from landed
group by user_id_x
order by user_id_x asc
), registered_min as (
select user_id_y, min(registered_at) as registered_at_min
from registered
group by user_id_y
)
select
    l.user_id_x, l.landed_at_min,
    r.registered_at_min
from landed_min as l
left join registered_min as r
on l.user_id_x = r.user_id_y and l.landed_at_min <= r.registered_at_min
order by l.user_id_x asc
;

 user_id_x | landed_at_min | registered_at_min
-----------+---------------+-------------------
         1 | 2018-07-01    | 2018-07-02
         2 | 2018-07-01    |
         3 | 2018-07-02    | 2018-07-02
         4 | 2018-07-01    |
         5 | 2018-07-10    | 2018-07-11
         6 | 2018-07-07    | 2018-07-10
(6 rows)


# first-first
landed %>%
  after_left_join(registered,
                  by_user = c("user_id_x" = "user_id_y"),
                  by_time = c("landed_at" = "registered_at"),
                  type = "first-first") %>%
  arrange(user_id_x)

# A tibble: 6 × 3
  user_id_x landed_at  registered_at
      <dbl> <date>     <date>
1         1 2018-07-01 2018-07-02
2         2 2018-07-01 NA
3         3 2018-07-02 2018-07-02
4         4 2018-07-01 NA
5         5 2018-07-10 2018-07-11
6         6 2018-07-07 2018-07-10
```

`first-firstafter`は参加前の各ユーザーの最も古い`x`と`y`を紐付ける。ただ、最初の`x`の前に`y`がある場合でも除かない。例えば、実験があったとして、最初の「参加」以降に、最初に「登録」した時を取得したい場合はこのタイプを利用する、というもの。

![ffa](https://user-images.githubusercontent.com/65038325/190848727-69ec110d-8c50-4b4e-85cf-c209dd0165fa.png)

```sql
with landed_min as (
select user_id_x, min(landed_at) as landed_at_min
from landed
group by user_id_x
order by user_id_x asc
)
select
    l.user_id_x, l.landed_at_min,
    min(r.registered_at) as registered_at_min
from landed_min as l
left join registered as r
on l.user_id_x = r.user_id_y and l.landed_at_min <= r.registered_at
group by user_id_x, landed_at_min
order by l.user_id_x asc
;

 user_id_x | landed_at_min | registered_at_min
-----------+---------------+-------------------
         1 | 2018-07-01    | 2018-07-02
         2 | 2018-07-01    |
         3 | 2018-07-02    | 2018-07-02
         4 | 2018-07-01    | 2018-07-02
         5 | 2018-07-10    | 2018-07-11
         6 | 2018-07-07    | 2018-07-10
(6 rows)

landed %>%
  after_left_join(registered,
                  by_user = c("user_id_x" = "user_id_y"),
                  by_time = c("landed_at" = "registered_at"),
                  type = "first-firstafter") %>%
  arrange(user_id_x)
# A tibble: 6 × 3
  user_id_x landed_at  registered_at
      <dbl> <date>     <date>
1         1 2018-07-01 2018-07-02
2         2 2018-07-01 NA
3         3 2018-07-02 2018-07-02
4         4 2018-07-01 2018-07-02
5         5 2018-07-10 2018-07-11
6         6 2018-07-07 2018-07-10
```

`lastbefore-firstafter`は、例えば、ラストクリック型の広告アトリビューションなんかで役に立つ。最初の CV の前で、最後にクリックした広告が必要なとき、というもの。

![lbfa](https://user-images.githubusercontent.com/65038325/190848729-e9378dea-3a40-41df-a2af-fa1cfb76683e.png)

すこしややこしいが、まず`x`以上の`y`を紐付けて、古いイベントを取得する。修正した`y`以下の`x`の新しいイベントを取得する。それを`x`に紐付けることで再現している。あまり自信がない。

```sql
with registered_mod as (
select user_id_y, min(registered_at) as registered_at
from landed as l
inner join registered as r
on l.user_id_x = r.user_id_y
    and l.landed_at <= r.registered_at
group by user_id_y
)
-- select * from registered_mod;
--  user_id_y | registered_at
-- -----------+---------------
--          1 | 2018-07-02
--          3 | 2018-07-02
--          4 | 2018-07-02
--          5 | 2018-07-11
--          6 | 2018-07-10
-- (5 rows)
, landed_mod as (
select l.user_id_x, max(l.landed_at) as landed_at
from registered_mod as rm
inner join landed as l
on rm.user_id_y = l.user_id_x
    and l.landed_at <= rm.registered_at
group by l.user_id_x
)
-- select * from landed_mod;
--  user_id_x | landed_at
-- -----------+------------
--          1 | 2018-07-01
--          3 | 2018-07-02
--          4 | 2018-07-01
--          5 | 2018-07-10
--          6 | 2018-07-08
-- (5 rows)
, landed_registered_mod as (
select user_id_x as user_id_key, landed_at as landed_at_key, registered_at
from landed_mod as lm
inner join registered_mod as rm
on rm.user_id_y = lm.user_id_x
    and lm.landed_at <= rm.registered_at
)
-- select * from landed_registered_mod;
--  user_id_key | landed_at_key | registered_at
-- -------------+---------------+---------------
--            1 | 2018-07-01    | 2018-07-02
--            3 | 2018-07-02    | 2018-07-02
--            4 | 2018-07-01    | 2018-07-02
--            5 | 2018-07-10    | 2018-07-11
--            6 | 2018-07-08    | 2018-07-10
-- (5 rows)
select user_id_x, landed_at, lmr.registered_at
from landed as l
left join landed_registered_mod as lmr
on l.user_id_x = lmr.user_id_key
    and l.landed_at = lmr.landed_at_key
order by user_id_x
;

 user_id_x | landed_at  | registered_at
-----------+------------+---------------
         1 | 2018-07-01 | 2018-07-02
         2 | 2018-07-01 |
         3 | 2018-07-02 | 2018-07-02
         4 | 2018-07-01 | 2018-07-02
         4 | 2018-07-04 |
         5 | 2018-07-10 | 2018-07-11
         5 | 2018-07-12 |
         6 | 2018-07-07 |
         6 | 2018-07-08 | 2018-07-10
(9 rows)

# lastbefore-firstafter
landed %>%
  after_left_join(registered,
                  by_user = c("user_id_x" = "user_id_y"),
                  by_time = c("landed_at" = "registered_at"),
                  type = "lastbefore-firstafter") %>%
  arrange(user_id_x)

# A tibble: 9 × 3
  user_id_x landed_at  registered_at
      <dbl> <date>     <date>
1         1 2018-07-01 2018-07-02
2         2 2018-07-01 NA
3         3 2018-07-02 2018-07-02
4         4 2018-07-01 2018-07-02
5         4 2018-07-04 NA
6         5 2018-07-10 2018-07-11
7         5 2018-07-12 NA
8         6 2018-07-07 NA
9         6 2018-07-08 2018-07-10
```

`any-firstafter`は、すべての`x`以降で最初の`y`が続くものを取る。例えば、誰かがホームページを訪問した回数と、その後に訪問した最初の製品ページをすべて取得したい場合などに役立つ、というもの。

![afa](https://user-images.githubusercontent.com/65038325/190848731-27ce2ad2-84ac-4bdb-bb0a-c133f79f915d.png)

```sql
select l.user_id_x, l.landed_at, min(r.registered_at) as registered_at
from landed as l
left join registered as r
on l.user_id_x = r.user_id_y
    and l.landed_at <= r.registered_at
group by l.user_id_x, l.landed_at
order by user_id_x, landed_at
;

 user_id_x | landed_at  | registered_at
-----------+------------+---------------
         1 | 2018-07-01 | 2018-07-02
         2 | 2018-07-01 |
         3 | 2018-07-02 | 2018-07-02
         4 | 2018-07-01 | 2018-07-02
         4 | 2018-07-04 |
         5 | 2018-07-10 | 2018-07-11
         5 | 2018-07-12 |
         6 | 2018-07-07 | 2018-07-10
         6 | 2018-07-08 | 2018-07-10
(9 rows)

# any-firstafter
landed %>%
  after_left_join(registered,
                  by_user = c("user_id_x" = "user_id_y"),
                  by_time = c("landed_at" = "registered_at"),
                  type = "any-firstafter") %>%
  arrange(user_id_x)

# A tibble: 9 × 3
  user_id_x landed_at  registered_at
      <dbl> <date>     <date>
1         1 2018-07-01 2018-07-02
2         2 2018-07-01 NA
3         3 2018-07-02 2018-07-02
4         4 2018-07-01 2018-07-02
5         4 2018-07-04 NA
6         5 2018-07-10 2018-07-11
7         5 2018-07-12 NA
8         6 2018-07-07 2018-07-10
9         6 2018-07-08 2018-07-10
```

`any-any`はすべての`x`以降ですべての`y`が続くものを取る。例えば、誰かがホームページを訪問した回数と、その後に見たすべての製品ページを表示する、というもの。

![aa](https://user-images.githubusercontent.com/65038325/190848732-4df1f42c-2f46-4935-bf89-87691680111e.png)

```sql
select l.user_id_x, l.landed_at, r.registered_at
from landed as l
left join registered as r
on l.user_id_x = r.user_id_y
    and l.landed_at <= r.registered_at
order by user_id_x, landed_at
;

 user_id_x | landed_at  | registered_at
-----------+------------+---------------
         1 | 2018-07-01 | 2018-07-02
         2 | 2018-07-01 |
         3 | 2018-07-02 | 2018-07-02
         4 | 2018-07-01 | 2018-07-02
         4 | 2018-07-04 |
         5 | 2018-07-10 | 2018-07-11
         5 | 2018-07-12 |
         6 | 2018-07-07 | 2018-07-10
         6 | 2018-07-07 | 2018-07-11
         6 | 2018-07-08 | 2018-07-10
         6 | 2018-07-08 | 2018-07-11
(11 rows)

# any-any
landed %>%
  after_left_join(registered,
                  by_user = c("user_id_x" = "user_id_y"),
                  by_time = c("landed_at" = "registered_at"),
                  type = "any-any") %>%
  arrange(user_id_x)

# A tibble: 11 × 3
   user_id_x landed_at  registered_at
       <dbl> <date>     <date>
 1         1 2018-07-01 2018-07-02
 2         2 2018-07-01 NA
 3         3 2018-07-02 2018-07-02
 4         4 2018-07-01 2018-07-02
 5         4 2018-07-04 NA
 6         5 2018-07-10 2018-07-11
 7         5 2018-07-12 NA
 8         6 2018-07-07 2018-07-10
 9         6 2018-07-07 2018-07-11
10         6 2018-07-08 2018-07-10
11         6 2018-07-08 2018-07-11
```

## :closed_book: Reference

- [Effective SQL](https://www.shoeisha.co.jp/book/detail/9784798153995)
