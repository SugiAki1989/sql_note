## :memo: Overview

Preppin' Data challenge の「2019: Week 18」の問題を SQL で解答する。問題文と解答は下記の公式サイトより確認下さい。

- [Question](https://preppindata.blogspot.com/2019/06/2019-week-18.html)
- [Answer](https://preppindata.blogspot.com/2019/06/2019-week-18-solution.html)

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`Preppin'Data challenge`

## :pencil2: Example

提供されたデータこちら。アニメのレーティングに関するデータセットで下記の集計を行うというお題。

- 平均評価 (小数点以下第 2 位まで)。
- 平均視聴率 (小数点以下第 0 位まで)。
- 最高評価

```sql
select * from p2019w18t1 limit 10;

 anime_id |                           name                            |                            genre                             | type  | episodes | rating | members
----------+-----------------------------------------------------------+--------------------------------------------------------------+-------+----------+--------+---------
    32281 | Kimi no Na wa.                                            | Drama, Romance, School, Supernatural                         | Movie | 1        |   9.37 |  200630
     5114 | Fullmetal Alchemist: Brotherhood                          | Action, Adventure, Drama, Fantasy, Magic, Military, Shounen  | TV    | 64       |   9.26 |  793665
    28977 | Gintama°                                                  | Action, Comedy, Historical, Parody, Samurai, Sci-Fi, Shounen | TV    | 51       |   9.25 |  114262
     9253 | Steins;Gate                                               | Sci-Fi, Thriller                                             | TV    | 24       |   9.17 |  673572
     9969 | Gintama&#039;                                             | Action, Comedy, Historical, Parody, Samurai, Sci-Fi, Shounen | TV    | 51       |   9.16 |  151266
    32935 | Haikyuu!!: Karasuno Koukou VS Shiratorizawa Gakuen Koukou | Comedy, Drama, School, Shounen, Sports                       | TV    | 10       |   9.15 |   93351
    11061 | Hunter x Hunter (2011)                                    | Action, Adventure, Shounen, Super Power                      | TV    | 148      |   9.13 |  425855
      820 | Ginga Eiyuu Densetsu                                      | Drama, Military, Sci-Fi, Space                               | OVA   | 110      |   9.11 |   80679
    15335 | Gintama Movie: Kanketsu-hen - Yorozuya yo Eien Nare       | Action, Comedy, Historical, Parody, Samurai, Sci-Fi, Shounen | Movie | 1        |    9.1 |   72534
    15417 | Gintama&#039;: Enchousen                                  | Action, Comedy, Historical, Parody, Samurai, Sci-Fi, Shounen | TV    | 13       |   9.11 |   81109
(10 rows)
```

まずはジャンルを展開して、データ構造を変換する。

```sql
with tmp as (
select
    anime_id,
    type,
    episodes,
    rating,
    members,
    name,
    regexp_split_to_table(genre, ', ') as genre_split
from
    p2019w18t1
where
    type in ('Movie', 'TV') and
    rating is not null and
    genre is not null and
    members >= 10000
)
select * from tmp limit 10;

 anime_id | type  | episodes | rating | members |               name               | genre_split
----------+-------+----------+--------+---------+----------------------------------+--------------
    32281 | Movie | 1        |   9.37 |  200630 | Kimi no Na wa.                   | Drama
    32281 | Movie | 1        |   9.37 |  200630 | Kimi no Na wa.                   | Romance
    32281 | Movie | 1        |   9.37 |  200630 | Kimi no Na wa.                   | School
    32281 | Movie | 1        |   9.37 |  200630 | Kimi no Na wa.                   | Supernatural
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood | Action
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood | Adventure
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood | Drama
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood | Fantasy
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood | Magic
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood | Military
(10 rows)
```

必要な最大レーティングや平均レーティング、平均視聴者数を計算する。

```sql
with tmp as (
select
    anime_id,
    type,
    episodes,
    rating,
    members,
    name,
    regexp_split_to_table(genre, ', ') as genre_split
from
    p2019w18t1
where
    type in ('Movie', 'TV') and
    rating is not null and
    genre is not null and
    members >= 10000
), tmp2 as (
select
    anime_id,
    type,
    episodes,
    rating,
    members,
    name,
    genre_split,
    max(rating) over (partition by genre_split, type) as max_rating,
    avg(rating) over (partition by genre_split, type) as avg_rating,
    avg(members) over (partition by genre_split, type) as avg_members
from
    tmp
where
    genre_split != ''
)
select * from tmp2 limit 10;

 anime_id | type  | episodes | rating | members |                 name                  |  genre_split  | max_rating |    avg_rating     |      avg_members
----------+-------+----------+--------+---------+---------------------------------------+---------------+------------+-------------------+------------------------
    33589 | TV    | 12       |   6.96 |   12345 | ViVid Strike!                         |               |       6.96 |              6.96 | 12345.0000000000000000
      257 | TV    | 13       |   6.62 |   88969 | Ikkitousen                            |  Martial Arts |       6.62 |              6.62 |     88969.000000000000
     5675 | TV    | 26       |   7.38 |   30323 | Basquash!                             |  Mecha        |       7.38 |              7.38 |     30323.000000000000
     9624 | TV    | 12       |   6.84 |   31253 | 30-sai no Hoken Taiiku                |  Parody       |       6.84 |              6.84 |     31253.000000000000
    12467 | TV    | 13       |    7.4 |   99788 | Nazo no Kanojo X                      |  Romance      |        7.4 |             6.865 |     76553.500000000000
    16397 | TV    | 13       |   6.33 |   53319 | Photokano                             |  Romance      |        7.4 |             6.865 |     76553.500000000000
    32282 | TV    | 13       |    8.5 |  185015 | Shokugeki no Souma: Ni no Sara        |  School       |       8.61 |             8.555 |    266983.000000000000
    28171 | TV    | 24       |   8.61 |  348951 | Shokugeki no Souma                    |  School       |       8.61 |             8.555 |    266983.000000000000
    32686 | TV    | 12       |   7.22 |   83438 | Keijo!!!!!!!!                         |  Shounen      |       7.22 |              7.22 |     83438.000000000000
    12115 | Movie | 1        |   8.33 |   65594 | Berserk: Ougon Jidai-hen III - Kourin | Action        |        9.1 | 7.656145833333333 |     49859.302083333333
(10 rows)
```

あとはジャンル、タイプごとに最大のレコードを条件付ければおしまい。

```sql
with tmp as (
select
    anime_id,
    type,
    episodes,
    rating,
    members,
    name,
    regexp_split_to_table(genre, ', ') as genre_split
from
    p2019w18t1
where
    type in ('Movie', 'TV') and
    rating is not null and
    genre is not null and
    members >= 10000
), tmp2 as (
select
    anime_id,
    type,
    episodes,
    rating,
    members,
    name,
    genre_split,
    max(rating) over (partition by genre_split, type) as max_rating,
    avg(rating) over (partition by genre_split, type) as avg_rating,
    avg(members) over (partition by genre_split, type) as avg_members
from
    tmp
where
    genre_split != ''
)
select
    anime_id,
    type,
    episodes,
    rating,
    members,
    name,
    genre_split,
    max_rating,
    round(avg_rating::numeric, 2) as avg_rating,
    round(avg_members::numeric, 2) as avg_members
from
    tmp2
where
    rating = max_rating
order by
    avg_rating desc
;

 anime_id | type  | episodes | rating | members |                            name                            |  genre_split  | max_rating | avg_rating | avg_members
----------+-------+----------+--------+---------+------------------------------------------------------------+---------------+------------+------------+-------------
    28171 | TV    | 24       |   8.61 |  348951 | Shokugeki no Souma                                         |  School       |       8.61 |       8.56 |   266983.00
    30346 | Movie | 1        |   8.53 |   28864 | Doukyuusei (Movie)                                         | Shounen Ai    |       8.53 |       8.29 |    21787.67
    15335 | Movie | 1        |    9.1 |   72534 | Gintama Movie: Kanketsu-hen - Yorozuya yo Eien Nare        | Parody        |        9.1 |       8.20 |    38956.40
     6675 | Movie | 1        |   8.33 |  109392 | Redline                                                    | Cars          |       8.33 |       8.16 |    74383.00
    15335 | Movie | 1        |    9.1 |   72534 | Gintama Movie: Kanketsu-hen - Yorozuya yo Eien Nare        | Samurai       |        9.1 |       8.13 |    44060.67
    22507 | TV    | 4        |   8.29 |   19702 | Initial D Final Stage                                      | Cars          |       8.29 |       8.09 |    35334.67
    12355 | Movie | 1        |   8.84 |  226193 | Ookami Kodomo no Ame to Yuki                               | Slice of Life |       8.84 |       8.06 |    78558.50
    28957 | Movie | 1        |   8.75 |   32266 | Mushishi Zoku Shou: Suzu no Shizuku                        | Seinen        |       8.75 |       8.00 |    46774.55
     9253 | TV    | 24       |   9.17 |  673572 | Steins;Gate                                                | Thriller      |       9.17 |       7.97 |   233757.22
     7311 | Movie | 1        |   8.81 |  240297 | Suzumiya Haruhi no Shoushitsu                              | Mystery       |       8.81 |       7.95 |    49333.93
    13117 | Movie | 1        |   7.94 |   12076 | Hakuouki Movie 1: Kyoto Ranbu                              | Josei         |       7.94 |       7.94 |    12076.00
     1365 | Movie | 1        |   8.42 |   28462 | Detective Conan Movie 06: The Phantom of Baker Street      | Police        |       8.42 |       7.94 |    35682.66
     4282 | Movie | 1        |   8.68 |  111074 | Kara no Kyoukai 5: Mujun Rasen                             | Thriller      |       8.68 |       7.92 |    97376.47
    32281 | Movie | 1        |   9.37 |  200630 | Kimi no Na wa.                                             | School        |       9.37 |       7.87 |    51702.33
    15335 | Movie | 1        |    9.1 |   72534 | Gintama Movie: Kanketsu-hen - Yorozuya yo Eien Nare        | Historical    |        9.1 |       7.85 |    51022.31
    32281 | Movie | 1        |   9.37 |  200630 | Kimi no Na wa.                                             | Romance       |       9.37 |       7.84 |    69988.89
    28735 | TV    | 13       |   8.59 |   71295 | Shouwa Genroku Rakugo Shinjuu                              | Josei         |       8.59 |       7.83 |    78050.72
     9617 | Movie | 1        |   8.34 |  115252 | K-On! Movie                                                | Music         |       8.34 |       7.81 |    29130.50
    31757 | Movie | 1        |   8.73 |   34347 | Kizumonogatari II: Nekketsu-hen                            | Vampire       |       8.73 |       7.79 |    58372.50
    32281 | Movie | 1        |   9.37 |  200630 | Kimi no Na wa.                                             | Supernatural  |       9.37 |       7.78 |    70307.25
     4565 | Movie | 1        |   8.64 |   82253 | Tengen Toppa Gurren Lagann Movie: Lagann-hen               | Space         |       8.64 |       7.78 |    27666.84
     6675 | Movie | 1        |   8.33 |  109392 | Redline                                                    | Sports        |       8.33 |       7.77 |    28045.10
    11981 | Movie | 1        |    8.5 |  135735 | Mahou Shoujo Madoka★Magica Movie 3: Hangyaku no Monogatari | Psychological |        8.5 |       7.77 |    64794.47
    32281 | Movie | 1        |   9.37 |  200630 | Kimi no Na wa.                                             | Drama         |       9.37 |       7.75 |    61593.33
     4565 | Movie | 1        |   8.64 |   82253 | Tengen Toppa Gurren Lagann Movie: Lagann-hen               | Mecha         |       8.64 |       7.74 |    53278.48
    18617 | Movie | 1        |   8.55 |   25641 | Girls und Panzer der Film                                  | Military      |       8.55 |       7.73 |    46656.64
    11981 | Movie | 1        |    8.5 |  135735 | Mahou Shoujo Madoka★Magica Movie 3: Hangyaku no Monogatari | Magic         |        8.5 |       7.72 |    50740.68
       19 | TV    | 74       |   8.72 |  247562 | Monster                                                    | Psychological |       8.72 |       7.69 |   170750.98
       32 | Movie | 1        |   8.45 |  215630 | Neon Genesis Evangelion: The End of Evangelion             | Dementia      |       8.45 |       7.69 |    61067.43
    32935 | TV    | 10       |   9.15 |   93351 | Haikyuu!!: Karasuno Koukou VS Shiratorizawa Gakuen Koukou  | Sports        |       9.15 |       7.68 |    59226.04
    10408 | Movie | 1        |   8.61 |  197439 | Hotarubi no Mori e                                         | Shoujo        |       8.61 |       7.68 |    51170.58
    15335 | Movie | 1        |    9.1 |   72534 | Gintama Movie: Kanketsu-hen - Yorozuya yo Eien Nare        | Action        |        9.1 |       7.66 |    49859.30
    28977 | TV    | 51       |   9.25 |  114262 | Gintama°                                                   | Historical    |       9.25 |       7.66 |    80000.01
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood                           | Drama         |       9.26 |       7.60 |    96193.39
    12115 | Movie | 1        |   8.33 |   65594 | Berserk: Ougon Jidai-hen III - Kourin                      | Demons        |       8.33 |       7.60 |    55483.36
    15335 | Movie | 1        |    9.1 |   72534 | Gintama Movie: Kanketsu-hen - Yorozuya yo Eien Nare        | Comedy        |        9.1 |       7.60 |    44909.84
    15335 | Movie | 1        |    9.1 |   72534 | Gintama Movie: Kanketsu-hen - Yorozuya yo Eien Nare        | Shounen       |        9.1 |       7.59 |    41688.14
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood                           | Shounen       |       9.26 |       7.59 |   109864.51
    28977 | TV    | 51       |   9.25 |  114262 | Gintama°                                                   | Samurai       |       9.25 |       7.59 |    79375.76
    24701 | TV    | 10       |   8.88 |   75894 | Mushishi Zoku Shou 2nd Season                              | Mystery       |       8.88 |       7.59 |   135315.61
    28977 | TV    | 51       |   9.25 |  114262 | Gintama°                                                   | Parody        |       9.25 |       7.59 |   101138.51
     2246 | TV    | 12       |   8.49 |   88850 | Mononoke                                                   | Demons        |       8.49 |       7.58 |   118917.19
    12115 | Movie | 1        |   8.33 |   65594 | Berserk: Ougon Jidai-hen III - Kourin                      | Horror        |       8.33 |       7.57 |    59092.78
      199 | Movie | 1        |   8.93 |  466254 | Sen to Chihiro no Kamikakushi                              | Adventure     |       8.93 |       7.56 |    53873.41
    15335 | Movie | 1        |    9.1 |   72534 | Gintama Movie: Kanketsu-hen - Yorozuya yo Eien Nare        | Sci-Fi        |        9.1 |       7.56 |    53118.22
       19 | TV    | 74       |   8.72 |  247562 | Monster                                                    | Police        |       8.72 |       7.54 |   134961.77
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood                           | Military      |       9.26 |       7.53 |    91809.75
        1 | TV    | 26       |   8.82 |  486824 | Cowboy Bebop                                               | Space         |       8.82 |       7.52 |    52434.73
    12355 | Movie | 1        |   8.84 |  226193 | Ookami Kodomo no Ame to Yuki                               | Fantasy       |       8.84 |       7.52 |    54021.06
    24701 | TV    | 10       |   8.88 |   75894 | Mushishi Zoku Shou 2nd Season                              | Seinen        |       8.88 |       7.52 |   104776.19
    32983 | TV    | 13       |   8.76 |   38865 | Natsume Yuujinchou Go                                      | Shoujo        |       8.76 |       7.50 |    75275.90
     4565 | Movie | 1        |   8.64 |   82253 | Tengen Toppa Gurren Lagann Movie: Lagann-hen               | Super Power   |       8.64 |       7.50 |    45886.84
     4181 | TV    | 24       |   9.06 |  456749 | Clannad: After Story                                       | Supernatural  |       9.06 |       7.49 |   134590.53
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood                           | Adventure     |       9.26 |       7.46 |    95339.56
     4181 | TV    | 24       |   9.06 |  456749 | Clannad: After Story                                       | Slice of Life |       9.06 |       7.45 |    87180.12
    23273 | TV    | 22       |   8.92 |  416397 | Shigatsu wa Kimi no Uso                                    | Music         |       8.92 |       7.44 |    69271.32
    11061 | TV    | 148      |   9.13 |  425855 | Hunter x Hunter (2011)                                     | Super Power   |       9.13 |       7.42 |   142164.81
     4181 | TV    | 24       |   9.06 |  456749 | Clannad: After Story                                       | Romance       |       9.06 |       7.41 |   106088.03
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood                           | Action        |       9.26 |       7.40 |   117129.48
    28977 | TV    | 51       |   9.25 |  114262 | Gintama°                                                   | Comedy        |       9.25 |       7.40 |    92609.00
    14397 | TV    | 25       |   8.52 |   86074 | Chihayafuru 2                                              | Game          |       8.52 |       7.40 |   117680.13
     5675 | TV    | 26       |   7.38 |   30323 | Basquash!                                                  |  Mecha        |       7.38 |       7.38 |    30323.00
    11123 | TV    | 12       |   8.31 |   69253 | Sekaiichi Hatsukoi 2                                       | Shounen Ai    |       8.31 |       7.37 |    54066.83
    32935 | TV    | 10       |   9.15 |   93351 | Haikyuu!!: Karasuno Koukou VS Shiratorizawa Gakuen Koukou  | School        |       9.15 |       7.37 |   109402.65
    28977 | TV    | 51       |   9.25 |  114262 | Gintama°                                                   | Sci-Fi        |       9.25 |       7.36 |    87058.53
    28755 | Movie | 1        |   8.03 |   74690 | Boruto: Naruto the Movie                                   | Martial Arts  |       8.03 |       7.36 |    51605.67
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood                           | Fantasy       |       9.26 |       7.36 |   105179.32
     6594 | TV    | 12       |   8.49 |  207241 | Katanagatari                                               | Martial Arts  |       8.49 |       7.36 |    97371.19
     5114 | TV    | 64       |   9.26 |  793665 | Fullmetal Alchemist: Brotherhood                           | Magic         |       9.26 |       7.32 |    89893.34
     2904 | TV    | 25       |   8.98 |  572888 | Code Geass: Hangyaku no Lelouch R2                         | Mecha         |       8.98 |       7.29 |    70173.68
    14353 | Movie | 1        |   8.04 |   95690 | Death Billiards                                            | Game          |       8.04 |       7.28 |    32336.43
       19 | TV    | 74       |   8.72 |  247562 | Monster                                                    | Horror        |       8.72 |       7.27 |   128069.49
     6586 | TV    | 50       |   8.07 |   36921 | Yume-iro Pâtissière                                        | Kids          |       8.07 |       7.27 |    45532.86
       30 | TV    | 26       |   8.32 |  461946 | Neon Genesis Evangelion                                    | Dementia      |       8.32 |       7.26 |   131044.33
    32686 | TV    | 12       |   7.22 |   83438 | Keijo!!!!!!!!                                              |  Shounen      |       7.22 |       7.22 |    83438.00
    30279 | TV    | 12       |   8.04 |   44335 | Yuru Yuri San☆Hai!                                         | Shoujo Ai     |       8.04 |       7.20 |    55000.00
    17074 | TV    | 26       |    8.8 |  205959 | Monogatari Series: Second Season                           | Vampire       |        8.8 |       7.18 |   141792.92
     9790 | Movie | 1        |   7.87 |   70391 | Sora no Otoshimono: Tokeijikake no Angeloid                | Harem         |       7.87 |       7.17 |    42259.80
     2397 | Movie | 1        |   7.81 |   37078 | Digimon Adventure: Bokura no War Game!                     | Kids          |       7.81 |       7.15 |    35733.25
      853 | TV    | 26       |   8.39 |  422271 | Ouran Koukou Host Club                                     | Harem         |       8.39 |       7.10 |   107866.78
    12467 | TV    | 13       |    7.4 |   99788 | Nazo no Kanojo X                                           |  Romance      |        7.4 |       6.87 |    76553.50
     9624 | TV    | 12       |   6.84 |   31253 | 30-sai no Hoken Taiiku                                     |  Parody       |       6.84 |       6.84 |    31253.00
      257 | TV    | 13       |   6.62 |   88969 | Ikkitousen                                                 |  Martial Arts |       6.62 |       6.62 |    88969.00
(83 rows)
```

## :closed_book: Reference

None
