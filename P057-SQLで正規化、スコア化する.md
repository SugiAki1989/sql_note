## :memo: Overview

ここでは値の範囲が異なる指標を比較するためにいくつかある正規化の方法をまとめておく。中でも、MinMax 正規化、標準化、シグモイド変換を扱う。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`data analysis`

## :pencil2: Example

MinMax 正規化は変数の最小と最大を用いて、変換後の値が 0~1 になるように値を変換する方法。計算式については、ネットで検索すればたくさん出てくるので、そちらを参照。

```sql
-- salの型がintegerなので型変換を先にしておく
with tmp as (
select
    empno,
    sal::real
from
    emp
order by
    sal asc
)
select
    empno,
    sal,
    (sal - min(sal) over ()) / (max(sal) over () - min(sal) over ()) as minmax
from
    tmp
;

 empno | sal  |   minmax
-------+------+-------------
  7369 |  800 |           0
  7900 |  950 | 0.035714287
  7876 | 1100 | 0.071428575
  7521 | 1250 |  0.10714286
  7654 | 1250 |  0.10714286
  7934 | 1300 |  0.11904762
  7844 | 1500 |  0.16666667
  7499 | 1600 |   0.1904762
  7782 | 2450 |  0.39285713
  7698 | 2850 |  0.48809522
  7566 | 2975 |  0.51785713
  7902 | 3000 |  0.52380955
  7788 | 3000 |  0.52380955
  7839 | 5000 |           1
(14 rows)
```

次は標準化を計算する。標準化したデータの平均は 0 で分散が 1 になる。

```sql
with tmp as (
select
    empno,
    sal::real
from
    emp
order by
    sal asc
)
select
    empno,
    sal,
    avg(sal) over (),
    stddev(sal) over (),
    (sal - avg(sal) over ()) / stddev(sal) over () as z
from
    tmp
;

 empno | sal  |        avg        |       stddev       |          z
-------+------+-------------------+--------------------+---------------------
  7369 |  800 | 2073.214285714286 | 1182.5032235162716 |  -1.076711048557041
  7900 |  950 | 2073.214285714286 | 1182.5032235162716 | -0.9498615000594373
  7876 | 1100 | 2073.214285714286 | 1182.5032235162716 | -0.8230119515618335
  7521 | 1250 | 2073.214285714286 | 1182.5032235162716 | -0.6961624030642298
  7654 | 1250 | 2073.214285714286 | 1182.5032235162716 | -0.6961624030642298
  7934 | 1300 | 2073.214285714286 | 1182.5032235162716 | -0.6538792202316953
  7844 | 1500 | 2073.214285714286 | 1182.5032235162716 |  -0.484746488901557
  7499 | 1600 | 2073.214285714286 | 1182.5032235162716 | -0.4001801232364879
  7782 | 2450 | 2073.214285714286 | 1182.5032235162716 |  0.3186339849165997
  7698 | 2850 | 2073.214285714286 | 1182.5032235162716 |  0.6568994475768762
  7566 | 2975 | 2073.214285714286 | 1182.5032235162716 |  0.7626074046582126
  7902 | 3000 | 2073.214285714286 | 1182.5032235162716 |  0.7837489960744799
  7788 | 3000 | 2073.214285714286 | 1182.5032235162716 |  0.7837489960744799
  7839 | 5000 | 2073.214285714286 | 1182.5032235162716 |  2.4750763093758623
(14 rows)
```

平均は 0 、分散が 1 かどうか確かめておく。

```sql
with tmp as (
select
    empno,
    sal::real
from
    emp
order by
    sal asc
), tmp2 as (
select
    empno,
    sal,
    avg(sal) over (),
    stddev(sal) over (),
    (sal - avg(sal) over ()) / stddev(sal) over () as z
from
    tmp
)
select
    avg(z),
    variance(z),
    stddev(z)
from
    tmp2
;

 avg | variance | stddev
-----+----------+--------
   0 |        1 |      1
(1 row)
```

正規化ではないかもしれないが、最後はシグモイド関数や `log` を用いた変換。例えば、メルマガのクリック日時からの日数差を変換する。現時点から直近であれば大きな重みをもたせて、30 日後、60 日後、90 日後など、それなりの日数が経過しているものは、ほとんど同じように値に変換することがシグモイド関数を利用すればできる。

クリックからの経過日数が記録されている都合の良いテーブルを用意する。今日時点から最新のアクション日時を引けば経過日数を計算できるので、ない場合は計算する。

```sql
create table sigmoid(cuid varchar(255), elapsed_days int);

insert into
    sigmoid(cuid, elapsed_days)
values
    ('1','0'),
    ('2','1'),
    ('3','2'),
    ('4','3'),
    ('5','4'),
    ('6','5'),
    ('7','10'),
    ('8','30'),
    ('9','60'),
    ('10','90');
```

ここでは画像のようなシグモイド関数(ゲイン`a=0.5`)や `log2` を使って異なる減衰を重みとして表したもので重み付けを行う。どちらが正解というのは特にない。シグモイド関数の方では、1 週間後のスコアは 0 なので、購買意欲と捉えると、1 週間後の購買意欲はクリックした日から考えるとないものとも考えられる。

![スクリーンショット 2022-08-24 16 20 26](https://user-images.githubusercontent.com/65038325/186356061-221ec9bd-84c7-4626-87f4-138161c83ac9.png)

```sql
select
    cuid,
    elapsed_days,
    round(
        1/log(2, 0.5 * elapsed_days + 2)
        , 2) as logconv,
    round(
        2 - (2 / (1 + exp(-0.5*elapsed_days)))
        , 2) as sigconv
from
    sigmoid
;

 cuid | elapsed_days | logconv | sigmoid
------+--------------+---------+---------
 1    |            0 |    1.00 |    1.00
 2    |            1 |    0.76 |    0.76
 3    |            2 |    0.63 |    0.54
 4    |            3 |    0.55 |    0.36
 5    |            4 |    0.50 |    0.24
 6    |            5 |    0.46 |    0.15
 7    |           10 |    0.36 |    0.01
 8    |           30 |    0.24 |    0.00
 9    |           60 |    0.20 |    0.00
 10   |           90 |    0.18 |    0.00
(10 rows)
```

## :closed_book: Reference

None
