## :memo: Overview

Excel のセル結合をしたような見た目を作成する方法。1 列に同じ値が複数表示された場合に、先頭業だけの値を残して、それ以下の行の値は非表示にしたい。下記のようなイメージ。

```sql
  species  |  sex
-----------+--------
 Adelie    | female
           | male
 Chinstrap | male
           | female
 Gentoo    | female
           | male
(6 rows)
```

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`lag`

## :pencil2: Example

下記のデータをここでは利用する

- [palmerpenguins](https://allisonhorst.github.io/palmerpenguins/)

"rowid","species","island","bill_length_mm","bill_depth_mm","flipper_length_mm","body_mass_g","sex","year"

```sql
create table penguins(
    rowid integer,
    species varchar(10),
    island varchar(10),
    bill_length_mm real,
    bill_depth_mm real,
    flipper_length_mm real,
    body_mass_g real,
    sex varchar(10),
    year integer
    );

-- seaborn-data
wget https://gist.githubusercontent.com/slopp/ce3b90b9168f2f921784de84fa445651/raw/4ecf3041f0ed4913e7c230758733948bc561f434/penguins.csv
sed -e 's/NA//g' penguins.csv > penguins_mod.csv

head -n10 penguins_mod.csv
"rowid","species","island","bill_length_mm","bill_depth_mm","flipper_length_mm","body_mass_g","sex","year"
"1","Adelie","Torgersen",39.1,18.7,181,3750,"male",2007
"2","Adelie","Torgersen",39.5,17.4,186,3800,"female",2007
"3","Adelie","Torgersen",40.3,18,195,3250,"female",2007
"4","Adelie","Torgersen",NA,NA,NA,NA,NA,2007
"5","Adelie","Torgersen",36.7,19.3,193,3450,"female",2007
"6","Adelie","Torgersen",39.3,20.6,190,3650,"male",2007
"7","Adelie","Torgersen",38.9,17.8,181,3625,"female",2007
"8","Adelie","Torgersen",39.2,19.6,195,4675,"male",2007
"9","Adelie","Torgersen",34.1,18.1,193,3475,NA,2007

realpath penguins_mod.csv
\copy penguins from '/Users/aki/Desktop/penguins_mod.csv' (encoding 'utf8', format csv, header true);
```

まずは、`species`と`sex`のユニークな組み合わせを作成する。

```sql
select distinct
    species,
    sex
from
    penguins
where
    sex is not null
order by
    species
;

  species  |  sex
-----------+--------
 Adelie    | female
 Adelie    | male
 Chinstrap | female
 Chinstrap | male
 Gentoo    | male
 Gentoo    | female
(6 rows)
```

`species`は各 2 行ずつレコードを持っているので 1 行目だけ値を残したいので、`lag`でずらして値が一致するところは`null`に変換する。

```sql
with tmp as (
select distinct
    species,
    sex
from
    penguins
where
    sex is not null
order by
    species
)
select
    case when lag(species) over (order by species) = species then null
    else species end as species,
    sex
from
    tmp
;

  species  |  sex
-----------+--------
 Adelie    | female
           | male
 Chinstrap | female
           | male
 Gentoo    | male
           | female
(6 rows)

```

実際のデータを`lag`でずらしたときのイメージは下記の通り。

```sql
with tmp as (
select distinct
    species,
    sex
from
    penguins
where
    sex is not null
order by
    species
)
select
    species,
    lag(species) over (partition by species order by species) as lagspecies,
    sex
from
    tmp
;
  species  | lagspecies |  sex
-----------+------------+--------
 Adelie    |            | female
 Adelie    | Adelie     | male
 Chinstrap |            | female
 Chinstrap | Chinstrap  | male
 Gentoo    |            | male
 Gentoo    | Gentoo     | female
(6 rows)
```

## :closed_book: Reference

- [SQL Cookbook](https://www.oreilly.com/library/view/sql-cookbook/0596009763/)
- [SQL クックブック 第 2 版](https://www.oreilly.co.jp/books/9784873119779/)
