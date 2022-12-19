## :memo: Overview

シェルスクリプトでデータベースから条件を変えながら csv をエクスポートする方法をまとめておく。私の諸々の能力がなく、実務で使えるようなものではない。

`copy`コマンドを利用して出力した方が良いという記述もあったが、そうすると`where`の条件値を変更できなかったり、出力表示で`(rows)`を削除するとヘッダーも消えたりと、全然、やりたいことができてないが、その過程をメモしておく。

## :floppy_disk: Database

PostgreSQL

## :bookmark: Tag

`shell`, `bash`

## :pencil2: Example

まずは、`~/.pgpass`にパスワードを保存して、ファイルの権限を`600`に変更する。

```
# hostname:port:database:username:password　というフォーマット
```

ここでは`sample.sh`を実行することで(権限変更も忘れずに)、抽出条件を変更しながら、csv をエクスポートを行う。

```sql
#!/usr/bin/sh

vals=(CLERK SALESMAN MANAGER ANALYST PRESIDENT)

for val in "${vals[@]}"
do
    res=$( \
    psql \
    -h HostName  \
    -p PortNumber \
    -d DatabaseName \
    -U Username \
    --no-align \
    -F, \
    -q \
    -c  \
    " \
    select * \
    from emp \
    where job = '${val}' \
    ; \
    " \
    )

    echo "Export Start! job is ${val}"
    echo "$res" > "/Users/aki/Desktop/output_${val}.csv"
    echo "Export End!"
done

# psql Options
# -A or --no-align 位置揃えなしの出力モードに切り替え
# -F or --field-separator tab $'\t' 区切り文字の指定。カンマ区切りは-F,
# -q or --quiet psqlがメッセージ出力なしで処理を行うように指示
# -X or --no-psqlrc 起動用ファイル（psqlrcファイルもしくはユーザ用の~/.psqlrcファイル）を読み込みません
# -c or --command
```

`shell`フォルダに保存しておいた`sample.sh`を実行する。

```sql
$ bash ~/shell/sample.sh

Export Start! job is CLERK
Export End!
Export Start! job is SALESMAN
Export End!
Export Start! job is MANAGER
Export End!
Export Start! job is ANALYST
Export End!
Export Start! job is PRESIDENT
Export End!


$ ls -a | grep output
output_ANALYST.csv
output_CLERK.csv
output_MANAGER.csv
output_PRESIDENT.csv
output_SALESMAN.csv
```

エクスポートされた内容を確認しておく。

```sql
$ cat output_ANALYST.csv
empno,ename,job,mgr,hiredate,sal,comm,deptno
7788,SCOTT,ANALYST,7566,1982-12-09,3000,,20
7902,FORD,ANALYST,7566,1981-12-03,3000,,20
(2 rows)

$ cat output_SALESMAN.csv
empno,ename,job,mgr,hiredate,sal,comm,deptno
7499,ALLEN,SALESMAN,7698,1981-02-20,1600,300,30
7521,WARD,SALESMAN,7698,1981-02-22,1250,500,30
7654,MARTIN,SALESMAN,7698,1981-09-28,1250,1400,30
7844,TURNER,SALESMAN,7698,1981-09-08,1500,0,30
(4 rows)
```

## :closed_book: Reference

- [How to bash array into psql query](https://stackoverflow.com/questions/70162137/how-to-bash-array-into-psql-query)
- [【PostgreSQL】psql でパスワードを省略する](https://dk521123.hatenablog.com/entry/2020/03/06/000000)
- [【Shell】【PostgreSQL】シェルで SQL 結果を受け取る](https://dk521123.hatenablog.com/entry/2021/08/16/231459)
