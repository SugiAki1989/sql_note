## :memo: Overview

ここでは Fivetran でデータ転送を行うための基本的な方法をまとめておく。構成としては、Fivetran を実行し、GoogleSpreadSheet のシートデータを AWS 環境下の MySQL にロードすることが目的。前回、Embulk で実行していた部分を Fivetran に置き換えたもの。

- [Fivetran](https://www.fivetran.com/)

ちなみに書いている本人は、データエンジニアでなければエンジニアでもなく、Embulk のことも初めて学んだので、内容が怪しい場合があるので注意。わからないなりにドキュメントを読んでまとめたもの。

## :floppy_disk: Database

MySQL

## :bookmark: Tag

`Fivetran`

## :pencil2: GoogleSpreadSheet to MySQL via Fivetran

まずは Fivetran のアカウント発行から始める。今回は無料プランを利用するので、Fivetran の料金自体はかからない。料金形態については下記の公式資料に記載されている。新規登録したら、認証のメールが返ってくるので、クリックしたら登録完了。

- [Fivetran 料金プラン完全ガイド](https://resources.fivetran.com/datasheets/pricing-guide-jp)

画面に表示されるスタートガイドに従って、連携設定を行う。

### :pencil2: Destinations

まずは Destination の設定から行う。画面左のメニュー「Destinations」から「Add Destination」を選択し、「MySQL RDS」を選択する。 Fivetran は転送先にデータウェアハウスを指定することを推奨しているので MySQL を選択すると、警告が表示される。

```
MySQL database can fail to perform basic queries for even medium volumes of data and is not appropriate as a data warehouse.
We support MySQL as a test environment. If you run into these limitations, you will need to migrate to a supported data warehouse.
```

設定画面に従って設定を行う。公式サイトの[ドキュメント](https://fivetran.com/docs/databases/mysql/rds-setup-guide)にも同じことが書いてある。まずは、設定画面の右側に出てくる Setup Guide に従って、AWS の MySQL の設定を変更する。下記の前提があるので、予め MySQL の設定を必要であれば変更しておく。

> Prerequisites
> To connect MySQL RDS to Fivetran, you need the following:
>
> - MySQL version 5.5 or above (5.5.40 is the earliest version tested)
> - Database host's IP (e.g., 1.2.3.4) or host (your.server.com)
> - Port (usually 3306)
> - Database administrator permissions to create a Fivetran-specific MySQL user
> - Fivetran role with the Create Destinations or Manage Destinations permissions
> - Provide at least 1024MB for innodb_buffer_pool_size. For more information about innodb_buffer_pool_size, see MySQL's Buffer Pool documentation.
> - Set the local_infile system variable to ON. For more infomation about local_infile, see Server System Variables > documentation. Check the variable status with SHOW GLOBAL VARIABLES LIKE 'local_infile' and switch the status to ON with SET GLOBAL local_infile = true.

今回は`db.t3.micro`を使用しており、パラメタグループから`innodb_buffer_pool_size`を変更しても Fivetran のエラーが解消できなかった。おそらく、`db.t3.micro`では`innodb_buffer_pool_size`の上限があるようで、設定を変えても反映されない模様。そのため、一時的にインスタンスタイプを`db.t3.small`以上にアップしている。下記のエラーが表示されていた。

> Connection tests failed.
> Warehouse User: The innodb_buffer_pool_size for your database is: 384 MB, which is very low. Please make sure its at least 1024 MB.

設定ガイドに従うと、まずはデータベースに直接アクセスするか、SSH トンネルを利用するのか、AWS PrivateLink 経由なのかを決める必要がある。接続方法によって設定が異なってくる。ここではプライベートサブネットに MySQL を配置しているので、EC2 を踏み台サーバーにして、SSH トンネルで接続する。

この方法で接続を行うために、EC2 サーバーの IP アドレスを RDS のセキュリティグループに設定する必要がある。ただ、これまでの構成として EC2 の IP アドレスは Elastic IP アドレスを設定していない。そのため、起動のたびに可変するので IP アドレスを固定できない。ただ、EC2 に設定している特定のセキュリティグループ内のサーバーからであれば MySQL にアクセスできるようにしているため、この設定はすでに設定済みとなる。

直接接続する場合は、Fivetran の[Fivetran IP Addresses](https://fivetran.com/docs/getting-started/ips)を参考に必要な IP アドレスをセキュリティグループのインバウンドに設定する。AWS の ap-northeast-1 であれば`43.207.101.168/29`をインバウンドとして許可する。今回は、ネットワーク ACL の構成はないので、スキップする。

次は踏み台となる EC2 サーバーでユーザーを追加する。`fivetran`というグループを作成し、`fivetran`というユーザーを`fivetran`グループに追加する。ユーザーを切り替えて、ssh キーを保存するディレクトリを作成し、権限を付与する。`authorized_keys`には、Fivetran の設定画面からコピーできるので、それをコピペしておく。

```
[ec2-user]$ sudo groupadd fivetran
[ec2-user]$ sudo useradd -m -g fivetran fivetran
[ec2-user]$ sudo su - fivetran
[fivetran]$ mkdir ~/.ssh
[fivetran]$ chmod 700 ~/.ssh
[fivetran]$ cd ~/.ssh
[fivetran]$ vim authorized_keys
[fivetran]$ chmod 600 authorized_keys
```

次は MySQL データベースに Fivetran ユーザーを作成する。

```
MySQL [(none)]> CREATE USER fivetran@'%' IDENTIFIED BY 'password';
MySQL [(none)]> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, CREATE TEMPORARY TABLES, CREATE VIEW ON *.* TO fivetran@'%';
```

これで準備が整った。最終的な設定は下記のようになる。これで Save&Test を選択して成功すれば Destination の設定は完了。

```
Host: yout_host.ap-northeast-1.rds.amazonaws.com
Port: 3306
Database: Ignored
User: fivetran -- MySQLのユーザー
Password: ****** -- MySQLのユーザーのパスワード
Connection method: Connect via an SSH
Tunnel Host: 111.11.111.11 -- EC2のipアドレス
Tunnel Port: 22
Tunnel User: fivetran -- EC2のユーザー
Public Key: ssh-rsa ***** -- コピペした公開鍵
Data processing location: Japan
Cloud service provider: AWS
Cloud region: ap-northeast-1
Destination Group ID: rabbit_bunt
Default Time Zone: Japan Standard Time JST (UTC +09)
```

Connection tests が下記のように段階を踏んでくれるので、どこでエラーが出たのかわかりやすい。

- Connecting to SSH tunnel
- Connecting to host
- Validating certificate
- Warehouse User
- Permission Test

補足として、`innodb_buffer_pool_size`は 1024MB の 2 倍を設定しても、`db.t3.micro`では駄目。

```
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 0
Dictionary memory allocated 551081
Buffer pool size   24574
Free buffers       23325
Database pages     1243
Old database pages 478
Modified db pages  0
Pending reads      0
```

### :pencil2: Connectors

次は Connector の設定を行う。ここでは GoogleSpreadSheet を利用するので、画面左のメニュー「Connectors」から「Add Connector」を選択し、「Google Sheets」を選択する。

こちらもガイドに従って必要な設定を行っていく。[Google Sheets Setup Guide](https://fivetran.com/docs/files/google-sheets/google-sheets-setup-guide)にも同じガイドが記載されている。

`Destination schema`には Fivetran が作成するデータベース名を入力し、`Destination table`にはテーブル名を入力する。ここでは`google_sheets`と`monitoring_myroom_co_2`とした。

今回も Service Account を利用するが、サービスアカウントは設定画面に表示されているメールアドレス`****-****-****@fivetran-production.iam.gserviceaccount.com`に対して、アクセスするシートを共有すれば良い。あとは Sync したいシート URL とセルの範囲を「名前付き範囲」として設定する。

これで Save&Test を選択して成功すれば Connector の設定は完了。

### :pencil2: Sync

Destination と Connector の設定が完了したので、Sync ボタンを押して Sync する。`Sync Frequency`は 5 分に設定したので、5 分ごとに Sync が実行される。

MySQL を確認すると、Sync の実行前に存在していなかった`google_sheets`というデータベースが作成されている。

```
ySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| event_auto_insert  |
| googlespreadsheet  |
| information_schema |
| mysql              |
| performance_schema |
| sftp_db            |
| sys                |
+--------------------+
7 rows in set (0.00 sec)

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| event_auto_insert  |
| google_sheets      | ---- fivetran が作ったテーブル
| googlespreadsheet  |
| information_schema |
| mysql              |
| performance_schema |
| sftp_db            |
| sys                |
+--------------------+
8 rows in set (0.00 sec)
```

また、`monitoring_myroom_co_2`テーブルのレコードも Sync されている。

```
MySQL [(none)]> use google_sheets;
MySQL [google_sheets]> show tables;
+-------------------------+
| Tables_in_google_sheets |
+-------------------------+
| fivetran_audit          |
| monitoring_myroom_co_2  |
+-------------------------+
2 rows in set (0.00 sec)
```

Sync される段階で、`_row`と`_fivetran_synced`カラムが追加される模様。

```
MySQL [google_sheets]> select * from monitoring_myroom_co_2 order by time desc limit 5;
+------+----------+---------------------+----------------------------+
| _row | co_2_ppm | time                | _fivetran_synced           |
+------+----------+---------------------+----------------------------+
| 3719 |      740 | 2023-07-07 18:21:38 | 2023-07-07 18:22:08.602000 |
| 3718 |      724 | 2023-07-07 18:16:28 | 2023-07-07 18:22:08.602000 |
| 3717 |      754 | 2023-07-07 18:11:20 | 2023-07-07 18:22:08.602000 |
| 3716 |      701 | 2023-07-07 18:06:16 | 2023-07-07 18:22:08.602000 |
| 3715 |      689 | 2023-07-07 18:01:12 | 2023-07-07 18:22:08.602000 |
+------+----------+---------------------+----------------------------+
5 rows in set (0.01 sec)

```

5 分経過してからテーブルを確認すると、内容が Sync していることがわかる。

```
MySQL [google_sheets]> select * from monitoring_myroom_co_2 order by time desc limit ５;
+------+----------+---------------------+----------------------------+
| _row | co_2_ppm | time                | _fivetran_synced           |
+------+----------+---------------------+----------------------------+
| 3720 |      738 | 2023-07-07 18:26:42 | 2023-07-07 18:28:47.471000 | -- add
| 3719 |      740 | 2023-07-07 18:21:38 | 2023-07-07 18:28:47.471000 |
| 3718 |      724 | 2023-07-07 18:16:28 | 2023-07-07 18:28:47.471000 |
| 3717 |      754 | 2023-07-07 18:11:20 | 2023-07-07 18:28:47.471000 |
| 3716 |      701 | 2023-07-07 18:06:16 | 2023-07-07 18:28:47.471000 |
+------+----------+---------------------+----------------------------+
5 rows in set (0.01 sec)

MySQL [google_sheets]> select * from monitoring_myroom_co_2 order by time desc limit ５;
+------+----------+---------------------+----------------------------+
| _row | co_2_ppm | time                | _fivetran_synced           |
+------+----------+---------------------+----------------------------+
| 3721 |      748 | 2023-07-07 18:31:46 | 2023-07-07 18:31:58.908000 | -- add
| 3720 |      738 | 2023-07-07 18:26:42 | 2023-07-07 18:31:58.908000 |
| 3719 |      740 | 2023-07-07 18:21:38 | 2023-07-07 18:31:58.908000 |
| 3718 |      724 | 2023-07-07 18:16:28 | 2023-07-07 18:31:58.908000 |
| 3717 |      754 | 2023-07-07 18:11:20 | 2023-07-07 18:31:58.908000 |
+------+----------+---------------------+----------------------------+
5 rows in set (0.01 sec)

```

MySQL のインスタンスタイプをもとに戻すと、しっかりとエラーになってしまう。こちらのログは Connectors の GoogleSheet の logs から確認できる。

```
{
  "event" : "sync_end",
  "data" : {
    "status" : "FAILURE_WITH_TASK",
    "reason" : "java.lang.RuntimeException: The innodb_buffer_pool_size for your database is: 384 MB, which is very low. Please make sure its at least 1024 MB.",
    "taskType" : "reconnect_warehouse"
  },
  "created" : "2023-07-07T09:56:43.688Z",
  "connector_type" : "google_sheets",
  "connector_id" : "empty_bount",
  "connector_name" : "google_sheets.monitoring_myroom_co_2",
  "sync_id" : "b4hyg080-a5c0-4qa3-a4f3-12eef82wqw9"
}
```

次の日にデータを確認すると、`id`が 3721 から 3906 まで進んでおり、問題なく Sync されていることが確認できる。

```
MySQL [google_sheets]> select * from monitoring_myroom_co_2 order by time desc limit 5;

+------+----------+---------------------+----------------------------+
| _row | co_2_ppm | time                | _fivetran_synced           |
+------+----------+---------------------+----------------------------+
| 3906 |      694 | 2023-07-08 10:26:20 | 2023-07-08 10:30:19.223000 |
| 3905 |      745 | 2023-07-08 10:21:16 | 2023-07-08 10:30:19.223000 |
| 3904 |      731 | 2023-07-08 10:16:12 | 2023-07-08 10:30:19.223000 |
| 3903 |      734 | 2023-07-08 10:11:08 | 2023-07-08 10:30:19.223000 |
| 3902 |      759 | 2023-07-08 10:06:03 | 2023-07-08 10:30:19.223000 |
+------+----------+---------------------+----------------------------+
5 rows in set (0.02 sec)

```

## :closed_book: Reference

- [Fivetran](https://www.fivetran.com/)
