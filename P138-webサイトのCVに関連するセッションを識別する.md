## :memo: Overview

ここではSQLを使って、webサイトのアクセス解析を行う方法をまとめておく。主に下記の書籍の第6章を参考にしており、SQLを理解するためにSQLの途中の実行結果をメモっている。

- [ビッグデータ分析・活用のためのSQLレシピ](https://book.mynavi.jp/supportsite/detail/9784839961268.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`cv`, `desc`, 

## :pencil2: Example


第6章にCVページより前のアクセスにフラグを立てるクエリが記載されているが、このクエリを個人的には直感的(注意力散漫…私の)ではなかったので、メモしておく。`sum() over()`の`order by stamp desc`が肝である。

```sql
select
  session
  , stamp
  , path
  , sum(case when path = '/complete' then 1 else 0 end) 
      over(partition by session order by stamp desc rows between unbounded preceding and current row) as has_conversion
from 
  activity_log
```

通常、サイトにアクセスして、何らかの行動を伴って、CVに至る。その時系列でレコードが並んでいる、もしくは並び替えた頭で、`rows between unbounded preceding and current row`を解釈しようとすると、実行結果の解釈が上手くできない。ウインドウフレームの動き方と実行結果が伴わないためである。ここで`order by stamp desc`である点に注目する必要がある。`desc`なので、このように書くことで、時系列を反対に遡ることができる。つまり、CVした場合、1を立てて、あとはまとめてパーティションのレコードに1を立てることできる。個人的には、`max()`で取ることのほうが好み。

```sql
with
activity_log_with_conversion_flag as (
  select
    session
    , stamp
    , path
    , sum(case when path = '/complete' then 1 else 0 end) over(partition by session order by stamp desc rows between unbounded preceding and current row) as has_conversion
    , max(case when path = '/complete' then 1 else 0 end) over(partition by session order by stamp asc rows between unbounded preceding and unbounded following) as has_conversion2
  from 
    activity_log
)
select * 
from
  activity_log_with_conversion_flag
order by 
  session, stamp
;

 session  |        stamp        |     path      | has_conversion | has_conversion2
----------+---------------------+---------------+----------------+-----------------
 0fe39581 | 2017-01-09 12:18:43 | /search_list  |              0 |               0
 111f2996 | 2017-01-09 12:18:43 | /search_list  |              0 |               0
 111f2996 | 2017-01-09 12:19:11 | /search_input |              0 |               0
 111f2996 | 2017-01-09 12:20:10 |               |              0 |               0
 111f2996 | 2017-01-09 12:21:14 | /search_input |              0 |               0
 1cf7678e | 2017-01-09 12:18:43 | /detail       |              0 |               0
 1cf7678e | 2017-01-09 12:19:04 |               |              0 |               0
 36dd0df7 | 2017-01-09 12:18:43 | /search_list  |              0 |               0
 36dd0df7 | 2017-01-09 12:19:49 | /detail       |              0 |               0
 3efe001c | 2017-01-09 12:18:43 | /detail       |              0 |               0
 47db0370 | 2017-01-09 12:18:43 | /search_list  |              0 |               0
 5d5b0997 | 2017-01-09 12:18:43 | /detail       |              0 |               0
 5eb2e107 | 2017-01-09 12:18:43 | /detail       |              0 |               0
 87b5725f | 2017-01-09 12:18:43 | /detail       |              0 |               0
 87b5725f | 2017-01-09 12:20:22 | /search_list  |              0 |               0
 87b5725f | 2017-01-09 12:20:46 |               |              0 |               0
 87b5725f | 2017-01-09 12:21:26 | /search_input |              0 |               0
 87b5725f | 2017-01-09 12:22:51 | /search_list  |              0 |               0
 87b5725f | 2017-01-09 12:24:13 | /detail       |              0 |               0
 87b5725f | 2017-01-09 12:25:25 |               |              0 |               0
 8cc03a54 | 2017-01-09 12:18:43 | /search_list  |              1 |               1
 8cc03a54 | 2017-01-09 12:18:44 | /input        |              1 |               1
 8cc03a54 | 2017-01-09 12:18:45 | /confirm      |              1 |               1
 8cc03a54 | 2017-01-09 12:18:46 | /complete     |              1 |               1
 989004ea | 2017-01-09 12:18:43 | /search_list  |              0 |               0
 989004ea | 2017-01-09 12:19:27 | /search_input |              0 |               0
 989004ea | 2017-01-09 12:20:03 | /search_list  |              0 |               0
 9afaf87c | 2017-01-09 12:18:43 | /search_list  |              1 |               1
 9afaf87c | 2017-01-09 12:20:18 | /detail       |              1 |               1
 9afaf87c | 2017-01-09 12:21:39 | /input        |              1 |               1
 9afaf87c | 2017-01-09 12:22:52 | /confirm      |              1 |               1
 9afaf87c | 2017-01-09 12:23:00 | /complete     |              1 |               1
 cabf98e8 | 2017-01-09 12:18:43 | /search_input |              0 |               0
 d45ec190 | 2017-01-09 12:18:43 | /detail       |              0 |               0
 eee2bb21 | 2017-01-09 12:18:43 | /detail       |              0 |               0
 fe05e1d8 | 2017-01-09 12:18:43 | /detail       |              0 |               0
(36 rows)
```

## :closed_book: Reference

- [ビッグデータ分析・活用のためのSQLレシピ](https://book.mynavi.jp/supportsite/detail/9784839961268.html)
