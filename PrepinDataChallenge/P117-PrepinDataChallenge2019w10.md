## :memo: Overview

Preppin' Data challenge の「2019: Week 10」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/04/2019-week-10.html)
- [Answer](https://preppindata.blogspot.com/2019/04/2019-week-10-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。マーケティングのメーリングリストがあって、メルマガの購読解除が記録されている。購読解除=顧客が興味を示さないことによる収益減を計算したい、というのがお題。

```sql
prepindata=> select * from p2019w10t1 limit 10;
          email           | liquid | bar | sign_up_date
--------------------------+--------+-----+--------------
 dmalone1@tumblr.com      |      1 |   0 | 14/01/2016
 dsworder6@is.gd          |      0 |   1 | 11/12/2016
 bandresen8@rakuten.co.jp |      1 |   1 | 07/11/2016
 cmiskin9@dion.ne.jp      |      1 |   1 | 30/07/2017
 tmenguyb@people.com.cn   |      0 |   1 | 30/10/2016
 ebaggaleyc@earthlink.net |      1 |   0 | 20/05/2018
 esooleyd@pcworld.com     |      1 |   0 | 03/12/2018
 kfoucarde@tinyurl.com    |      1 |   1 | 27/08/2017
 grichardoni@google.fr    |      1 |   0 | 24/12/2016
 uimpyk@marriott.com      |      1 |   1 | 07/03/2018
(10 rows)

prepindata=> select * from p2019w10t2 limit 10;
 first_name | last_name |    date
------------+-----------+------------
 Donielle   | Malone    | 23.04.2018
 Dorry      | Sworder   | 26.12.2018
 Benedict   | Andresen  | 24.12.2018
 Cornell    | Miskin    | 10.12.2018
 Theresita  | Menguy    | 28.08.2018
 Ethan      | Baggaley  | 14.12.2018
 Efren      | Sooley    | 28.04.2018
 Kara       | Foucard   | 15.06.2018
 Gherardo   | Richardon | 16.11.2018
 Ursula     | Impy      | 25.04.2018
(10 rows)

prepindata=> select * from p2019w10t3 limit 10;
          email           | liquid_sales_to_date | bar_sales_to_date
--------------------------+----------------------+-------------------
 dmalone1@tumblr.com      |                 9380 |              3927
 dsworder6@is.gd          |                 8731 |              4300
 bandresen8@rakuten.co.jp |                 9655 |               611
 cmiskin9@dion.ne.jp      |                 8371 |              1814
 tmenguyb@people.com.cn   |                  567 |              1300
 ebaggaleyc@earthlink.net |                 8409 |              1590
 esooleyd@pcworld.com     |                 6248 |              4716
 kfoucarde@tinyurl.com    |                 1965 |              4768
 grichardoni@google.fr    |                  708 |                14
 uimpyk@marriott.com      |                 2233 |              1801
(10 rows)
```

まずはテーブルのキーが使えないので、これを作りだす。そして、購読日、解除日を使ってステータスを判定する。

```sql
with unsub as (
select
    -- [Dora|Phipard-Shears]はハイフンがあるとひもづかない
    regexp_replace(lower(left(first_name, 1) || last_name), '[[:punct:]]', '', 'g') as join_key,
    first_name,
    last_name,
    to_date(date, 'DD.MM.YYYY') as unsubscribe_date
from
    p2019w10t2
), mlist as (
select
    email,
    liquid,
    bar,
    regexp_replace(
    left(split_part(email, '@', 1), char_length(split_part(email, '@', 1))-1),
    '[[:digit:]]', '', 'g'
    ) as join_key,
    to_date(sign_up_date, 'DD.MM.YYYY') as sign_up_date
from
    p2019w10t1
), tmp1 as (
select
    t1.email,
    t1.liquid,
    t1.bar,
    t1.join_key,
    t1.sign_up_date,
    t2.first_name,
    t2.last_name,
    t2.unsubscribe_date,
    case
        when t2.unsubscribe_date is null then 'Subscribed'
        when t2.unsubscribe_date < t1.sign_up_date then 'Resubscribed'
        else 'Unsubscribed' end as status
from
    mlist as t1
left join
    unsub as t2
on
    t1.join_key = t2.join_key
)
select * from tmp1;

                 email                 | liquid | bar |    join_key    | sign_up_date | first_name |   last_name    | unsubscribe_date |    status
---------------------------------------+--------+-----+----------------+--------------+------------+----------------+------------------+--------------
 ggrogono5d@reference.com              |      1 |   1 | ggrogono       | 2018-08-14   | Gorden     | Grogono        | 2018-11-28       | Unsubscribed
 rcornelissen5h@wix.com                |      0 |   1 | rcornelissen   | 2017-03-01   | Reg        | Cornelissen    | 2019-03-04       | Unsubscribed
 cguiduzzi5i@plala.or.jp               |      1 |   1 | cguiduzzi      | 2018-06-19   | Colleen    | Guiduzzi       | 2018-05-24       | Resubscribed
 bmatisoff5j@flickr.com                |      1 |   0 | bmatisoff      | 2018-12-20   | Bard       | Matisoff       | 2018-10-04       | Resubscribed
 pvenart0@posterous.com                |      1 |   0 | pvenart        | 2018-08-16   | (null)     | (null)         | (null)           | Subscribed
 erosedale2@google.com.hk              |      0 |   1 | erosedale      | 2016-01-13   | (null)     | (null)         | (null)           | Subscribed

```

そして、解除した人たちは、解除するまでに何ヶ月かかっているのかを計算する。

```sql
with unsub as (
select
    -- [Dora|Phipard-Shears]はハイフンがあるとひもづかない
    regexp_replace(lower(left(first_name, 1) || last_name), '[[:punct:]]', '', 'g') as join_key,
    first_name,
    last_name,
    to_date(date, 'DD.MM.YYYY') as unsubscribe_date
from
    p2019w10t2
), mlist as (
select
    email,
    liquid,
    bar,
    regexp_replace(
    left(split_part(email, '@', 1), char_length(split_part(email, '@', 1))-1),
    '[[:digit:]]', '', 'g'
    ) as join_key,
    to_date(sign_up_date, 'DD.MM.YYYY') as sign_up_date
from
    p2019w10t1
), tmp1 as (
select
    t1.email,
    t1.liquid,
    t1.bar,
    t1.join_key,
    t1.sign_up_date,
    t2.first_name,
    t2.last_name,
    t2.unsubscribe_date,
    case
        when t2.unsubscribe_date is null then 'Subscribed'
        when t2.unsubscribe_date < t1.sign_up_date then 'Resubscribed'
        else 'Unsubscribed' end as status
from
    mlist as t1
left join
    unsub as t2
on
    t1.join_key = t2.join_key
), tmp2 as (
select
    email,
    liquid,
    bar,
    join_key,
    sign_up_date,
    first_name,
    last_name,
    unsubscribe_date,
    status,
    case when status = 'Unsubscribed'
    then extract(year from age(unsubscribe_date, sign_up_date))*12 + extract(month from age(unsubscribe_date, sign_up_date))
    else null end as months_before_unsubscribed
from
    tmp1
)
select * from tmp2 limit 10;

          email           | liquid | bar |  join_key  | sign_up_date | first_name | last_name | unsubscribe_date |    status    | months_before_unsubscribed
--------------------------+--------+-----+------------+--------------+------------+-----------+------------------+--------------+----------------------------
 dmalone1@tumblr.com      |      1 |   0 | dmalone    | 2016-01-14   | Donielle   | Malone    | 2018-04-23       | Unsubscribed |                         27
 dsworder6@is.gd          |      0 |   1 | dsworder   | 2016-12-11   | Dorry      | Sworder   | 2018-12-26       | Unsubscribed |                         24
 bandresen8@rakuten.co.jp |      1 |   1 | bandresen  | 2016-11-07   | Benedict   | Andresen  | 2018-12-24       | Unsubscribed |                         25
 cmiskin9@dion.ne.jp      |      1 |   1 | cmiskin    | 2017-07-30   | Cornell    | Miskin    | 2018-12-10       | Unsubscribed |                         16
 tmenguyb@people.com.cn   |      0 |   1 | tmenguy    | 2016-10-30   | Theresita  | Menguy    | 2018-08-28       | Unsubscribed |                         21
 ebaggaleyc@earthlink.net |      1 |   0 | ebaggaley  | 2018-05-20   | Ethan      | Baggaley  | 2018-12-14       | Unsubscribed |                          6
 esooleyd@pcworld.com     |      1 |   0 | esooley    | 2018-12-03   | Efren      | Sooley    | 2018-04-28       | Resubscribed |                     (null)
 kfoucarde@tinyurl.com    |      1 |   1 | kfoucard   | 2017-08-27   | Kara       | Foucard   | 2018-06-15       | Unsubscribed |                          9
 grichardoni@google.fr    |      1 |   0 | grichardon | 2016-12-24   | Gherardo   | Richardon | 2018-11-16       | Unsubscribed |                         22
 uimpyk@marriott.com      |      1 |   1 | uimpy      | 2018-03-07   | Ursula     | Impy      | 2018-04-25       | Unsubscribed |                          1
(10 rows)
```

月数をグループにまとめて、

```sql
with unsub as (
select
    -- [Dora|Phipard-Shears]はハイフンがあるとひもづかない
    regexp_replace(lower(left(first_name, 1) || last_name), '[[:punct:]]', '', 'g') as join_key,
    first_name,
    last_name,
    to_date(date, 'DD.MM.YYYY') as unsubscribe_date
from
    p2019w10t2
), mlist as (
select
    email,
    liquid,
    bar,
    regexp_replace(
    left(split_part(email, '@', 1), char_length(split_part(email, '@', 1))-1),
    '[[:digit:]]', '', 'g'
    ) as join_key,
    to_date(sign_up_date, 'DD.MM.YYYY') as sign_up_date
from
    p2019w10t1
), tmp1 as (
select
    t1.email,
    t1.liquid,
    t1.bar,
    t1.join_key,
    t1.sign_up_date,
    t2.first_name,
    t2.last_name,
    t2.unsubscribe_date,
    case
        when t2.unsubscribe_date is null then 'Subscribed'
        when t2.unsubscribe_date < t1.sign_up_date then 'Resubscribed'
        else 'Unsubscribed' end as status
from
    mlist as t1
left join
    unsub as t2
on
    t1.join_key = t2.join_key
), tmp2 as (
select
    email,
    liquid,
    bar,
    join_key,
    sign_up_date,
    first_name,
    last_name,
    unsubscribe_date,
    status,
    case when status = 'Unsubscribed'
    then extract(year from age(unsubscribe_date, sign_up_date))*12 + extract(month from age(unsubscribe_date, sign_up_date))
    else null end as months_before_unsubscribed
from
    tmp1
), tmp3 as (
select
    email,
    liquid,
    bar,
    join_key,
    sign_up_date,
    first_name,
    last_name,
    unsubscribe_date,
    months_before_unsubscribed,
    case
    when months_before_unsubscribed is null then '---'
    when months_before_unsubscribed  < 3  then '0-2'
    when months_before_unsubscribed  < 6  then '3-5'
    when months_before_unsubscribed  < 12 then '6-11'
    when months_before_unsubscribed  < 24 then '12-23'
    else '24+' end as months_before_unsubscribed_group
from
    tmp2
)
select * from tmp3 limit 10;

          email           | liquid | bar |  join_key  | sign_up_date | first_name | last_name | unsubscribe_date | months_before_unsubscribed | months_before_unsubscribed_group
--------------------------+--------+-----+------------+--------------+------------+-----------+------------------+----------------------------+----------------------------------
 dmalone1@tumblr.com      |      1 |   0 | dmalone    | 2016-01-14   | Donielle   | Malone    | 2018-04-23       |                         27 | 24+
 dsworder6@is.gd          |      0 |   1 | dsworder   | 2016-12-11   | Dorry      | Sworder   | 2018-12-26       |                         24 | 24+
 bandresen8@rakuten.co.jp |      1 |   1 | bandresen  | 2016-11-07   | Benedict   | Andresen  | 2018-12-24       |                         25 | 24+
 cmiskin9@dion.ne.jp      |      1 |   1 | cmiskin    | 2017-07-30   | Cornell    | Miskin    | 2018-12-10       |                         16 | 12-23
 tmenguyb@people.com.cn   |      0 |   1 | tmenguy    | 2016-10-30   | Theresita  | Menguy    | 2018-08-28       |                         21 | 12-23
 ebaggaleyc@earthlink.net |      1 |   0 | ebaggaley  | 2018-05-20   | Ethan      | Baggaley  | 2018-12-14       |                          6 | 6-11
 esooleyd@pcworld.com     |      1 |   0 | esooley    | 2018-12-03   | Efren      | Sooley    | 2018-04-28       |                     (null) | ---
 kfoucarde@tinyurl.com    |      1 |   1 | kfoucard   | 2017-08-27   | Kara       | Foucard   | 2018-06-15       |                          9 | 6-11
 grichardoni@google.fr    |      1 |   0 | grichardon | 2016-12-24   | Gherardo   | Richardon | 2018-11-16       |                         22 | 12-23
 uimpyk@marriott.com      |      1 |   1 | uimpy      | 2018-03-07   | Ursula     | Impy      | 2018-04-25       |                          1 | 0-2
(10 rows)
```

LTV が計算されているテーブルと紐付けて、集計して完了。

```sql
with unsub as (
select
    -- [Dora|Phipard-Shears]はハイフンがあるとひもづかない
    regexp_replace(lower(left(first_name, 1) || last_name), '[[:punct:]]', '', 'g') as join_key,
    first_name,
    last_name,
    to_date(date, 'DD.MM.YYYY') as unsubscribe_date
from
    p2019w10t2
), mlist as (
select
    email,
    liquid,
    bar,
    regexp_replace(
    left(split_part(email, '@', 1), char_length(split_part(email, '@', 1))-1),
    '[[:digit:]]', '', 'g'
    ) as join_key,
    to_date(sign_up_date, 'DD.MM.YYYY') as sign_up_date
from
    p2019w10t1
), tmp1 as (
select
    t1.email,
    t1.liquid,
    t1.bar,
    t1.join_key,
    t1.sign_up_date,
    t2.first_name,
    t2.last_name,
    t2.unsubscribe_date,
    case
        when t2.unsubscribe_date is null then 'Subscribed'
        when t2.unsubscribe_date < t1.sign_up_date then 'Resubscribed'
        else 'Unsubscribed' end as status
from
    mlist as t1
left join
    unsub as t2
on
    t1.join_key = t2.join_key
), tmp2 as (
select
    email,
    liquid,
    bar,
    join_key,
    sign_up_date,
    first_name,
    last_name,
    unsubscribe_date,
    status,
    case when status = 'Unsubscribed'
    then extract(year from age(unsubscribe_date, sign_up_date))*12 + extract(month from age(unsubscribe_date, sign_up_date))
    else null end as months_before_unsubscribed
from
    tmp1
), tmp3 as (
select
    t1.email,
    t1.liquid,
    t1.bar,
    t1.join_key,
    t1.sign_up_date,
    t1.first_name,
    t1.last_name,
    t1.unsubscribe_date,
    t1.months_before_unsubscribed,
    case
    when t1.months_before_unsubscribed is null then '---'
    when t1.months_before_unsubscribed  < 3  then '0-2'
    when t1.months_before_unsubscribed  < 6  then '3-5'
    when t1.months_before_unsubscribed  < 12 then '6-11'
    when t1.months_before_unsubscribed  < 24 then '12-23'
    else '24+' end as months_before_unsubscribed_group,
    t2.liquid_sales_to_date,
    t2.bar_sales_to_date
from
    tmp2 as t1
inner join
    p2019w10t3 as t2
on
    t1.email = t2.email
)
select * from tmp3 limit 10;

          email           | liquid | bar |  join_key  | sign_up_date | first_name | last_name | unsubscribe_date | months_before_unsubscribed | months_before_unsubscribed_group | liquid_sales_to_date | bar_sales_to_date
--------------------------+--------+-----+------------+--------------+------------+-----------+------------------+----------------------------+----------------------------------+----------------------+-------------------
 dmalone1@tumblr.com      |      1 |   0 | dmalone    | 2016-01-14   | Donielle   | Malone    | 2018-04-23       |                         27 | 24+                              |                 9380 |              3927
 dsworder6@is.gd          |      0 |   1 | dsworder   | 2016-12-11   | Dorry      | Sworder   | 2018-12-26       |                         24 | 24+                              |                 8731 |              4300
 bandresen8@rakuten.co.jp |      1 |   1 | bandresen  | 2016-11-07   | Benedict   | Andresen  | 2018-12-24       |                         25 | 24+                              |                 9655 |               611
 cmiskin9@dion.ne.jp      |      1 |   1 | cmiskin    | 2017-07-30   | Cornell    | Miskin    | 2018-12-10       |                         16 | 12-23                            |                 8371 |              1814
 tmenguyb@people.com.cn   |      0 |   1 | tmenguy    | 2016-10-30   | Theresita  | Menguy    | 2018-08-28       |                         21 | 12-23                            |                  567 |              1300
 ebaggaleyc@earthlink.net |      1 |   0 | ebaggaley  | 2018-05-20   | Ethan      | Baggaley  | 2018-12-14       |                          6 | 6-11                             |                 8409 |              1590
 esooleyd@pcworld.com     |      1 |   0 | esooley    | 2018-12-03   | Efren      | Sooley    | 2018-04-28       |                     (null) | ---                              |                 6248 |              4716
 kfoucarde@tinyurl.com    |      1 |   1 | kfoucard   | 2017-08-27   | Kara       | Foucard   | 2018-06-15       |                          9 | 6-11                             |                 1965 |              4768
 grichardoni@google.fr    |      1 |   0 | grichardon | 2016-12-24   | Gherardo   | Richardon | 2018-11-16       |                         22 | 12-23                            |                  708 |                14
 uimpyk@marriott.com      |      1 |   1 | uimpy      | 2018-03-07   | Ursula     | Impy      | 2018-04-25       |                          1 | 0-2                              |                 2233 |              1801
(10 rows)

```

最終的に集計した結果がこちら。

```sql
with unsub as (
select
    -- [Dora|Phipard-Shears]はハイフンがあるとひもづかない
    regexp_replace(lower(left(first_name, 1) || last_name), '[[:punct:]]', '', 'g') as join_key,
    first_name,
    last_name,
    to_date(date, 'DD.MM.YYYY') as unsubscribe_date
from
    p2019w10t2
), mlist as (
select
    email,
    liquid,
    bar,
    regexp_replace(
    left(split_part(email, '@', 1), char_length(split_part(email, '@', 1))-1),
    '[[:digit:]]', '', 'g'
    ) as join_key,
    to_date(sign_up_date, 'DD.MM.YYYY') as sign_up_date
from
    p2019w10t1
), tmp1 as (
select
    t1.email,
    t1.liquid,
    t1.bar,
    t1.join_key,
    t1.sign_up_date,
    t2.first_name,
    t2.last_name,
    t2.unsubscribe_date,
    case
        when t2.unsubscribe_date is null then 'Subscribed'
        when t2.unsubscribe_date < t1.sign_up_date then 'Resubscribed'
        else 'Unsubscribed' end as status
from
    mlist as t1
left join
    unsub as t2
on
    t1.join_key = t2.join_key
), tmp2 as (
select
    email,
    liquid,
    bar,
    join_key,
    sign_up_date,
    first_name,
    last_name,
    unsubscribe_date,
    status,
    case when status = 'Unsubscribed'
    then extract(year from age(unsubscribe_date, sign_up_date))*12 + extract(month from age(unsubscribe_date, sign_up_date))
    else null end as months_before_unsubscribed
from
    tmp1
), tmp3 as (
select
    t1.email,
    t1.liquid,
    t1.bar,
    t1.join_key,
    t1.sign_up_date,
    t1.first_name,
    t1.last_name,
    t1.unsubscribe_date,
    t1.status,
    t1.months_before_unsubscribed,
    case
    when t1.months_before_unsubscribed is null then '---'
    when t1.months_before_unsubscribed  < 3  then '0-2'
    when t1.months_before_unsubscribed  < 6  then '3-5'
    when t1.months_before_unsubscribed  < 12 then '6-11'
    when t1.months_before_unsubscribed  < 24 then '12-23'
    else '24+' end as months_before_unsubscribed_group,
    t2.liquid_sales_to_date,
    t2.bar_sales_to_date
from
    tmp2 as t1
inner join
    p2019w10t3 as t2
on
    t1.email = t2.email
)
select
    months_before_unsubscribed_group,
    status,
    liquid,
    bar,
    count(email) as cnt_email,
    sum(liquid_sales_to_date) as sum_liquid_sales_to_date,
    sum(bar_sales_to_date) as sum_bar_sales_to_date
from
    tmp3
group by
    months_before_unsubscribed_group,
    status,
    liquid,
    bar
order by
    months_before_unsubscribed_group asc,
    status asc
;

 months_before_unsubscribed_group |    status    | liquid | bar | cnt_email | sum_liquid_sales_to_date | sum_bar_sales_to_date
----------------------------------+--------------+--------+-----+-----------+--------------------------+-----------------------
 ---                              | Resubscribed |      0 |   1 |         1 |                     7244 |                  2690
 ---                              | Resubscribed |      1 |   0 |         4 |                    18081 |                 15290
 ---                              | Resubscribed |      1 |   1 |         1 |                     8056 |                  3454
 ---                              | Subscribed   |      0 |   0 |         1 |                     6761 |                   201
 ---                              | Subscribed   |      0 |   1 |        34 |                   144420 |                 81908
 ---                              | Subscribed   |      1 |   0 |        33 |                   162304 |                 82356
 ---                              | Subscribed   |      1 |   1 |        27 |                   151519 |                 64693
 0-2                              | Unsubscribed |      0 |   1 |         2 |                     4202 |                  2613
 0-2                              | Unsubscribed |      1 |   0 |         3 |                    18419 |                  5865
 0-2                              | Unsubscribed |      1 |   1 |         2 |                     3852 |                  3907
 12-23                            | Unsubscribed |      0 |   0 |         1 |                     9618 |                  3264
 12-23                            | Unsubscribed |      0 |   1 |        14 |                    63407 |                 35087
 12-23                            | Unsubscribed |      1 |   0 |        15 |                    81107 |                 38064
 12-23                            | Unsubscribed |      1 |   1 |        11 |                    59902 |                 30024
 24+                              | Unsubscribed |      0 |   1 |        10 |                    51113 |                 24934
 24+                              | Unsubscribed |      1 |   0 |        12 |                    53472 |                 31870
 24+                              | Unsubscribed |      1 |   1 |         5 |                    31269 |                  5895
 3-5                              | Unsubscribed |      0 |   1 |         2 |                     3927 |                  6190
 3-5                              | Unsubscribed |      1 |   0 |         3 |                    19665 |                 11449
 3-5                              | Unsubscribed |      1 |   1 |         5 |                    25420 |                  9041
 6-11                             | Unsubscribed |      0 |   1 |         7 |                    37817 |                 19359
 6-11                             | Unsubscribed |      1 |   0 |         4 |                    33594 |                 12114
 6-11                             | Unsubscribed |      1 |   1 |         3 |                    15484 |                 11854
(23 rows)

```

## :closed_book: Reference

None
