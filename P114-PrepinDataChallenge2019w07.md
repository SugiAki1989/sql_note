## :memo: Overview

Preppin' Data challenge の「2019: Week 7」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/03/2019-week-7.html)
- [Answer](https://preppindata.blogspot.com/2019/04/2019-week-7-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。出荷する荷物の情報があって、これらが出荷条件を満たしているかを判定する、というお題。

```sql
select * from p2019w07t1 limit 10;

  salesperson  |      departure_id       | date_logged |     product_type     | weight_allocated | volume_allocated
---------------+-------------------------+-------------+----------------------+------------------+------------------
 Daisy Steiner | Tanker-05-04-05-2019    | 21/2/2019   | Spurs Jerseys        |              111 |               10
 Tim Bisley    | Freighter-01-18-06-2019 | 14/1/2019   | Phone Catalogues     |              756 |               48
 Daisy Steiner | Tanker-04-27-04-2019    | 26/1/2019   | Oil                  |              220 |               28
 Colin Chien   | Freighter-02-10-04-2019 | 11/3/2019   | Rubber Ducks         |              536 |               20
 Colin Chien   | Tanker-05-20-05-2019    | 27/1/2019   | Vinyl Records        |              148 |               30
 Mike Watt     | Tanker-05-03-05-2019    | 22/2/2019   | Legumes              |              221 |               10
 Twist Morgan  | Tanker-05-08-06-2019    | 17/1/2019   | Tiny Violins         |               68 |                2
 Colin Chien   | Tug-01-29-04-2019       | 13/1/2019   | University Textbooks |               99 |               10
 Twist Morgan  | Tug-04-18-06-2019       | 11/2/2019   | Rubber Ducks         |               61 |                7
 Colin Chien   | Tug-04-12-06-2019       | 4/2/2019    | C&BS Co Soap         |               95 |                1
(10 rows)

select * from p2019w07t2 limit 10;

   ship_id    | departure_date | max_weight | max_volume
--------------+----------------+------------+------------
 Tug-05       | 27/3/2019      |        500 |         50
 Freighter-04 | 31/3/2019      |       5000 |        500
 Freighter-03 | 1/4/2019       |       5000 |        500
 Freighter-01 | 6/4/2019       |       5000 |        500
 Tanker-02    | 7/4/2019       |       1000 |        200
 Tanker-04    | 10/4/2019      |       1000 |        200
 Freighter-02 | 10/4/2019      |       5000 |        500
 Tug-05       | 10/4/2019      |        500 |         50
 Freighter-03 | 10/4/2019      |       5000 |        500
 Freighter-04 | 12/4/2019      |       5000 |        500
(10 rows)
```

`departure_id`が`id-no-DD-MM-YYYY`というフォーマットなので、下記のようにして日付と ID を取り出す。

```sql
select
    to_date(right(departure_id, 10), 'DD-MM-YYYY') as departure_date,
    departure_id,
    char_length(departure_id) as _charlen,
    left(departure_id, char_length(departure_id) - 11) as shipid_other,
    split_part(departure_id, '-', 1) || '-' || split_part(departure_id, '-', 2) as shipid
from
    p2019w07t1
limit 10
;

 departure_date |      departure_id       | _charlen | shipid_other |    shipid
----------------+-------------------------+----------+--------------+--------------
 2019-05-04     | Tanker-05-04-05-2019    |       20 | Tanker-05    | Tanker-05
 2019-06-18     | Freighter-01-18-06-2019 |       23 | Freighter-01 | Freighter-01
 2019-04-27     | Tanker-04-27-04-2019    |       20 | Tanker-04    | Tanker-04
 2019-04-10     | Freighter-02-10-04-2019 |       23 | Freighter-02 | Freighter-02
 2019-05-20     | Tanker-05-20-05-2019    |       20 | Tanker-05    | Tanker-05
 2019-05-03     | Tanker-05-03-05-2019    |       20 | Tanker-05    | Tanker-05
 2019-06-08     | Tanker-05-08-06-2019    |       20 | Tanker-05    | Tanker-05
 2019-04-29     | Tug-01-29-04-2019       |       17 | Tug-01       | Tug-01
 2019-06-18     | Tug-04-18-06-2019       |       17 | Tug-04       | Tug-04
 2019-06-12     | Tug-04-12-06-2019       |       17 | Tug-04       | Tug-04
(10 rows)
```

取り出した日付と ID で紐付ける。その後は、`ship_id, departure_date`でグループ化して、しきい値を超えていないか、各指標を合計して判定する。

```sql
with tmp as (
select
    salesperson,
    date_logged,
    product_type,
    weight_allocated,
    volume_allocated,
    to_date(right(departure_id, 10), 'DD-MM-YYYY') as departure_date,
    split_part(departure_id, '-', 1) || '-' || split_part(departure_id, '-', 2) as ship_id
from
    p2019w07t1
), tmp2 as (
select
    ship_id,
    to_date(departure_date, 'DD/MM/YYYY') as departure_date,
    max_weight,
    max_volume
from
    p2019w07t2
), tmp3 as (
select
    t1.salesperson,
    t1.date_logged,
    t1.product_type,
    t1.weight_allocated,
    t1.volume_allocated,
    t1.departure_date,
    t1.ship_id,
    t2.max_weight,
    t2.max_volume
from
    tmp as t1
left join
    tmp2 as t2
on
    t1.departure_date = t2.departure_date and
    t1.ship_id = t2.ship_id
)
select * from tmp3 where ship_id = 'Freighter-05' order by departure_date asc
;

  salesperson  | date_logged |     product_type     | weight_allocated | volume_allocated | departure_date |   ship_id    | max_weight | max_volume
---------------+-------------+----------------------+------------------+------------------+----------------+--------------+------------+------------
 Daisy Steiner | 14/4/2019   | Luxury Goods         |              844 |               40 | 2019-05-23     | Freighter-05 |       5000 |        500
 Mike Watt     | 1/2/2019    | Spices               |              152 |                8 | 2019-05-23     | Freighter-05 |       5000 |        500
 Tim Bisley    | 4/3/2019    | Vinyl Records        |              504 |               70 | 2019-05-23     | Freighter-05 |       5000 |        500
 Brian Topp    | 19/2/2019   | Produce              |              368 |               58 | 2019-05-23     | Freighter-05 |       5000 |        500
 Mike Watt     | 25/4/2019   | Vinyl Records        |              776 |               24 | 2019-05-23     | Freighter-05 |       5000 |        500
 Tim Bisley    | 15/4/2019   | University Textbooks |              200 |               26 | 2019-05-23     | Freighter-05 |       5000 |        500
 Mike Watt     | 1/3/2019    | Rubber Ducks         |              208 |               72 | 2019-05-23     | Freighter-05 |       5000 |        500
 Daisy Steiner | 18/1/2019   | Spurs Jerseys        |               84 |               78 | 2019-05-26     | Freighter-05 |       5000 |        500
 Brian Topp    | 27/2/2019   | DVD Boxsets          |              236 |               90 | 2019-05-26     | Freighter-05 |       5000 |        500
 Tim Bisley    | 20/1/2019   | Rubber Ducks         |              624 |               42 | 2019-05-26     | Freighter-05 |       5000 |        500
 Daisy Steiner | 25/3/2019   | University Textbooks |              196 |               96 | 2019-05-26     | Freighter-05 |       5000 |        500
 Tim Bisley    | 14/3/2019   | Beanie Babies        |              868 |               42 | 2019-06-06     | Freighter-05 |       5000 |        500
 Colin Chien   | 3/2/2019    | Livestock            |              804 |               40 | 2019-06-06     | Freighter-05 |       5000 |        500
 Colin Chien   | 16/1/2019   | DVD Boxsets          |              632 |              100 | 2019-06-06     | Freighter-05 |       5000 |        500
 Brian Topp    | 10/2/2019   | Vinyl Records        |              228 |               98 | 2019-06-06     | Freighter-05 |       5000 |        500
 Tim Bisley    | 27/1/2019   | Produce              |              680 |               94 | 2019-06-06     | Freighter-05 |       5000 |        500
 Colin Chien   | 2/3/2019    | Tiny Violins         |              716 |               76 | 2019-06-10     | Freighter-05 |       5000 |        500
 Brian Topp    | 1/1/2019    | Legumes              |              568 |               60 | 2019-06-10     | Freighter-05 |       5000 |        500
 Daisy Steiner | 18/1/2019   | Motor vehicles       |              760 |               82 | 2019-06-10     | Freighter-05 |       5000 |        500
 Daisy Steiner | 26/2/2019   | Luxury Goods         |              200 |               70 | 2019-06-10     | Freighter-05 |       5000 |        500
 Marsha Klein  | 2/2/2019    | Oil                  |              508 |               12 | 2019-06-10     | Freighter-05 |       5000 |        500
 Marsha Klein  | 28/3/2019   | Spices               |              556 |               20 | 2019-06-10     | Freighter-05 |       5000 |        500
 Marsha Klein  | 4/1/2019    | DVD Boxsets          |              188 |               60 | 2019-06-10     | Freighter-05 |       5000 |        500
 Colin Chien   | 6/3/2019    | Livestock            |              872 |               54 | 2019-06-19     | Freighter-05 |       5000 |        500
 Brian Topp    | 1/4/2019    | Spices               |              764 |               42 | 2019-06-19     | Freighter-05 |       5000 |        500
 Daisy Steiner | 4/4/2019    | Legumes              |              940 |               58 | 2019-06-26     | Freighter-05 |       5000 |        500
 Tim Bisley    | 1/1/2019    | Timber               |              364 |               54 | 2019-06-26     | Freighter-05 |       5000 |        500
 Colin Chien   | 11/3/2019   | C&BS Co Soap         |              252 |               82 | 2019-06-26     | Freighter-05 |       5000 |        500
 Twist Morgan  | 3/1/2019    | Gold Bullion         |              968 |               72 | 2019-06-26     | Freighter-05 |       5000 |        500
 Daisy Steiner | 22/4/2019   | Oil                  |              780 |               16 | 2019-06-26     | Freighter-05 |       5000 |        500
(30 rows)
```

テーブルの定義までわからないので、`ship_id, departure_dat`でユニークになのか不明なので、`max(max_weight), max(max_volume)`して問題ないかの確認。

```sql
select count(1) from (
    select distinct ship_id, departure_date from p2019w07t2
) as subq
;
 count
-------
    58
(1 row)

select count(1) from p2019w07t2;
 count
-------
    58
(1 row)

select
    ship_id,
    departure_date,
    min(max_weight),
    max(max_weight)
from
    p2019w07t2
group by
    ship_id,
    departure_date
having
    case when min(max_weight) != max(max_weight) then 1 else 0 end = 1
;

 ship_id | departure_date | min | max
---------+----------------+-----+-----
(0 rows)


select
    ship_id,
    departure_date,
    min(max_volume),
    max(max_volume)
from
    p2019w07t2
group by
    ship_id,
    departure_date
having
    case when min(max_volume) != max(max_volume) then 1 else 0 end = 1
;

 ship_id | departure_date | min | max
---------+----------------+-----+-----
(0 rows)
```

グループ化した結果がこれ。

```sql
with tmp as (
select
    salesperson,
    date_logged,
    product_type,
    weight_allocated,
    volume_allocated,
    to_date(right(departure_id, 10), 'DD-MM-YYYY') as departure_date,
    split_part(departure_id, '-', 1) || '-' || split_part(departure_id, '-', 2) as ship_id
from
    p2019w07t1
), tmp2 as (
select
    ship_id,
    to_date(departure_date, 'DD/MM/YYYY') as departure_date,
    max_weight,
    max_volume
from
    p2019w07t2
), tmp3 as (
select
    t1.salesperson,
    t1.date_logged,
    t1.product_type,
    t1.weight_allocated,
    t1.volume_allocated,
    t1.departure_date,
    t1.ship_id,
    t2.max_weight,
    t2.max_volume
from
    tmp as t1
left join
    tmp2 as t2
on
    t1.departure_date = t2.departure_date and
    t1.ship_id = t2.ship_id
)
select
    ship_id,
    departure_date,
    sum(volume_allocated) as sum_volume_allocated,
    sum(weight_allocated) as sum_weight_allocated,
    max(max_weight) as max_weight,
    max(max_volume) as max_max_volume
from
    tmp3
group by
    ship_id,
    departure_date
limit 10
;
   ship_id    | departure_date | sum_volume_allocated | sum_weight_allocated | max_weight | max_max_volume
--------------+----------------+----------------------+----------------------+------------+----------------
 Freighter-01 | 2019-04-06     |                  112 |                 1336 |       5000 |            500
 Freighter-01 | 2019-06-18     |                  556 |                 4580 |       5000 |            500
 Freighter-02 | 2019-04-10     |                  396 |                 3972 |       5000 |            500
 Freighter-02 | 2019-04-19     |                  200 |                 1932 |       5000 |            500
 Freighter-02 | 2019-05-26     |                  294 |                 4252 |       5000 |            500
 Freighter-02 | 2019-06-03     |                  112 |                 2976 |       5000 |            500
 Freighter-02 | 2019-06-11     |                  102 |                 1056 |       5000 |            500
 Freighter-03 | 2019-04-01     |                  232 |                 2392 |       5000 |            500
 Freighter-03 | 2019-04-10     |                  456 |                 3476 |       5000 |            500
 Freighter-04 | 2019-03-31     |                  106 |                  616 |       5000 |            500
(10 rows)
```

あとはしきい値を超えているものがないかを判定する。

```sql
with tmp as (
select
    salesperson,
    date_logged,
    product_type,
    weight_allocated,
    volume_allocated,
    to_date(right(departure_id, 10), 'DD-MM-YYYY') as departure_date,
    split_part(departure_id, '-', 1) || '-' || split_part(departure_id, '-', 2) as ship_id
from
    p2019w07t1
), tmp2 as (
select
    ship_id,
    to_date(departure_date, 'DD/MM/YYYY') as departure_date,
    max_weight,
    max_volume
from
    p2019w07t2
), tmp3 as (
select
    t1.salesperson,
    t1.date_logged,
    t1.product_type,
    t1.weight_allocated,
    t1.volume_allocated,
    t1.departure_date,
    t1.ship_id,
    t2.max_weight,
    t2.max_volume
from
    tmp as t1
left join
    tmp2 as t2
on
    t1.departure_date = t2.departure_date and
    t1.ship_id = t2.ship_id
), tmp4 as (
select
    ship_id,
    departure_date,
    sum(volume_allocated) as sum_volume_allocated,
    sum(weight_allocated) as sum_weight_allocated,
    max(max_weight) as max_weight,
    max(max_volume) as max_max_volume
from
    tmp3
group by
    ship_id,
    departure_date
), tmp5 as (
select
    ship_id,
    departure_date,
    sum_volume_allocated,
    max_max_volume,
    case when sum_volume_allocated > max_max_volume then true else false end as is_max_volume_exceeded,
    sum_weight_allocated,
    max_weight,
    case when sum_weight_allocated > max_weight then true else false end as is_max_weight_exceeded
from
    tmp4
)
select * from tmp5
where
    is_max_volume_exceeded = true or
    is_max_weight_exceeded = true
;

   ship_id    | departure_date | sum_volume_allocated | max_max_volume | is_max_volume_exceeded | sum_weight_allocated | max_weight | is_max_weight_exceeded
--------------+----------------+----------------------+----------------+------------------------+----------------------+------------+------------------------
 Freighter-01 | 2019-06-18     |                  556 |            500 | t                      |                 4580 |       5000 | f
 Tanker-01    | 2019-06-04     |                  228 |            200 | t                      |                 1063 |       1000 | t
 Tanker-04    | 2019-04-27     |                  276 |            200 | t                      |                  924 |       1000 | f
 Tanker-04    | 2019-06-27     |                  216 |            200 | t                      |                  839 |       1000 | f
 Tanker-05    | 2019-05-03     |                  142 |            200 | f                      |                 1007 |       1000 | t
 Tug-02       | 2019-04-25     |                   64 |             50 | t                      |                  506 |        500 | t
 Tug-03       | 2019-04-17     |                   65 |             50 | t                      |                  521 |        500 | t
 Tug-04       | 2019-06-18     |                   53 |             50 | t                      |                  435 |        500 | f
 Tug-05       | 2019-03-27     |                   68 |             50 | t                      |                  707 |        500 | t
(9 rows)
```

## :closed_book: Reference

None
