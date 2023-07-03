## :memo: Overview

この SQL ノートでは、README にも書いたように動けばよいレベルのメモなので、フォーマットに関して考慮していない。これでは仕事だと色々大変なので、そこで便利なのが、SQLFluff。SQLFluff は SQL をリントしてくれるツール。ここではオンライン版を実行する。

- [The SQL Linter for Humans](https://docs.sqlfluff.com/en/stable/)

下記の書籍でサブクエリで書かれているものを`with`で書き直して、SQLFluff を使用してみる。

- [SQL for Data Analysis](https://www.oreilly.com/library/view/sql-for-data/9781492088776/)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`SQLFluff`

## :pencil2: Example

サンプルデータは下記の通り用意する。

```sql
-- create table statement from here
-- https://github.com/cathytanimura/sql_book/tree/master/Chapter%204:%20Cohorts
COPY legislators FROM '~/Desktop/legislators.csv' DELIMITER ',' CSV HEADER;
COPY legislators_terms FROM '~/Desktop/legislators_terms.csv' DELIMITER ',' CSV HEADER;
```

とりあえず、何も気にせず書き直して、[SQLFluff Online](https://online.sqlfluff.com/)にリントをお願いする。

```sql
with tmp as (
select id_bioguide, term_start, min(term_start) over (partition by id_bioguide) as first_term
from legislators_terms
),
 tmp2 as (
select id_bioguide, term_start, first_term, date_part('year',age(term_start, first_term)) as period
from tmp
),
 tmp3 as (
select period, count(distinct id_bioguide) as cohort_retained
from tmp2
group by period
)
select
period,
first_value(cohort_retained) over (order by period) as cohort_size,
cohort_retained,
cohort_retained * 1.0 / first_value(cohort_retained) over (order by period) as pct_retained
from tmp3
;
```

インデントされてない、`select`のカラムは行ごとに改行する、行が長すぎる、空白行がない、などなど色々指摘してくれる。

```sql
Code	Line / Position	Description
L003	2 / 1	Expected 1 indentation, found 0 [compared to line 01]
L036	2 / 1	Select targets should be on a new line unless there is only one select target.
L016	2 / 94	Line is too long.
L003	3 / 1	Expected 1 indentation, found 0 [compared to line 01]
L003	5 / 2	Expected 0 indentations, found more than 0 [compared to line 04]
L022	5 / 2	Blank line expected but not found after CTE closing bracket.
L003	6 / 1	Expected 1 indentation, found 0 [compared to line 04]
L036	6 / 1	Select targets should be on a new line unless there is only one select target.
L008	6 / 62	Commas should be followed by a single whitespace unless followed by a comment.
L016	6 / 100	Line is too long.
L003	7 / 1	Expected 1 indentation, found 0 [compared to line 04]
L003	9 / 2	Expected 0 indentations, found more than 0 [compared to line 08]
L022	9 / 2	Blank line expected but not found after CTE closing bracket.
L003	10 / 1	Expected 1 indentation, found 0 [compared to line 08]
L036	10 / 1	Select targets should be on a new line unless there is only one select target.
L003	11 / 1	Expected 1 indentation, found 0 [compared to line 08]
L003	12 / 1	Expected 1 indentation, found 0 [compared to line 08]
L022	14 / 1	Blank line expected but not found after CTE closing bracket.
L034	14 / 1	Select wildcards then simple targets before calculations and aggregates.
L003	15 / 1	Expected 1 indentation, found 0 [compared to line 14]
L003	16 / 1	Expected 1 indentation, found 0 [compared to line 14]
L003	17 / 1	Expected 1 indentation, found 0 [compared to line 14]
L003	18 / 1	Expected 1 indentation, found 0 [compared to line 14]
L016	18 / 92	Line is too long.
L052	19 / 10	Statements must end with a semi-colon.
```

これが修正された SQL がこちら。

```sql
with tmp as (
    select
        id_bioguide,
        term_start,
        min(term_start) over (partition by id_bioguide) as first_term
    from legislators_terms
),

tmp2 as (
    select
        id_bioguide,
        term_start,
        first_term,
        date_part('year', age(term_start, first_term)) as period
    from tmp
),

tmp3 as (
    select
        period,
        count(distinct id_bioguide) as cohort_retained
    from tmp2
    group by period
)

select
    period,
    cohort_retained,
    first_value(cohort_retained) over (order by period) as cohort_size,
    cohort_retained * 1.0 / first_value(
        cohort_retained
    ) over (order by period) as pct_retained
from tmp3;
```

お仕事で SQL を書く際は気をつけます…。

## :closed_book: Reference

- [SQL for Data Analysis](https://www.oreilly.com/library/view/sql-for-data/9781492088776/)
