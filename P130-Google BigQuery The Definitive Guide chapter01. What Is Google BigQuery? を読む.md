## :memo: Overview

下記の書籍の内容の理解を深めるために各章に分けてメモを残しておく。BigQueryでSQLは記述できても、BigQueryそのものにはあまり詳しくないので、メモをまとめておく。600 ページほどあるが、非常に実践的な内容が多く、ありがたい一冊。ただ、日本語翻訳版の書籍がなく、見返すのに時間がかかるので、メモを残している。誤訳している内容をメモしている可能性もある。かつ、書籍の内容をそのままメモしているわけでもない。

- [Google BigQuery: The Definitive Guide](https://www.oreilly.com/library/view/google-bigquery-the/9781492044451/)

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`BigQuery`

## :pencil2: BigQuery: A Serverless, Distributed SQL Engine

サンプルで登場する`bigquery-public-data.new_york_citibike.citibike_trips`データは、行数が58,937,715で、合計論理バイト数は7.47 GB。BigQueryでは、ご法度ではあるが、`select *`すると7.47 GBスキャンされる。

ただ、プレビューでデータの中身を見ればわかるが、`NULL`のみのレコードが非常に多い。下記のSQLで件数を調べると、58,937,715 - 53,108,721 = 5,828,994は、`tripduration`が`NULL`の行である。

```sql
SELECT
  count(1)
FROM
  `bigquery-public-data.new_york_citibike.citibike_trips`
WHERE
  tripduration IS NOT NULL
;
```

ここで紹介されている内容は後の章のイントロ要素が多いので、各章で必要な部分はメモすることにする。なので、この章ではあまりまとめることがない。

## :closed_book: Reference

- [Google BigQuery: The Definitive Guide](https://www.oreilly.com/library/view/google-bigquery-the/9781492044451/)
- [bigquery-oreilly-book](https://github.com/GoogleCloudPlatform/bigquery-oreilly-book)