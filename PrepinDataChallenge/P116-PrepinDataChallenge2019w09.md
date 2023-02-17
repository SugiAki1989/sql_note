## :memo: Overview

Preppin' Data challenge の「2019: Week 9」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/04/2019-week-9.html)
- [Answer](https://preppindata.blogspot.com/2019/04/2019-week-9-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。ツイッター上の企業への苦情に関するツイートのデータ。ここから一般的な単語を除外して、特定の単語のみにする、というお題。

```sql
 select * from p2019w09t1 limit 10;
                                                   tweet
-----------------------------------------------------------------------------------------------------------
 Hey @C&BSudsCo you suds are soap...I expected beer!
 WTF?! You’re Soap has just filled my bathroom full of bubbles!! Way to bubbly for me @C&BSudsCo
 What kind of moron name is @C&BSudsCo?
 No where near enough bubbles from your Soap Bar @C&BSudsCo. I wanted a bar of soap, not a chocolate bar
 My wife has accused me of having an affair you morons. You’ve over perfumed your Soap Bar @C&BSudsCo
 I just wanted a bar of soap, not to smell like a brothel?! Do you even smell your own products @C&BSudsCo
 Your soap has made my beard itchy, what the hell do you put in it?
 @C&BSudsCo when r u coming to Paris?
 I just bought my first @C&BSudsCo soap! Because I'm worth it! Actually I'm worth a lot more!
 @C&BSudsCo Who thought glitter in a beard shampoo was a good idea???
(10 rows)

select * from p2019w09t2 limit 10;
 rank | word
------+------
    1 | the
    2 | of
    3 | to
    4 | and
    5 | a
    6 | in
    7 | is
    8 | it
    9 | you
   10 | that
(10 rows)

```

企業名、句読点、連続空白を置換する。おそらくストップワードで除去されるので、`You’re`は`You re`として扱っておく。

```sql
select
    tweet,
	trim(
		regexp_replace(
			regexp_replace(
				replace(tweet, '@C&BSudsCo', ''), -- 固有名詞を先に削除,
				'[[:punct:]]', ' ', 'g'), -- 句読点を空白に置換すると空白が2個以上続くケースがでる
			'[ ]{2,}', ' ', 'g') -- 空白が2個以上続くケースを1つの空白に変換
	 ) as tweet_mod
from
	p2019w09t1
limit 5
;

                                                  tweet                                                  |                                         tweet_mod
---------------------------------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------
 Hey @C&BSudsCo you suds are soap...I expected beer!                                                     | Hey you suds are soap I expected beer
 WTF?! You’re Soap has just filled my bathroom full of bubbles!! Way to bubbly for me @C&BSudsCo         | WTF You re Soap has just filled my bathroom full of bubbles Way to bubbly for me
 What kind of moron name is @C&BSudsCo?                                                                  | What kind of moron name is
 No where near enough bubbles from your Soap Bar @C&BSudsCo. I wanted a bar of soap, not a chocolate bar | No where near enough bubbles from your Soap Bar I wanted a bar of soap not a chocolate bar
 My wife has accused me of having an affair you morons. You’ve over perfumed your Soap Bar @C&BSudsCo    | My wife has accused me of having an affair you morons You ve over perfumed your Soap Bar
(5 rows)

```

区切り文字の WideToLong 変換は`regexp_split_to_table`関数が便利なのでこれを利用する。このような関数がなければ文字を分割して、ユニオンする。この方法だと分割数が多くなると大変ではある。

```sql
with tmp as (
select
    tweet,
	trim(
		regexp_replace(
			regexp_replace(
				replace(tweet, '@C&BSudsCo', ''), -- 固有名詞を先に削除
				'[[:punct:]]', ' ', 'g'), -- 句読点を空白に置換すると空白が2個以上続くケースがでる
			'[ ]{2,}', ' ', 'g') -- 空白が2個以上続くケースを1つの空白に変換。[ ]で空白を指定し、{2,}で2個以上
	 ) as tweet_mod
from
	p2019w09t1
), tmp2 as (
select
    regexp_split_to_table(tweet_mod, ' ') as word
from
    tmp
)
select * from tmp2 limit 15
;

   word
----------
 Hey
 you
 suds
 are
 soap
 I
 expected
 beer
 WTF
 You
 re
 Soap
 has
 just
 filled
(15 rows)
```

あとはコモンワードを除外すれば一旦完成。ただ、小文字大文字が別単語扱いなので、`lower`関数で処理してからまとめても良いかもしれない。

```sql
with tmp as (
select
    tweet,
	trim(
		regexp_replace(
			regexp_replace(
				replace(tweet, '@C&BSudsCo', ''), -- 固有名詞を先に削除
				'[[:punct:]]', ' ', 'g'), -- 句読点を空白に置換すると空白が2個以上続くケースがでる
			'[ ]{2,}', ' ', 'g') -- 空白が2個以上続くケースを1つの空白に変換。[ ]で空白を指定し、{2,}で2個以上
	 ) as tweet_mod
from
	p2019w09t1
), tmp2 as (
select
    regexp_split_to_table(tweet_mod, ' ') as word
from
    tmp
)
select
    t1.word,
    count(1) as cnt
from
    tmp2 as t1
left join
    p2019w09t2 as t2
on
    t1.word = t2.word
where
    t2.rank is null
group by
    t1.word
order by
    cnt desc
;

  word    | cnt
-----------+-----
 soap      |   6
 Soap      |   3
 beard     |   3
 bar       |   3
 smell     |   2
 You       |   2
 worth     |   2
 bubbles   |   2
 not       |   2
 wanted    |   2
 Bar       |   2
 m         |   2
 Who       |   1
 Your      |   1
 accused   |   1
 affair    |   1
 bathroom  |   1
 beer      |   1

[略]
```

## :closed_book: Reference

None
