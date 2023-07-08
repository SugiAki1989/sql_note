## :memo: Overview

ここではデータ転送を行う際のデータ変換に関して、dbt で実行する基本的な方法をまとめようとしたのだが、先に Slowly Changing Dimensions という考えたをおさらいしておきたかったので、Slowly Changing Dimensions についてまとめておく。下記の記事を参考にさせて頂いた。

- [Slowly Changing Dimensions](https://towardsdatascience.com/data-analysts-primer-to-slowly-changing-dimensions-d087c8327e08)

## :floppy_disk: Database

None

## :bookmark: Tag

`Slowly Changing Dimensions`, `Dimensional Modeling`, `Fact Table`

## :pencil2: Slowly Changing Dimensions(SCD)

Slowly Changing Dimensions は、データをどのように DWH に記録するかを考える際に役立つもので、ディメンションに最新のデータと過去のデータの両方が含まれる DWH の概念を指す。どれかが必ずしも正しい、というものではなく、正しいかどうかは時と場合による。記録のされ方によって分析 SQL が簡単に記述できたり、ひと工夫が必要になる場合がある。

例えば、会員ステータスの変化を調べたい場合、どのようなデータが記録されていれば分析することができるだろうか。このような問題を考える際に SCD は役立つ。またファクトテーブルにもいくつかタイプがあるのでそちらもあわせてまとめておく。

### SCD Type 0: 元の値をそのまま利用する方法

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-d0.png)

このパターンは値が何も変わらないので、前後で変化することはない。

### SCD Type 1: 値を上書きして利用する方法

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-d1.png)

このパターンは値が更新されると過去の値を上書きすることで、値を更新する。そのため、アプリーケーションでは常にユーザーの最新の値が参照される必要があるので、このような更新が行われる。ただ、分析の場合は前の値が要件によっては必要になるため、上書きされてしまうとどうしようもなくなる。

### SCD Type 2: 新しいレコードを追加して利用する方法

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-d2.png)

このパターンは、更新があると新しいレコードを追加することで、変化に対応する。この例であれば、最新のレコードを識別するカラムがあるため、この値を利用しながら必要な時点のレコードを紐付ける。

### SCD Type 3: 新しい属性を追加して利用する方法

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-d3.png)

このパターンは、更新があると予め用意していおいたカラムに前の値を保存することで、値を更新する。時点が 2 つであれば、2 カラムで足りるが、時点が増えると予め用意するカラムの数も増えるのと、予め更新数が固定されているようなケースであれば使われたりする。最新時点とその前しか常に持たないなど。

### SCD Type 4: 履歴テーブルを使用する方法

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-d4.png)

このパターンは、更新がある部分に関してディメンションテーブルを分割する形でデータの変化をトラックする。紐付ける際は、必要なパーテーションのデータを指定して紐付ける。

### SCD Other: スナップショットを利用する方法

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-d5.png)

Type 4 と類似するが、このパターンは毎日、毎週などの期間でまるごとスナップショットを作成して、値の変化をトラックする。Type 4 との違いは、更新がないものもすべて保存する。

### Fact Table Type 1: トランザクション系ファクトテーブル

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-f1.png)

このパターンは、POS や EC 、Web のアクセスログなどのトランザクションデータでよく見かける最も一般的なトランザクションテーブル。トランザクションごとにレコードが生成されていく。ビジネスにあわせて急速にレコードが増加するが、値の変更はない。

### Fact Table Type 2: 定期系ファクトテーブル

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-f2.png)

このパターンは、定期的に値が変化する過程を含めたトランザクションデータ。トランザクション系ファクトテーブルよりもレコードが増えるの速くない。銀行預金とか、何とかスコアの推移とか、SaaS サブスクの支払い系とかでも見るかも。

### Fact Table Type 3: 蓄積系ファクトテーブル

![](https://github.com/SugiAki1989/sql_note/blob/main/image/p122-f3.png)

このパターンは、時点によって値が更新されていく。EC 通販を考えるとわかりやすい。注文時点では、発送、受取時点は確定していない。時間が進むごとに、発送時点が確定し、さらに時間が進むごとに受取時点が確定する。確定するごとに値が更新されていく。画像は更新間隔が粗いがイメージはあんな感じ。

## :closed_book: Reference

- [Slowly Changing Dimensions](https://towardsdatascience.com/data-analysts-primer-to-slowly-changing-dimensions-d087c8327e08)
