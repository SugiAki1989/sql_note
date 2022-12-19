## :memo: Overview

ここでは 2 商品のアソシエーション分析を行う。アソシエーション分析は、相関ルールを抽出する方法の 1 つで、A を買った会員は、B を買っているなどの購買傾向を分析する方法。Support(支持度)、Lift(リフト)、Confidence(確信度)などの各指標の計算方法や意味合いは下記でわかりやすく解説されているので、下記を参照。

- [商品分析の手法（ABC 分析、アソシエーション分析）](https://www.albert2005.co.jp/knowledge/marketing/customer_product_analysis/abc_association)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

サンプルデータを用意しておく。

```sql
create table basket(
    item varchar(12),
    orderid varchar(12)
);

insert into basket values
  ('diapers','ord_01'),
  ('beer','ord_01'),
  ('beer','ord_02'),
  ('diapers','ord_02'),
  ('diapers','ord_02'),
  ('apple','ord_02'),
  ('apple','ord_02'),
  ('banana','ord_02'),
  ('orange','ord_03'),
  ('banana','ord_04'),
  ('beer','ord_04'),
  ('banana','ord_05'),
  ('banana','ord_06'),
  ('diapers','ord_07'),
  ('diapers','ord_08'),
  ('banana','ord_09'),
  ('beer','ord_10'),
  ('banana','ord_11'),
  ('diapers','ord_11'),
  ('banana','ord_12'),
  ('banana','ord_12'),
  ('diapers','ord_13'),
  ('beer','ord_13'),
  ('orange','ord_14'),
  ('orange','ord_14'),
  ('diapers','ord_15'),
  ('beer','ord_15'),
  ('beer','ord_15'),
  ('diapers','ord_15'),
  ('diapers','ord_15'),
  ('apple','ord_15'),
  ('apple','ord_15'),
  ('banana','ord_15'),
  ('orange','ord_15'),
  ('banana','ord_15'),
  ('beer','ord_15'),
  ('banana','ord_15'),
  ('banana','ord_15'),
  ('diapers','ord_15'),
  ('diapers','ord_15'),
  ('banana','ord_15'),
  ('beer','ord_15'),
  ('banana','ord_15'),
  ('diapers','ord_15'),
  ('banana','ord_15'),
  ('banana','ord_15'),
  ('diapers','ord_15'),
  ('beer','ord_15'),
  ('orange','ord_15'),
  ('orange','ord_15');
```

以前ブログに書いていたものをそのまま転機する。

```sql
-- ASsociation Analysis
-- deleate duplication
with log_u as (
select
    orderid, item
from
    basket
group by
    orderid, item
),
-- log_u--Root --Branch_X-Merge_XY
--              └Branch_Y┘
-- Root
tmp1 as (
select
  item,
  count(item) as cnt_from_XY  -- アイテムごとの購入数
from
  log_u
group by
  item
),
-- Branch_X
-- 会員ごとの商品組み合わせパターン一覧に、アイテムごとの購入数がついている
X as (
select
  log_u.orderid as X_uid,
  log_u.item as X_item,
  tmp1.cnt_from_XY as cnt_from_X  -- {X → Y}
from
  log_u
inner join
  tmp1
on
  log_u.item = tmp1.item
),
-- Branch_Y
-- 会員ごとの商品組み合わせパターン一覧に、アイテムごとの購入数がついている
Y as (
select
  log_u.orderid as Y_uid,
  log_u.item as Y_item,
  tmp1.cnt_from_XY as cnt_from_Y
from
  log_u
inner join
  tmp1
on
  log_u.item = tmp1.item
),
-- merge XY
-- Xからみた各アイテムごとの購入数がcnt_from_Xにはついている
-- Yからみた各アイテムごとの購入数がcnt_from_Yにはついている
XY as (
select
  X.X_item,
  Y.Y_item,
  max(cnt_from_X) as X_cnt,   -- Xからみたアイテムごとの購入数
  max(cnt_from_Y) as Y_cnt,   -- Yからみたアイテムごとの購入数
  count(X.X_item) as XY_cnt,  -- XかつYの同時購入数, count(br_Y.Y_item)でも同じ
  (select count(distinct orderid) from log_u) as user_uu_cnt
from
  X
inner join
  Y
on
  X.X_uid = Y.Y_uid and X.X_item <> Y.Y_item
group by
  X.X_item,
  Y.Y_item
)
select
  XY.X_item,
  XY.Y_item,
  XY.X_cnt,
  XY.Y_cnt,
  XY.XY_cnt,
  XY.user_uu_cnt,
  -- 全体の中でXもYも買われる確率：Support
  round(XY.XY_cnt::numeric / XY.user_uu_cnt::numeric, 2) as Support,
  -- Xを買った人がYも買う確率：Confidence
  round(XY.XY_cnt::numeric / XY.X_cnt::numeric, 2) as Confidence,
  -- Xを買った人にYを薦めることで、何もない場合に比べて、Yの購入確率がどれほどあがるのかを表す
  round((XY.XY_cnt::numeric / XY.X_cnt::numeric) / (XY.Y_cnt::numeric / XY.user_uu_cnt::numeric), 2) as Lift
from
  XY
ORDER BY
  XY.X_item,
  XY.Y_item
;

 x_item  | y_item  | x_cnt | y_cnt | xy_cnt | user_uu_cnt | support | confidence | lift
---------+---------+-------+-------+--------+-------------+---------+------------+------
 apple   | banana  |     2 |     8 |      2 |          15 |    0.13 |       1.00 | 1.88
 apple   | beer    |     2 |     6 |      2 |          15 |    0.13 |       1.00 | 2.50
 apple   | diapers |     2 |     7 |      2 |          15 |    0.13 |       1.00 | 2.14
 apple   | orange  |     2 |     3 |      1 |          15 |    0.07 |       0.50 | 2.50
 banana  | apple   |     8 |     2 |      2 |          15 |    0.13 |       0.25 | 1.88
 banana  | beer    |     8 |     6 |      3 |          15 |    0.20 |       0.38 | 0.94
 banana  | diapers |     8 |     7 |      3 |          15 |    0.20 |       0.38 | 0.80
 banana  | orange  |     8 |     3 |      1 |          15 |    0.07 |       0.13 | 0.63
 beer    | apple   |     6 |     2 |      2 |          15 |    0.13 |       0.33 | 2.50
 beer    | banana  |     6 |     8 |      3 |          15 |    0.20 |       0.50 | 0.94
 beer    | diapers |     6 |     7 |      4 |          15 |    0.27 |       0.67 | 1.43
 beer    | orange  |     6 |     3 |      1 |          15 |    0.07 |       0.17 | 0.83
 diapers | apple   |     7 |     2 |      2 |          15 |    0.13 |       0.29 | 2.14
 diapers | banana  |     7 |     8 |      3 |          15 |    0.20 |       0.43 | 0.80
 diapers | beer    |     7 |     6 |      4 |          15 |    0.27 |       0.57 | 1.43
 diapers | orange  |     7 |     3 |      1 |          15 |    0.07 |       0.14 | 0.71
 orange  | apple   |     3 |     2 |      1 |          15 |    0.07 |       0.33 | 2.50
 orange  | banana  |     3 |     8 |      1 |          15 |    0.07 |       0.33 | 0.63
 orange  | beer    |     3 |     6 |      1 |          15 |    0.07 |       0.33 | 0.83
 orange  | diapers |     3 |     7 |      1 |          15 |    0.07 |       0.33 | 0.71
(20 rows)
```

![20200124125307](https://user-images.githubusercontent.com/65038325/186362861-7e94897e-2c93-4d0e-bcbe-6bb7ac077e21.png)

## :closed_book: Reference

None
