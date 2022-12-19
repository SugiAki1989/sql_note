## :memo: Overview

秒数を`HH:MM:SS`形式に変換して表示する方法についてまとめておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`floor`, `round`, `lpad`

## :pencil2: Example

秒数が入っているサンプルデータを用意する。

```sql
create table formatter(ss int);
insert into formatter(ss)
values
    ('0'),
    ('5'),
    ('10'),
    ('30'),
    ('59'),
    ('60'),
    ('61'),
    ('90'),
    ('120'),
    ('3600'),
    ('86400'),
    ('172800'),
    ('172801')
    ;
```

秒数が各時間単位でどれだけに当たるかを計算する。

```sql
select
    ss,
    floor(ss/3600) as h,
    floor(mod((ss/60), 60)) as m,
    round(mod(ss,60)) as s
from
    formatter
;

   ss   | h  | m | s
--------+----+---+----
      0 |  0 | 0 |  0
      5 |  0 | 0 |  5
     10 |  0 | 0 | 10
     30 |  0 | 0 | 30
     59 |  0 | 0 | 59
     60 |  0 | 1 |  0
     61 |  0 | 1 |  1
     90 |  0 | 1 | 30
    120 |  0 | 2 |  0
   3600 |  1 | 0 |  0
  86400 | 24 | 0 |  0
 172800 | 48 | 0 |  0
 172801 | 48 | 0 |  1
(13 rows)
```

あとは見た目の都合上、2 桁揃えになるように 0 埋めを行う。

```sql
select
    ss,
    lpad(floor(ss/3600)::text, 2, '0') as h,
    lpad(floor(mod((ss/60), 60))::text, 2, '0') as m,
    lpad(round(mod(ss,60))::text, 2, '0') as s
from
    formatter
;

   ss   | h  | m  | s
--------+----+----+----
      0 | 00 | 00 | 00
      5 | 00 | 00 | 05
     10 | 00 | 00 | 10
     30 | 00 | 00 | 30
     59 | 00 | 00 | 59
     60 | 00 | 01 | 00
     61 | 00 | 01 | 01
     90 | 00 | 01 | 30
    120 | 00 | 02 | 00
   3600 | 01 | 00 | 00
  86400 | 24 | 00 | 00
 172800 | 48 | 00 | 00
 172801 | 48 | 00 | 01
(13 rows)

```

これを連結すれば、狙ったフォーマットで表示できる。

```sql
select
    ss,
    lpad(floor(ss/3600)::text, 2, '0') || ':' ||
    lpad(floor(mod((ss/60), 60))::text, 2, '0') || ':' ||
    lpad(round(mod(ss,60))::text, 2, '0') as hhmmss
from
    formatter
;

   ss   |  hhmmss
--------+----------
      0 | 00:00:00
      5 | 00:00:05
     10 | 00:00:10
     30 | 00:00:30
     59 | 00:00:59
     60 | 00:01:00
     61 | 00:01:01
     90 | 00:01:30
    120 | 00:02:00
   3600 | 01:00:00
  86400 | 24:00:00
 172800 | 48:00:00
 172801 | 48:00:01
(13 rows)
```

秒数が大きすぎると 2 桁に収まらないので注意。

```sql
select
    604800,
    lpad(floor(604800/3600)::text, 4, '0') || ':' ||
    lpad(floor(mod((604800/60), 60))::text, 2, '0') || ':' ||
    lpad(round(mod(604800,60))::text, 2, '0') as hhmmss
;
 ?column? |   hhmmss
----------+------------
   604800 | 0168:00:00
```

## :closed_book: Reference

None
