## :memo: Overview

ここでは BigQuery のマテリアライズドビュー(materialized view)についてまとめておく。マテリアライズドビュー自体は別に新しい概念でもなく、昔からあるデータベースの機能。ただ、BigQueryはほとんど仕事では縁がなく、`CREATE VIEW`コマンドを利用しなくても、UIからビューが作れる、マテリアライズドビューでも色々と制限があるなど、少し戸惑う部分があったのでまとめておく。公式ドキュメントは下記の通り。

- [マテリアライズド（実体化）ビューの概要](https://cloud.google.com/bigquery/docs/materialized-views-intro?hl=ja)
- [マテリアライズド ビューの作成](https://cloud.google.com/bigquery/docs/materialized-views-create?hl=ja)
- [マテリアライズド ビューを使用する](https://cloud.google.com/bigquery/docs/materialized-views-use?hl=ja)
- [マテリアライズド ビューの管理](https://cloud.google.com/bigquery/docs/materialized-views-manage?hl=ja)

上記のドキュメントを読みながら気になったところをまとめいく。

## :floppy_disk: Database

BigQuery

## :bookmark: Tag

`bq`

## :pencil2: Materialized View

> BigQuery のマテリアライズド ビューは事前に計算されたビューで、パフォーマンスと効率を向上させるためにクエリの結果を定期的にキャッシュに保存します。

BigQueryの場合、マテリアライズドビューはリフレッシュしなくても、定期的に更新される仕様で、ベーステーブルから差分を読み取って、最新の結果を計算するとのこと。ベーステーブルが変更されたときに、リフレッシュマテリアライズドビュー(`REFRESH MATERIALIZED VIEW`)をしなくて良いのは、更新のメンテンスが不要になるので楽。マテリアライズドビューのユースケースとして下記が紹介されている。

> マテリアライズド ビューでは、コンピューティング コストが高く、データセットの結果が小さいクエリを最適化できます。マテリアライズド ビューが有効なプロセスには、ETL（抽出、変換、ロード）処理やビジネス インテリジェンス（BI）パイプラインのように、予測可能なクエリを繰り返して大量の処理を必要とするオンライン分析処理（OLAP）オペレーションなどがあります。

通常のマテリアライズドビューの使い方ではあるが、このユースケースを読む限り、大きなデータを集計したデータマートを作るときは便利そう。ビューだとベーステーブルが変更されるたびに、毎回計算に時間がかかってしまうので、マテリアライズドビューにしておけば、レスポンスが高速化できる。料金に関しては、マテリアライズドビューに対するクエリ(マテリアライズドビューとベーステーブルの必要な部分によって処理されたバイト数)、マテリアライズドビューのメンテナンス(マテリアライズドビュー テーブルの保存)、マテリアライズドビューテーブルの保存(マテリアライズドビューに保存されているバイト数)に関して費用が発生する。

さらっと読む限りは自動リフレッシュしてくれて、便利そうな気もするが、色々と制約があるらしい。ドキュメントに記載されている制限事項を読んでおく。

> マテリアライズド ビューのデータを、COPY、EXPORT、LOAD、WRITE、データ操作言語（DML）ステートメントなどのオペレーションを使用して直接更新することや操作することはできません。

作成済みのマテリアライズドビューに何らかの変更を加えることはできないとのこと。ただ、マテリアライズドビューを変更することはないので、とくに問題ではなさそう。

> 既存のマテリアライズド ビューを、同じ名前のマテリアライズド ビューで置き換えることはできません。

これも大丈夫そう。ビューの命名規則を組織で持っていれば、同じ名前をつける必要はなさそう。

> マテリアライズド ビューの作成後にビュー SQL の更新はできません。(The view SQL cannot be updated after the materialized view is created.)

日本語翻訳の内容がよくわからなかったので、英語版のドキュメントを読んでみたが、よくわからなかった。

> マテリアライズド ビューは、ベーステーブルと同じ組織に存在する必要があります。プロジェクトが組織に属していない場合は、同じプロジェクト内に存在する必要があります。

これも大丈夫そう。仕事で使うことになるのであれば、特に問題はないはず。ただ、何らかの要件では、そういうことが起こりうるのだろうか。

> 各ベーステーブルを参照できるマテリアライズド ビューの数には上限があります。詳細については、マテリアライズド ビューの上限をご覧ください。

ベーステーブルへの参照数に上限があるようで、同じデータセットからは20個、同じプロジェクトからは100個、組織全体からは500個まで、マテリアライズドビューから参照できるとのこと。

> スマートな調整では、同じデータセットからのマテリアライズド ビューのみが考慮されます。

「スマートな調整」という機能があるらしく、特定の条件に合致する場合はクエリが書き換えられるとのこと。[スマートな調整の例](https://cloud.google.com/bigquery/docs/materialized-views-use?hl=ja#smart_tuning_examples)を参照のこと。

> パーティション分割テーブルの実体化されたビューは、パーティション分割できます。マテリアライズド ビューのパーティション分割は、クエリがパーティションのサブセットにアクセスすることが多い場合にメリットが得られる点で、通常のテーブルのパーティショニングに似ています。

元がパーティションテーブルであればマテリアライズドビューでもパーティションできる。公式ドキュメントには下記の例が記載されている。ベーステーブルは日別パーティションを使用して `transaction_time` 列でパーティション分割し、マテリアライズドビューは同じ列でパーティション分割し、さらに`employee_id` 列でクラスタ化している。

```sql
CREATE TABLE my_project.my_dataset.my_base_table(
  employee_id INT64,
  transaction_time TIMESTAMP)
  PARTITION BY DATE(transaction_time)
  OPTIONS (partition_expiration_days = 2);

CREATE MATERIALIZED VIEW my_project.my_dataset.my_mv_table
  PARTITION BY DATE(transaction_time)
  CLUSTER BY employee_id
AS (
  SELECT
    employee_id,
    transaction_time,
    COUNT(employee_id) AS cnt
  FROM
    my_dataset.my_base_table
  GROUP BY
    employee_id, transaction_time
);
```

> パーティションの有効期限をマテリアライズド ビューに設定することはできません。マテリアライズド ビューは、ベーステーブルからパーティションの有効期限を暗黙的に継承します。

ただ、パーティションは設定できるが有効期限はベーステーブルの期限を継承するとのこと。ベーステーブルがなくなったら作りようがないので、当たり前といえば当たり前かも。

> マテリアライズド ビューを他のマテリアライズド ビューにネストすることはできません。

これは気をつけましょう、という感じですね。クエリを設計するときに、こうならないように意識すれば問題なさそう。

> マテリアライズド ビューでは、外部やワイルドカードのテーブル、ビュー、スナップショットに対してクエリを実行できません。

外部テーブルやワイルドカードでテーブル指定は普通にしてしまいそうなので、普通に気が付かず沼りそう。

> マテリアライズド ビューでは、GoogleSQL 言語のみサポートされています。
GoogleSQL 言語というのは、おそらく[こちら](https://cloud.google.com/spanner/docs/reference/standard-sql/overview)で解説されている話のことだと思われる。

> マテリアライズド ビューの説明を設定できますが、マテリアライズド ビューの個々の列の説明は設定できません。

カラム個々には説明がつけれないとのこと。このあたりはbdtのデータカタログ機能を組み合わせると良いのかも。

> 先にマテリアライズド ビューを削除せずにベーステーブルを削除した場合、マテリアライズド ビューのクエリと更新は失敗します。ベーステーブルを再作成する場合は、マテリアライズド ビューも再作成する必要があります。

データ分析基盤のデータベースとかがそうかもしれないが、長年使われているデータベースとかになると、分析SQLとかなんだが管理されてないクエリとかテーブルとかあるので、こういうことは普通に起こりそう。

> マテリアライズド ビューは、制限付き SQL 構文と一部の集計関数を使用します。詳細については、[サポートされているマテリアライズド ビュー](https://cloud.google.com/bigquery/docs/materialized-views?hl=ja#supported-mvs)をご覧ください。


## :pencil2: Materialized Viewの制約

これが結構やっかいかもしれない。マテリアライズドビューは、制限付きSQL構文を使用する必要がある。制限付きSQL構文は下記の通り。

```sql
[ WITH cte [, …]]
SELECT  [{ ALL | DISTINCT }]
  expression [ [ AS ] alias ] [, ...]
FROM from_item [, ...]
[ WHERE bool_expression ]
[ GROUP BY expression [, ...] ]

from_item:
    {
      table_name [ as_alias ]
      | { join_operation | ( join_operation ) }
      | field_path
      | unnest_operator
      | cte_name [ as_alias ]
    }

as_alias:
    [ AS ] alias
```

まずは集計要件について。実体化されたクエリ結果は出力である必要がある。つまり、集計結果を利用した計算、フィルタはサポートされていない。下記、公式ドキュメントの例の通りで、`COUNT(*) / 10`は出力ではなく、出力を利用した計算になっているため、これはサポートされていない。

```sql
SELECT TIMESTAMP_TRUNC(ts, HOUR) AS ts_hour, COUNT(*) / 10 AS cnt
FROM mydataset.mytable
GROUP BY ts_hour;
```

分析で使いそうなものに絞っているので、サポートされている集計関数の詳細は公式ドキュメントを参照。サポートされているのは、`APPROX_COUNT_DISTINCT`、`ARRAY_AGG`、`AVG`、`COUNT`、`COUNTIF`、`HLL_COUNT.INIT`、`MAX`、`MIN`、`SUM`、`LOGICAL_OR`、`LOGICAL_AND`など。他にも、WITH 句と共通テーブル式をサポートされている。

サポートされていない SQL 機能がいくつか存在しているので、何も意識せず使ってると沼りそうかも。`Left/right/full outer joins`、`Self-joins (joins using the same table more than once).`、`Window functions`、`ARRAY subqueries`、`Non-deterministic functions such as RAND(), CURRENT_DATE(), SESSION_USER(), or CURRENT_TIME()`、`User-defined functions (UDFs)`、`TABLESAMPLE`、`FOR SYSTEM_TIME AS OF`がドキュメントには記載されている。つまり、`join`系、ウインドウ関数、実行するまで値が確定しない関数やサンプリング、UDFなどがサポートされていない。

制約に関しては、2023年4月頃にアップデートがあったみたいで、`OUTER JOIN`、`UNION`や分析関数などがマテリアライズドビューで利用可能になった。ただ、マテリアライズドビュー作成時に `allow_non_incremental_definition`や`max_staleness`オプションを有効にして、「非増分マテリアライズドビュー」を作成する必要があるとのこと。非増分マテリアライズドビューの名の通り、増分更新がサポートされないものの、`max_staleness`オプションを使用すると、自動的にメンテナンスされ、未更新保証が組み込まれた任意の複雑なマテリアライズドビューを構築できる。アップデートを読むだけではよくわからんので、公式ドキュメントを読んでおく。

- [非増分マテリアライズドビュー](https://cloud.google.com/bigquery/docs/materialized-views-create?hl=ja#non-incremental)

> 非増分マテリアライズド ビューは、`allow_non_incremental_definition` オプションを使用して作成できます。このオプションには `max_staleness`オプションを指定する必要があります。マテリアライズド ビューを定期的に更新するには、更新ポリシーも構成する必要があります。更新ポリシーを使用しない場合は、マテリアライズド ビューを手動で更新する必要があります。

非増分マテリアライズドビューは、`allow_non_incremental_definition`オプションを使用して作成できるマテリアライズドビューのことで、あわせて`max_staleness`オプションを指定する必要がある。

> そして、マテリアライズド ビューは、常に `max_staleness` 間隔内のベーステーブルの状態を表します。最後の更新が古すぎて `max_staleness` 間隔内のベーステーブルを表していない場合、クエリはベーステーブルを読み取ります。

どうやら、`max_staleness`というのがベーステーブルの対象範囲を決める模様で、マテリアライズドビューを定期的に更新するには、更新ポリシーも構成する必要があり、更新ポリシーを使用しない場合は、マテリアライズドビューを手動で更新する必要がある。更新ポリシーは下記の通り設定できる。

```sql
CREATE MATERIALIZED VIEW dataset.mv
OPTIONS (
  allow_non_incremental_definition = true
  max_staleness = INTERVAL "4" HOUR,
  enable_refresh = true, 
  refresh_interval_minutes = 60,
  )
AS SELECT
  s_store_sk,
  SUM(ss_net_paid) AS sum_sales,
  APPROX_QUANTILES(ss_net_paid, 2)[safe_offset(1)] median
FROM dataset.store
LEFT OUTER JOIN dataset.store_sales
  ON ss_store_sk = s_store_sk
GROUP BY s_store_sk
HAVING median < 40 OR median is NULL
;
```

いまいち`max_staleness`オプションがよくわからないが、ドキュメントに説明が載っているので見ておく。そもそも`staleness`は、老化によって純粋さと新鮮さを失ってしまうことらしい。

> `max_staleness` マテリアライズド ビュー オプションにより、大規模で頻繁に変化するデータセットを処理するときに、制御されたコストで一貫して高いパフォーマンスを実現できます。`max_staleness` パラメータを使用すると、結果の鮮度を調整してクエリのパフォーマンスを調整できます。この動作は、データの鮮度が必須ではないダッシュボードやレポートに役立ちます。

このオプションのユースケースとして、データの鮮度が必須ではないダッシュボードやレポートに役立つとあるので、毎日は見る必要はないが月1くらいでベーステーブルを更新して、各種アドホックな複雑な分析に利用するビューとして使えそうではある。

> `max_staleness` を使用してマテリアライズド ビューにクエリを実行すると、BigQuery は、`max_staleness` 間隔内で実行されたマテリアライズド ビュークエリの結果と一致するデータを返します。

> 最後の更新が `max_staleness` 間隔内にある場合、BigQuery は、ベーステーブルを読み取ることなく、実体化されたビューから直接データを返します。

> 最後の更新が `max_staleness` 間隔の範囲外の場合、クエリはベーステーブルからデータを読み取り、未更新間隔内の結果を返します。

`max_staleness = INTERVAL "4" HOUR`と設定すると、最後の更新が4時間以内の間隔内にある場合、ベーステーブルは読まず実体化されたものを参照するということだと思われる。

[未更新と更新頻度の管理](https://cloud.google.com/bigquery/docs/materialized-views-create?hl=ja#manage_staleness_and_refresh_frequency)に記載されているが、ベーステーブルのビュー化に仮に1時間かかるのであれば、 1 時間のバッファが必要になる。

```sql
CREATE MATERIALIZED VIEW project-id.my_dataset.my_mv_table
OPTIONS (
  enable_refresh = true, 
  refresh_interval_minutes = 120, 
  max_staleness = INTERVAL "4:0:0" HOUR TO SECOND)
AS SELECT
  employee_id,
  DATE(transaction_time),
  COUNT(1) AS cnt
FROM my_dataset.my_base_table
GROUP BY 1, 2;
```

他にも`max_staleness`を短くすればするほど、新しい状態のベーステーブルから結果を返すことができそうではあるが、これをするとコストが掛かってしまうが、データの鮮度やコストとトレードオフという感じ。

## :closed_book: Reference

- [マテリアライズド（実体化）ビューの概要](https://cloud.google.com/bigquery/docs/materialized-views-intro?hl=ja)
- [マテリアライズド ビューの作成](https://cloud.google.com/bigquery/docs/materialized-views-create?hl=ja)
- [マテリアライズド ビューを使用する](https://cloud.google.com/bigquery/docs/materialized-views-use?hl=ja)
- [マテリアライズド ビューの管理](https://cloud.google.com/bigquery/docs/materialized-views-manage?hl=ja)
