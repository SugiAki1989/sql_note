## :memo: Overview

PostgreSQL には`cube`関数という引数の組み合わせに応じたクロス集計を行ってくれる関数がある。Web サイト上で何らかの複数のアクションを行ったユーザーを検索したい場合に、便利に利用できる。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`cube`

## :pencil2: Example

例えば、下記のようなアクセスログがあった場合に、どのようなアクションの組み合わせが何人によって行われているのかを知りたいとする。加えて、特定のアクションを除外した全ての組み合わせでの結果も知りたい、というケースがあった場合に、組み合わせもととなるカラムが多くなればなるほど、SQL を書くのが辛くなる。

```sql
create table cube(id integer, act varchar(5));
insert into cube
    (id, act)
values
    ('1', 'act_x'),
    ('1', 'act_y'),
    ('1', 'act_z'),

    ('2', 'act_x'),
    ('2', 'act_x'),
    ('2', 'act_x'),

    ('3', 'act_y'),
    ('4', 'act_z'),

    ('5', 'act_x'),
    ('5', 'act_y'),
    ('5', 'act_z')
;

with tmp as (
select
    id,
    sign(sum(case when act = 'act_x' then 1 else 0 end)) has_x,
    sign(sum(case when act = 'act_y' then 1 else 0 end)) has_y,
    sign(sum(case when act = 'act_z' then 1 else 0 end)) has_z
from
    cube
group by
    id
) select * from tmp;
 id | has_x | has_y | has_z
----+-------+-------+-------
  1 |     1 |     1 |     1
  2 |     1 |     0 |     0
  3 |     0 |     1 |     0
  4 |     0 |     0 |     1
  5 |     1 |     1 |     1
```

実際に SQL を書いてみるが、必要な組み合わせは、「x,y,z を行ったユーザー」「y,z を行ったユーザー」「z,x を行ったユーザー」「x,y を行ったユーザー」…という形で書いてユニオンすることになる。

```sql
with tmp as (
select
    id,
    sign(sum(case when act = 'act_x' then 1 else 0 end)) has_x,
    sign(sum(case when act = 'act_y' then 1 else 0 end)) has_y,
    sign(sum(case when act = 'act_z' then 1 else 0 end)) has_z
from
    cube
group by
    id
)
-- 0 act
select null as has_x, null as has_y, null as has_z, count(*) as cnt
from tmp

-- 3 act
union all
select has_x, has_y, has_z, count(*) as cnt
from tmp
group by has_x, has_y, has_z

-- 2 act
union all
select null as has_x, has_y, has_z, count(*) as cnt
from tmp
group by has_y, has_z
union all
select has_x, null as has_y, has_z, count(*) as cnt
from tmp
group by has_x, has_z
union all
select has_x, has_y, null as has_z, count(*) as cnt
from tmp
group by has_x, has_y

-- 1 act
union all
select null as has_x, null as has_y, has_z, count(*) as cnt
from tmp
group by has_z
union all
select has_x, null as has_y, null as has_z, count(*) as cnt
from tmp
group by has_x
union all
select null as has_x, has_y, null as has_z, count(*) as cnt
from tmp
group by has_y
;

 has_x | has_y | has_z | cnt
-------+-------+-------+-----
       |       |       |   5
     1 |     1 |     1 |   2
     0 |     1 |     0 |   1
     0 |     0 |     1 |   1
     1 |     0 |     0 |   1
       |     0 |     0 |   1
       |     1 |     1 |   2
       |     1 |     0 |   1
       |     0 |     1 |   1
     0 |       |     0 |   1
     1 |       |     1 |   2
     1 |       |     0 |   1
     0 |       |     1 |   1
     0 |     0 |       |   1
     1 |     1 |       |   2
     1 |     0 |       |   1
     0 |     1 |       |   1
       |       |     0 |   2
       |       |     1 |   3
     0 |       |       |   2
     1 |       |       |   3
       |     0 |       |   2
       |     1 |       |   3
(23 rows)
```

正直コレは 3 カラムだから良かったが、4,5 となると流石にきついし、組み合わせを書き忘れる可能性はある。このような作業を簡単に行ってくれるのが`cube`関数。`group by cube()`として利用する。

```sql
with tmp as (
select
    id,
    sign(sum(case when act = 'act_x' then 1 else 0 end)) has_x,
    sign(sum(case when act = 'act_y' then 1 else 0 end)) has_y,
    sign(sum(case when act = 'act_z' then 1 else 0 end)) has_z
from
    cube
group by
    id
)
select has_x, has_y, has_z, count(*) as cnt
from tmp
group by cube(has_x, has_y, has_z)
;

 has_x | has_y | has_z | cnt
-------+-------+-------+-----
       |       |       |   5
     1 |     1 |     1 |   2
     0 |     1 |     0 |   1
     0 |     0 |     1 |   1
     1 |     0 |     0 |   1
     0 |     0 |       |   1
     1 |     1 |       |   2
     1 |     0 |       |   1
     0 |     1 |       |   1
     0 |       |       |   2
     1 |       |       |   3
       |     0 |     0 |   1
       |     1 |     1 |   2
       |     1 |     0 |   1
       |     0 |     1 |   1
       |     0 |       |   2
       |     1 |       |   3
     0 |       |     0 |   1
     1 |       |     1 |   2
     0 |       |     1 |   1
     1 |       |     0 |   1
       |       |     0 |   2
       |       |     1 |   3
(23 rows)
```

あとは`case`式で見やすくすれば完成。

```sql
with tmp as (
select
    id,
    sign(sum(case when act = 'act_x' then 1 else 0 end)) has_x,
    sign(sum(case when act = 'act_y' then 1 else 0 end)) has_y,
    sign(sum(case when act = 'act_z' then 1 else 0 end)) has_z
from
    cube
group by
    id
), tmp2 as (
select has_x, has_y, has_z, count(*) as cnt
from tmp
group by cube(has_x, has_y, has_z)
)
select
    case has_x when 1 then 'act x' when 0 then 'not act x'  else '-' end as has_x,
    case has_y when 1 then 'act y' when 0 then 'not act y'  else '-' end as has_y,
    case has_z when 1 then 'act z' when 0 then 'not act z'  else '-' end as has_z,
    cnt
from
    tmp2
;
--少し並び替えた
   has_x   |   has_y   |   has_z   | cnt
-----------+-----------+-----------+-----
 -         | -         | -         |   5
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 act x     | act y     | act z     |   2　
 act x     | not act y | not act z |   1
 not act x | act y     | not act z |   1
 not act x | not act y | act z     |   1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 -         | act y     | act z     |   2
 -         | act y     | not act z |   1
 -         | not act y | act z     |   1
 -         | not act y | not act z |   1 -- only action x
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 act x     | -         | act z     |   2
 act x     | -         | not act z |   1
 not act x | -         | act z     |   1
 not act x | -         | not act z |   1 -- only action y
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 act x     | act y     | -         |   2
 act x     | not act y | -         |   1
 not act x | act y     | -         |   1
 not act x | not act y | -         |   1 -- only action z
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 act x     | -         | -         |   3
 not act x | -         | -         |   2
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 -         | act y     | -         |   3
 -         | not act y | -         |   2
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 -         | -         | act z     |   3
 -         | -         | not act z |   2
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
(23 rows)
```

## :closed_book: Reference

- [7.2.4. GROUPING SETS、CUBE、ROLLUP](https://www.postgresql.jp/document/13/html/queries-table-expressions.html#QUERIES-GROUPING-SETS)
