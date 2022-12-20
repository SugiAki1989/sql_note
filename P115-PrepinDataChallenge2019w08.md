## :memo: Overview

Preppin' Data challenge の「2019: Week 8」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/04/2019-week-8.html)
- [Answer](https://preppindata.blogspot.com/2019/04/2019-week-8-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。

```sql
select * from p2019w08t1;

   type   |     action     |    date    | quantity | store_id | crime_ref_number
----------+----------------+------------+----------+----------+------------------
 Bar      | Theft          | 19/03/2019 |       10 | OX1      | S04P1
 Bar      | Stock Adjusted | 25/03/2019 |      -10 | OX1      | S04P1
 Bar      | Theft          | 22/03/2019 |        5 | OX1      | S04P2
 Bar      | Stock Adjusted | 25/03/2019 |       -4 | OX1      | S04P2
 Liquid   | Theft          | 02/03/2019 |      100 | WM1      | S04P3
 Liquid   | Stock Adjusted | 02/03/2019 |     -100 | WM1      | S04P3
 Bar      | Theft          | 03/04/2019 |        5 | WM2      | S04P4
 Luquid   | Theft          | 22/03/2019 |       14 | OX1      | S04P5
 Liquid   | Theft          | 02/02/2019 |        2 | WM2      | S04P6
 Liquid   | Stock Adjusted | 03/04/2019 |       -2 | WM2      | S04P6
 Soap Bar | Theft          | 28/02/2019 |       22 | WM1      | S04P7
 Soap Bar | Stock Adjusted | 01/03/2019 |      -20 | WM1      | S04P7
 Liquid   | Theft          | 27/02/2019 |      105 | OX1      | S04P8
 Liquid   | Stock Adjusted | 01/03/2019 |     -105 | OX1      | S04P8
(14 rows)

select * from p2019w08t2;

      branch_id
---------------------
 OX1 - Oxford Street
 WM1 - Wimbledon 1
 WM2 - Wimbledon 2
 ST1 - Stratford
(4 rows)
```

```sql

```

## :closed_book: Reference

None
