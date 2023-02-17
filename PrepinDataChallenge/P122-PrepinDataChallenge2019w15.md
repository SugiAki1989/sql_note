## :memo: Overview

Preppin' Data challenge の「2019: Week 15」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/05/2019-week-15.html)
- [Answer](https://preppindata.blogspot.com/2019/05/2019-week-15-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。5 つのエリアの売上データを分析し、構成する総売上の割合を計算するというお題。

```sql
-- t1: Central Stock Purchases
-- t2: East Stock Purchases
-- t3: North Stock Purchases
-- t4: South Stock Purchases
-- t5: West Stock Purchases

select * from p2019w15t1 limit 10;

 customer_id | first_name | last_name  |  sales  | order_date |                     stock
-------------+------------+------------+---------+------------+-----------------------------------------------
           1 | Berny      | Robjant    | 6087.89 | 2018-08-01 | Park Sterling Corporation
           2 | Fanni      | Ackenhead  | 6294.87 | 2018-01-08 | Titan International, Inc.
           3 | Keelby     | Northcote  | 5768.85 | 2019-01-20 | Brookfield Real Assets Income Fund Inc.
           4 | Riva       | Rosenhaupt |  4073.2 | 2018-03-22 | AllianzGI Equity & Convertible Income Fund
           5 | Shoshanna  | Jeacop     | 4586.99 | 2017-07-10 | Aratana Therapeutics, Inc.
           6 | Bonnibelle | Bourdon    | 5877.19 | 2017-04-20 | RCM Technologies, Inc.
           7 | Dyna       | Christol   | 1574.96 | 2017-05-19 | Electrum Special Acquisition Corporation
           8 | Gusta      | Mahmood    |  304.99 | 2017-01-07 | Fusion Telecommunications International, Inc.
           9 | Delores    | Nieass     | 7798.91 | 2018-03-03 | Panhandle Royalty Company
          10 | Pansy      | Newbury    |  508.47 | 2019-01-05 | La Jolla Pharmaceutical Company
(10 rows)
```

ユニオンして、単位ごとの合計を計算して比率を計算すれば終わり。

```sql
with tmp as (
select *, 'Central' as region from p2019w15t1
union all
select *, 'East' as region from p2019w15t2
union all
select *, 'North' as region from p2019w15t3
union all
select *, 'South' as region from p2019w15t4
union all
select *, 'West' as region from p2019w15t5
), tmp2 as (
select
    customer_id, first_name, last_name, sales, order_date, stock,
    sum(sales) over (partition by stock) as total_sales,
    (sales/sum(sales) over (partition by stock)) * 100 as percent_of_total_sales,
    sum(sales) over (partition by region, stock) as total_regional_sales,
    (sales/sum(sales) over (partition by region, stock)) * 100 as percent_of_regional_sales
from
    tmp
)
select *
from tmp2
where
    sales != total_regional_sales
;

 customer_id |  first_name   |   last_name    |  sales  | order_date |                             stock        |    total_sales     | percent_of_total_sales | total_regional_sales | percent_of_regional_sales
-------------+---------------+----------------+---------+------------+------------------------------------------+--------------------+------------------------+----------------------+---------------------------
         632 | Christoffer   | Chetwind       |  2875.8 | 2017-09-27 | A V Homes, Inc.                          |            8857.25 |     32.468316915521186 |              8857.25 |        32.468316915521186
          56 | Dallas        | Blowes         | 5981.45 | 2019-02-16 | A V Homes, Inc.                          |            8857.25 |      67.53168308447881 |              8857.25 |         67.53168308447881
         466 | Anabel        | Diggens        | 2754.66 | 2018-08-04 | Abeona Therapeutics Inc.                 |           20128.61 |     13.685296699573394 |              7530.21 |        36.581449919723354
         955 | Lyn           | Nudds          | 4775.55 | 2018-06-22 | Abeona Therapeutics Inc.                 |           20128.61 |      23.72518519659331 |              7530.21 |        63.418550080276646
         711 | Free          | Philot         | 8706.56 | 2018-03-10 | Adient plc                               |            9957.68 |      87.43562757590121 |              9957.68 |         87.43562757590121
          85 | Theodosia     | Wellsman       | 1251.12 | 2017-12-16 | Adient plc                               |            9957.68 |     12.564372424098785 |              9957.68 |        12.564372424098785
         617 | Lonee         | Feaviour       | 4765.47 | 2018-06-28 | Advanced Semiconductor Engineering, Inc. |            5845.46 |      81.52429406753275 |              5845.46 |         81.52429406753275

```

## :closed_book: Reference

None
