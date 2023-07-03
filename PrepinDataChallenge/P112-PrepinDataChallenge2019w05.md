## :memo: Overview

Preppin' Data challenge の「2019: Week 5」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/03/2019-week-5.html)
- [Answer](https://preppindata.blogspot.com/2019/03/2019-week-5-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。クレームの問い合せデータがあって、そこから問い合わせ方法や、苦情内容をルールにそって分類するというお題。

```sql
select * from p2019w05t1;
   date    | customer_id |                                                                               notes
-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Monday    |       29439 | Called about their policy #4899. Wanted to know the balance.
 Tuesday   |       39822 | Called regarding policy #4030. Change of Address requested
 Tuesday   |       83219 | Emailed about the recommendation scheme. Wants to get the bonus for Customer 23813
 Tuesday   |       27316 | Called complaining. Had to wait on the line and called back multiple times. Phone lines are too busy apparently. Gave $40 to shut them up. Added to account #3001
 Wednesday |       12219 | Email about #2001. Raised a complaint.
 Thursday  |       39822 | Called about policy #4030. Gave the wrong postcode! Jeez!
 Thursday  |       49291 | Email asking to Change Address. Change made to policy #9220
 Friday    |       40201 | Emailed requesting a new savings account. Policy created #6090
(8 rows)

select * from p2019w05t2;
   date    | customer_id |                                                         notes
-----------+-------------+------------------------------------------------------------------------------------------------------------------------
 Monday    |       72617 | Emailed. #2080 asking for a statement
 Monday    |       39822 | Call came in correcting the incorrect corrected postcode. #4030 updated and I've sent the client a link to an optician
 Monday    |       29439 | Called about their policy #4899. Wanted to know the balance.
 Tuesday   |       29439 | Called about incorrect balance on policy #4899
 Tuesday   |       12219 | Email about #2001. Complaint about complaint note being dealt with fast enough.
 Wednesday |       34399 | Email regarding #4002. Statement requested.
 Thursday  |       99999 | Call. Wrong number.
 Thursday  |       99999 | Call. Still the wrong number
 Thursday  |       99999 | Email. Asking for the correct number for Pizza Junction.
 Friday    |       29439 | Email about #4899. Customer thanking me for my awesome work. Apparently I rock!
 Friday    |       72617 | Email. Wants to close policy #2080.
(11 rows)
```

解答の SQL はこれ。細かい部分で面倒な処理が入っているので、それは以降でまとめている。一部、これで大丈夫なのか？という処理もあるが、データの入力内容が特定の形式に従っていると仮定する。

```sql
with tmp as (
select *, 'Week commencing 17th June 19' as tablename from p2019w05t1
union all
select *, 'Week commencing 24th June 19' as tablename from p2019w05t2
), tmp2 as (
select
    customer_id,
    to_date(replace(tablename, 'Week commencing ', ''), 'DDth　MON　YY') +
    case date
        when 'Monday' then 0
        when 'Tuesday' then 1
        when 'Wednesday' then 2
        when 'Thursday' then 3
        when 'Friday' then 4
        when 'Saturday' then 5
        when 'Sunday' then 6
    else null end as date_mod,
    substring(notes, '.#(\d+)') as policy_number,
    case when lower(notes) like '%email%' then 'email' else 'call' end  as contact_method,
    case when lower(notes) like '%statement%' then 1 else 0 end  as is_statement,
    case when lower(notes) like '%balance%' then 1 else 0 end  as is_balance,
    case when lower(notes) like '%complain%' then 1 else 0 end  as is_complain
from
    tmp
)
select * from tmp2
;

 customer_id |  date_mod  | policy_number | contact_method | is_statement | is_balance | is_complain
-------------+------------+---------------+----------------+--------------+------------+-------------
       29439 | 2019-06-17 | 4899          | call           |            0 |          1 |           0
       39822 | 2019-06-18 | 4030          | call           |            0 |          0 |           0
       83219 | 2019-06-18 | (null)        | email          |            0 |          0 |           0
       27316 | 2019-06-18 | 3001          | call           |            0 |          0 |           1
       12219 | 2019-06-19 | 2001          | email          |            0 |          0 |           1
       39822 | 2019-06-20 | 4030          | call           |            0 |          0 |           0
       49291 | 2019-06-20 | 9220          | email          |            0 |          0 |           0
       40201 | 2019-06-21 | 6090          | email          |            0 |          0 |           0
       72617 | 2019-06-24 | 2080          | email          |            1 |          0 |           0
       39822 | 2019-06-24 | 4030          | call           |            0 |          0 |           0
       29439 | 2019-06-24 | 4899          | call           |            0 |          1 |           0
       29439 | 2019-06-25 | 4899          | call           |            0 |          1 |           0
       12219 | 2019-06-25 | 2001          | email          |            0 |          0 |           1
       34399 | 2019-06-26 | 4002          | email          |            1 |          0 |           0
       99999 | 2019-06-27 | (null)        | call           |            0 |          0 |           0
       99999 | 2019-06-27 | (null)        | call           |            0 |          0 |           0
       99999 | 2019-06-27 | (null)        | email          |            0 |          0 |           0
       29439 | 2019-06-28 | 4899          | email          |            0 |          0 |           0
       72617 | 2019-06-28 | 2080          | email          |            0 |          0 |           0
(19 rows)
```

まずは、TableauPrep でユニオンすると、`tablename`というカラムがファイル名を識別子として作成される。この文字列を週の開始日がファイル名として使われているので、これを処理して週の開始日を計算する。

```sql
with tmp as (
select *, 'Week commencing 17th June 19' as tablename from p2019w05t1
union all
select *, 'Week commencing 24th June 19' as tablename from p2019w05t2
)
select
    date,
    customer_id,
    -- notes,
    tablename,
    substring(replace(tablename, 'Week commencing ', ''), '\d+') ||
        substring(replace(tablename, 'Week commencing ', ''), '(\s.*)'),
    to_date(
        substring(replace(tablename, 'Week commencing ', ''), '\d+') ||
        substring(replace(tablename, 'Week commencing ', ''), '(\s.*)'),
         'DD　MON　YY') as week_start_date,
    to_date(replace(tablename, 'Week commencing ', ''), 'DDth　MON　YY') as week_start_date2
from
    tmp
;

   date    | customer_id |          tablename           |  ?column?  | week_start_date | week_start_date2
-----------+-------------+------------------------------+------------+-----------------+------------------
 Monday    |       29439 | Week commencing 17th June 19 | 17 June 19 | 2019-06-17      | 2019-06-17
 Tuesday   |       39822 | Week commencing 17th June 19 | 17 June 19 | 2019-06-17      | 2019-06-17
 Tuesday   |       83219 | Week commencing 17th June 19 | 17 June 19 | 2019-06-17      | 2019-06-17
 Tuesday   |       27316 | Week commencing 17th June 19 | 17 June 19 | 2019-06-17      | 2019-06-17
 Wednesday |       12219 | Week commencing 17th June 19 | 17 June 19 | 2019-06-17      | 2019-06-17
 Thursday  |       39822 | Week commencing 17th June 19 | 17 June 19 | 2019-06-17      | 2019-06-17
 Thursday  |       49291 | Week commencing 17th June 19 | 17 June 19 | 2019-06-17      | 2019-06-17
 Friday    |       40201 | Week commencing 17th June 19 | 17 June 19 | 2019-06-17      | 2019-06-17
 Monday    |       72617 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Monday    |       39822 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Monday    |       29439 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Tuesday   |       29439 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Tuesday   |       12219 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Wednesday |       34399 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Thursday  |       99999 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Thursday  |       99999 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Thursday  |       99999 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Friday    |       29439 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
 Friday    |       72617 | Week commencing 24th June 19 | 24 June 19 | 2019-06-24      | 2019-06-24
(19 rows)
```

週の開始日なので、問い合わせがあった日付ではないので、曜日を利用して、週の開始日を問い合わせ日に変更する。

```sql
with tmp as (
select *, 'Week commencing 17th June 19' as tablename from p2019w05t1
union all
select *, 'Week commencing 24th June 19' as tablename from p2019w05t2
), tmp2 as (
select
    date,
    customer_id,
    notes,
    to_date(replace(tablename, 'Week commencing ', ''), 'DDth　MON　YY') as week_start_date
from
    tmp
), tmp3 as (
select
    date,
    customer_id,
    -- notes,
    week_start_date,
    case date
        when 'Monday' then 0
        when 'Tuesday' then 1
        when 'Wednesday' then 2
        when 'Thursday' then 3
        when 'Friday' then 4
        when 'Saturday' then 5
        when 'Sunday' then 6
    else null end asadddate,
    week_start_date +
    case date
        when 'Monday' then 0
        when 'Tuesday' then 1
        when 'Wednesday' then 2
        when 'Thursday' then 3
        when 'Friday' then 4
        when 'Saturday' then 5
        when 'Sunday' then 6
    else null end as date_mod
from
    tmp2
)
select * from tmp3
;

   date    | customer_id | week_start_date | asadddate |  date_mod
-----------+-------------+-----------------+-----------+------------
 Monday    |       29439 | 2019-06-17      |         0 | 2019-06-17
 Tuesday   |       39822 | 2019-06-17      |         1 | 2019-06-18
 Tuesday   |       83219 | 2019-06-17      |         1 | 2019-06-18
 Tuesday   |       27316 | 2019-06-17      |         1 | 2019-06-18
 Wednesday |       12219 | 2019-06-17      |         2 | 2019-06-19
 Thursday  |       39822 | 2019-06-17      |         3 | 2019-06-20
 Thursday  |       49291 | 2019-06-17      |         3 | 2019-06-20
 Friday    |       40201 | 2019-06-17      |         4 | 2019-06-21
 Monday    |       72617 | 2019-06-24      |         0 | 2019-06-24
 Monday    |       39822 | 2019-06-24      |         0 | 2019-06-24
 Monday    |       29439 | 2019-06-24      |         0 | 2019-06-24
 Tuesday   |       29439 | 2019-06-24      |         1 | 2019-06-25
 Tuesday   |       12219 | 2019-06-24      |         1 | 2019-06-25
 Wednesday |       34399 | 2019-06-24      |         2 | 2019-06-26
 Thursday  |       99999 | 2019-06-24      |         3 | 2019-06-27
 Thursday  |       99999 | 2019-06-24      |         3 | 2019-06-27
 Thursday  |       99999 | 2019-06-24      |         3 | 2019-06-27
 Friday    |       29439 | 2019-06-24      |         4 | 2019-06-28
 Friday    |       72617 | 2019-06-24      |         4 | 2019-06-28
(19 rows)
```

あとは問い合わせ内容から必要な情報を取り出せば OK。

```sql
with tmp as (
select *, 'Week commencing 17th June 19' as tablename from p2019w05t1
union all
select *, 'Week commencing 24th June 19' as tablename from p2019w05t2
), tmp2 as (
select
    date,
    customer_id,
    notes,
    to_date(replace(tablename, 'Week commencing ', ''), 'DDth　MON　YY') as week_start_date
from
    tmp
), tmp3 as (
select
    date,
    customer_id,
    -- notes,
    week_start_date +
    case date
        when 'Monday' then 0
        when 'Tuesday' then 1
        when 'Wednesday' then 2
        when 'Thursday' then 3
        when 'Friday' then 4
        when 'Saturday' then 5
        when 'Sunday' then 6
    else null end as date_mod,
    substring(notes, '.#(\d+)') as policy_number,
    case when lower(notes) like '%email%' then 'email' else 'call' end  as contact_method,
    case when lower(notes) like '%statement%' then 1 else 0 end  as is_statement,
    case when lower(notes) like '%balance%' then 1 else 0 end  as is_balance,
    case when lower(notes) like '%complain%' then 1 else 0 end  as is_complain
from
    tmp2
)
select * from tmp3
;

   date    | customer_id |  date_mod  | policy_number | contact_method | is_statement | is_balance | is_complain
-----------+-------------+------------+---------------+----------------+--------------+------------+-------------
 Monday    |       29439 | 2019-06-17 | 4899          | call           |            0 |          1 |           0
 Tuesday   |       39822 | 2019-06-18 | 4030          | call           |            0 |          0 |           0
 Tuesday   |       83219 | 2019-06-18 | (null)        | email          |            0 |          0 |           0
 Tuesday   |       27316 | 2019-06-18 | 3001          | call           |            0 |          0 |           1
 Wednesday |       12219 | 2019-06-19 | 2001          | email          |            0 |          0 |           1
 Thursday  |       39822 | 2019-06-20 | 4030          | call           |            0 |          0 |           0
 Thursday  |       49291 | 2019-06-20 | 9220          | email          |            0 |          0 |           0
 Friday    |       40201 | 2019-06-21 | 6090          | email          |            0 |          0 |           0
 Monday    |       72617 | 2019-06-24 | 2080          | email          |            1 |          0 |           0
 Monday    |       39822 | 2019-06-24 | 4030          | call           |            0 |          0 |           0
 Monday    |       29439 | 2019-06-24 | 4899          | call           |            0 |          1 |           0
 Tuesday   |       29439 | 2019-06-25 | 4899          | call           |            0 |          1 |           0
 Tuesday   |       12219 | 2019-06-25 | 2001          | email          |            0 |          0 |           1
 Wednesday |       34399 | 2019-06-26 | 4002          | email          |            1 |          0 |           0
 Thursday  |       99999 | 2019-06-27 | (null)        | call           |            0 |          0 |           0
 Thursday  |       99999 | 2019-06-27 | (null)        | call           |            0 |          0 |           0
 Thursday  |       99999 | 2019-06-27 | (null)        | email          |            0 |          0 |           0
 Friday    |       29439 | 2019-06-28 | 4899          | email          |            0 |          0 |           0
 Friday    |       72617 | 2019-06-28 | 2080          | email          |            0 |          0 |           0
(19 rows)
```

## :closed_book: Reference

None
