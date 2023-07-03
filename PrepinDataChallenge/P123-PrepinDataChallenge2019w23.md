## :memo: Overview

Preppin' Data challenge の「2019: Week 23」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/07/2019-week-23.html)
- [Answer](https://preppindata.blogspot.com/2019/07/2019-week-23-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。

```sql
select * from p2019w23t1 limit 10;
    day    |                      notes
-----------+-------------------------------------------------
 Monday    | MRS THEODORE WANTS £2500 OF JASMINE LIQUID SOAP
 Monday    | mr smith wants £400 of lavender soap bars
 Tuesday   | mr rankin wants £250 of jasmine liquid soap
 Wednesday | MR BUMBLE WANTS £4000 OF HONEY SOAP BARS
 Friday    | MRS FLORAL WANTS £200 OF LAVENDER SOAP BARS
 Friday    | mrs bumble wants £100 of honey liquid soap
(6 rows)

select * from p2019w23t2 limit 10;
    day    |                     notes
-----------+------------------------------------------------
 Monday    | mrs bumble wants £1000 of honey soap bars
 Tuesday   | MR BUMBLE WANTS £250 OF HONEY LIQUID SOAP
 Tuesday   | SIR LORDY WANTS £5000 OF LAVENDER SOAP BARS
 Wednesday | mr smith wants £250 of lavender liquid soap
 Friday    | mrs theodore wants £250 of jasmine liquid soap
(5 rows)

select * from p2019w23t3 limit 10;
    day    |                     notes                      |
-----------+------------------------------------------------+
 Monday    | mr adam wants £200 of jasmine soap bars        |
 Monday    | ms martin wants £400 of lavender liquid soap   |
 Tuesday   | mr allchin wants £100 of espresso coffee beans |
 Tuesday   | LADY LORDY WANTS £2000 OF JASMINE SOAP BARS    |
 Wednesday | MR SMITH WANTS £400 OF LAVENDER LIQUID SOAP    |
 Friday    | MRS BUMBLE WANTS £1000 OF HONEY LIQUID SOAP    |
 Friday    | MS COHEN WANTS £250 OF LEMONGRASS SOAP BARS    |
(7 rows)

```

```sql
with tmp as (
select day, notes from p2019w23t1
union all
select day, notes from p2019w23t2
union all
select day, notes from p2019w23t3
), tmp2 as (
select * from tmp
)
select * from tmp2



```

## :closed_book: Reference

None
