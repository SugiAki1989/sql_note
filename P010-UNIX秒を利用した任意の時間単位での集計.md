## :memo: Overview

任意の単位時間で集計したい場合に日付を丸める方法をまとめる。例えば 15 分ごと、30 分ごとなど任意の時間単位で集計する方法のことで、ここでは UNIX 秒を使った方法をまとめておく。

## :floppy_disk: Database

PostgreSQL / MySQL

## :bookmark: Tag

`from_unixtime`, `truncate`, `unix_timestamp`, `to_timestamp`, `extract(epoch)`, `trunc`

## :pencil2: Example

UNIX秒を扱う簡単なおさらいから始める。システムのタイムゾーンが日本に設定されている場合の例。

```sql
SET timezone TO 'Asia/Tokyo';
show timezone;
  TimeZone  
------------
 Asia/Tokyo
(1 row)
```

`+09`は、UTCからのオフセットを示す記号で、UTCとの時差に基づいて表示される。この場合、日本時間はUTCより9時間進んでいることを意味する。つまり、`+09`は`UTC+09:00`であり、日本標準時を指す。
ということで、`1970-01-01 00:00:00+09`は、日本時間の`1970-01-01 00:00:00`時点のことであり、イギリス(UTC)基準から9時間進んでいて、イギリスでは`1970-12-31 15:00:00`時点だよ、ということ。

そのため、UTCで「1970年1月1日午前0時0分0秒」を基準にして、そこからの経過秒数で日時を表現するUNIX時間は、JSTの`1970-01-01 00:00:00+09`の場合、9時間分の時差を考慮して`-32400`となる。イギリス(UTC)ではまだ「1970年1月1日午前0時0分0秒」を迎えておらず、`1970-12-31 15:00:00`時点で、`-32400`秒前となる。

つまり、日本標準時に、ここから`+32400`秒を足し込むと、UNIX時間は0秒となる。日本標準時でのUNIX時間が`0`秒の時点は、`1970-01-01 09:00:00+09`(朝9時時点)である。このタイミングで、イギリス(UTC)では、やっと「1970年1月1日午前0時0分0秒」を迎えるので、UNIX時間は`0`秒となる。

```sql
with tmp as (
SELECT 
'1970-01-01 00:00:00'::timestamp with time zone as jst
)
select
    jst,
    extract(epoch from jst) as jst_unix,
    extract(epoch from jst) + 32400 as jst_unix_plus9,
    to_timestamp(extract(epoch from jst) + 32400) as jst_plus9,
		-- RedShiftの場合
    -- convert_timezone('jst', timestamp 'epoch' + cast(extract(epoch from jst)+ 32400 as bigint) * interval '1 second') as jst_plus9_2
    extract(epoch from to_timestamp(extract(epoch from jst) + 32400)) as jst_unix_valid
from
    tmp
;
-[ RECORD 1 ]--+-----------------------
jst            | 1970-01-01 00:00:00+09
jst_unix       | -32400.000000
jst_unix_plus9 | 0.000000
jst_plus9      | 1970-01-01 09:00:00+09
jst_unix_valid | 0.000000
```

たまにテーブル定義書の説明に「UNIX時間(JST)」と書かれている場合がある。例えば、このカラムの値が`0`であれば、これは`1970-01-01 09:00:00+09`時点のことを指しており、`1970-01-01 00:00:00+09`時点のことではないはず(たぶんあってる)。なので、`0`を`1970-01-01 09:00:00+09`に戻せばよいと思われる。なんでUTC基準のUNIX時間をJSTのUNIX秒にして保存する必要があるのかはわからない。


ここからは任意の単位時間で集計したい場合に日付を丸める方法をまとめる。ここでは、15 分単位で時間を区切って、集計する。まずは、`2022-07-26 15:30:00`から`2022-07-26 16:30:00`までの値をもつサンプルテーブルを`generate_series`を利用して作成する。

```sql
select createdat, round((random() * (1 - 100))::numeric, 0) + 100 as value
from generate_series('2022-07-26 15:30:00'::timestamp with time zone,
					 '2022-07-26 16:30:00'::timestamp with time zone,
					 '1 minute') as createdat
;

       createdat        | value
------------------------+-------
 2022-07-26 15:30:00+09 |    89
 2022-07-26 15:31:00+09 |    88
 2022-07-26 15:32:00+09 |    83
略
 2022-07-26 16:28:00+09 |    29
 2022-07-26 16:29:00+09 |    42
 2022-07-26 16:30:00+09 |    32
```

このテーブルに対して、下記のようなクエリを発行する。内容は、日時を UNIX 秒に変換して、900 秒で割り、端数を切り落とす。ここまでの処理で、15 分単位でグループ化されているので、900 秒をかけて 15 分単位の UNIX 秒に戻したものを日時に再度、変換する。

```sql
-- 15分ごとだと15分×60秒で1800秒
-- 30分ごとだと30分×60秒で1800秒
with logs as (select createdat, round((random() * (1 - 100))::numeric, 0) + 100 as value
from generate_series('2022-07-26 15:30:00'::timestamp with time zone,
                     '2022-07-26 16:30:00'::timestamp with time zone,
                     '1 minute') as createdat
)
select
    createdat,
    extract(epoch from createdat) as unixtimestamp,
    extract(epoch from createdat) / 900 as unixtimestamp900,
    trunc(extract(epoch from createdat) / 900, 0) unixtimestamp900trunc,
    trunc(extract(epoch from createdat) / 900, 0) * 900 as unixtimestamp900trunc900,
    to_timestamp(trunc(extract(epoch from createdat) / 900, 0) * 900) as every15min
from
    logs
;
       createdat        |   unixtimestamp   |   unixtimestamp900   | unixtimestamp900trunc | unixtimestamp900trunc900 |       every15min
------------------------+-------------------+----------------------+-----------------------+--------------------------+------------------------
 2022-07-26 15:30:00+09 | 1658817000.000000 | 1843130.000000000000 |               1843130 |               1658817000 | 2022-07-26 15:30:00+09
 2022-07-26 15:31:00+09 | 1658817060.000000 | 1843130.066666666667 |               1843130 |               1658817000 | 2022-07-26 15:30:00+09
 2022-07-26 15:32:00+09 | 1658817120.000000 | 1843130.133333333333 |               1843130 |               1658817000 | 2022-07-26 15:30:00+09
略
 2022-07-26 15:44:00+09 | 1658817840.000000 | 1843130.933333333333 |               1843130 |               1658817000 | 2022-07-26 15:30:00+09
 2022-07-26 15:45:00+09 | 1658817900.000000 | 1843131.000000000000 |               1843131 |               1658817900 | 2022-07-26 15:45:00+09
 2022-07-26 15:46:00+09 | 1658817960.000000 | 1843131.066666666667 |               1843131 |               1658817900 | 2022-07-26 15:45:00+09
略
 2022-07-26 15:59:00+09 | 1658818740.000000 | 1843131.933333333333 |               1843131 |               1658817900 | 2022-07-26 15:45:00+09
 2022-07-26 16:00:00+09 | 1658818800.000000 | 1843132.000000000000 |               1843132 |               1658818800 | 2022-07-26 16:00:00+09
 2022-07-26 16:01:00+09 | 1658818860.000000 | 1843132.066666666667 |               1843132 |               1658818800 | 2022-07-26 16:00:00+09
略
 2022-07-26 16:14:00+09 | 1658819640.000000 | 1843132.933333333333 |               1843132 |               1658818800 | 2022-07-26 16:00:00+09
 2022-07-26 16:15:00+09 | 1658819700.000000 | 1843133.000000000000 |               1843133 |               1658819700 | 2022-07-26 16:15:00+09
 2022-07-26 16:16:00+09 | 1658819760.000000 | 1843133.066666666667 |               1843133 |               1658819700 | 2022-07-26 16:15:00+09
略
 2022-07-26 16:29:00+09 | 1658820540.000000 | 1843133.933333333333 |               1843133 |               1658819700 | 2022-07-26 16:15:00+09
略
 2022-07-26 16:30:00+09 | 1658820600.000000 | 1843134.000000000000 |               1843134 |               1658820600 | 2022-07-26 16:30:00+09
```

15 分単位で集計する場合は、`every15min`カラムを利用して、集計すれば目的を達成できる。

```sql
with logs as (select createdat, round((random() * (1 - 100))::numeric, 0) + 100 as value
from generate_series('2022-07-26 15:30:00'::timestamp with time zone,
					 '2022-07-26 16:30:00'::timestamp with time zone,
					 '1 minute') as createdat
), log2 as (
select
	createdat,
	to_timestamp(trunc(extract(epoch from createdat) / 900, 0) * 900) as every15min,
	value
from
	logs
)
select
	every15min,
	sum(value)
from
	log2
group by
	every15min
order by
	every15min asc
;
       every15min       | sum
------------------------+-----
 2022-07-26 15:30:00+09 | 728
 2022-07-26 15:45:00+09 | 613
 2022-07-26 16:00:00+09 | 720
 2022-07-26 16:15:00+09 | 710
 2022-07-26 16:30:00+09 |  90
```

MySQL で行う場合は対応する関数に書き直せば OK。30 分単位に変更した結果を記載しておく。`generate_series`を利用できないので、探してみたところ、MySQL でも擬似的に`generate_series`を再現できるそうなので、下記を参考にさせていただきました。

- [裏 MySQL クエリー入門(15) 応用編 3 MySQL で連番の仮想表を作成](https://it7c.hatenadiary.org/entry/20100713/1278950305)

```sql
with logs as (
select
	date_add('2022-07-26 15:30:00', interval td.generate_series minute) as createdat,
	round((rand() * (1 - 100)), 0) + 100 as value
from
	(
	select 0 generate_series from dual where (@num:=1-1)*0
	union all
	select @num:=@num+1 from `information_schema`.columns limit 60
	) as td
), log2 as (
select
    createdat,
	from_unixtime(truncate(unix_timestamp(createdat) / 1800, 0) * 1800) as every15min,
    value
from
	logs
)
select
	every15min,
    sum(value)
from
	log2
group by
	every15min
order by
	every15min asc
;

+---------------------+------------+
| every15min          | sum(value) |
+---------------------+------------+
| 2022-07-26 15:30:00 |       1239 |
| 2022-07-26 16:00:00 |       1468 |
| 2022-07-26 16:30:00 |         86 |
+---------------------+------------+
```

## :closed_book: Reference

- [裏 MySQL クエリー入門(15) 応用編 3 MySQL で連番の仮想表を作成](https://it7c.hatenadiary.org/entry/20100713/1278950305)
- [MySQL 日時ごとの集計まとめ](https://qiita.com/yakatsuka/items/2906011803500ebd4390)
- [9.9. 日付/時刻関数と演算子](https://www.postgresql.jp/document/13/html/functions-datetime.html)
- [12.7 日付および時間関数](https://dev.mysql.com/doc/refman/8.0/ja/date-and-time-functions.html#function_date-format)
