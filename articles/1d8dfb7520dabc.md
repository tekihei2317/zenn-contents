---
title: "データベースのロックの基礎からデッドロックまで"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["データベース", "MySQL"]
published: true
publication_name: "gibjapan"
---

データベースのロックについて、資料を読んだり実際に試してみたので、学んだことを整理してみようと思います。はじめにロックについての基本的な知識を整理して、最終的にはデッドロックとその対策について説明します。

## 使用したソフトウェアのバージョン

- MySQL 8.0.31

この記事ではMySQLを使用しています。その他のデータベースについては、基本的な部分は共通していると思いますが、異なる点があることをご了承ください。

## ロックとは何か（概要）

ロックはデータの更新を正しく行うための仕組みの一つで、あるデータに対する更新処理を制御するためのものです。ここで、データを正しく更新するとはどういうことかを説明するために、口座への振込を例に考えてみます。

例えば、口座Aから口座Bに1万円の振込を行うとします。このとき、口座Aの出金処理と口座Bの入金処理は、必ず両方が成功しなければなりません。このための仕組みが、みなさんご存じの通りトランザクションです。

![](https://i.gyazo.com/131ddcde34079c1fd3973c630d484da1.png =480x)

また、口座Aから口座Bに振り込むのと同じタイミングに、口座Cから口座Bに1万円が振り込まれるとします。このとき、参照と更新が次のような順序で行われると、Bの残高が2万円増えるべきところが1万円しか増えないことが起こってしまいます。

![](https://i.gyazo.com/30a908afae0530b55b3e2ae8b7d7bdbf.png =480x)

このような同時更新による問題を防ぐための仕組みがロックです。

## ロックとは何か（詳細）

ロックとは「あるデータに対する更新処理を制御するための仕組み」でした。データに対する更新処理を制御するというのは、同時に行われると問題が起きる処理を順番に実行するということです。そうするためには、あるデータを変更できるトランザクションを1つまでに制限します。

### 2種類のロック

ロックには排他ロックと共有ロックの2種類があります。先述の「変更できるトランザクションを1つまでに制限する機能」は、排他ロックのことです。排他ロックは、「これからこのデータを変更するので、他のトランザクションは変更しないでね」ということを表します。

共有ロックは、トランザクション内で読み取りを行なっていることを示すためのロックです。つまり、「今このデータを読み取っているので、他のトランザクションは変更しないでね」ということを表します。

排他ロックと共有ロックは、ロックをかけられたデータが他のトランザクションから変更できないという点は同じです。異なる点は、排他ロックがデータを変更することが目的なのに対し、共有ロックはそうではない点です。

排他ロックと共有ロックの共通点や違いをまとめてみると、次のようになります（トランザクションをTXと略しています）。

|            | 他のTXから読み取れる | 他のTXから変更できる | 自分のTXから変更できる | 他のTXから排他ロックを取得できる | 他のTXから共有ロックを取得できる |
| ---------- | -------------------- | -------------------- | ---------------------- | -------------------------------- | -------------------------------- |
| 排他ロック | o                    | x                    | o                      | x                                | x                                |
| 共有ロック | o                    | x                    | △                      | x                                | o                                |

排他ロックはデータを順番に更新するための仕組みなので、複数のトランザクションが取得することはできません。一方で、共有ロックは複数のトランザクションが取得できます。

:::details 補足: 共有ロックの△の部分について

共有ロックで自身からの変更が△になっているのは、変更ができる場合とできない場合があるためです。具体的には、他のトランザクションが共有ロックを取得していなければ変更でき、取得している場合はできません。

しかし、共有ロックを取得したデータを変更することは、後述するデッドロックの原因になるため避けたほうがよいと考えられます。

:::

### ロックの取得するにはどうすればよいか

ロックを取得する代表的な方法には、次の2つがあります。

- トランザクション内でロック読み取りする（`SELECT ... FOR UPDATE`、`SELECT ... FOR SHARE`）
- `UPDATE`や`DELETE`を実行する

これらの方法では、検索条件に主キーなどの一意なインデックスを使った場合は、該当するレコードにロックがかけられます。

:::message
範囲検索を使った場合や、一意でないインデックスを使った場合は異なる挙動になりますが、ここでは簡単のためにレコードロックのみを考えることにします。
:::

### ロックを取得してみる

それでは、実際に排他ロックと共有ロックを取得してみます。

まずは、ロック読み取りで排他ロックを取得してみます。

```sql
mysql(T1)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(T1)> select * from numbers where id = 1 for update;
+----+-------+
| id | value |
+----+-------+
|  1 |    30 |
+----+-------+
1 row in set (0.00 sec)
```

別のセッションで、同じレコードに対して排他ロックの取得を試してみます。

```sql
mysql(T2)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(T2)> select * from numbers where id = 1 for update;

```

すると、実行結果がすぐ返ってきません。このように、ロックが取得できない場合はロックが待機状態（ロックの解放待ち）になります。待機状態のロックは、`innodb_lock_wait_timeout`秒（デフォルトでは50秒）が経過すると取得に失敗します。

ロックを解放するために、T1のトランザクションを終了します。すると、T2のクエリの実行結果が表示され、ロックも取得できました。

```sql
mysql(T1)> commit;
Query OK, 0 rows affected (0.00 sec)

mysql(T2)> select * from numbers where id = 1 for update;
+----+-------+
| id | value |
+----+-------+
|  1 |    30 |
+----+-------+
1 row in set (5.47 sec)
```

次に、ロック読み取りで共有ロックを取得してみます。共有ロックは、複数のトランザクションから取得できます。

```sql
mysql(T1)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(T1)> select * from numbers where id = 1 for share;
+----+-------+
| id | value |
+----+-------+
|  1 |    30 |
+----+-------+
1 row in set (0.00 sec)

mysql(T2)> begin;
Query OK, 0 rows affected (0.01 sec)

mysql(T2)> select * from numbers where id = 1 for share;
+----+-------+
| id | value |
+----+-------+
|  1 |    30 |
+----+-------+
1 row in set (0.00 sec)
```

また、ロックの取得状態は`performance_schema.data_locks`テーブルで確認できます。`S, REC_NOT_GAP`となっているのがレコードの共有ロックです。2つのトランザクションから取得できていることが分かります。

```sql
mysql(T1)> select THREAD_ID ,OBJECT_NAME, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA from performance_schema.data_locks;
+-----------+-------------+------------+-----------+---------------+-------------+-----------+
| THREAD_ID | OBJECT_NAME | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+-----------+-------------+------------+-----------+---------------+-------------+-----------+
|        48 | numbers     | NULL       | TABLE     | IS            | GRANTED     | NULL      |
|        48 | numbers     | PRIMARY    | RECORD    | S,REC_NOT_GAP | GRANTED     | 1         |
|        50 | numbers     | NULL       | TABLE     | IS            | GRANTED     | NULL      |
|        50 | numbers     | PRIMARY    | RECORD    | S,REC_NOT_GAP | GRANTED     | 1         |
+-----------+-------------+------------+-----------+---------------+-------------+-----------+
4 rows in set (0.00 sec)
```

カラムの詳細については、[InnoDBの行ロック状態を確認する［その1］ | gihyo.jp](https://gihyo.jp/dev/serial/01/mysql-road-construction-news/0145)などを参照してみて下さい。

### ここまでの整理

ここまでの内容を、Q & A形式で整理してみます。

:::details データベースのロックは何のための仕組みなのか？
データベースのロックは、あるデータに対する更新処理を制御するための仕組みです。ロックを使うことで、同時に実行すると問題が発生するトランザクションの一部の処理を、順番に実行させることができます。
:::

:::details ロックをすると、他のトランザクションから読み取れなくなるのか？
いいえ！ロックを取得しない読み取り（通常のSELECT）は行えます。
:::

:::details 排他ロックと共有ロックはそれぞれどういう目的のロックなのか？
排他ロックはデータを変更するためのロックで、共有ロックはデータを読み取っていることを示すためのロックです。
:::

:::details 排他ロックと共有ロックの違いは何か？
排他ロックは、データを更新できるトランザクションを1つまでに制限する仕組みです。そのため、排他ロックは単一のトランザクションしか取得できません。一方で、共有ロックは複数のトランザクションから取得できます。
:::

:::details 排他ロックと共有ロックの共通点は何か？
どちらも、他のトランザクションからの変更を禁止します。
:::

:::details ロックを取得するにはどのようにすればよいのか？
`SELECT … FOR UPDATE`（排他ロック）や、`SELECT … FOR SHARE`（共有ロック）で取得できます。また、`UPDATE`や`DELETE`の実行時には、対象のレコードに排他ロックがかけられます。
:::

:::details ロックが解放されるのはどのタイミングか？
ロックは、トランザクションが終了したタイミングで解放されます。
:::

## 最初の問題をロックで解決する

それでは、最初の振込の問題をロックを使って解決してみましょう。どのような問題だったかというと、口座A→口座Bへの入金と、口座C→口座Bへの入金が同じタイミングで起こり、口座Bの入金金額がおかしくなってしまうという問題です。

解決するためには、入金処理を開始するときに2つの口座の排他ロックを取得すればよいです（`{}`はプレースホルダを表すこととします）。

```sql
begin;
select * from accounts where id = {id_from} for update;
select * from accounts where id = {id_to} for update;
update accounts set balance = balance - {amount} where id = {id_from};
update accounts set balance = balance + {amount} where id = {id_to};
commit;
```

こうすることで、同じ口座に関する処理は順番に行われるため、口座Bの金額がおかしくなることを防げます（赤線が口座Bの更新処理で、順番に行われていることがわかります）。

![](https://i.gyazo.com/bec52788017801365f848fe5400510d0.png)

## デッドロックとは何か

ロックを使うことでデータを正しく更新できるようになりました。しかし、ロックはまた別の問題を引き起こすことがあります。それはデッドロックです。

デッドロックとは、複数のトランザクションが、互いに相手が所有するロックが解放されるのを待機して処理が進まなくなってしまうことをいいます。実は、先ほどの解決策でもデッドロックが起きる可能性があります。

定義から分かるように、デッドロックは共有ロックだけを使う場合は起きません（共有ロックだけではロックの待機が起こらないため）。つまり、共有ロック・排他ロックが両方使われる場合と、排他ロックが複数使われる場合に発生します。

少々無理やりになってしまいますが、以下の単純なテーブルを使ってデッドロックを発生させてみます。

```sql
CREATE TABLE numbers (
  id INT PRIMARY KEY AUTO_INCREMENT,
  value INT NOT NULL
);
```

### 共有ロックと排他ロックによるデッドロック（変換デッドロック）

T1とT2でそれぞれ共有ロックを取得したあと、T1とT2の両方で排他ロックを取得しようとするとデッドロックになります。排他ロックを取得するときに、共有ロックの解放待ちになるためです。

![](https://i.gyazo.com/81794a0002c8383de7b657a4ee303337.png =480x)

:::details 実行例

```sql
mysql(A)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(A)> select * from numbers where id = 1 for share;
+----+-------+
| id | value |
+----+-------+
|  1 |    30 |
+----+-------+
1 row in set (0.00 sec)

mysql(B)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(B)> select * from numbers where id = 1 for share;
+----+-------+
| id | value |
+----+-------+
|  1 |    30 |
+----+-------+
1 row in set (0.00 sec)

-- ロック待ちになり、 相手がデッドロックがロールバックされてから実行される
mysql(A)> update numbers set value = 100 where id = 1;
Query OK, 1 row affected (4.34 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql(B)> update numbers set value = 100 where id = 1;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

:::

なお、共有ロックを取得している状態で排他ロックを取得しようとして起こるデッドロックを、「変換デッドロック」と呼ぶようです。

### 複数の排他ロックによるデッドロック（サイクルデッドロック）

T1とT2でそれぞれ別のデータの排他ロックを取得したあと、相手側が持つデータの排他ロックを取得しようとするとデッドロックになります。

![](https://i.gyazo.com/f57ff22ea2d96a4dcaae868d1d0883ad.png =480x)

:::details 実行例

```sql
mysql(A)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(B)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(A)> select * from numbers where id = 1 for update;
+----+-------+
| id | value |
+----+-------+
|  1 |    30 |
+----+-------+
1 row in set (0.02 sec)

mysql(B)> select * from numbers where id = 2 for update;
+----+-------+
| id | value |
+----+-------+
|  2 |    10 |
+----+-------+
1 row in set (0.00 sec)

mysql(A)> select * from numbers where id = 2 for update;
-- ロック待ちになり、Bがデッドロックでロールバックされてから表示される
+----+-------+
| id | value |
+----+-------+
|  2 |    10 |
+----+-------+
1 row in set (6.82 sec)

mysql(B)> select * from numbers where id = 1 for update;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```
:::

このようなデッドロックを、排他ロックの待機がサイクル状になるため「サイクルデッドロック」と呼びます。
## デッドロックが起こらないようにするためには

デッドロックが起こる原因は、共有ロックを取得したデータを更新してしまうことや、トランザクションでの更新順序が一定でないことです。つまり、更新する場合は排他ロックを取得することと、トランザクションでの更新順序を一定にすることを守れば防げるはずです。

例えば、振込の次の実装にはデッドロックが起きる可能性があるという話をしました。具体的には、口座A→口座Bの振込と、口座B→口座Aの振込が同時に起きるとサイクルデッドロックが発生します。

```sql
begin;
select * from accounts where id = {id_from} for update;
select * from accounts where id = {id_to} for update;
update accounts set balance = balance - {amount} where id = {id_from};
update accounts set balance = balance + {amount} where id = {id_to};
commit;
```

デッドロックを回避するには、例えばIDが小さい順番にロックを取得する方法があります。

## （おまけ）在庫管理システムでのデッドロック

最後に、私が現在開発している在庫管理システムで起こりそうだった例を紹介しようと思います。少々長くなるので、[まとめ](#まとめ)まで読み飛ばしていただいても問題ありません。

まず、テーブルについて説明します。商品には複数の在庫があり、在庫は現在の在庫数を持っています。在庫→商品への外部キーと、入荷→在庫への外部キーには、外部キー制約が設定されています。

:::details DDL

```sql
-- 商品
CREATE TABLE products (
  id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(191) NOT NULL
);

-- 在庫
CREATE TABLE inventories (
  id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
  product_id INT NOT NULL,
  current_quantity INT NOT NULL,

  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT ON UPDATE CASCADE
);

-- 入荷
CREATE TABLE arrivals (
  id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
  inventory_id INT NOT NULL,
  quantity INT NOT NULL,
  arrived_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (inventory_id) REFERENCES inventories(id) ON DELETE RESTRICT ON UPDATE CASCADE
);

insert into products (id, name) values (1, 'ハサミ'), (2, 'ノート'), (3, 'ボールペン');
insert into inventories (product_id, current_quantity) values (1, 10), (2, 30);
```

:::

入荷があるごとに入荷レコードを登録し、関連する在庫の在庫数を更新します。具体的には次のような実装です。

```sql
begin;
insert arrivals (inventory_id, quantity) values ({inventory_id}, {quantity});
update inventories set current_quantity = current_quantity - quantity;
commit;
```

UPDATE文で排他ロックがかけらるため、在庫数の更新は正しく行えます。しかし、これは同時に実行されたときにデッドロックが発生する可能性があります。実際に起こしてみましょう。

```sql
-- T1で入荷を登録
mysql(T1)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(T1)> insert into arrivals (inventory_id, quantity) values (1, 10);
Query OK, 1 row affected (0.04 sec)

-- T2で入荷を登録
mysql(T2)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(T2)> insert into arrivals (inventory_id, quantity) values (1, 20);
Query OK, 1 row affected (0.02 sec)

--- T1で在庫を更新しようとすると、ロック待ちになる(→なぜ?)
mysql(T1)> update inventories set current_quantity = current_quantity + 10 where id = 1;

---T2でも在庫を更新すると、デッドロックになる
mysql(T2)> update inventories set current_quantity = current_quantity + 20 where id = 1;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

一体なぜ、在庫の更新の部分でロック待ちになるのでしょうか？原因は、**入荷にINSERTするときに、外部キーで参照する在庫レコードに共有ロックが設定されるから**です。

デッドロックが発生したのは、共有ロックがかかったデータに、2つのトランザクションから排他ロックを取得しようとしたためです。つまり、変換デッドロックが発生しています。

外部キーによる共有ロックの取得については、MySQLのドキュメントに次のように書かれています。

> FOREIGN KEY 制約がテーブル上で定義されている場合は、制約条件をチェックする必要がある挿入、更新、または削除が行われると、制約をチェックするために、参照されるレコード上に共有レコードレベルロックが設定されます。
[MySQL :: MySQL 8.0 リファレンスマニュアル :: 15.7.3 InnoDB のさまざまな SQL ステートメントで設定されたロック](https://dev.mysql.com/doc/refman/8.0/ja/innodb-locks-set.html)

共有ロックは、「データを参照するので更新しないでねというロック」でした。そのため、子テーブルへのINSERTで親テーブルの関連レコードに共有ロックがかかるのは、外部キー制約の挙動としては妥当だと考えられます。

デッドロックの原因は、共有ロックを取得した在庫レコードを更新していることです。つまり、最初に更新のための排他ロックを取得すれば、デッドロックを防げます。

```sql
begin;
-- 最初に排他ロックを取得する
select id from inventories where id = {inventory_id} for update;
-- 排他ロックを取得しているため、ここで共有ロックは取得されない
insert arrivals (inventory_id, quantity) values ({inventory_id}, {quantity});
update inventories set current_quantity = current_quantity - quantity;
commit;
```

## その他のデッドロックの例

Zennの記事では、次の2つの記事が勉強になりました。どちらの例も、意図しない共有ロックの取得によって変換デッドロックが起きています。

https://zenn.dev/tockn/articles/4268398c8ec9a9

https://zenn.dev/shuntagami/articles/ea44a20911b817

デッドロックの対策として、「親レコードを排他ロックする」「再試行する」「デッドロックを許容する」「Redis Mutexを使う」「分離レベルを下げる」などの方法が紹介されています。

## まとめ

- データを正しく更新するためのデータベースの仕組みには、トランザクションとロックがある
- ロックは、あるデータに対する更新処理を制御するための仕組み
- ロックには、データの更新を行うための排他ロックと、データを読み取っていることを示すための共有ロックがある
- ロックしようとしているデータが既にロックされている場合は、ロックの解放待ち（トランザクションの終了待ち）になる。ただし、共有ロックは複数取得できる。
- レコードロックを取得する代表的な方法には、ロック読み取り（`SELECT … FOR UPDATE`、`SELECT … FOR SHARE`）と、`UPDATE`文・`DELETE`文がある。`UPDATE`や`DELETE`では、レコードに排他ロックが設定される。
- 複数のトランザクションが互いに終了待ちになることをデッドロックという。デッドロックが発生すると、片方のトランザクションがロールバックで強制終了される。
- デッドロックの原因には、データの更新を行うのに共有ロックを取得していることや、データを更新する順序が一定でないことがある。前者で起きるデッドロックを変換デッドロック、後者をサイクルデッドロックという。

## 参考

- [Database-Exclusion-Control.pdf](https://fintan.jp/wp-content/uploads/2022/03/Database-Exclusion-Control.pdf)
- [データベースのデッドロック - モノノフ日記](https://mononofu.hatenablog.com/entry/20071128/1196212796)
- [リレーショナルデータベース　デッドロックの具体例 - Qiita](https://qiita.com/Black_Kite/items/8795d03aacfe9d35ceaf)
