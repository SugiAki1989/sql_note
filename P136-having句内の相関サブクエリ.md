## :memo: Overview

[Effective SQL](https://www.shoeisha.co.jp/book/detail/9784798153995)のP133を読んでいて、理解に苦しんだサブクエリがあったので、その時のメモ。


## :floppy_disk: Database

MySQL

## :bookmark: Tag

`subquery`

## :pencil2: SQL実行の流れ

日頃からサブクエリは避けて`with`句を使っていたので、サブクエリで記述している内容が理解できなかった。EFFECTIVE SQLのP133には下記のSQLが紹介されている。このクエリの目的は「2015年Q4において、そのカテゴリの平均よりも売れた商品をリスト化」するためである。

```sql
SET search_path = SalesOrdersSample;

SELECT 
  C.CategoryDescription, P.ProductName, 
  SUM(OD.QuotedPrice * OD.QuantityOrdered) AS TotalSales
FROM Products AS P 
  INNER JOIN Order_Details AS OD 
     ON P.ProductNumber=OD.ProductNumber
  INNER JOIN Categories AS C
     ON C.CategoryID = P.CategoryID
  INNER JOIN Orders AS O
     ON O.OrderNumber = OD.OrderNumber
WHERE 
  O.OrderDate BETWEEN '2015-10-01' AND '2015-12-31'
  --サブクエリの＊と対応している
  --カテゴリ条件はない
GROUP BY 
  P.CategoryID, C.CategoryDescription, P.ProductName
HAVING 
SUM(OD.QuotedPrice * OD.QuantityOrdered) > 
  (
    SELECT
      AVG(SumCategory) 
    FROM 
       (
          SELECT 
            P2.CategoryID, 
            SUM(OD2.QuotedPrice * OD2.QuantityOrdered) AS SumCategory 
          FROM 
            Products AS P2 
            INNER JOIN Order_Details AS OD2 
              ON P2.ProductNumber = OD2.ProductNumber 
            INNER JOIN Orders AS O2
              ON O2.OrderNumber = OD2.OrderNumber
          WHERE 
            O2.OrderDate BETWEEN '2015-10-01' AND '2015-12-31'
            -- メインクエリの＊と対応している
            AND P2.CategoryID = P.CategoryID
          GROUP BY 
            P2.CategoryID, P2.ProductNumber
          ORDER BY
            P2.CategoryID ASC
       ) AS S 
    GROUP BY CategoryID
  )
ORDER BY CategoryDescription, ProductName;

 categorydescription |           productname            | totalsales
---------------------+----------------------------------+-------------
 Accessories         | Cycle-Doc Pro Repair Stand       |  32595.7600
 Accessories         | Dog Ear Aero-Flow Floor Pump     |  15539.1500
 Accessories         | Glide-O-Matic Cycling Helmet     |  23640.0000
 Accessories         | King Cobra Helmet                |  27847.2600
 Accessories         | Viscount CardioSport Sport Watch |  16469.7900
 Bikes               | GT RTS-2 Mountain Bike           | 527703.0000
 Bikes               | Trek 9000 Mountain Bike          | 954516.0000
 Car racks           | Ultimate Export 2G Car Rack      |  31014.0000
 Clothing            | StaDry Cycling Pants             |   8641.5600
 Components          | AeroFlo ATB Wheels               |  37709.2800
 Components          | Cosmic Elite Road Warrior Wheels |  32064.4500
 Components          | Eagle SA-120 Clipless Pedals     |  17003.8500
 Skateboards         | Viscount Skateboard              | 196964.3000
 Tires               | Ultra-2K Competition Tire        |   5216.2800
(14 rows)
```

目的は複雑ではないので、読みにくいと思いながらも、サブクエリの内側から手を動かしながら、読み進めいていくと、サブクエリの洗礼に早速出会う。`P2.CategoryID = P.CategoryID`がエラーで`P`テーブルが存在しないと言われる。

```sql
SELECT P2.CategoryID, 
 SUM(OD2.QuotedPrice * OD2.QuantityOrdered) AS SumCategory 
FROM Products AS P2 
INNER JOIN Order_Details AS OD2 
  ON P2.ProductNumber = OD2.ProductNumber 
INNER JOIN Orders AS O2
  ON O2.OrderNumber = OD2.OrderNumber
WHERE P2.CategoryID = P.CategoryID
AND O2.OrderDate BETWEEN '2015-10-01' AND '2015-12-31'
GROUP BY P2.CategoryID, P2.ProductNumber
;

ERROR:  missing FROM-clause entry for table "p"
LINE 8: WHERE P2.CategoryID = P.CategoryID
```

つまり、`P`テーブルが存在していないとこのクエリの検証はできない。そもそも`P`テーブルはメインクエリに存在している。

```sql
SELECT C.CategoryDescription, P.ProductName, 
  SUM(OD.QuotedPrice * OD.QuantityOrdered) AS TotalSales
FROM Products AS P 
(snip)
GROUP BY CategoryID)
;
```

つまり、サブクエリを検証するには、`P`テーブルを定義する必要がある。ただ、このサブクエリは`having`句内で定義されているので、簡単に書き直すのは手間だし、そもそも無理な気がする。

```sql
SELECT C.CategoryDescription, P.ProductName, 
  SUM(OD.QuotedPrice * OD.QuantityOrdered) AS TotalSales
FROM Products AS P 
WHERE O.OrderDate BETWEEN '2015-10-01' AND '2015-12-31'
GROUP BY P.CategoryID, C.CategoryDescription, P.ProductName
HAVING SUM(OD.QuotedPrice * OD.QuantityOrdered) > 
  (SELECT AVG(SumCategory) 
   FROM 
     (SELECT P2.CategoryID, 
       SUM(OD2.QuotedPrice * OD2.QuantityOrdered) 
       AS SumCategory 
      FROM Products AS P2 
      INNER JOIN Order_Details AS OD2 
        ON P2.ProductNumber = OD2.ProductNumber 
      INNER JOIN Orders AS O2
        ON O2.OrderNumber = OD2.OrderNumber
      WHERE P2.CategoryID = P.CategoryID
      AND O2.OrderDate BETWEEN '2015-10-01' AND '2015-12-31'
      GROUP BY P2.CategoryID, P2.ProductNumber) AS S 
GROUP BY CategoryID)
;
```

そもそも`P2.CategoryID = P.CategoryID`の必要性は、サブクエリで集計対象となるカテゴリーを、メインクエリで選択したカテゴリーに一致させるために必要となる(相関サブクエリ)。相関サブクエリについは、「P018-0-相関サブクエリ」を参照。

言葉では理解しにくいので、下記をみるとわかりよい。ただ、これは、カテゴリをメインクエリで限定していないにも関わらず必要である。あと、`having`でベクトルを利用するのは一般的なんだろうか。

```sql
SELECT 
  C.CategoryDescription, P.ProductName, 
  SUM(OD.QuotedPrice * OD.QuantityOrdered) AS TotalSales
FROM Products AS P 
  INNER JOIN Order_Details AS OD 
     ON P.ProductNumber=OD.ProductNumber
  INNER JOIN Categories AS C
     ON C.CategoryID = P.CategoryID
  INNER JOIN Orders AS O
     ON O.OrderNumber = OD.OrderNumber
WHERE 
  O.OrderDate BETWEEN '2015-10-01' AND '2015-12-31'
  -- サブクエリの＊と対応している
  AND P.CategoryID = 1
GROUP BY 
  P.CategoryID, C.CategoryDescription, P.ProductName
HAVING 
SUM(OD.QuotedPrice * OD.QuantityOrdered) > 
  (
    SELECT
      AVG(SumCategory) 
    FROM 
       (
          SELECT 
            P2.CategoryID, 
            SUM(OD2.QuotedPrice * OD2.QuantityOrdered) AS SumCategory 
          FROM 
            Products AS P2 
            INNER JOIN Order_Details AS OD2 
              ON P2.ProductNumber = OD2.ProductNumber 
            INNER JOIN Orders AS O2
              ON O2.OrderNumber = OD2.OrderNumber
          WHERE 
            O2.OrderDate BETWEEN '2015-10-01' AND '2015-12-31'
            -- メインクエリの＊と対応している
            -- メインクエリのカテゴリ条件とあわせる
            AND P2.CategoryID = 1
          GROUP BY 
            P2.CategoryID, P2.ProductNumber
          ORDER BY
            P2.CategoryID ASC
       ) AS S 
    GROUP BY CategoryID
  )
ORDER BY CategoryDescription, ProductName;

 categorydescription |           productname            | totalsales
---------------------+----------------------------------+------------
 Accessories         | Cycle-Doc Pro Repair Stand       | 32595.7600
 Accessories         | Dog Ear Aero-Flow Floor Pump     | 15539.1500
 Accessories         | Glide-O-Matic Cycling Helmet     | 23640.0000
 Accessories         | King Cobra Helmet                | 27847.2600
 Accessories         | Viscount CardioSport Sport Watch | 16469.7900
(5 rows)
```

`with`句で書き直すとこのようになる。個人的な感想ではあるが、`with`句の方が、その他プラグラミング言語同様に上から下に処理が実行される流れがわかりやすく、テーブルの処理や組み上がりがイメージしやすい。確かにクエリが長くなり、パフォーマンスの観点のデメリットはあるが、その分、クエリを理解する時間がへるので、相当なサブクエリ使い以外は、トレードオフな気もする。

```sql
WITH CatProdData AS (
    SELECT 
        C.CategoryID, C.CategoryDescription, P.ProductName, OD.QuotedPrice, OD.QuantityOrdered
    FROM 
        Products AS P 
        INNER JOIN Order_Details AS OD ON P.ProductNumber = OD.ProductNumber
        INNER JOIN Categories AS C ON C.CategoryID = P.CategoryID
        INNER JOIN Orders AS O ON O.OrderNumber = OD.OrderNumber
    WHERE 
        O.OrderDate BETWEEN DATE '2015-10-01' AND DATE '2015-12-31'
)
-- select * from CatProdData limit 10;
--  categoryid | categorydescription |         productname         | quotedprice | quantityordered
-- ------------+---------------------+-----------------------------+-------------+-----------------
--           2 | Bikes               | Trek 9000 Mountain Bike     |   1164.0000 |               6
--           3 | Clothing            | StaDry Cycling Pants        |     66.9300 |               5
--           2 | Bikes               | Trek 9000 Mountain Bike     |   1164.0000 |               6
--           7 | Skateboards         | Viscount Skateboard         |    615.9500 |               5
--           2 | Bikes               | GT RTS-2 Mountain Bike      |   1600.5000 |               6
--           1 | Accessories         | Dog Ear Monster Grip Gloves |     14.5500 |               6
--           1 | Accessories         | King Cobra Helmet           |    139.0000 |               4
--           1 | Accessories         | Ultra-Pro Knee Pads         |     45.0000 |               2
--           1 | Accessories         | HP Deluxe Panniers          |     37.8300 |               6
--           5 | Car racks           | Ultimate Export 2G Car Rack |    174.6000 |               6
-- (10 rows)
, CategorySales AS (
    SELECT CategoryID, CategoryDescription, ProductName, SUM(QuotedPrice * QuantityOrdered) AS TotalSales
    FROM CatProdData 
    GROUP BY CategoryID, CategoryDescription, ProductName
)
-- select * from CategorySales limit 30;
--  categoryid | categorydescription |              productname              | totalsales
-- ------------+---------------------+---------------------------------------+-------------
--           1 | Accessories         | Cycle-Doc Pro Repair Stand            |  32595.7600
--           1 | Accessories         | Dog Ear Aero-Flow Floor Pump          |  15539.1500
-- (snip)
--           1 | Accessories         | Viscount Microshell Helmet            |   2052.3600
--           1 | Accessories         | Viscount Tru-Beat Heart Transmitter   |   6932.9700
--           2 | Bikes               | Eagle FS-3 Mountain Bike              |  40860.0000
--           2 | Bikes               | GT RTS-2 Mountain Bike                | 527703.0000
--           2 | Bikes               | Trek 9000 Mountain Bike               | 954516.0000
--           3 | Clothing            | Kool-Breeze Rocket Top Jersey         |   4427.5200
--           3 | Clothing            | StaDry Cycling Pants                  |   8641.5600
--           3 | Clothing            | Wonder Wool Cycle Socks               |   2921.8200
--           4 | Components          | AeroFlo ATB Wheels                    |  37709.2800
--           4 | Components          | Cosmic Elite Road Warrior Wheels      |  32064.4500
--           4 | Components          | Eagle SA-120 Clipless Pedals          |  17003.8500
--           4 | Components          | ProFormance Toe-Klips 2G              |    502.3800
--           4 | Components          | Shinoman 105 SC Brakes                |   1180.3000
-- (30 rows)
, AverageCategorySalesPre AS (
    SELECT CategoryID, ProductName, SUM(QuotedPrice * QuantityOrdered) AS SumCategory 
    FROM CatProdData
    GROUP BY CategoryID, ProductName
)
, AverageCategorySales AS (
    SELECT CategoryID, AVG(SumCategory) AS AvgCategorySales
    FROM AverageCategorySalesPre
    GROUP BY CategoryID
)
-- select * from AverageCategorySales limit 10;
--  categoryid |   avgcategorysales
-- ------------+-----------------------
--           3 | 5330.3000000000000000
--           5 |    30493.125000000000
--           4 |    16193.426666666667
--           6 | 3942.2333333333333333
--           2 |   507693.000000000000
--           7 |   104473.920000000000
--           1 | 9726.6615789473684211
-- (7 rows)
SELECT 
    CS.CategoryDescription, CS.ProductName, CS.TotalSales, ACS.AvgCategorySales,
    case when CS.TotalSales > ACS.AvgCategorySales then 'having is true' else null end as check
FROM 
    CategorySales AS CS 
        INNER JOIN AverageCategorySales AS ACS ON CS.CategoryID = ACS.CategoryID
ORDER BY 
    CS.CategoryDescription, CS.ProductName
;
 categorydescription |              productname              | totalsales  |   avgcategorysales    |     check
---------------------+---------------------------------------+-------------+-----------------------+----------------
 Accessories         | Cycle-Doc Pro Repair Stand            |  32595.7600 | 9726.6615789473684211 | having is true
 Accessories         | Dog Ear Aero-Flow Floor Pump          |  15539.1500 | 9726.6615789473684211 | having is true
 Accessories         | Dog Ear Cyclecomputer                 |    888.7500 | 9726.6615789473684211 |
 Accessories         | Dog Ear Helmet Mount Mirrors          |    532.8500 | 9726.6615789473684211 |
 Accessories         | Dog Ear Monster Grip Gloves           |   1532.5500 | 9726.6615789473684211 |
 Accessories         | Glide-O-Matic Cycling Helmet          |  23640.0000 | 9726.6615789473684211 | having is true
 Accessories         | HP Deluxe Panniers                    |   8154.9000 | 9726.6615789473684211 |
 Accessories         | King Cobra Helmet                     |  27847.2600 | 9726.6615789473684211 | having is true
 Accessories         | Kryptonite Advanced 2000 U-Lock       |   3188.5000 | 9726.6615789473684211 |
 Accessories         | Nikoma Lok-Tight U-Lock               |   5763.1200 | 9726.6615789473684211 |
 Accessories         | Pro-Sport 'Dillo Shades               |   9294.7000 | 9726.6615789473684211 |
 Accessories         | ProFormance Knee Pads                 |   6636.8400 | 9726.6615789473684211 |
 Accessories         | TransPort Bicycle Rack                |   3925.2600 | 9726.6615789473684211 |
 Accessories         | True Grip Competition Gloves          |   3297.1400 | 9726.6615789473684211 |
 Accessories         | Ultra-Pro Knee Pads                   |   8398.8000 | 9726.6615789473684211 |
 Accessories         | Viscount C-500 Wireless Bike Computer |   8115.8700 | 9726.6615789473684211 |
 Accessories         | Viscount CardioSport Sport Watch      |  16469.7900 | 9726.6615789473684211 | having is true
 Accessories         | Viscount Microshell Helmet            |   2052.3600 | 9726.6615789473684211 |
 Accessories         | Viscount Tru-Beat Heart Transmitter   |   6932.9700 | 9726.6615789473684211 |
 Bikes               | Eagle FS-3 Mountain Bike              |  40860.0000 |   507693.000000000000 |
 Bikes               | GT RTS-2 Mountain Bike                | 527703.0000 |   507693.000000000000 | having is true
 Bikes               | Trek 9000 Mountain Bike               | 954516.0000 |   507693.000000000000 | having is true
 Car racks           | Road Warrior Hitch Pack               |  29972.2500 |    30493.125000000000 |
 Car racks           | Ultimate Export 2G Car Rack           |  31014.0000 |    30493.125000000000 | having is true
 Clothing            | Kool-Breeze Rocket Top Jersey         |   4427.5200 | 5330.3000000000000000 |
 Clothing            | StaDry Cycling Pants                  |   8641.5600 | 5330.3000000000000000 | having is true
 Clothing            | Wonder Wool Cycle Socks               |   2921.8200 | 5330.3000000000000000 |
 Components          | AeroFlo ATB Wheels                    |  37709.2800 |    16193.426666666667 | having is true
 Components          | Cosmic Elite Road Warrior Wheels      |  32064.4500 |    16193.426666666667 | having is true
 Components          | Eagle SA-120 Clipless Pedals          |  17003.8500 |    16193.426666666667 | having is true
 Components          | ProFormance Toe-Klips 2G              |    502.3800 |    16193.426666666667 |
 Components          | Shinoman 105 SC Brakes                |   1180.3000 |    16193.426666666667 |
 Components          | Shinoman Deluxe TX-30 Pedal           |   8700.3000 |    16193.426666666667 |
 Skateboards         | Shinoman Skateboard                   |  11983.5400 |   104473.920000000000 |
 Skateboards         | Viscount Skateboard                   | 196964.3000 |   104473.920000000000 | having is true
 Tires               | Turbo Twin Tires                      |   3404.0200 | 3942.2333333333333333 |
 Tires               | Ultra-2K Competition Tire             |   5216.2800 | 3942.2333333333333333 | having is true
 Tires               | X-Pro All Weather Tires               |   3206.4000 | 3942.2333333333333333 |
(38 rows)

```

## :closed_book: Reference

- [Effective SQL](https://www.shoeisha.co.jp/book/detail/9784798153995)
- [Listing 5.015.sql](https://github.com/TexanInParis/Effective-SQL/blob/master/PostgreSQL/Chapter%2005/Listing%205.015.sql)
