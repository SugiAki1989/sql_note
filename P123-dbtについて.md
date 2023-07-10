## :memo: Overview

ここではデータ転送を行う際のデータ変換の部分を担当する dbt の基本的な部分をまとめておく。ドキュメントを見つつ、dbt を触りつつ、気になった内容をまとめておく。

- [dbt](https://www.getdbt.com/)

基本的には`dbt init`すると生成されるフォルダごとにわけて記載しているが、出来ることがあまりにも多いので、詳細な情報までは記載しておらず、各機能の役割を簡単にまとめているだけ。

dbt は ETL のデータ変換の部分のみを担当するものだと勘違いしていたが、dbt はデータ変換にとどまらず、ドキュメントの生成もでき、データ変換にまつわる諸問題も対処してくれる。さらに、マクロやパッケージを利用することでデータ変換のパイプラインを堅牢にできる。データ変換にまつわる様々な便利な機能を提供してくれるが dbt ということがわかった。

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

## :pencil2: seeds & snapshots

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

## :pencil2: Macros

Macros には Python でコードが記述できる。例えば、`my_macro`としてマクロを定義すれば、

```
{% macro my_macro(model) %}
    SELECT * FROM {{ model }} WHERE
    {% for col in adapter.get_columns_in_relation(model) -%}
        {{ col.column }} IS NULL OR
    {% endfor %}
    FALSE
{% endmacro %}
```

`tests`フォルダの`.sql`でテストを記述する際にマクロを呼び出すことができる。

```
{{ my_macro(ref('dim_foge')) }}
```

また、`macros/check.sql`として保存しているマクロは`models/schema.yml`の中でも呼び出すことができる。

```
models:
  - name: dim_hoge
      (snip)
      - name: macro_name
        tests:
          - check
```

## :pencil2: Packages

サードパーティのパッケージを取り込むことができる。下記のサイトから様々なパッケージを検索して利用できる。

- [Jumpstart your warehouse](https://hub.getdbt.com/)

利用方法はプロジェクトのディレクトリに`packages.yml`を作成し、その中に下記のように記述することで利用できる。

```
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

`dbt deps`コマンドでインストールできる。

```
$ dbt deps
```

例えば、`dbt_utils.generate_surrogate_key`マクロは、下記のように select 句の中で呼び出すことで利用できる。このマクロは、指定されたカラムを使用してハッシュ化されたサロゲートキーを生成する。

```
SELECT
  {{ dbt_utils.generate_surrogate_key(['a', 'b', 'c', 's']) }} AS foge_id,
  *
FROM
  piyo
```

## :pencil2: Documentation

dbt のドキュメンテーション機能は非常に便利。`schema.yml`や`sources.yml`の情報をもとに HTML ドキュメントを生成してくれる。`schema.yml`には`description`を追加でき、テーブルやカラムの説明を追記できる。

```
models:
  - name: table_name
    description: description of table
    columns:
      - name: a
        tests:
          - unique
          - not_null

      - name: b
        description: description of b
        tests:
          - not_null
          - relationships:
              to: ref('hoge')
              field: hoge_id
```

`dbt docs generate`コマンドでドキュメンテーションを生成できる。`pj/target/catalog.json`に HTML の元となる情報が出力され、`pj/target/index.html`を開けばドキュメントが確認できる。

```
$ dbt docs generate
```

また、画像をドキュメントを呼び出して、組み込みこともできる。`schema.yml`には下記のように記述する。

```
- name: fuga
  description: '{{ doc("hoge__fuga") }}'
  tests:
    - check
```

`models/docs.md`とすることで、ドキュメントを参照先から呼び出すことができる。

```
{% docs hoge__fuga %}
hoge hoge.
fuga fuga.

piyo piyo.

![image](path_to_image)
{% enddocs %}
```

画像などは`assets`として管理できる。`dbt_project.yml`に下記のパスを追加し、ディレクトリに画像を保存する。

```
model-paths: ['models']
analysis-paths: ['analyses']
test-paths: ['tests']
seed-paths: ['seeds']
macro-paths: ['macros']
snapshot-paths: ['snapshots']
asset-paths: ['assets'] -- 追加したもの
```

呼び出す際は`assets/image.png`と記述すれば OK。

```
{% docs hoge__fuga %}
hoge hoge.
fuga fuga.

piyo piyo.

![image](assets/image.png)
{% enddocs %}
```

また、HTML ドキュメントを作成すれば自動で Data Lineage Graph も生成される。データリネージグラフはテーブルやビューの依存関係がわかるので非常に便利。

## :pencil2: Analyses

analyses ディレクトリには一時的に利用する分析 SQL を保存できる。ここに保存されているものは実体化されるわけでないらしい。この段階では参照するテーブルは`ref()`で指定しておき、`analyses/hoge.sql`として保存し`dbt compile`コマンドでコンパイルすれば、`target/compiled/pj_name/analyses/hoge.sql`には、`ref()`の部分がテーブル名に置き換えられたものが作成される。この`hoge.sql`をデータウェアハウスに流せばアドホックな分析が実行できる。

## :pencil2: Hooks

Hooks 事前に定義されたタイミングで SQL を実行できる機能で、プロジェクト、サブフォルダ、モデルレベルで定義できる。`dbt_project.yml`に下記のに記述することで、ユーザーに権限を付与できる。タイミングも`on_run_start`、`on_run_end`、`pre-hook`、`post-hook`の 4 種類ある。

```
# Configuring models
models:
  pj_name:
    +materialized: view
    +post-hook:
      - 'GRANT SELECT ON {{ this }} TO ROLE user_name'
    dim:
      +materialized: table
    src:
      +materialized: ephemeral
```

## :closed_book: Reference

- [dbt](https://www.getdbt.com/)
- [Jumpstart your warehouse](https://hub.getdbt.com/)
