## :memo: Overview

ここでは完全一致や不等号判定ではなく、類似する文字列での結合を行う結合の方法のことを`fuzzy join`という。ここでは、SQL で `pg_trgm` モジュールを使った` fuzzy join``を行う方法をまとめておく。pg_trgm ` モジュールのドキュメントは下記。

- [F.33. pg_trgm](https://www.postgresql.jp/document/14/html/pgtrgm.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`fuzzy join`

## :pencil2: Example

まずはサンプルデータを用意する。ここではミススペルと正解のスペルが記録されているテーブルを用意する。[Wikipedia:Lists of common misspellings/For machines](https://en.wikipedia.org/wiki/Wikipedia:Lists_of_common_misspellings/For_machines)を参考にしている。

```sql
create table fuzzy(
    rownum integer,
    misspelling varchar(255),
    correct varchar(255)
);

copy fuzzy from '/Users/aki/Desktop/misspellings.csv' with csv header;

select * from fuzzy where rownum < 10;

 rownum | misspelling |  correct
--------+-------------+------------
      1 | abandonned  | abandoned
      2 | aberation   | aberration
      3 | abilties    | abilities
      4 | abilty      | ability
      5 | abondon     | abandon
      6 | abbout      | about
      7 | abotu       | about
      8 | abouta      | about a
      9 | aboutit     | about it
(9 rows)
```

`pg_trgm` モジュールを有効にする。

```sql
CREATE EXTENSION pg_trgm;

\dx
                                     List of installed extensions
   Name    | Version |   Schema   |                            Description
-----------+---------+------------+-------------------------------------------------------------------
 pg_trgm   | 1.6     | public     | text similarity measurement and index searching based on trigrams

```

pg_trgm モジュールの関数は、基本的に`show_trgm`関数で与えられる文字列内のすべてのトライグラムを利用して、文字列の類似度をスコア化する。

```sql
select misspelling, show_trgm(misspelling) from fuzzy limit 10;
 misspelling |                      show_trgm
-------------+-----------------------------------------------------
 abandonned  | {"  a"," ab",aba,and,ban,don,"ed ",ndo,ned,nne,onn}
 aberation   | {"  a"," ab",abe,ati,ber,era,ion,"on ",rat,tio}
 abilties    | {"  a"," ab",abi,bil,"es ",ies,ilt,lti,tie}
 abilty      | {"  a"," ab",abi,bil,ilt,lty,"ty "}
 abondon     | {"  a"," ab",abo,bon,don,ndo,"on ",ond}
 abbout      | {"  a"," ab",abb,bbo,bou,out,"ut "}
 abotu       | {"  a"," ab",abo,bot,otu,"tu "}
 abouta      | {"  a"," ab",abo,bou,out,"ta ",uta}
 aboutit     | {"  a"," ab",abo,bou,"it ",out,tit,uti}
 aboutthe    | {"  a"," ab",abo,bou,"he ",out,the,tth,utt}
(10 rows)

select correct, show_trgm(correct) from fuzzy limit 10;
  correct   |                       show_trgm
------------+-------------------------------------------------------
 abandoned  | {"  a"," ab",aba,and,ban,don,"ed ",ndo,ned,one}
 aberration | {"  a"," ab",abe,ati,ber,err,ion,"on ",rat,rra,tio}
 abilities  | {"  a"," ab",abi,bil,"es ",ies,ili,iti,lit,tie}
 ability    | {"  a"," ab",abi,bil,ili,ity,lit,"ty "}
 abandon    | {"  a"," ab",aba,and,ban,don,ndo,"on "}
 about      | {"  a"," ab",abo,bou,out,"ut "}
 about      | {"  a"," ab",abo,bou,out,"ut "}
 about a    | {"  a"," a "," ab",abo,bou,out,"ut "}
 about it   | {"  a","  i"," ab"," it",abo,bou,"it ",out,"ut "}
 about the  | {"  a","  t"," ab"," th",abo,bou,"he ",out,the,"ut "}
(10 rows)
```

ドキュメントにスコアリングの簡単な紹介がある。

> 最初の文字列では、トライグラムの集合は{" w"," wo","wor","ord","rd "}です。 二番目の文字列では、順序付きトライグラムの集合は{" t"," tw",two,"wo "," w"," wo","wor","ord","rds", "ds "}です。 二番目の文字列中の順序付きトライグラムの集合の中で最も類似度の高い範囲は、{" w"," wo","wor","ord"}で、類似度は 0.8 となります。

`misspelling`の言葉に近い言葉をスコアを利用して紐付ける。ここでは処理の関係上、単語を 10 単語に限定している。類似度スコアが 0.3 より大きいもの表示しているが、似ている単語を紐付けられていることがわかる。

```sql
with tmp as (
select
    f.misspelling as target_word,
    ff.misspelling as candidate_word,
    f.correct
from fuzzy as f
cross join (select misspelling from fuzzy) as ff
where f.rownum <= 10 and f.misspelling != ff.misspelling
order by f.misspelling asc
)
select
    target_word,
    candidate_word,
    correct,
    similarity(target_word, candidate_word) as score
from tmp
where similarity(target_word, candidate_word) > 0.3
;

 target_word | candidate_word |  correct   |   score
-------------+----------------+------------+------------
 abandonned  | abondoned      | abandoned  |        0.4
 abandonned  | adbandon       | abandoned  | 0.33333334
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 abbout      | aboutit        | about      | 0.36363637
 abbout      | abouta         | about      |        0.4
 abbout      | aboutthe       | about      | 0.33333334
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 aberation   | incorperation  | aberration | 0.33333334
 aberation   | imigration     | aberration |     0.3125
 aberation   | imigration     | aberration |     0.3125
 aberation   | coorperation   | aberration |  0.3529412
 aberation   | coorperation   | aberration |  0.3529412
 aberation   | avation        | aberration |  0.3846154
 aberation   | assocation     | aberration |     0.3125
 aberation   | aplication     | aberration |     0.3125
 aberation   | adminstration  | aberration | 0.33333334
 aberation   | absorbtion     | aberration |     0.3125
 aberation   | seperation     | aberration |        0.4
 aberation   | refridgeration | aberration | 0.31578946
 aberation   | preperation    | aberration |      0.375
 aberation   | preliferation  | aberration | 0.33333334
 aberation   | accelleration  | aberration |  0.4117647
 aberation   | abreviation    | aberration |      0.375
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 abilties    | abilty         | abilities  | 0.45454547
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 abilty      | abilties       | ability    | 0.45454547
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 abondon     | adbandon       | abandon    | 0.30769232
 abondon     | abondoning     | abandon    |  0.5833333
 abondon     | abondoned      | abandon    |  0.6363636
 abondon     | abondons       | abandon    |        0.7
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 abotu       | abouta         | about      |        0.3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 abouta      | abbout         | about a    |        0.4
 abouta      | aboutthe       | about a    | 0.45454547
 abouta      | aboutit        | about a    |        0.5
 abouta      | abotu          | about a    |        0.3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 aboutit     | abouta         | about it   |        0.5
 aboutit     | aboutthe       | about it   | 0.41666666
 aboutit     | abbout         | about it   | 0.36363637
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 aboutthe    | aboutit        | about the  | 0.41666666
 aboutthe    | abbout         | about the  | 0.33333334
 aboutthe    | abouta         | about the  | 0.45454547
(38 rows)
```

より類似度の高い単語を紐付けたければ、類似度スコアを 1 に近づける。

```sql
-- where similarity(target_word, candidate_word) > 0.5
 target_word | candidate_word | correct |   score
-------------+----------------+---------+-----------
 abondon     | abondoned      | abandon | 0.6363636
 abondon     | abondoning     | abandon | 0.5833333
 abondon     | abondons       | abandon |       0.7
(3 rows)
```

## :closed_book: Reference

- [F.33. pg_trgm](https://www.postgresql.jp/document/14/html/pgtrgm.html)
