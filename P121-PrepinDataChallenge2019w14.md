## :memo: Overview

Preppin' Data challenge の「2019: Week 14」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/05/2019-week-14.html)
- [Answer](https://preppindata.blogspot.com/2019/05/each-week-i-normally-like-to-outline.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。カフェの売上データを分析するというお題。

```sql
select * from p2019w14t1 limit 10;

 ticketid |    date    | memberid |  desc  | price | type
----------+------------+----------+--------+-------+-------
   203508 | 2013-09-23 |   994797 | Banana |     1 | Snack
   203552 | 2013-09-23 |   990653 | Banana |     1 | Snack
   203599 | 2013-09-23 |   (null) | Banana |     1 | Snack
   203603 | 2013-09-23 |   (null) | Banana |     1 | Snack
   203608 | 2013-09-30 |   (null) | Banana |     1 | Snack
   203609 | 2013-09-30 |   (null) | Banana |     1 | Snack
   203632 | 2013-09-30 |   (null) | Banana |     1 | Snack
   203636 | 2013-09-30 |   (null) | Banana |     1 | Snack
   203688 | 2013-09-30 |   (null) | Banana |     1 | Snack
   203725 | 2013-10-07 |   (null) | Banana |     1 | Snack
(10 rows)
```

一枚のチケット ID 内に数量が記録されるタイプではなく、数量分レコードが生成されるタイプ。

```sql
select * from p2019w14t1 where ticketid = 102464;

 ticketid |    date    | memberid |         desc          | price | type
----------+------------+----------+-----------------------+-------+-------
   102464 | 2013-07-01 |   (null) | Pear                  |     1 | Snack
   102464 | 2013-07-01 |   (null) | Vegan Sandwich        |     3 | Main
   102464 | 2013-07-01 |   (null) | Crisps Prawn Cocktail |     1 | Snack
   102464 | 2013-07-01 |   (null) | Apple Juice           |     2 | Drink
   102464 | 2013-07-01 |   (null) | Apple Juice           |     2 | Drink
   102464 | 2013-07-01 |   (null) | Apple Juice           |     2 | Drink
   102464 | 2013-07-01 |   (null) | Apple Juice           |     2 | Drink
   102464 | 2013-07-01 |   (null) | Tea                   |     2 | Drink
   102464 | 2013-07-01 |   (null) | Carrot Cake           |     2 | Snack
   102464 | 2013-07-01 |   (null) | Carrot Cake           |     2 | Snack
   102464 | 2013-07-01 |   (null) | Vegetable Soup        |     3 | Main
   102464 | 2013-07-01 |   (null) | Vegan Brownie         |     2 | Snack
   102464 | 2013-07-01 |   (null) | Vegan Brownie         |     2 | Snack
   102464 | 2013-07-01 |   (null) | Vegan Brownie         |     2 | Snack
   102464 | 2013-07-01 |   (null) | Crisps Ready Salted   |     1 | Snack
(15 rows)
```

まずはこれを処理するために Long 型から Wide 型に変換して、レコードを数量とみなす。

```sql
with tmp1 as (
select
    ticketid, date, memberid, "desc", price,
    sum(case when type = 'Drink' then 1 else 0 end) as drink,
    sum(case when type = 'Snack' then 1 else 0 end) as snack,
    sum(case when type = 'Main' then 1 else 0 end) as main
from
    p2019w14t1
group by
    ticketid, date, memberid, "desc", price
), tmp2 as (
select
    ticketid, date, memberid, "desc",
    drink,
    snack,
    main,
    drink * price as total_drink_price,
    snack * price as total_snack_price,
    main  * price as total_main_price,
    (drink * price) + (snack * price) + (main  * price) as total_ticket_price
from
    tmp1
)
select * from tmp2 limit 10;

 ticketid |    date    | memberid |        desc         | drink | snack | main | total_drink_price  | total_snack_price | total_main_price | total_ticket_price
----------+------------+----------+---------------------+-------+-------+------+--------------------+-------------------+------------------+--------------------
   100004 | 2013-01-07 |   (null) | Apple Juice         |     1 |     0 |    0 |                  2 |                 0 |                0 |                  2
   100004 | 2013-01-07 |   (null) | Carrot Cake         |     0 |     1 |    0 |                  0 |                 2 |                0 |                  2
   100004 | 2013-01-07 |   (null) | Rocky Road          |     0 |     2 |    0 |                  0 |                 4 |                0 |                  4
   100004 | 2013-01-07 |   (null) | Tea                 |     1 |     0 |    0 |                  2 |                 0 |                0 |                  2
   100004 | 2013-01-07 |   (null) | Tuna Sandwich       |     0 |     0 |    3 |                  0 |                 0 |             10.5 |               10.5
   100004 | 2013-01-07 |   (null) | Vegan Brownie       |     0 |     2 |    0 |                  0 |                 4 |                0 |                  4
   100004 | 2013-01-07 |   (null) | Vegetable Soup      |     0 |     0 |    1 |                  0 |                 0 |                3 |                  3
   100006 | 2013-01-07 |   996536 | Can Pop             |     3 |     0 |    0 | 3.5999999999999996 |                 0 |                0 | 3.5999999999999996
   100006 | 2013-01-07 |   996536 | Crisps Ready Salted |     0 |     1 |    0 |                  0 |                 1 |                0 |                  1
   100006 | 2013-01-07 |   996536 | Grapes              |     0 |     1 |    0 |                  0 |                 1 |                0 |                  1
(10 rows)
```

これ以降は日付要素は不要なので、チケット ID、メンバー ID 単位で纏める。行あたりの価格の計算を行なう。

```sql
with tmp1 as (
select
    ticketid, date, memberid, "desc", price,
    sum(case when type = 'Drink' then 1 else 0 end) as drink,
    sum(case when type = 'Snack' then 1 else 0 end) as snack,
    sum(case when type = 'Main' then 1 else 0 end) as main
from
    p2019w14t1
group by
    ticketid, date, memberid, "desc", price
), tmp2 as (
select
    ticketid, date, memberid, "desc",
    drink,
    snack,
    main,
    drink * price as total_drink_price,
    snack * price as total_snack_price,
    main  * price as total_main_price,
    (drink * price) + (snack * price) + (main  * price) as total_ticket_price
from
    tmp1
), tmp3 as (
select
    ticketid, memberid,
    sum(total_drink_price) as total_drink_price,
    sum(total_snack_price) as total_snack_price,
    sum(total_main_price)  as total_main_price,
    sum(drink) as drink,
    sum(snack) as snack,
    sum(main) as main,
    sum(total_ticket_price) as total_ticket_price
from
    tmp2
group by
    ticketid, memberid
)
select * from tmp3 limit 10;

 ticketid | memberid | total_drink_price | total_snack_price | total_main_price | drink | snack | main | total_ticket_price
----------+----------+-------------------+-------------------+------------------+-------+-------+------+--------------------
   100004 |   (null) |                 4 |                10 |             13.5 |     2 |     5 |    4 |               27.5
   100006 |   996536 |               5.6 |                 4 |               16 |     4 |     3 |    5 |               25.6
   100008 |   (null) |                 2 |                 6 |             19.5 |     1 |     4 |    6 |               27.5
   100010 |   990952 |                 6 |                 0 |               16 |     3 |     0 |    5 |                 22
   100011 |   (null) |               1.2 |                 5 |               16 |     1 |     3 |    5 |               22.2
   100012 |   (null) |                 2 |                 7 |               13 |     1 |     4 |    4 |                 22
   100013 |   (null) |                 4 |                 0 |                7 |     2 |     0 |    2 |                 11
   100020 |   995020 |               5.2 |                 5 |                9 |     3 |     3 |    3 |               19.2
   100023 |   (null) |                 2 |                 5 |             16.5 |     1 |     4 |    5 |               23.5
   100025 |   996103 |               2.4 |                 6 |                9 |     2 |     4 |    3 |               17.4
(10 rows)
```

各チケットに含まれる食事の取引数を計算する。この数は、各チケットの任意のアイテムの最小数になるので、ドリンク 2 つ、メイン 4 つ、スナック 3 つがある場合、2 つのドリンクしかなく、2 つの可能な食事の取引があることになる。食事の費用は 5 ポンドであるため、食事の合計費用も計算できる。食事クーポンがビジネスにどれだけのコストをもたらすかを見積もるために、各チケットの各アイテムタイプがどれくらいのコストになるかを平均で計算する。

```sql
with tmp1 as (
select
    ticketid, date, memberid, "desc", price,
    sum(case when type = 'Drink' then 1 else 0 end) as drink,
    sum(case when type = 'Snack' then 1 else 0 end) as snack,
    sum(case when type = 'Main' then 1 else 0 end) as main
from
    p2019w14t1
group by
    ticketid, date, memberid, "desc", price
), tmp2 as (
select
    ticketid, date, memberid, "desc",
    drink,
    snack,
    main,
    drink * price as total_drink_price,
    snack * price as total_snack_price,
    main  * price as total_main_price,
    (drink * price) + (snack * price) + (main  * price) as total_ticket_price
from
    tmp1
), tmp3 as (
select
    ticketid, memberid,
    sum(total_drink_price) as total_drink_price,
    sum(total_snack_price) as total_snack_price,
    sum(total_main_price)  as total_main_price,
    sum(drink) as drink,
    sum(snack) as snack,
    sum(main) as main,
    sum(total_ticket_price) as total_ticket_price
from
    tmp2
group by
    ticketid, memberid
), tmp4 as (
select
    ticketid,
    memberid,
    drink,
    snack,
    main,
    case when main = 0  then 0 else total_main_price / main   end as avg_main_price,
    case when drink = 0 then 0 else total_drink_price / drink end as avg_drink_price,
    case when snack = 0 then 0 else total_snack_price / snack end as avg_snack_price,
    total_ticket_price,
    least(main, drink, snack) as num_of_meal_deals,
    least(main, drink, snack)*5 as total_meal_deal_earnings
from
    tmp3
where
    least(main, drink, snack) != 0
)
select * from tmp4 limit 10;

 ticketid | memberid | drink | snack | main | avg_main_price |  avg_drink_price   |  avg_snack_price   | total_ticket_price | num_of_meal_deals | total_meal_deal_earnings
----------+----------+-------+-------+------+----------------+--------------------+--------------------+--------------------+-------------------+--------------------------
   100004 |   (null) |     2 |     5 |    4 |          3.375 |                  2 |                  2 |               27.5 |                 2 |                       10
   100006 |   996536 |     4 |     3 |    5 |            3.2 |                1.4 | 1.3333333333333333 |               25.6 |                 3 |                       15
   100008 |   (null) |     1 |     4 |    6 |           3.25 |                  2 |                1.5 |               27.5 |                 1 |                        5
   100011 |   (null) |     1 |     3 |    5 |            3.2 |                1.2 | 1.6666666666666667 |               22.2 |                 1 |                        5
   100012 |   (null) |     1 |     4 |    4 |           3.25 |                  2 |               1.75 |                 22 |                 1 |                        5
   100020 |   995020 |     3 |     3 |    3 |              3 | 1.7333333333333334 | 1.6666666666666667 |               19.2 |                 3 |                       15
   100023 |   (null) |     1 |     4 |    5 |            3.3 |                  2 |               1.25 |               23.5 |                 1 |                        5
   100025 |   996103 |     2 |     4 |    3 |              3 |                1.2 |                1.5 |               17.4 |                 2 |                       10
   100034 |   (null) |     1 |     3 |    2 |           3.25 |                  2 | 1.6666666666666667 |               13.5 |                 1 |                        5
   100035 |   (null) |     4 |     4 |    4 |          3.125 |                1.8 |                1.5 |               25.7 |                 4 |                       20
(10 rows)
```

食事券に含まれていないスナック菓子の数を確認する。そのチケットのその種類のアイテムの平均価格を使用して、この超過分の価値を見積もる。

```sql
with tmp1 as (
select
    ticketid, date, memberid, "desc", price,
    sum(case when type = 'Drink' then 1 else 0 end) as drink,
    sum(case when type = 'Snack' then 1 else 0 end) as snack,
    sum(case when type = 'Main' then 1 else 0 end) as main
from
    p2019w14t1
group by
    ticketid, date, memberid, "desc", price
), tmp2 as (
select
    ticketid, date, memberid, "desc",
    drink,
    snack,
    main,
    drink * price as total_drink_price,
    snack * price as total_snack_price,
    main  * price as total_main_price,
    (drink * price) + (snack * price) + (main  * price) as total_ticket_price
from
    tmp1
), tmp3 as (
select
    ticketid, memberid,
    sum(total_drink_price) as total_drink_price,
    sum(total_snack_price) as total_snack_price,
    sum(total_main_price)  as total_main_price,
    sum(drink) as drink,
    sum(snack) as snack,
    sum(main) as main,
    sum(total_ticket_price) as total_ticket_price
from
    tmp2
group by
    ticketid, memberid
), tmp4 as (
select
    ticketid,
    memberid,
    main,
    drink,
    snack,
    case when main = 0  then 0 else total_main_price / main   end as avg_main_price,
    case when drink = 0 then 0 else total_drink_price / drink end as avg_drink_price,
    case when snack = 0 then 0 else total_snack_price / snack end as avg_snack_price,
    total_ticket_price,
    least(main, drink, snack) as num_of_meal_deals,
    least(main, drink, snack)*5 as total_meal_deal_earnings
from
    tmp3
where
    least(main, drink, snack) != 0
), tmp5 as (
select
    ticketid,
    memberid,
    main,
    drink,
    snack,
    avg_main_price,
    avg_drink_price,
    avg_snack_price,
    total_ticket_price,
    total_meal_deal_earnings,
    num_of_meal_deals,
    (main  - num_of_meal_deals) * avg_main_price as excess_main_value,
    (drink - num_of_meal_deals) * avg_drink_price as excess_drink_value,
    (snack - num_of_meal_deals) * avg_snack_price as excess_snack_value,
    ((main  - num_of_meal_deals) * avg_main_price) +
    ((drink - num_of_meal_deals) * avg_drink_price) +
    ((snack - num_of_meal_deals) * avg_snack_price) as total_excess
from
    tmp4
)
select * from tmp5 limit 10;
 ticketid | memberid | main | drink | snack | avg_main_price |  avg_drink_price   |  avg_snack_price   | total_meal_deal_earnings | num_of_meal_deals | excess_main_value | excess_drink_value | excess_snack_value |    total_excess
----------+----------+------+-------+-------+----------------+--------------------+--------------------+--------------------------+-------------------+-------------------+--------------------+--------------------+--------------------
   100004 |   (null) |    4 |     2 |     5 |          3.375 |                  2 |                  2 |                       10 |                 2 |              6.75 |                  0 |                  6 |              12.75
   100006 |   996536 |    5 |     4 |     3 |            3.2 |                1.4 | 1.3333333333333333 |                       15 |                 3 |               6.4 |                1.4 |                  0 |  7.800000000000001
   100008 |   (null) |    6 |     1 |     4 |           3.25 |                  2 |                1.5 |                        5 |                 1 |             16.25 |                  0 |                4.5 |              20.75
   100011 |   (null) |    5 |     1 |     3 |            3.2 |                1.2 | 1.6666666666666667 |                        5 |                 1 |              12.8 |                  0 | 3.3333333333333335 | 16.133333333333333
   100012 |   (null) |    4 |     1 |     4 |           3.25 |                  2 |               1.75 |                        5 |                 1 |              9.75 |                  0 |               5.25 |                 15
   100020 |   995020 |    3 |     3 |     3 |              3 | 1.7333333333333334 | 1.6666666666666667 |                       15 |                 3 |                 0 |                  0 |                  0 |                  0
   100023 |   (null) |    5 |     1 |     4 |            3.3 |                  2 |               1.25 |                        5 |                 1 |              13.2 |                  0 |               3.75 |              16.95
   100025 |   996103 |    3 |     2 |     4 |              3 |                1.2 |                1.5 |                       10 |                 2 |                 3 |                  0 |                  3 |                  6
   100034 |   (null) |    2 |     1 |     3 |           3.25 |                  2 | 1.6666666666666667 |                        5 |                 1 |              3.25 |                  0 | 3.3333333333333335 |  6.583333333333334
   100035 |   (null) |    4 |     4 |     4 |          3.125 |                1.8 |                1.5 |                       20 |                 4 |                 0 |                  0 |                  0 |                  0
(10 rows)
```

最後に、元のチケット価格と、代わりに食事代を支払った場合のチケット価格との差を計算する、total_meal_deal_earnings と total_excess を合計することで、支払った金額を計算できる。これを total_ticket_price から差し引くと、ticket_price_variance_to_meal_deal_earnings が残る。

```sql
with tmp1 as (
select
    ticketid, date, memberid, "desc", price,
    sum(case when type = 'Drink' then 1 else 0 end) as drink,
    sum(case when type = 'Snack' then 1 else 0 end) as snack,
    sum(case when type = 'Main' then 1 else 0 end) as main
from
    p2019w14t1
group by
    ticketid, date, memberid, "desc", price
), tmp2 as (
select
    ticketid, date, memberid, "desc",
    drink,
    snack,
    main,
    drink * price as total_drink_price,
    snack * price as total_snack_price,
    main  * price as total_main_price,
    (drink * price) + (snack * price) + (main  * price) as total_ticket_price
from
    tmp1
), tmp3 as (
select
    ticketid, memberid,
    sum(total_drink_price) as total_drink_price,
    sum(total_snack_price) as total_snack_price,
    sum(total_main_price)  as total_main_price,
    sum(drink) as drink,
    sum(snack) as snack,
    sum(main) as main,
    sum(total_ticket_price) as total_ticket_price
from
    tmp2
group by
    ticketid, memberid
), tmp4 as (
select
    ticketid,
    memberid,
    main,
    drink,
    snack,
    case when main = 0  then 0 else total_main_price / main   end as avg_main_price,
    case when drink = 0 then 0 else total_drink_price / drink end as avg_drink_price,
    case when snack = 0 then 0 else total_snack_price / snack end as avg_snack_price,
    total_ticket_price,
    least(main, drink, snack) as num_of_meal_deals,
    least(main, drink, snack)*5 as total_meal_deal_earnings
from
    tmp3
where
    least(main, drink, snack) != 0
), tmp5 as (
select
    ticketid,
    memberid,
    main,
    drink,
    snack,
    avg_main_price,
    avg_drink_price,
    avg_snack_price,
    total_ticket_price,
    total_meal_deal_earnings,
    num_of_meal_deals,
    (main  - num_of_meal_deals) * avg_main_price as excess_main_value,
    (drink - num_of_meal_deals) * avg_drink_price as excess_drink_value,
    (snack - num_of_meal_deals) * avg_snack_price as excess_snack_value,
    ((main  - num_of_meal_deals) * avg_main_price) +
    ((drink - num_of_meal_deals) * avg_drink_price) +
    ((snack - num_of_meal_deals) * avg_snack_price) as total_excess
from
    tmp4
)
select
    *,
    total_ticket_price - (total_meal_deal_earnings + total_excess) as ticket_price_variance_to_meal_deal_earnings
from
    tmp5
limit 10;

 ticketid | memberid | main | drink | snack | avg_main_price |  avg_drink_price   |  avg_snack_price   | total_ticket_price | total_meal_deal_earnings | num_of_meal_deals | excess_main_value | excess_drink_value | excess_snack_value |    total_excess    |    ticket_price_variance_to_meal_deal_earnings
----------+----------+------+-------+-------+----------------+--------------------+--------------------+--------------------+--------------------------+-------------------+-------------------+--------------------+--------------------+--------------------+--------------------
   100004 |   (null) |    4 |     2 |     5 |          3.375 |                  2 |                  2 |               27.5 |                       10 |                 2 |              6.75 |                  0 |                  6 |              12.75 |               4.75
   100006 |   996536 |    5 |     4 |     3 |            3.2 |                1.4 | 1.3333333333333333 |               25.6 |                       15 |                 3 |               6.4 |                1.4 |                  0 |  7.800000000000001 | 2.8000000000000007
   100008 |   (null) |    6 |     1 |     4 |           3.25 |                  2 |                1.5 |               27.5 |                        5 |                 1 |             16.25 |                  0 |                4.5 |              20.75 |               1.75
   100011 |   (null) |    5 |     1 |     3 |            3.2 |                1.2 | 1.6666666666666667 |               22.2 |                        5 |                 1 |              12.8 |                  0 | 3.3333333333333335 | 16.133333333333333 | 1.0666666666666664
   100012 |   (null) |    4 |     1 |     4 |           3.25 |                  2 |               1.75 |                 22 |                        5 |                 1 |              9.75 |                  0 |               5.25 |                 15 |                  2
   100020 |   995020 |    3 |     3 |     3 |              3 | 1.7333333333333334 | 1.6666666666666667 |               19.2 |                       15 |                 3 |                 0 |                  0 |                  0 |                  0 |  4.199999999999999
   100023 |   (null) |    5 |     1 |     4 |            3.3 |                  2 |               1.25 |               23.5 |                        5 |                 1 |              13.2 |                  0 |               3.75 |              16.95 | 1.5500000000000007
   100025 |   996103 |    3 |     2 |     4 |              3 |                1.2 |                1.5 |               17.4 |                       10 |                 2 |                 3 |                  0 |                  3 |                  6 | 1.3999999999999986
   100034 |   (null) |    2 |     1 |     3 |           3.25 |                  2 | 1.6666666666666667 |               13.5 |                        5 |                 1 |              3.25 |                  0 | 3.3333333333333335 |  6.583333333333334 |  1.916666666666666
   100035 |   (null) |    4 |     4 |     4 |          3.125 |                1.8 |                1.5 |               25.7 |                       20 |                 4 |                 0 |                  0 |                  0 |                  0 |  5.699999999999999
(10 rows)
```

最後は集計しておわり。

```sql
with tmp1 as (
select
    ticketid, date, memberid, "desc", price,
    sum(case when type = 'Drink' then 1 else 0 end) as drink,
    sum(case when type = 'Snack' then 1 else 0 end) as snack,
    sum(case when type = 'Main' then 1 else 0 end) as main
from
    p2019w14t1
group by
    ticketid, date, memberid, "desc", price
), tmp2 as (
select
    ticketid, date, memberid, "desc",
    drink,
    snack,
    main,
    drink * price as total_drink_price,
    snack * price as total_snack_price,
    main  * price as total_main_price,
    (drink * price) + (snack * price) + (main  * price) as total_ticket_price
from
    tmp1
), tmp3 as (
select
    ticketid, memberid,
    sum(total_drink_price) as total_drink_price,
    sum(total_snack_price) as total_snack_price,
    sum(total_main_price)  as total_main_price,
    sum(drink) as drink,
    sum(snack) as snack,
    sum(main) as main,
    sum(total_ticket_price) as total_ticket_price
from
    tmp2
group by
    ticketid, memberid
), tmp4 as (
select
    ticketid,
    memberid,
    main,
    drink,
    snack,
    case when main = 0  then 0 else total_main_price / main   end as avg_main_price,
    case when drink = 0 then 0 else total_drink_price / drink end as avg_drink_price,
    case when snack = 0 then 0 else total_snack_price / snack end as avg_snack_price,
    total_ticket_price,
    least(main, drink, snack) as num_of_meal_deals,
    least(main, drink, snack)*5 as total_meal_deal_earnings
from
    tmp3
where
    least(main, drink, snack) != 0
), tmp5 as (
select
    ticketid,
    memberid,
    main,
    drink,
    snack,
    avg_main_price,
    avg_drink_price,
    avg_snack_price,
    total_ticket_price,
    total_meal_deal_earnings,
    num_of_meal_deals,
    (main  - num_of_meal_deals) * avg_main_price as excess_main_value,
    (drink - num_of_meal_deals) * avg_drink_price as excess_drink_value,
    (snack - num_of_meal_deals) * avg_snack_price as excess_snack_value,
    ((main  - num_of_meal_deals) * avg_main_price) +
    ((drink - num_of_meal_deals) * avg_drink_price) +
    ((snack - num_of_meal_deals) * avg_snack_price) as total_excess
from
    tmp4
)
select
    sum(total_ticket_price) as total_ticket_price,
    sum(total_ticket_price - (total_meal_deal_earnings + total_excess)) as ticket_price_variance_to_meal_deal_earnings
from
    tmp5
;

 total_ticket_price | ticket_price_variance_to_meal_deal_earnings
--------------------+---------------------------------------------
 401674.00000005943 |                           50019.85530469316
(1 row)
```

## :closed_book: Reference

None
