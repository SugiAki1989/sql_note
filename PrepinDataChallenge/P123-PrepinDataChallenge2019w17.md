## :memo: Overview

Preppin' Data challenge の「2019: Week 17」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/06/2019-week-17.html)
- [Answer](https://preppindata.blogspot.com/2019/06/2019-week-17-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。多数決の計算方法を変えた場合の勝者を求める、というお題

- FPTP 投票システムを使用した場合の勝者を計算
- AV 投票システムを使用した場合の勝者を計算
- Borda 投票システムを使用した場合の勝者を計算

```sql
select * from p2019w17t1 limit 10;

 votingpreferences |  voter
-------------------+---------
 ABC               | Mona
 ABC               | Zara
 ABC               | Natalie
 ABC               | Camilla
 ACB               | Colin
 BCA               | Jane
 BCA               | Hashem
 CBA               | Coral
 CBA               | Kenny
 CBA               | Tim
 CBA               | Hafiz
(11 rows)
```

Borda 投票システムを使用した場合の勝者を計算していく。

```sql
with tmp as (
select
    voter,
    votingpreferences,
    left(votingpreferences, 1) as choice1,
    substring(votingpreferences, 2, 1) as choice2,
    right(votingpreferences, 1) as choice3
from
    p2019w17t1
), tmp2 as (
select voter, votingpreferences, choice1 as choice, '1st' as choice_type, 3 as points from tmp
union all
select voter, votingpreferences, choice2 as choice, '2nd' as choice_type, 2 as points from tmp
union all
select voter, votingpreferences, choice3 as choice, '3rd' as choice_type, 1 as points  from tmp
)
select
    *
from
    tmp2
;

  voter  | votingpreferences | choice | choice_type | points
---------+-------------------+--------+-------------+--------
 Mona    | ABC               | A      | 1st         |      3
 Zara    | ABC               | A      | 1st         |      3
(snip)
 Mona    | ABC               | B      | 2nd         |      2
 Zara    | ABC               | B      | 2nd         |      2
(snip)
 Mona    | ABC               | C      | 3rd         |      1
 Zara    | ABC               | C      | 3rd         |      1
(snip)
(33 rows)
```

Borda 投票システムの場合、B が勝者。

```sql
with tmp as (
select
    voter,
    votingpreferences,
    left(votingpreferences, 1) as choice1,
    substring(votingpreferences, 2, 1) as choice2,
    right(votingpreferences, 1) as choice3
from
    p2019w17t1
), tmp2 as (
select voter, votingpreferences, choice1 as choice, '1st' as choice_type, 3 as points from tmp
union all
select voter, votingpreferences, choice2 as choice, '2nd' as choice_type, 2 as points from tmp
union all
select voter, votingpreferences, choice3 as choice, '3rd' as choice_type, 1 as points  from tmp
)
select
    choice,
    sum(points) as points
from
    tmp2
group by
    choice
order by
    points desc
;

 choice | points
--------+--------
 B      |     23
 C      |     22
 A      |     21
(3 rows)
```

FPTP 投票システムを使用した場合の勝者を計算していく。FPTP 投票システムの場合、A が勝者。

```sql
select
    left(votingpreferences, 1) as choice1,
    count(1) as points
from
    p2019w17t1
group by
    left(votingpreferences, 1)
order by
    points desc
;

 choice1 | points
---------+--------
 A       |      5
 C       |      4
 B       |      2
(3 rows)
```

AV 投票システムを使用した場合の勝者を計算していく。ファーストラウンドで半数を取れていないので、セカンドラウンドに回る。

```sql
with tmp as (
    select count(1) as total_votes from p2019w17t1
)
select
    left(votingpreferences, 1) as choice1,
    count(1) as points,
    max(total_votes) as total_votes,
    case when count(1) > max(total_votes)/2 then 1 else null end as judge
from
    p2019w17t1
cross join
    tmp
group by
    left(votingpreferences, 1)
order by
    points desc
;

 choice1 | points | total_votes | judge
---------+--------+-------------+--------
 A       |      5 |          11 | (null)
 C       |      4 |          11 | (null)
 B       |      2 |          11 | (null)
(3 rows)
```

ラウンド 1 で最下位の B を除外して計算し直すと、勝者は C となる。

```sql
with tmp as (
    select count(1) as total_votes from p2019w17t1
)
select
    left(replace(votingpreferences, 'B', ''),1) as new_top_choice,
    count(1) as points,
    max(total_votes) as total_votes,
    case when count(1) > max(total_votes)/2 then 1 else null end as judge
from
    p2019w17t1
cross join
    tmp
group by
    left(replace(votingpreferences, 'B', ''),1)
order by
    points desc
;

 new_top_choice | points | total_votes | judge
----------------+--------+-------------+--------
 C              |      6 |          11 |      1
 A              |      5 |          11 | (null)
(2 rows)
```

## :closed_book: Reference

None
