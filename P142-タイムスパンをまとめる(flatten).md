## :memo: Overview

ここでは下記のブログで紹介されているSQLを参考にさせていただきながら、SQLやデータベースへの理解を深める。

- [EXPLAIN EXTENDED](https://explainextended.com/)

このブログ主は、最近、一部界隈で話題となっていた[
How to create fast database queries Happy New Year: GPT in 500 lines of SQL](https://explainextended.com/2023/12/31/happy-new-year-15/)を書かれた方で、他の記事も非常に勉強になる内容のものが多い。

今回は、こちらの記事に関するものを参考にさせていただく。

- [Flattening timespans: PostgreSQL 8.4](https://explainextended.com/2009/07/14/flattening-timespans-postgresql-8-4/#more-2115)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`union all`, `max() over()`

## :pencil2: Example

今回はタイムスパンを平坦化するSQL。言葉での説明が難しいので、イメージものせておく。各タイムスパンが重複している場合、それは同じタイムスパンと考え、それが途切れた時点から、異なるタイムスパンとして扱うというもの。2枚目の最後の方に、ブロックが重なっていない箇所があるが、それがタイムスパンの切れ目である。

![start](https://github.com/SugiAki1989/sql_note/blob/main/image/p141-start.png)
![end](https://github.com/SugiAki1989/sql_note/blob/main/image/p141-end.png)
 
参照元ブログにも記載されている通り、まずは、サンプルデータを生成する。クエリは下記の通り。

```sql
CREATE TABLE t_span
  (
  s_start TIMESTAMP NOT NULL,
  s_end TIMESTAMP NOT NULL
  );
 
CREATE INDEX ix_span_start ON t_span (s_start);
 
SELECT setseed(0.5);
 
INSERT
INTO t_span
SELECT  CAST('2009-07-14' AS TIMESTAMP) - ((s * 30) || ' SECOND')::interval,
  CAST('2009-07-14' AS TIMESTAMP) - ((s * 30) || ' SECOND')::interval  + ((15 + RANDOM() * 300) || ' SECOND')::interval
FROM generate_series (1, 1440000) s;
 
ANALYZE t_span;
```

ブログで紹介されているクエリはこちら。

```sql
WITH    q AS
        (
        SELECT  s_start,
                s_end,
                MAX(s_end) OVER (ORDER BY s_start) AS s_maxend,
                ROW_NUMBER() OVER (ORDER BY s_start) AS s_rn
        FROM    t_span s
        ORDER BY
                s_start
        )
SELECT  q1.s_start,
        q2.s_end
FROM    (
        SELECT  COALESCE(LAG(s_rn) OVER (ORDER BY s_rn), 1) AS s_prn, s_rn - 1 AS s_crn
        FROM    (
                SELECT  s_rn
                FROM    (
                        SELECT  q.*, LAG(s_maxend) OVER (ORDER BY s_rn) AS s_maxpend
                        FROM    q
                        ) qo
                WHERE   s_start > s_maxpend
                UNION ALL
                SELECT  MAX(s_rn) + 1
                FROM    q
                ) qo
        ) g
JOIN    q q1
ON      q1.s_rn = g.s_prn
JOIN    q q2
ON      q2.s_rn = g.s_crn
ORDER BY
        q1.s_start

    s_start    |  s_end
---------------------+----------------------------
 2008-03-01 00:00:00 | 2008-03-01 06:19:24.034759
 2008-03-01 06:19:30 | 2008-03-04 09:04:57.467786
 2008-03-04 09:05:00 | 2008-03-05 13:03:58.64022
 2008-03-05 13:04:00 | 2008-03-09 22:16:47.591515
 2008-03-09 22:17:00 | 2008-03-10 04:38:29.880287
 2008-03-10 04:38:30 | 2008-03-21 14:11:56.317819
 (snip)
 2009-06-14 22:59:00 | 2009-07-08 12:00:46.772354
 2009-07-08 12:01:00 | 2009-07-12 10:17:15.255521
 2009-07-12 10:17:30 | 2009-07-13 01:29:29.724301
 2009-07-13 01:29:30 | 2009-07-14 00:00:59.973126
(80 rows)
```

今回もこのクエリを分解して理解を深めていく。`with`句では、`s_start`を並び替えて、`max()over()`で最大日時を取得している。ウインドウフレームのデフォルトは`rows between unbounded preceding and current row`なので、カーソルが進むたびに拡大する範囲における最大値が`s_maxend`である。先ほどのイメージでいうとブロックの右端を追い続けるイメージ。

```sql
SELECT  
  s_start,
  s_end,
  MAX(s_end) OVER (ORDER BY s_start) AS s_maxend,
  ROW_NUMBER() OVER (ORDER BY s_start) AS s_rn
FROM 
  t_span s
ORDER BY
  s_start
limit 10;

       s_start       |           s_end            |          s_maxend          | s_rn
---------------------+----------------------------+----------------------------+------
 2008-03-01 00:00:00 | 2008-03-01 00:02:46.766558 | 2008-03-01 00:02:46.766558 |    1
 2008-03-01 00:00:30 | 2008-03-01 00:01:26.193727 | 2008-03-01 00:02:46.766558 |    2
 2008-03-01 00:01:00 | 2008-03-01 00:04:18.014022 | 2008-03-01 00:04:18.014022 |    3
 2008-03-01 00:01:30 | 2008-03-01 00:02:18.863313 | 2008-03-01 00:04:18.014022 |    4
 2008-03-01 00:02:00 | 2008-03-01 00:04:40.446329 | 2008-03-01 00:04:40.446329 |    5
 2008-03-01 00:02:30 | 2008-03-01 00:03:15.233501 | 2008-03-01 00:04:40.446329 |    6
 2008-03-01 00:03:00 | 2008-03-01 00:03:51.000514 | 2008-03-01 00:04:40.446329 |    7
 2008-03-01 00:03:30 | 2008-03-01 00:04:47.341508 | 2008-03-01 00:04:47.341508 |    8
 2008-03-01 00:04:00 | 2008-03-01 00:07:51.55771  | 2008-03-01 00:07:51.55771  |    9
 2008-03-01 00:04:30 | 2008-03-01 00:06:54.468112 | 2008-03-01 00:07:51.55771  |   10
(10 rows)
```

サブクエリの中で使用されるテーブル`qo`は、さきほどのテーブルの`s_maxend`をずらしたレコードを持つ。

```sql
WITH q AS
  (
  SELECT  
    s_start,
    s_end,
    MAX(s_end) OVER (ORDER BY s_start) AS s_maxend,
    ROW_NUMBER() OVER (ORDER BY s_start) AS s_rn
  FROM 
    t_span s
  ORDER BY
    s_start
  )
SELECT  q.*, LAG(s_maxend) OVER (ORDER BY s_rn) AS s_maxpend FROM q limit 10;

       s_start       |           s_end            |          s_maxend          | s_rn |         s_maxpend
---------------------+----------------------------+----------------------------+------+----------------------------
 2008-03-01 00:00:00 | 2008-03-01 00:02:46.766558 | 2008-03-01 00:02:46.766558 |    1 |
 2008-03-01 00:00:30 | 2008-03-01 00:01:26.193727 | 2008-03-01 00:02:46.766558 |    2 | 2008-03-01 00:02:46.766558
 2008-03-01 00:01:00 | 2008-03-01 00:04:18.014022 | 2008-03-01 00:04:18.014022 |    3 | 2008-03-01 00:02:46.766558
 2008-03-01 00:01:30 | 2008-03-01 00:02:18.863313 | 2008-03-01 00:04:18.014022 |    4 | 2008-03-01 00:04:18.014022
 2008-03-01 00:02:00 | 2008-03-01 00:04:40.446329 | 2008-03-01 00:04:40.446329 |    5 | 2008-03-01 00:04:18.014022
 2008-03-01 00:02:30 | 2008-03-01 00:03:15.233501 | 2008-03-01 00:04:40.446329 |    6 | 2008-03-01 00:04:40.446329
 2008-03-01 00:03:00 | 2008-03-01 00:03:51.000514 | 2008-03-01 00:04:40.446329 |    7 | 2008-03-01 00:04:40.446329
 2008-03-01 00:03:30 | 2008-03-01 00:04:47.341508 | 2008-03-01 00:04:47.341508 |    8 | 2008-03-01 00:04:40.446329
 2008-03-01 00:04:00 | 2008-03-01 00:07:51.55771  | 2008-03-01 00:07:51.55771  |    9 | 2008-03-01 00:04:47.341508
 2008-03-01 00:04:30 | 2008-03-01 00:06:54.468112 | 2008-03-01 00:07:51.55771  |   10 | 2008-03-01 00:07:51.55771
(10 rows)
```

次にこれまで準備したテーブルを利用して、下記の部分が実行される。`s_start > s_maxpend`に合う条件でレコードを取ってきたのはわかるが、もう少し理解を深めたい。

```sql
WITH q AS
  (
  SELECT  
    s_start,
    s_end,
    MAX(s_end) OVER (ORDER BY s_start) AS s_maxend,
    ROW_NUMBER() OVER (ORDER BY s_start) AS s_rn
  FROM 
    t_span s
  ORDER BY
    s_start
  )
SELECT
  s_rn
FROM (
  SELECT  q.*, LAG(s_maxend) OVER (ORDER BY s_rn) AS s_maxpend
  FROM q
  ) qo
WHERE
  s_start > s_maxpend
limit 10;

  s_rn
--------
    760
   9731
  13089
  25715
  26478
  59305
  66113
  90361
 142470
 146489
(10 rows)
```

`s_start > s_maxpend`に合う条件のレコードというのは、イメージ画像でいうと、ブロックの右端を追いかけ続けて最大値を更新してきた中で、1つ下のブロックの左端と右端の関係が、右側になる部分である。重複していると、右側ではなく、左側に位置する関係になる。つまり、最大値を超えた部分が条件にあうレコードである。また、タイムスパンの切れ目でもある。

```sql
    s_start          | s_rn |   s_maxpend
---------------------+------+---------------------
 2008-03-01 00:00:00 |    1 | (null)              -- null
 2008-03-01 00:00:30 |    2 | 2008-03-01 00:02:46 -- false
 2008-03-01 00:01:00 |    3 | 2008-03-01 00:02:46 -- false
 2008-03-01 00:01:30 |    4 | 2008-03-01 00:04:18 -- false
 2008-03-01 00:02:00 |    5 | 2008-03-01 00:04:18 -- false
 (snip)
 2008-03-01 06:18:30 |  758 | 2008-03-01 06:19:17 -- false
 2008-03-01 06:19:00 |  759 | 2008-03-01 06:19:19 -- false
 2008-03-01 06:19:30 |  760 | 2008-03-01 06:19:24 -- true -- break in timespan
 2008-03-01 06:20:00 |  761 | 2008-03-01 06:20:15 -- false
```

その後に、帳尻を合わせるために、下記をユニオンしている。

```sql
WITH q AS
  (
  SELECT
    s_start,
    s_end,
    MAX(s_end) OVER (ORDER BY s_start) AS s_maxend,
    ROW_NUMBER() OVER (ORDER BY s_start) AS s_rn
  FROM 
    t_span s
  ORDER BY
    s_start
  )
SELECT 
  MAX(s_rn) + 1 as s_rn
FROM 
  q
;
 ?column?
----------
  1440001
(1 row) 
```

ここでは、タイムスパンの切れ目を表す開始、終了のレコードを識別するインデックスを作成したテーブルを作っている。

```sql
WITH q AS
  (
  SELECT
    s_start,
    s_end,
    MAX(s_end) OVER (ORDER BY s_start) AS s_maxend,
    ROW_NUMBER() OVER (ORDER BY s_start) AS s_rn
  FROM 
    t_span s
  ORDER BY
    s_start
  )
SELECT  
  COALESCE(LAG(s_rn) OVER (ORDER BY s_rn), 1) AS s_prn, 
  s_rn - 1 AS s_crn
FROM (
  SELECT  
    s_rn
  FROM (
    SELECT  q.*, LAG(s_maxend) OVER (ORDER BY s_rn) AS s_maxpend
    FROM q
 ) qo
  WHERE 
    s_start > s_maxpend
  UNION ALL
  SELECT MAX(s_rn) + 1
  FROM q
  ) qo
limit 10
;
 s_prn  | s_crn
--------+--------
      1 |    759
    760 |   9730
   9731 |  13088
  13089 |  25714
  25715 |  26477
  26478 |  59304
  59305 |  66112
  66113 |  90360
  90361 | 142469
 142470 | 146488
(10 rows)
```

最後は、タイムスパンの切れ目を表す開始、終了のレコードを識別するインデックスをキーにして、テーブルを紐づけて、必要な開始時刻、終了時刻を取得している。

```sql
WITH    q AS
        (
        SELECT  s_start,
                s_end,
                MAX(s_end) OVER (ORDER BY s_start) AS s_maxend,
                ROW_NUMBER() OVER (ORDER BY s_start) AS s_rn
        FROM    t_span s
        ORDER BY
                s_start
        )
SELECT  q1.s_start,
        q2.s_end
FROM    (
        SELECT  COALESCE(LAG(s_rn) OVER (ORDER BY s_rn), 1) AS s_prn, s_rn - 1 AS s_crn
        FROM    (
                SELECT  s_rn
                FROM    (
                        SELECT  q.*, LAG(s_maxend) OVER (ORDER BY s_rn) AS s_maxpend
                        FROM    q
                        ) qo
                WHERE   s_start > s_maxpend
                UNION ALL
                SELECT  MAX(s_rn) + 1
                FROM    q
                ) qo
        ) g
JOIN    q q1
ON      q1.s_rn = g.s_prn
JOIN    q q2
ON      q2.s_rn = g.s_crn
ORDER BY
        q1.s_start
;

    s_start    |  s_end
---------------------+----------------------------
 2008-03-01 00:00:00 | 2008-03-01 06:19:24.034759
 2008-03-01 06:19:30 | 2008-03-04 09:04:57.467786
 2008-03-04 09:05:00 | 2008-03-05 13:03:58.64022
 2008-03-05 13:04:00 | 2008-03-09 22:16:47.591515
 2008-03-09 22:17:00 | 2008-03-10 04:38:29.880287
 (snip)
 2009-06-14 07:47:30 | 2009-06-14 22:58:47.321115
 2009-06-14 22:59:00 | 2009-07-08 12:00:46.772354
 2009-07-08 12:01:00 | 2009-07-12 10:17:15.255521
 2009-07-12 10:17:30 | 2009-07-13 01:29:29.724301
 2009-07-13 01:29:30 | 2009-07-14 00:00:59.973126
(80 rows)
```

`max() over()`でタイムスパンの切れ目を判定できるようする点は学びが多かった。なるほどな～。

## :closed_book: Reference

- [EXPLAIN EXTENDED](https://explainextended.com/)





