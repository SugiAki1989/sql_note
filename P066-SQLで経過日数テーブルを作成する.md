## :memo: Overview

サービス利用履歴データからユーザーのカプランマイヤー曲線(生存曲線)を計算する方法を 2 回にわけてまとめておく。ここでは、まずサービス利用履歴からカプランマイヤー曲線を計算するためのベースとなるテーブルを作成する。次回はそのテーブルを使ってカプランマイヤー曲線を計算する。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

カプランマイヤー曲線を計算するためのベースとなるテーブルには、`userid`で一意となり、生存日数を表す`duration_days`、イベントの発生を示す`churn`の 3 列あればよいので、このカラムを作成していく。今回扱うのは、SNS などのサービスログなので、明確なイベントの発生がわからないため、データの最新日`2017-02-21`から 30 日前の間でサービスの利用がなければ、チャーンしたと判定する。

```sql
create table km_log(
    timestamp timestamp with time zone,
    userid varchar(255),
    os varchar(255),
    country varchar(255)
);

-- csv data from : https://exploratory.io/data/BWz1Bar4JF/Survival-Access-Log-cNT1hYC9KH
\copy km_log from '/Users/aki/Desktop/km_log.csv' (encoding 'utf8', format csv, header true);

select * from km_log limit 10;
       timestamp        |        userid        | os  |    country
------------------------+----------------------+-----+---------------
 2016-05-18 06:04:54+09 | acee2775986fdd62bc51 | Mac | United States
 2016-05-18 06:07:32+09 | ccbba04e6bda228bb538 | Mac | United States
 2016-05-18 06:50:39+09 | 8e0f0538e74f227369c8 | Mac | Ireland
 2016-05-18 07:51:35+09 | ac12f9936963901f617f | Mac | Japan
 2016-05-18 09:10:03+09 | 7034f5aebbaee32c1510 | Mac | Australia
 2016-05-18 10:54:40+09 | 72d1cc6abf6ea213ff5e | Mac | Japan
 2016-05-18 11:05:26+09 | 406d6e9d806e487649bb | Mac | United States
 2016-05-18 12:02:40+09 | c3d5e22f93b5a7979c8c | Mac | United States
 2016-05-18 12:06:15+09 | ed7b17c954bbb2cc4e45 | Mac | Japan
 2016-05-18 12:54:00+09 | b2166bdf22ca03ec79df | Mac | Japan
(10 rows)
```

```sql
create table km_v as (
with tmp as (
    select max(timestamp::date)  as censor_date from km_log
), tmp2 as (
select
    userid as user_id,
    min(timestamp::date) as min_date,
    max(timestamp::date) as max_date
from
    km_log
group by
    userid
)
select
    user_id,
    min_date,
    max_date,
    tmp.censor_date,
    (max_date - min_date) as duration_days,
    tmp.censor_date - max_date as cond_days,
    case when (tmp.censor_date - max_date) > 30 then 1 else 0 end as churn
from
    tmp2
cross join
    tmp
order by
    user_id asc
);
```

難しいことはしていないので、特に説明はないが、処理のイメージを載せておく。

![クイックノート P1](https://user-images.githubusercontent.com/65038325/186905983-8f180ed2-c7b7-45cf-8387-37fbfa5629d9.png)

```sql
select
    *
from
    km_v
where user_id in (
'0014163a53056db3f10e',
'00164e47967b306239e8',
'0016e5b445a29e9cbfcc',
'004536fa45ac76c2642e',
'012e70bd197a5ff0af5b',
'0156a46ca6be1bc36c33')
;

       user_id        |  min_date  |  max_date  | censor_date | duration_days | cond_days | churn
----------------------+------------+------------+-------------+---------------+-----------+-------
 0014163a53056db3f10e | 2016-08-11 | 2016-10-31 | 2017-02-21  |            81 |       113 |     1
 00164e47967b306239e8 | 2016-05-19 | 2016-05-26 | 2017-02-21  |             7 |       271 |     1
 0016e5b445a29e9cbfcc | 2016-06-07 | 2017-02-13 | 2017-02-21  |           251 |         8 |     0
 004536fa45ac76c2642e | 2016-12-23 | 2017-02-02 | 2017-02-21  |            41 |        19 |     0
 012e70bd197a5ff0af5b | 2017-01-27 | 2017-01-27 | 2017-02-21  |             0 |        25 |     0
 0156a46ca6be1bc36c33 | 2016-06-22 | 2016-12-10 | 2017-02-21  |           171 |        73 |     1
(6 rows)

-- csv export
\copy km_v to '~/Desktop/km_v.csv' WITH CSV DELIMITER ',' HEADER;
```

このテーブルを次回、利用する。

## :closed_book: Reference

None
