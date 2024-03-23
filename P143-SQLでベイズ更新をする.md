## :memo: Overview

ここでは下記のブログで紹介されているSQLを参考にさせていただきながら、SQLやデータベースへの理解を深める。

- [EXPLAIN EXTENDED](https://explainextended.com/)

このブログ主は、最近、一部界隈で話題となっていた[
How to create fast database queries Happy New Year: GPT in 500 lines of SQL](https://explainextended.com/2023/12/31/happy-new-year-15/)を書かれた方で、他の記事も非常に勉強になる内容のものが多い。

今回は、こちらの記事に関するものを参考にさせていただく。

- [Bayesian classification](https://explainextended.com/2010/03/25/bayesian-classification/#more-4598)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`ln()`, `sum()`, `exp()`

## :pencil2: Example
 
参照元ブログにも記載されている通り、まずは、サンプルデータを生成する。クエリは下記の通り。

```sql
create table filler (
id serial primary key
);

create table t_user (
id serial primary key,
name varchar(20) not null,
gender char(1) not null
);

create table t_site (
id serial primary key,
name varchar(20) not null,
male NUMERIC(3, 2) not null
);

create table t_visit (
u_id int not null,
s_id int not null,
primary key (u_id, s_id)
);

create or replace function prc_filler(cnt int)
returns void as $$
declare
    _cnt int := 1;
begin
    while _cnt <= cnt loop
        insert into filler values (_cnt);
        _cnt := _cnt + 1;
    end loop;
end;
$$ language plpgsql;

select prc_filler(1000);

insert into t_user (id, name, gender)
select id, concat('user ', id), case when random() > 0.5 then 'm' else 'f' end
from filler;

insert into t_site (id, name, male)
select id, concat('site ', id), random() * 0.94 + 0.03
from filler;
UPDATE t_site SET male = ROUND(male::numeric, 1);


insert into t_visit (u_id, s_id)
select u.id as u_id, s.id as s_id
from t_user u
cross join t_site s
where random() < 0.05
and random() < case when u.gender = 'm' then s.male else 1 - s.male end;
```

今回使用するデータはこちら。

```sql
-- ユーザーのマスタ情報で、実際の性別を持っている
select * from t_user limit 10;

 id |  name   | gender
----+---------+--------
  1 | User 1  | F
  2 | User 2  | M
  3 | User 3  | F
  4 | User 4  | F
  5 | User 5  | M
  6 | User 6  | M
  7 | User 7  | M
  8 | User 8  | M
  9 | User 9  | F
 10 | User 10 | M
(10 rows)

-- サイトのマスタ情報で、男性という条件のもとで、サイトに訪問する確率
select * from t_site limit 10;
 id |  name   | male
----+---------+------
 60 | Site 60 |  0.6
 61 | Site 61 |  0.8
 62 | Site 62 |  0.4
 63 | Site 63 |  0.2
 64 | Site 64 |  0.9
 65 | Site 65 |  0.3
 66 | Site 66 |  0.1
 67 | Site 67 |  0.2
 68 | Site 68 |  0.2
 69 | Site 69 |  0.1
(10 rows)

-- ユーザーが訪問したサイトが記録されている
select * from t_visit where u_id = 838;
 u_id | s_id
------+------
  838 |  102
  838 |  177
  838 |  225
  838 |  301
  838 |  381
  838 |  442
  838 |  450
  838 |  556
  838 |  572
  838 |  648
  838 |  650
  838 |  709
  838 |  786
  838 |  863
(14 rows)
```

ブログで紹介されているクエリはこちら。これを今回は紐解いていく。

```sql
with vars as (
select 0.5 as prior
), posterior_calculation as (
select  
  u.*,
  prior * exp(sum(ln(male))) / (prior * exp(sum(ln(male))) + (1 - prior) * exp(sum(ln(1 - male)))) as posterior
from 
  vars
cross join 
  t_user u
left join 
  t_visit v 
on 
  v.u_id = u.id
left join 
  t_site s 
on 
  s.id = v.s_id
group by 
  u.id, 
  vars.prior
), sub as (
select  
  *,
  case
      when posterior < 0.05 then 'F'
      when posterior > 0.95 then 'M'
      else 'U'
  end as guessed
from 
  posterior_calculation
)
select
  *
from 
  sub
where 
  guessed <> gender
;

  id  |   name    | gender |               posterior               | guessed
------+-----------+--------+---------------------------------------+---------
  160 | User 160  | F      |          0.25486361234823265862511250 | U
  770 | User 770  | M      |      0.910458342185417870145997715876 | U
  534 | User 534  | M      |        0.3331001527074949115422096598 | U
  337 | User 337  | M      |  0.0000000018156813851024790613775316 | F
  571 | User 571  | F      |        0.9979426862040474707248457988 | M
(snip)
   28 | User 28   | F      |        0.5422924101608050521348847984 | U
  628 | User 628  | M      |          0.51268163114038306143993482 | U
  548 | User 548  | M      |         0.040530904337455502083130676 | F
  251 | User 251  | M      |        0.5898903132026947971231025718 | U
  825 | User 825  | M      |         0.745627965351788899640493397 | U
(598 rows)
```

まずはSQLではなく、ベイズ更新の整理から始める。イベントの説明は下記の通りである。

- `P(B)`  : サイトに訪問する確率
- `P(A)`  : 訪問者が男性である事前確率
- `P(B|A)`: 男性という条件のもとで、サイトに訪問する確率
- `P(¬A)` : 訪問者が女性である事前確率
- `P(B|¬A)`: 女性という条件のもとで、サイトに訪問する確率

今回計算したいのが、サイトの訪問履歴から逐次更新した事後確率`P(A|B)`である。つまり、訪問者(B)が男性(A)である事後確率である。

- `P(A|B) = P(B|A)P(A) / P(B) = P(B|A)×P(A) / (P(B|A)×P(A) + P(B|¬A)×P(¬A)`

例えば、下記のユーザーであれば、サイト訪問ごとに、計算した事後確率を事前分布に割り当てて、事後確率を更新していく。初期値となる事前確率は0.5とする。

```sql
with vars as (
select 0.5 as prior
)
select  
  *
from 
  vars
cross join 
  t_user u
left join 
  t_visit v 
on 
  v.u_id = u.id
left join 
  t_site s 
on 
  s.id = v.s_id
where
  u.id = 838 and v.s_id in (102, 177, 225, 301, 381)
;

 prior | id  |   name   | gender | u_id | s_id | id  |   name   | male
-------+-----+----------+--------+------+------+-----+----------+------
   0.5 | 838 | User 838 | F      |  838 |  102 | 102 | site 102 | 0.18
   0.5 | 838 | User 838 | F      |  838 |  177 | 177 | site 177 | 0.96
   0.5 | 838 | User 838 | F      |  838 |  225 | 225 | site 225 | 0.41
   0.5 | 838 | User 838 | F      |  838 |  301 | 301 | site 301 | 0.37
   0.5 | 838 | User 838 | F      |  838 |  381 | 381 | site 381 | 0.18
(5 rows)
```

実際の計算は下記となり、事後確率を事前分布として利用し、逐次更新していく。

```
(male*prior) / ((male*prior) + ((1-male)*(1-prior)))
↓
サイト訪問1回目: (0.18*0.50)/((0.18*0.50) + ((1-0.18)*(1-0.50))) = 0.18
サイト訪問2回目: (0.96*0.18)/((0.96*0.18) + ((1-0.96)*(1-0.18))) = 0.84
サイト訪問3回目: (0.41*0.84)/((0.41*0.84) + ((1-0.41)*(1-0.84))) = 0.78
サイト訪問4回目: (0.37*0.78)/((0.37*0.78) + ((1-0.37)*(1-0.78))) = 0.67
サイト訪問5回目: (0.18*0.67)/((0.18*0.67) + ((1-0.18)*(1-0.67))) = 0.32
```

ここで問題となるのが、SQLでは簡単には逐次更新できない。事後確率がカラムとして存在しており、レコードをずらすだけで良い状況であれば、それでもよいが、今回はそうではない。ただ、逐次更新をしていく場合、総乗を使った式でも同じように計算できる。

![formula](https://github.com/SugiAki1989/sql_note/blob/main/image/p143-formula.png)

総乗を使った関数はないものの、対数変換して、総和して、指数変換で戻せば、総乗はSQLでも可能である。

```sql
with vars as (
select 0.5 as prior
)
select  
  u.*
  , prior * exp(sum(ln(male))) / (prior * exp(sum(ln(male))) + (1 - prior) * exp(sum(ln(1 - male)))) as posterior
from 
  vars
cross join 
  t_user u
left join 
  t_visit v 
on 
  v.u_id = u.id
left join 
  t_site s 
on 
  s.id = v.s_id
where
  -- in の中身を変更する
  u.id = 838 and v.s_id in (102, 177, 225, 301, 381)
group by 
  u.id, 
  vars.prior
;

-- 102
 id  |   name   | gender |       posterior
-----+----------+--------+------------------------
 838 | User 838 | F      | 0.18000000000000000000
(1 row)

-- 102, 177
 id  |   name   | gender |       posterior
-----+----------+--------+------------------------
 838 | User 838 | F      | 0.84046692607003890740
(1 row)

-- 102, 177, 225
 id  |   name   | gender |       posterior
-----+----------+--------+------------------------
 838 | User 838 | F      | 0.78545454545454544503
(1 row)

-- 102, 177, 225, 301
 id  |   name   | gender |       posterior
-----+----------+--------+------------------------
 838 | User 838 | F      | 0.68255188316679476499
(1 row)

-- 102, 177, 225, 301, 381
 id  |   name   | gender |       posterior
-----+----------+--------+------------------------
 838 | User 838 | F      | 0.32064192577733199599
(1 row)
```

あとは、計算された事後確率をもとに`case`文で男性、女性、判定不可に振り分け直し、一致しないものを表示すれば、参照先のブログでやっていることの再現となる。

```sql
with vars as (
select 0.5 as prior
), posterior_calculation as (
select  
  u.*,
  prior * exp(sum(ln(male))) / (prior * exp(sum(ln(male))) + (1 - prior) * exp(sum(ln(1 - male)))) as posterior
from 
  vars
cross join 
  t_user u
left join 
  t_visit v 
on 
  v.u_id = u.id
left join 
  t_site s 
on 
  s.id = v.s_id
group by 
  u.id, 
  vars.prior
), sub as (
select  
  *,
  case
      when posterior < 0.05 then 'F'
      when posterior > 0.95 then 'M'
      else 'U'
  end as guessed
from 
  posterior_calculation
)
select
  *
from 
  sub
where 
  guessed <> gender
;

  id  |   name    | gender |               posterior               | guessed
------+-----------+--------+---------------------------------------+---------
  160 | User 160  | F      |          0.25486361234823265862511250 | U
  770 | User 770  | M      |      0.910458342185417870145997715876 | U
  534 | User 534  | M      |        0.3331001527074949115422096598 | U
  337 | User 337  | M      |  0.0000000018156813851024790613775316 | F
  571 | User 571  | F      |        0.9979426862040474707248457988 | M
(snip)
   28 | User 28   | F      |        0.5422924101608050521348847984 | U
  628 | User 628  | M      |          0.51268163114038306143993482 | U
  548 | User 548  | M      |         0.040530904337455502083130676 | F
  251 | User 251  | M      |        0.5898903132026947971231025718 | U
  825 | User 825  | M      |         0.745627965351788899640493397 | U
(598 rows)
```

## :closed_book: Reference

- [EXPLAIN EXTENDED](https://explainextended.com/)





