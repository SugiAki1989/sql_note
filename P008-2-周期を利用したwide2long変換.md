## :memo: Overview

周期のある横長データを周期から導かれる規則を使うことで、縦場変換を行う。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`union`

## :pencil2: Example

アクセス時間と曜日のクロス表のヒートマップのようなデータがあるとする。ここでは 24 時間を 4 つに分割して列ごとに管理しているとする。つまり 1 行目は 0~3 時、2 行目は 4~7 時という感じ。

```sql
create table yokonaga (s int, t1 int, t2 int, t3 int, t4 int);

insert into yokonaga
  (s, t1, t2, t3, t4)
values
  ('1', '3', '1', '2', '0'),
  ('2', '1', '5', '1', '7'),
  ('3', '4', '4', '6', '5'),
  ('4', '5', '3', '3', '4'),
  ('5', '7', '1', '7', '2'),
  ('6', '3', '2', '2', '4');
;

 s | t1 | t2 | t3 | t4
---+----+----+----+----
 1 |  3 |  1 |  2 |  0
 2 |  1 |  5 |  1 |  7
 3 |  4 |  4 |  6 |  5
 4 |  5 |  3 |  3 |  4
 5 |  7 |  1 |  7 |  2
 6 |  3 |  2 |  2 |  4
(6 rows)
```

このテーブルは周期があるので、その周期を利用すると横長の扱いにくい状態から扱いやすい縦長に簡単に変換できる。左上の`t1=3`が 0 になるように、`t1`では`s * 4 - 4`という計算式で各行の開始時間を計算できる。

```sql
select s * 4 - 4 as cycle from yokonaga;

 cycle
-------
     0
     4
     8
    12
    16
    20
(6 rows)
```

`t2=1`が 1 になるように、`t2`では`s * 4 - 3`とする。

```sql
select s * 4 - 3 as cycle from yokonaga;
 cycle
-------
     1
     5
     9
    13
    17
    21
(6 rows)
```

あとはこれを使って`union`すれば OK。

```sql
select s * 4 - 4 as hour, t1 as value from yokonaga
union all select s * 4 - 3 as hour, t2  as value from yokonaga
union all select s * 4 - 2 as hour, t3  as value from yokonaga
union all select s * 4 - 1 as hour, t4  as value from yokonaga
order by hour asc;

 hour | value
------+-------
    0 |     3
    1 |     1
    2 |     2
    3 |     0
    4 |     1
    5 |     5
    6 |     1
    7 |     7
    8 |     4
    9 |     4
   10 |     6
   11 |     5
   12 |     5
   13 |     3
   14 |     3
   15 |     4
   16 |     7
   17 |     1
   18 |     7
   19 |     2
   20 |     3
   21 |     2
   22 |     2
   23 |     4
(24 rows)
```

体裁を整えておく。

```sql
select lpad((s * 4 - 4)::text, 2 , '0') || 'H' as hour, t1 as value from yokonaga
union all
select lpad((s * 4 - 3)::text, 2 , '0') || 'H' as hour, t2  as value from yokonaga
union all
select lpad((s * 4 - 2)::text, 2 , '0') || 'H' as hour, t3  as value from yokonaga
union all
select lpad((s * 4 - 1)::text, 2 , '0') || 'H' as hour, t4  as value from yokonaga
order by hour asc;

 hour | value
------+-------
 00H  |     3
 01H  |     1
 02H  |     2
 03H  |     0
 04H  |     1
 05H  |     5
 06H  |     1
 07H  |     7
 08H  |     4
 09H  |     4
 10H  |     6
 11H  |     5
 12H  |     5
 13H  |     3
 14H  |     3
 15H  |     4
 16H  |     7
 17H  |     1
 18H  |     7
 19H  |     2
 20H  |     3
 21H  |     2
 22H  |     2
 23H  |     4
(24 rows)
```

他にも`cross join`を使用する方法もある。横長のカラム数に合わせて連番テーブルを作成し、`cross join`でくっつける。こうすることで、連番テーブルの行数分、紐づけ先の1行を拡張できる。

```sql
with c as (
  select 1 as time
  union all select 2 as time
  union all select 3 as time
  union all select 4 as time
) 
select * from yokonaga cross join c;

 s | t1 | t2 | t3 | t4 | time 
---+----+----+----+----+------
 1 |  3 |  1 |  2 |  0 |    1
 1 |  3 |  1 |  2 |  0 |    2
 1 |  3 |  1 |  2 |  0 |    3
 1 |  3 |  1 |  2 |  0 |    4
---------------------------------
 2 |  1 |  5 |  1 |  7 |    1
 2 |  1 |  5 |  1 |  7 |    2
 2 |  1 |  5 |  1 |  7 |    3
 2 |  1 |  5 |  1 |  7 |    4
---------------------------------
 3 |  4 |  4 |  6 |  5 |    1
 3 |  4 |  4 |  6 |  5 |    2
 3 |  4 |  4 |  6 |  5 |    3
 3 |  4 |  4 |  6 |  5 |    4
---------------------------------
 4 |  5 |  3 |  3 |  4 |    1
 4 |  5 |  3 |  3 |  4 |    2
 4 |  5 |  3 |  3 |  4 |    3
 4 |  5 |  3 |  3 |  4 |    4
---------------------------------
 5 |  7 |  1 |  7 |  2 |    1
 5 |  7 |  1 |  7 |  2 |    2
 5 |  7 |  1 |  7 |  2 |    3
 5 |  7 |  1 |  7 |  2 |    4
---------------------------------
 6 |  3 |  2 |  2 |  4 |    1
 6 |  3 |  2 |  2 |  4 |    2
 6 |  3 |  2 |  2 |  4 |    3
 6 |  3 |  2 |  2 |  4 |    4
(24 rows)
```

あとはこのテーブルを使って、ラベルや値を取得する。

```sql
with c as (
  -- select generate_series(1, 4) as time だと1行でかける
  select 1 as time
  union all select 2 as time
  union all select 3 as time
  union all select 4 as time
) 
select 
  case 
    when c.time = 1 then 't1'
    when c.time = 2 then 't2' 
    when c.time = 3 then 't3' 
    when c.time = 4 then 't4'
  end as quarter,
  case 
    when c.time = 1 then y.t1
    when c.time = 2 then y.t2 
    when c.time = 3 then y.t3 
    when c.time = 4 then y.t4
  end as value
from
  yokonaga as y
cross join
  c
;

 quarter | value 
---------+-------
 t1      |     3
 t2      |     1
 t3      |     2
 t4      |     0
 t1      |     1
 t2      |     5
 t3      |     1
 t4      |     7
 t1      |     4
 t2      |     4
 t3      |     6
 t4      |     5
 t1      |     5
 t2      |     3
 t3      |     3
 t4      |     4
 t1      |     7
 t2      |     1
 t3      |     7
 t4      |     2
 t1      |     3
 t2      |     2
 t3      |     2
 t4      |     4
(24 rows)
```

## :closed_book: Reference

None
