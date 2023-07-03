## :memo: Overview

Preppin' Data challenge の「2019: Week 16」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/05/2019-week-16.html)
- [Answer](https://preppindata.blogspot.com/2019/06/2019-week-16-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。5 つの地域の顧客の売上が記録されているテーブルがあり、特定の期間の売上げランキングを作成する、というお題。

```sql
-- t1: Sales_BarSoap.csv
-- t2: Sales_BudgetSoap.csv
-- t3: Sales_LiquidSoap.csv
-- t4: Sales_PlasmaSoap.csv
-- t5: Sales_SoapAccessories.csv

select * from p2019w16t1 limit 10;
           email           | order_total | order_date
---------------------------+-------------+------------
 tgeerebe@friendfeed.com   |         2.8 | 2018-09-01
 dbarthel84@youku.com      |        88.6 | 2018-10-01
 zwyard2n@cbsnews.com      |        44.6 | 2018-10-04
 dshewan3y@guardian.co.uk  |        99.3 | 2018-06-25
 mcarcas40@studiopress.com |        26.3 | 2019-02-18
 ienburygj@yale.edu        |         7.3 | 2018-08-24
 abenedetti8t@hud.gov      |        66.9 | 2019-01-02
 troylancefu@imgur.com     |        77.5 | 2018-07-03
 fohmsk1@cdbaby.com        |        25.1 | 2018-08-02
 tfireman5k@harvard.edu    |        46.7 | 2018-08-13
(10 rows)
```

まずはユニオンして、お題の指定通り直近 6 ヶ月にデータをフィルタする。

```sql
with tmp as (
select * from p2019w16t1
union all
select * from p2019w16t2
union all
select * from p2019w16t3
union all
select * from p2019w16t4
union all
select * from p2019w16t5
), tmp2 as (
select
    *,
    ((max(order_date)  over () + 1) - interval '6 month')::date as limit_date
from
    tmp
), tmp3 as (
select
    email,
    sum(order_total) as order_total
from
    tmp2
where
    order_date >= limit_date
group by
    email
)
select *
from tmp3 limit 10;

              email               |    order_total
----------------------------------+--------------------
 xsiely60@virginia.edu            |              160.8
 sottee2j@twitter.com             |               82.7
 mhuke2l@yale.edu                 | 253.70000000000002
 kmemmory4e@pcworld.com           |              114.2
 rlegerwoodlz@dot.gov             | 116.80000000000001
 cselvestergd@reddit.com          |               29.3
 adeane25@twitter.com             | 328.59999999999997
 tdewitt6p@vk.com                 |               35.5
 aohogertiedb@cargocollective.com |                172
 bstannislawskiqt@cafepress.com   |              120.3
(10 rows)
```

自分以上のアカウントをセルフジョインで紐付けて、ランクを作成するためのレコードを作成する。SQL であればランク系の関数や row_number を使えばこの処理は楽ができる。ここではランクを自作する。

```sql
with tmp as (
select * from p2019w16t1
union all
select * from p2019w16t2
union all
select * from p2019w16t3
union all
select * from p2019w16t4
union all
select * from p2019w16t5
), tmp2 as (
select
    *,
    ((max(order_date)  over () + 1) - interval '6 month')::date as limit_date
from
    tmp
), tmp3 as (
select
    email,
    sum(order_total) as order_total
from
    tmp2
where
    order_date >= limit_date
group by
    email
), tmp4 as (
select
    t1.email,
    t1.order_total,
    t2.email as t2_email,
    t2.order_total as t2_order_total
from
    tmp3 as t1
inner join
    tmp3 as t2
on
    t1.order_total <= t2.order_total
)
select *
from tmp4
where email = 'cchessil8b@sitemeter.com'
;

          email           |    order_total     |          t2_email           |   t2_order_total
--------------------------+--------------------+-----------------------------+--------------------
 cchessil8b@sitemeter.com | 386.40000000000003 | kcathesyedrl@parallels.com  |              411.1
 cchessil8b@sitemeter.com | 386.40000000000003 | lgillmor4v@blinklist.com    | 456.99999999999994
 cchessil8b@sitemeter.com | 386.40000000000003 | dmathison6d@twitter.com     |                389
 cchessil8b@sitemeter.com | 386.40000000000003 | rhabin95@washingtonpost.com | 467.70000000000005
 cchessil8b@sitemeter.com | 386.40000000000003 | gsanbrookb8@mlb.com         |  521.6999999999999
 cchessil8b@sitemeter.com | 386.40000000000003 | cchessil8b@sitemeter.com    | 386.40000000000003
(6 rows)
```

あとはトップ 8％のみで良いので、そのための処理を行なう。

```sql
with tmp as (
select * from p2019w16t1
union all
select * from p2019w16t2
union all
select * from p2019w16t3
union all
select * from p2019w16t4
union all
select * from p2019w16t5
), tmp2 as (
select
    *,
    ((max(order_date)  over () + 1) - interval '6 month')::date as limit_date
from
    tmp
), tmp3 as (
select
    email,
    sum(order_total) as order_total
from
    tmp2
where
    order_date >= limit_date
group by
    email
), tmp4 as (
select
    t1.email,
    t1.order_total,
    count(1) as last6months_rank,
    count(1) over () as num_of_row,
    count(1) over () * 0.08 as top_8_percent_filter
from
    tmp3 as t1
inner join
    tmp3 as t2
on
    t1.order_total <= t2.order_total
group by
    t1.email,
    t1.order_total
), tmp5 as (
select
    email,
    order_total,
    last6months_rank
from
    tmp4
)
select *
from tmp4
where email = 'cchessil8b@sitemeter.com'
;

          email           |    order_total     | last6months_rank | num_of_row | top_8_percent_filter
--------------------------+--------------------+------------------+------------+----------------------
 cchessil8b@sitemeter.com | 386.40000000000003 |                6 |        909 |                72.72
(1 row)
```

あとはランキングを作成すれば終了。

```sql
with tmp as (
select * from p2019w16t1
union all
select * from p2019w16t2
union all
select * from p2019w16t3
union all
select * from p2019w16t4
union all
select * from p2019w16t5
), tmp2 as (
select
    *,
    ((max(order_date)  over () + 1) - interval '6 month')::date as limit_date
from
    tmp
), tmp3 as (
select
    email,
    sum(order_total) as order_total
from
    tmp2
where
    order_date >= limit_date
group by
    email
), tmp4 as (
select
    t1.email,
    t1.order_total,
    count(1) as last6months_rank,
    count(1) over () as num_of_row,
    count(1) over () * 0.08 as top_8_percent_filter
from
    tmp3 as t1
inner join
    tmp3 as t2
on
    t1.order_total <= t2.order_total
group by
    t1.email,
    t1.order_total
), tmp5 as (
select
    email,
    order_total,
    last6months_rank
from
    tmp4
where
    top_8_percent_filter > last6months_rank
order by
    last6months_rank asc
)
select *
from tmp5
;

             email              |    order_total     | last6months_rank
--------------------------------+--------------------+------------------
 gsanbrookb8@mlb.com            |  521.6999999999999 |                1
 rhabin95@washingtonpost.com    | 467.70000000000005 |                2
 lgillmor4v@blinklist.com       | 456.99999999999994 |                3
 kcathesyedrl@parallels.com     |              411.1 |                4
 dmathison6d@twitter.com        |                389 |                5
 cchessil8b@sitemeter.com       | 386.40000000000003 |                6
 chavock70@freewebs.com         | 383.29999999999995 |                7
 dstitcherj0@hubpages.com       |                383 |                8
 rrippingalegr@dailymail.co.uk  | 382.70000000000005 |                9
 ecuppleditchnq@princeton.edu   |              376.9 |               10
 mharrhyc0@java.com             |              376.1 |               11
 hcheversfi@edublogs.org        |              374.9 |               12
 dhyndlr@sun.com                |              368.9 |               13
 kmedlingdt@freewebs.com        |              366.1 |               14
 djaquiss7w@sphinn.com          | 357.20000000000005 |               15
 rmarkel9n@wp.com               |              356.2 |               16
 gbeebeel7@ca.gov               |              352.2 |               17
 gburrassnw@google.fr           |              347.4 |               18
 tenzley6w@wisc.edu             |              344.6 |               19
 rbarockro@liveinternet.ru      |              343.4 |               20
 mvancastelel6@surveymonkey.com | 343.29999999999995 |               21
 sbollerjj@dedecms.com          |              340.8 |               22
 cstendell50@cnbc.com           |              340.5 |               23
 rsutherdenht@amazon.co.jp      |              336.6 |               24
 kduxburypz@java.com            |              336.2 |               25
 lclaypoole5@youtube.com        | 335.20000000000005 |               26
 lrubinowiczmn@over-blog.com    | 330.40000000000003 |               27
 adeane25@twitter.com           | 328.59999999999997 |               28
 xhamelynq4@cpanel.net          |              328.5 |               29
 ldomingues3h@ihg.com           |              327.9 |               30
 abenedetti8t@hud.gov           | 324.20000000000005 |               31
 lobradainaq@admin.ch           | 322.70000000000005 |               32
 rarundelli4@netvibes.com       |                321 |               33
 tchesnay9x@github.io           |              317.8 |               34
 fpiscottiaj@toplist.cz         | 310.00000000000006 |               35
 ehealks17@privacy.gov.au       |              309.7 |               36
 rmcgarahan1w@soup.io           | 308.29999999999995 |               37
 bmassow3z@list-manage.com      | 308.20000000000005 |               38
 bgagesr8@ucla.edu              | 307.70000000000005 |               39
 wprobbingsdh@ft.com            |              306.2 |               40
 stynewellw@kickstarter.com     |              304.5 |               41
 nguthrum51@google.fr           |              303.7 |               42
 fmulliscy@usda.gov             | 302.70000000000005 |               43
 ucorpes2m@usnews.com           |              301.2 |               45
 edayesgs@hugedomains.com       |              301.2 |               45
 mdefrainc5@plala.or.jp         |              300.4 |               46
 rperotti6o@simplemachines.org  | 299.90000000000003 |               47
 dsconcepa@shop-pro.jp          | 299.79999999999995 |               48
 sstothardgc@dyndns.org         |              299.6 |               49
 dfonzonepf@instagram.com       |              297.1 |               50
 tgeerebe@friendfeed.com        |              295.3 |               51
 asimysonns@sohu.com            |              293.6 |               52
 rleyshl@prweb.com              |              293.5 |               53
 gdurrell9a@blinklist.com       | 293.49999999999994 |               54
 thawker3a@vinaora.com          |                291 |               55
 cpaladinil5@fc2.com            |              288.8 |               56
 btirreyv@geocities.com         |              288.4 |               57
 dalesbrook2h@mlb.com           |              288.3 |               58
 feddinsqy@about.me             |              287.1 |               59
 ggilhoolie3s@seattletimes.com  |              286.9 |               60
 hohickeedx@cbsnews.com         |              286.4 |               61
 mboltepb@irs.gov               |                284 |               62
 pmilbankbo@prlog.org           |              282.3 |               63
 rbradbornehe@soundcloud.com    |              282.2 |               64
 gkettridge7h@yale.edu          |              281.8 |               65
 aheffordeh6@wiley.com          | 281.09999999999997 |               66
 traimanl2@washingtonpost.com   |                280 |               67
 mdouchjw@gnu.org               |              278.8 |               68
 clewinsdn@geocities.jp         | 278.29999999999995 |               69
 ahawsbycx@irs.gov              | 278.09999999999997 |               70
 ggodsell6j@umn.edu             |              275.8 |               71
 bllewellynlu@howstuffworks.com | 275.59999999999997 |               72
(72 rows)
```

## :closed_book: Reference

None
