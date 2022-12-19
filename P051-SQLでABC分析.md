## :memo: Overview

ABC 分析を SQL で行う。ABC 分析は商品や取引先などごとの売上金額を集計し、自社の売上構成を分析する手法。上位 0%~70%を A ランク、70%~90%を B ランク、その他を C ランクとして分類する。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

サンプルデータは下記のサイトよりお借りした。

- [UC Irvine Machine Learning Repository!](https://archive.ics.uci.edu/ml/datasets/online+retail#)

```sql
-- From https://archive.ics.uci.edu/ml/datasets/online+retail#
select * from onlinestore limit 10;
 invoiceno | stockcode |             description             | quantity |     invoicedate     | unitprice | customerid |    country
-----------+-----------+-------------------------------------+----------+---------------------+-----------+------------+----------------
 536365    | 85123A    | WHITE HANGING HEART T-LIGHT HOLDER  |        6 | 2010-12-01 08:26:00 |      2.55 | 17850      | United Kingdom
 536365    | 71053     | WHITE METAL LANTERN                 |        6 | 2010-12-01 08:26:00 |      3.39 | 17850      | United Kingdom
 536365    | 84406B    | CREAM CUPID HEARTS COAT HANGER      |        8 | 2010-12-01 08:26:00 |      2.75 | 17850      | United Kingdom
 536365    | 84029G    | KNITTED UNION FLAG HOT WATER BOTTLE |        6 | 2010-12-01 08:26:00 |      3.39 | 17850      | United Kingdom
 536365    | 84029E    | RED WOOLLY HOTTIE WHITE HEART.      |        6 | 2010-12-01 08:26:00 |      3.39 | 17850      | United Kingdom
 536365    | 22752     | SET 7 BABUSHKA NESTING BOXES        |        2 | 2010-12-01 08:26:00 |      7.65 | 17850      | United Kingdom
 536365    | 21730     | GLASS STAR FROSTED T-LIGHT HOLDER   |        6 | 2010-12-01 08:26:00 |      4.25 | 17850      | United Kingdom
 536366    | 22633     | HAND WARMER UNION JACK              |        6 | 2010-12-01 08:28:00 |      1.85 | 17850      | United Kingdom
 536366    | 22632     | HAND WARMER RED POLKA DOT           |        6 | 2010-12-01 08:28:00 |      1.85 | 17850      | United Kingdom
 536367    | 84879     | ASSORTED COLOUR BIRD ORNAMENT       |       32 | 2010-12-01 08:34:00 |      1.69 | 13047      | United Kingdom

```

ABC 分析の計算過程は下記の通りなので、この順番で SQL を書いていく。

1. 興味のあるグループごとに売上を合計
2. 売上の降順で並び替え
3. グループごとに構成比を計算
4. 構成比の累計を計算
5. ランク分け

4 までを計算すると下記のようになる。

```sql
with tmp as (
select
    *
from
    onlinestore
where
    invoicedate between '2010-12-01' and '2010-12-31'
), tmp1 as (
select
    stockcode,
    sum(unitprice) as unitprice_sum
from
    tmp
group by
    stockcode
order by
    sum(unitprice) desc
)
select
    stockcode,
    unitprice_sum,
    sum(unitprice_sum) over() as ttl,
    100 * unitprice_sum / sum(unitprice_sum) over() as ratio,
    100 * sum(unitprice_sum) over(order by unitprice_sum desc rows between unbounded preceding and current row) / sum(unitprice_sum) over() as cumlative_ratio
from
    tmp1
limit
    20
;

  stockcode   | unitprice_sum |    ttl    |        ratio        |  cumlative_ratio
--------------+---------------+-----------+---------------------+--------------------
 AMAZONFEE    |      66325.74 | 260521.33 |   25.45885308694436 |  25.45885308694436
 DOT          |      24671.19 | 260521.33 |   9.469930784817581 | 34.928783122063244
 M            |       4897.11 | 260521.33 |  1.8797347221151819 |  36.80851765675375
 22423        |      2676.039 | 260521.33 |   1.027186173876719 |  37.83570383063047
 22655        |          1510 | 260521.33 |   0.579607055924224 |  38.41531088655469
 POST         |          1341 | 260521.33 |   0.514737127148599 |  38.93004801370329
 21258        |       1256.87 | 260521.33 | 0.48244418380752774 | 39.412491119818945
 BANK CHARGES |     1077.2999 | 260521.33 | 0.41351697940097876 |  39.82600692781571
 22112        |     931.26025 | 260521.33 |  0.3574602742157927 | 40.183466264908134
 22111        |     927.56036 | 260521.33 |  0.3560400871764638 | 40.539507172067545
 22625        |           846 | 260521.33 |  0.3247334896105255 | 40.864240661678075
 22752        |     803.77997 | 260521.33 | 0.30852751060598743 | 41.172768664273825
 22837        |      767.5598 | 260521.33 |  0.2946245591396818 | 41.467394254249214
 21485        |      764.2101 | 260521.33 |   0.293338779019712 |  41.76073336126211
 85123A       |      734.3705 | 260521.33 | 0.28188497605312424 |  42.04261707219868
 21843        |     717.70026 | 260521.33 | 0.27548618054154034 | 42.318104353860186
 D            |     693.98004 | 260521.33 |  0.2663812773021523 | 42.584484295761534
 22656        |           675 | 260521.33 | 0.25909586937010015 | 42.843580165131634
 22835        |      658.9499 | 260521.33 |  0.2529351031945261 | 43.096516510014624
 84029G       |     639.05005 | 260521.33 | 0.24529663403278223 |  43.34181192578703
(20 rows)
```

あとはこのデータにランクをふれば ABC 分析のための集計は完了。

```sql
with tmp as (
select
    *
from
    onlinestore
where
    invoicedate between '2010-12-01' and '2010-12-31'
), tmp1 as (
select
    stockcode,
    sum(unitprice) as unitprice_sum
from
    tmp
group by
    stockcode
order by
    sum(unitprice) desc
), tmp2 as (
select
    stockcode,
    unitprice_sum,
    sum(unitprice_sum) over() as ttl,
    100 * unitprice_sum / sum(unitprice_sum) over() as ratio,
    100 * sum(unitprice_sum) over(order by unitprice_sum desc rows between unbounded preceding and current row) / sum(unitprice_sum) over() as cumlative_ratio
from
    tmp1
)
select
    stockcode,
    unitprice_sum,
    round(ratio::numeric, 2) as ratio,
    round(cumlative_ratio::numeric, 2) as cumlative_ratio,
    case
    when cumlative_ratio between 0 and 70 then 'A Rank'
    when cumlative_ratio between 70 and 90 then 'B Rank'
    when cumlative_ratio between 90 and 100 then 'C Rank'
    else null end as abcrank
from
    tmp2
limit 20
;

  stockcode   | unitprice_sum | ratio | cumlative_ratio | abcrank
--------------+---------------+-------+-----------------+---------
 AMAZONFEE    |      66325.74 | 25.46 |           25.46 | A Rank
 DOT          |      24671.19 |  9.47 |           34.93 | A Rank
 M            |       4897.11 |  1.88 |           36.81 | A Rank
 22423        |     2676.0393 |  1.03 |           37.84 | A Rank
 22655        |          1510 |  0.58 |           38.42 | A Rank
 POST         |          1341 |  0.51 |           38.93 | A Rank
 21258        |       1256.87 |  0.48 |           39.41 | A Rank
 BANK CHARGES |     1077.2999 |  0.41 |           39.83 | A Rank
~~~~~~
 84946        |     161.66998 |  0.06 |           69.80 | A Rank
 22451        |     161.63997 |  0.06 |           69.86 | A Rank
 22170        |     161.54999 |  0.06 |           69.92 | A Rank
 84968A       |        161.44 |  0.06 |           69.99 | A Rank
 22783        |         161.4 |  0.06 |           70.05 | B Rank
 84997D       |        161.37 |  0.06 |           70.11 | B Rank
 22187        |     161.14001 |  0.06 |           70.17 | B Rank
 82583        |     160.72998 |  0.06 |           70.23 | B Rank
 22471        |        160.69 |  0.06 |           70.30 | B Rank
~~~~~~
 85226A       |             0 |  0.00 |          100.00 | C Rank
 35951        |             0 |  0.00 |          100.00 | C Rank
 84977        |             0 |  0.00 |          100.00 | C Rank
 21653        |             0 |  0.00 |          100.00 | C Rank
 21431        |             0 |  0.00 |          100.00 | C Rank
 84968B       |             0 |  0.00 |          100.00 | C Rank
 21661        |             0 |  0.00 |          100.00 | C Rank
 85044        |             0 |  0.00 |          100.00 | C Rank
(2822 rows)
```

## :closed_book: Reference

None
