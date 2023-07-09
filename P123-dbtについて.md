## :memo: Overview

ここではデータ転送を行う際のデータ変換の部分を担当する dbt の基本的な部分をまとめておく。ドキュメントを見つつ、dbt を触りつつ、気になった内容をまとめておく。

- [dbt](https://www.getdbt.com/)

`dbt init`すると生成されるフォルダごとにわけて記載している。

## :floppy_disk: Database

Snowflake

## :bookmark: Tag

`dbt`

## :pencil2: models

このディレクトリにデータを変換するビジネスロジックを記述していく。モデルは`.sql`ファイル群で定義される。

例えば下記の画像のような形のビジネスロジックがあったとする。ローデータテーブル(`raw`)からソースビュー(`source`)を作り、ディメンションビュー(`dimention`)やファクトビュー(`fact`)を作成。そして、ディメンションビュー(`dimention`)を組み合わせてテーブル化し、追加のテーブルとファクトテーブルを組み合わせてマートを作成。そして、レポートが作られる、みたいな構造。

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p123-dbt-models.png)

これらを`with`句を分割して、役割ごとに`.sql`ファイルを保存する。つまり、1 つの SQL にしようと思えばできなくもないが、分けることで役割を明確にでき、疎結合にもつながるため、管理しやすくなる。ファイルの名前がモデル名として使用される。

画像はあくまでも一例で、より複雑なもの実現できるし、ビューやマテリアルビュー、テーブル化するなど柔軟に設定できる。下記のようにディレクトリにわけて`.sql`ファイルを作成しておくことで、データウェアハウス側でも`dbt run`コマンド実行時にビューやテーブルが分けて作られる。

```
$ tree ./models
./models
├── dim
│   ├── dim_hoge.sql
│   ├── dim_fuga.sql
│   └── dim_hoge_fuga.sql
├── fct
│   └── fct_piyo.sql
├── mart
│   └── mart_hoge.sql
├── schema.yml
├── sources.yml
└── src
    ├── src_hoge.sql
    ├── src_fuga.sql
    └── src_piyo.sql
```

dbt ではテーブルの選択を`from {{ ref('hoge') }}`という形で指定できる。`models/sources.yml`に設定を入力することで、テーブルを抽象化して構造化をはかれる。

## :pencil2: dbt_project.yml

`dbt_project`ではさきほどの`models/*.sql`に対する設定を記述できる。

```
# Configuring models
models:
  pj_name:
    +materialized: view
    dim:
      +materialized: table # `dim/`以下の全てのモデルに適用されます
    src:
      +materialized: ephemeral # `src/`以下の全てのモデルに適用されます
```

サブディレクトリに設定された内容が優先される。また、下記のように`.sql`の先頭で直接設定することも可能。

```
{{
  config(materialized = 'view')
}}
```

## :pencil2: seeds 　& snapshots

seeds には追加のデータを保存することで、データウェアハウスにインサートして使用できる。`seeds/seed_hoge.csv`があるときに,`dbt seed`コマンドを実行すると、データウェアハウスに`seed_hoge`というテーブルが作成される。

snapshots には`.sql`ファイルを保存することで、SCD に対応するためにスナップショットを作ることができる。`dbt snapshot`コマンドを実行すると、`scd_raw_hoge`テーブルにスナップショットが作成される。

```
{% snapshot scd_raw_hoge %}

{{
   config(
       target_schema='dev',
       unique_key='id',
       strategy='timestamp',
       updated_at='updated_at',
       invalidate_hard_deletes=True
   )
}}

select * FROM {{ source('name', 'hoge') }}

{% endsnapshot %}
```

## :pencil2: tests

tests にはテスト内容を記述した`.sql`を保存する。`dbt test`コマンドを実行することで、テストが行われる。テーブルを紐付けることで不整合が発生していないかどうかや、そもそも取りうるはずがない値が記録されていないかどうかなど、`.sql`にテストしたい SQL を書くことで柔軟にテストができる。

また、`models/schema.yml`にもテストを書くことができる。NotNull、ユニーク、リレーションに関するテストやカーディナリティに関するテストが記述できる。

```
version: 2

models:
  - name: dim_hoge
    columns:
      - name: hoge_id
        tests:
          - unique
          - not_null

      - name: fuga_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_fuga')
              field: fuga_id

      - name: piyo
        tests:
          - accepted_values:
              values: ['x', 'y', 'z']
```

## :pencil2: CustomTests

## :pencil2: Packages

## :pencil2: Macros

## :pencil2: Analyses

## :closed_book: Reference

- [dbt](https://www.getdbt.com/)
