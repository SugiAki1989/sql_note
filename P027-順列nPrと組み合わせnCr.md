## :memo: Overview

SQL で組み合わせ、順列を計算する方法をまとめる。基本的に、SQL で順列や組み合わせを用いた計算を行うことはあまりなく、必要あれば、R や Python に
おまかせすることになるが、頭の体操としては良いのでまとめておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`permutation`, `combination`

## :pencil2: Example

下記のページを参考にさせて頂いた。

- [組合せの基本と順列との関係、二項係数 nCr の等式の証明](https://examist.jp/mathematics/baainokazu/kumiawase/)

```sql
create table ball(color varchar(5));
insert into ball
    (color)
values
    ('red'),
    ('blue'),
    ('black')
;
```

順列は、いくつかのものを、順序をつけて 1 列に並べることで、組み合わせは、異なる n 個のものの中から r 個取り出したときの、組み合わせ数のことで順序は考慮しない。

たとえば、`red`、`blue`、`black`の 3 つのボールから 2 つ選ぶ選び方は組み合わせは、「`red`、`blue`」「`red`、`black`」「`blue`、`black`」の 3 通り。一方で順列は「`red`、`blue`」と「`blue`、`red`」は別のものとして考えるので、「`red`、`blue`」「`blue`、`red`」「`red`、`black`」「`black`、`red`」「`blue`、`black`」「`black`、`blue`」の 6 通りあります。

ここまで書くと SQL ではどうすればよいかなんとなくわかってくる。直積すればなんとかできそう。3 個から 2 個を選んで並べる順列の総数は 6 通りなので、これを SQL で計算してみる。

```sql
select b1.color as b1color, b2.color as b2color
from ball as b1 cross join ball as b2;

 b1color | b2color
---------+---------
 red     | red
 red     | blue
 red     | black
 blue    | red
 blue    | blue
 blue    | black
 black   | red
 black   | blue
 black   | black
(9 rows)
```

これでは順列ではなく、重複順列なので、自分を選んだら次には自分を選べないという表現を加える。

```sql
select b1.color as b1color, b2.color as b2color
from ball as b1 cross join ball as b2
where b1.color != b2.color;

 b1color | b2color
---------+---------
 red     | blue
 red     | black
 blue    | red
 blue    | black
 black   | red
 black   | blue
(6 rows)
```

お次は組み合わせを考える。3 個から 2 個を選ぶ組み合わせの数は 3 通りなので、これを SQL で計算してみる。さきほどの順列の計算から同じ組み合わせを除外すればよい。数学的に考えると`nCr = nPr / r!`となり、順序は異なっても同じ要素は`r!`個あるのでこれを除外すればよい。

ということでさきほどの順列の結果を眺めていると、下記の各組み合わせの部分はどちらかが取れれば良いとうことがわかる。

```sql
 b1color | b2color
---------+---------
 red     | blue　--  red   > blue → true
 blue    | red 　--  red   > blue → false

 red     | black --  red   > black → true
 black   | red   --  black > red   → false

 blue    | black --  blue  > black → true
 black   | blue  --  black > blue  → false
(6 rows)
```

これをさらに条件として追加すれば、組み合わせが計算できる。

```sql
select b1.color as b1color, b2.color as b2color
from ball as b1 cross join ball as b2
where b1.color != b2.color and b1.color > b2.color;

 b1color | b2color
---------+---------
 red     | blue
 red     | black
 blue    | black
(3 rows)
```

これを拡張して 4 つから 3 つの順列と組み合わせを計算してみる。

```sql
create table ball2(color varchar(5));
insert into ball2
    (color)
values
    ('red'),
    ('blue'),
    ('black'),
    ('white')
;
```

まずはさきほどと同じように直積を求める。`4^3=64`が返ってくる。

```sql
select
    b1.color as b1color,
    b2.color as b2color,
    b3.color as b3color
from
    ball2 as b1
cross join ball2 as b2
cross join ball2 as b3
;

 b1color | b2color | b3color
---------+---------+---------
 red     | red     | red
 red     | red     | blue
 red     | red     | black
 red     | red     | white
 red     | blue    | red
 red     | blue    | blue
 red     | blue    | black
 red     | blue    | white
 red     | black   | red
 red     | black   | blue
 red     | black   | black
 red     | black   | white
 red     | white   | red
 red     | white   | blue
 red     | white   | black
 red     | white   | white
 略
 white   | black   | white
 white   | white   | red
 white   | white   | blue
 white   | white   | black
 white   | white   | white
(64 rows)
```

実際は`4P3=24`なので、先ほどと同じように自分とは結合できない条件を加える。

```sql
select
    b1.color as b1color,
    b2.color as b2color,
    b3.color as b3color
from
    ball2 as b1
cross join ball2 as b2
cross join ball2 as b3
where
    b1.color != b2.color and
    b2.color != b3.color and
    b3.color != b1.color
;

 b1color | b2color | b3color
---------+---------+---------
 red     | blue    | black
 red     | blue    | white
 red     | black   | blue
 red     | black   | white
 red     | white   | blue
 red     | white   | black
 blue    | red     | black
 blue    | red     | white
 blue    | black   | red
 blue    | black   | white
 blue    | white   | red
 blue    | white   | black
 black   | red     | blue
 black   | red     | white
 black   | blue    | red
 black   | blue    | white
 black   | white   | red
 black   | white   | blue
 white   | red     | blue
 white   | red     | black
 white   | blue    | red
 white   | blue    | black
 white   | black   | red
 white   | black   | blue
(24 rows)
```

これは直感的にわかりにくいが、下記のようなことをしている。

```sql
with tmp as (
select
    b1.color as b1color,
    b2.color as b2color,
    b3.color as b3color
from
    ball2 as b1
cross join ball2 as b2
cross join ball2 as b3
)
select
    b1color, b2color, b3color,
    case when b1color != b2color and b2color != b3color and b3color != b1color then 1
    else 0 end
from
    tmp
;

 b1color | b2color | b3color | case
---------+---------+---------+------
 red     | red     | red     |    0 `red`を選んで、`red`を選んで、`red`を選んでいるのでNG
 red     | red     | blue    |    0 `red`を選んで、`red`を選んでいるのでNG
 red     | red     | black   |    0 `red`を選んで、`red`を選んでいるのでNG
 red     | red     | white   |    0 `red`を選んで、`red`を選んでいるのでNG
 red     | blue    | red     |    0 `red`を選んで、`blue`を選んでいるが、`red`を選んでいるのでNG
 red     | blue    | blue    |    0 `red`を選んで、`blue`を選んでいるが、`blue`を選んでいるのでNG
 red     | blue    | black   |    1
 red     | blue    | white   |    1
 red     | black   | red     |    0 `red`を選んで、`black`を選んでいるが、`red`を選んでいるのでNG
 red     | black   | blue    |    1
 red     | black   | black   |    0 `red`を選んで、`black`を選んでいるが、`black`を選んでいるのでNG
 red     | black   | white   |    1
 red     | white   | red     |    0 `red`を選んで、`white`を選んでいるが、`red`を選んでいるのでNG
 red     | white   | blue    |    1
 red     | white   | black   |    1
 red     | white   | white   |    0 `red`を選んで、`white`を選んでいるが、`white`を選んでいるのでNG
略
```

さきほどの順列の計算から同じ組み合わせを除外すればよく、順序は異なっても同じ要素は`r!`個あるのでこれを除外すればよい。

```sql
select
    b1.color as b1color,
    b2.color as b2color,
    b3.color as b3color
from
    ball2 as b1
cross join ball2 as b2
cross join ball2 as b3
where
    b1.color != b2.color and b2.color != b3.color and b3.color != b1.color and
    b1.color >  b2.color and b2.color >  b3.color
;

 b1color | b2color | b3color
---------+---------+---------
 red     | blue    | black
 white   | red     | blue
 white   | red     | black
 white   | blue    | black
(4 rows)
```

これも直感的にわかりにくいが、下記のようなことをしている。下記の各組み合わせの部分は 1 つ取れれば良いとうことがわかる。

```sql
with tmp as (
select
    b1.color as b1color,
    b2.color as b2color,
    b3.color as b3color
from
    ball2 as b1
cross join ball2 as b2
cross join ball2 as b3
where b1.color != b2.color and b2.color != b3.color and b3.color != b1.color
)
select
    b1color, b2color, b3color,
    case when b1color >  b2color and b2color >  b3color then 1
    else 0 end
from
    tmp
;
 b1color | b2color | b3color | case
---------+---------+---------+------
-- {red, blue, black}
 red     | blue    | black   |    1 --true  & true  → true
 red     | black   | blue    |    0 --true  & false → false
 blue    | red     | black   |    0 --false & false → false
 blue    | black   | red     |    0 --false & false → false
 black   | red     | blue    |    0 --false & false → false
 black   | blue    | red     |    0 --false & false → false

-- {white, red, blue}
 white   | red     | blue    |    1　--true  & true  → true
 white   | blue    | red     |    0 --false & false → false
 red     | blue    | white   |    0 --false & false → false
 red     | white   | blue    |    0 --false & false → false
 blue    | red     | white   |    0 --false & false → false
 blue    | white   | red     |    0 --false & false → false

-- {white, red, black}
 white   | red     | black   |    1　--true  & true  → true
 white   | black   | red     |    0 --false & false → false
 red     | black   | white   |    0 --false & false → false
 red     | white   | black   |    0 --false & false → false
 black   | red     | white   |    0 --false & false → false
 black   | white   | red     |    0 --false & false → false

-- {white, blue, black}
 white   | blue    | black   |    1　--true  & true  → true
 white   | black   | blue    |    0 --false & false → false
 blue    | black   | white   |    0 --false & false → false
 blue    | white   | black   |    0 --false & false → false
 black   | blue    | white   |    0 --false & false → false
 black   | white   | blue    |    0 --false & false → false
(24 rows)
```

文字で考えると少しわかりにくいが、数字に置き換えるとわかりよい。組み合わせは下記が必要。

```sql
{1，2，3}
{1，2，4}
{1，3，4}
{2，3，4}
```

ここで順列を組み合わせでグループ化して考えると、`{1，2，3}`の要素の順列は必ず、`1,2,3`のいずれかが並んでいるので、左から大きいもの順に並んでいるものは、`★`しか残らない。

```sql
-- {1，2，3}の要素の順列
{1，2，3}，
{1，3，2}，
{2，1，3}，
{2，3，1}，
{3，1，2}，
{3，2，1} -- ★

-- {1，2，4}の要素の順列
{1，2，4}，
{1，4，2}，
{2，1，4}，
{2，4，1}，
{4，1，2}，
{4，2，1} -- ★

-- {1，3, 4}の要素の順列
{1，3，4}，
{1，4，3}，
{3，1，4}，
{3，4，1}，
{4，1，3}，
{4，3，1} -- ★

-- {2，3, 4}の要素の順列
{2，3，4}，
{2，4，3}，
{3，2，4}，
{3，4，2}，
{4，2，3}，
{4，3，2} -- ★
```

## :closed_book: Reference

None
