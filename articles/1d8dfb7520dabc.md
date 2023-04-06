---
title: "データベースのロックの基礎からデッドロックまで"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["データベース", "MySQL"]
published: false
publication_name: "gibjapan"
---

データベースのロックについて、資料を読んだり実際に試してみたので、学んだことを整理してみようと思います。はじめにロックについての基本的な知識を整理して、最終的にはデッドロックとその対策について説明します。

## 使用したソフトウェアのバージョン

- MySQL 8.0.31

## ロックとは何か（概要）

ロックはデータの更新を正しく行うための仕組みの一つで、トランザクションの実行順を制御するためのものです。ここで、データを正しく更新するとはどういうことかを説明するために、振込を例に考えてみます。

例えば、口座Aから口座Bに1万円の振込を行うとします。このとき、口座Aの出金処理と口座Bの入金処理は、必ず両方が成功しなければなりません。このための仕組みが、みなさんご存じの通りトランザクションです。

![トランザクションによる更新](https://i.gyazo.com/2ff22fe194bbfa0c4ec925dcf70026f9.png =480x)

また、口座Aから口座Bに振り込むのと同じタイミングに、口座Cから口座Bに1万円振り込むとします。このとき、参照と更新が次のような順序で行われると、Bが2万円増えるべきところが1万円しか増えないことが起こってしまいます。

![正しく更新できていない](https://i.gyazo.com/caba32292d7ffdf8c372ac6be89c30a8.png =480x)

このような同時更新による問題を防ぐための仕組みが、ロックです。

## ロックとは何か（詳細）

ロックとは「トランザクションの実行順を制御するための仕組み」でした。トランザクションの実行順を制御するというのは、同時に行われると問題が起きる処理を直列に実行するということです。そうするためには、あるデータを変更できるトランザクションを1つまでに制限すれば良いです。

### ロックの種類には何があるか

ロックには排他ロックと共有ロックの2種類あります。先述の「変更できるトランザクションを1つまでに制限する仕組み」は、排他ロックのことです。排他ロックは、「これからこのデータを変更するので、他のトランザクションは処理を待ってね」ということを表します。

共有ロックは、トランザクション内で読み取りを行なっていることを示すためのロックです。つまり、「今このデータを読み取っているので、他のトランザクションは変更しないでね」といことを表します。

排他ロックと共有ロックは、かけられたデータが他のトランザクションから変更できないという点は同じです。異なる点は、排他ロックをがデータを変更することが目的なのに対し、共有ロックはそうではない点です。

排他ロックと共有ロックの共通点や違いをまとめてみると、次のようになります。

|            | 他のTXから読み取れる | 他のTXから書き込める | 自分のTXから書き込める | 他のTXから排他ロックを取得できる | 他のTXから共有ロックを取得できる |
| ---------- | -------------------- | -------------------- | ---------------------- | -------------------------------- | -------------------------------- |
| 排他ロック | o                    | x                    | o                      | x                                | x                                |
| 共有ロック | o                    | x                    | △                      | x                                | o                                |

共有ロックで△になっているのは、他のTXが共有ロックを取得していなければでき、取得している場合はできないからです。しかし、共有ロックを取得したデータを変更することは、後述するデッドロックの原因になるため避けるべきです。

### ロックの取得するにはどうすればよいか

メモ: ここまでの内容を確認できるようにしたいです。共有ロックは複数のTXから取得できること、排他ロックは1つのTXしか取得できないこと、ロックを取得できない場合は解放待ちになること、TXを終了するとロックが解放されること、などです。排他ロックを取得するにはSELECT FOR UPDATE、共有ロックを取得するためにはSELECT FOR SHARE。またUPDATEやDELETEでも排他ロックが取得されることを確認したいです。

### ここまでの整理

ここまでの内容を、Q & A形式で整理してみます。

:::details データベースのロックは何のための仕組みなのか？
トランザクションの実行順序を制御するための仕組みです。ロックを使うことで、同じデータを更新するトランザクションを直列に実行させることができます。
:::

:::details ロックをすると、他のトランザクションから読み取れなくなるのか？
いいえ！ロックを取得しない読み取り（通常のSELECT）は行えます。
:::

:::details 排他ロックと共有ロックはそれぞれどういう目的のロックなのか？
排他ロックは「データを変更するため」のロックで、共有ロックは「読み取りを行っていることを示す」ためのロックです。
:::

:::details 排他ロックと共有ロックの違いは何か？
変更行うトランザクションは1つまでのため、排他ロックは単一のトランザクションしか取得できません。一方で、共有ロックは複数のトランザクションから取得できます。
:::

:::details 排他ロックと共有ロックの共通点は何か？
どちらも、他のトランザクションからの変更を禁止します。
:::

:::details ロックを取得するにはどのようにすればよいのか？
SELECT … FOR UPDATE（排他ロック）や、SELECT … FOR SHARE（共有ロック）で取得できます。また、UPDATEやDELETEの実行時には、対象のレコードに排他ロックがかけられます。
:::

:::details ロックが解放されるのはどのタイミングか？
ロックは、トランザクションが終了したタイミングで解放されます。
:::

## 最初の問題をロックで解決する

それでは、最初の振込の問題をロックを使って解決してみましょう。どのような問題だったかというと、口座A→口座B、口座C→口座Bへの入金が同じタイミングで起こり、口座Bの入金金額がおかしくなってしまうという問題です。

解決するためには、入金処理を開始するときに2つの口座の排他ロックを取得すればよいです。

```sql
begin;
select * from accounts where id = {id_from} for update;
select * from accounts where id = {id_to} for update;
update accounts set balance = balance - {amount} where id = {id_from};
update accounts set balance = balance + {amount} where id = {id_to};
commit;
```

こうすることで、同じ口座に関する処理は順番に行われるため、口座Bの金額がおかしくなることを防げます（赤線がBの更新処理で、順番に行われていることがわかります）。

![ロックで解決](https://i.gyazo.com/4f51b5891a7afcbe9b875fb5aa97dc5a.png)

## デッドロックとは何か

最後に、デッドロックについて説明します。デッドロックとは、複数のトランザクションが互いに完了待ちになってしまうことをいいます。定義から分かるように、デッドロックは共有ロックだけを使う場合は起きません（解放待ちにならないため）。つまり、共有ロック・排他ロックが両方使われる場合と、排他ロックが複数使われる場合に発生します。

少々無理やりですが、この2つの場合のデッドロックを起こしてみます。

### 共有ロックと排他ロックでデッドロックが発生する例

T1とT2でそれぞれ共有ロックを取得したあと、T1とT2の両方で排他ロックを取得しようとするとデッドロックになります。排他ロックを取得するときに、共有ロックの解放待ちになるためです。

![共有ロックと排他ロックによるデッドロック](https://i.gyazo.com/94b2e08b1d9289c56e80b735e1f75711.png =480x)

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

### 複数の排他ロックでデッドロックが発生する例

T1とT2でそれぞれ排他ロックを取得したあと、相手側が持つデータの排他ロックを取得しようとするとデッドロックになります。

![複数の排他ロックによるデッドロック](https://i.gyazo.com/835300c455f81372e39a1716cc1a7a6d.png =480x)

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

## デッドロックが起こらないようにするためには

デッドロックが起こる原因は、更新するのに共有ロックを取得している場合や、トランザクションでの更新順序が一定でないことです。つまり、更新する場合は排他ロックを取得することと、トランザクションでの更新順序を一定にすることを守れば防げるはずです。

理解を深めるために、私が現在開発している在庫管理システムで遭遇した例を紹介します。

### 在庫管理システムでのデッドロック

商品には複数の在庫があり、在庫は現在の在庫数を持っています。入荷があるごとに入荷レコードを登録し、関連する在庫の在庫数を更新します。

:::details DDL

```sql
-- 商品
create table products (
  id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(191) NOT NULL
);

-- 在庫
create table inventories (
  id INT NOT NULL PRIMARY KEY AUTO_INCREMENT,
  product_id INT NOT NULL,
  current_quantity INT NOT NULL,

  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT ON UPDATE CASCADE
);

-- 入荷
create table arrivals (
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

入荷の処理を、次のようなコードで実装していました。

```sql
begin;
insert arrivals (inventory_id, quantity) values ({inventory_id}, {quantity});
update inventories set current_quantity = current_quantity - quantity;
commit;
```

在庫を更新する処理では排他ロックがかかっているので、在庫数の更新は正しく行えます。しかし、これは同時に実行されたときにデッドロックが発生する可能性があります。実際に起こしてみましょう。

```sql
mysql(T1)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(T1)> insert into arrivals (inventory_id, quantity) values (1, 10);
Query OK, 1 row affected (0.04 sec)

mysql(T2)> begin;
Query OK, 0 rows affected (0.00 sec)

mysql(T2)> insert into arrivals (inventory_id, quantity) values (1, 20);
Query OK, 1 row affected (0.02 sec)

--- ロック待ちになる(???)
mysql(T1)> update inventories set current_quantity = current_quantity + 10 where id = 1;

---- →デッドロック
mysql(T2)> update inventories set current_quantity = current_quantity + 20 where id = 1;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

一体なぜ、在庫の更新の部分でロック待ちになるのでしょうか？原因は、**入荷にINSERTするときに、外部キーによる共有ロックが取得されるから**です。

デッドロックが発生したのは、共有ロックがかかったデータに2つのトランザクションから排他ロックを取得しようとしたためです。これは、デッドロックの1つ目の例と同じパターンです。

外部キーによる共有ロックの取得については、MySQLのドキュメントに次のように書かれています。

> FOREIGN KEY 制約がテーブル上で定義されている場合は、制約条件をチェックする必要がある挿入、更新、または削除が行われると、制約をチェックするために、参照されるレコード上に共有レコードレベルロックが設定されます。
[MySQL :: MySQL 8.0 リファレンスマニュアル :: 15.7.3 InnoDB のさまざまな SQL ステートメントで設定されたロック](https://dev.mysql.com/doc/refman/8.0/ja/innodb-locks-set.html)

共有ロックは、「データを参照するので更新しないでねというロック」でした。そのため、子テーブルへのINSERTで親テーブルに共有ロックをかけるのは、外部キー制約の挙動としては妥当だと考えられます。

デッドロックの原因は、同じトランザクションで在庫テーブルを更新していることです。そのため、最初に更新のための排他ロックを取得すれば、デッドロックを防げます。

```sql
begin;
select id from inventories where id = {inventory_id} for update;
insert arrivals (inventory_id, quantity) values ({inventory_id}, {quantity});
update inventories set current_quantity = current_quantity - quantity;
commit;
```

### その他のデッドロックの例

Zennの記事では、次の2つの記事が勉強になりました。どちらの例も、デッドロックの原因は意図しない共有ロックの取得です。

- [「トランザクション張っておけば大丈夫」と思ってませんか？ バグの温床になる、よくある実装パターン](https://zenn.dev/tockn/articles/4268398c8ec9a9)
- [MySQLで発生し得る思わぬデッドロックと対応方法](https://zenn.dev/shuntagami/articles/ea44a20911b817)

デッドロックの対策として、「親レコードを排他ロックする」「再試行する」「デッドロックを許容する」「Redis Mutexを使う」「分離レベルを下げる」などの方法が紹介されています。

## まとめ

- データを正しく更新するためのデータベースの仕組みには、トランザクションとロックがある
- ロックは、あるデータに対するトランザクション内の更新順序を制御するためのもの
- ロックには、データの更新を行うための排他ロックと、データを読み取っている（= 変更しないでほしい）ことを示すための共有ロックがある
- ロックするが既にロックされている場合は、ロックの解放待ち（トランザクションの終了待ち）になる。
- レコードロックを取得する代表的な方法には、ロック読み取り（SELECT … FOR UPDATE、SELECT … FOR SHARE）と、UPDATE文・DELETE文がある。UPDATEやDELETEでは、レコードに排他ロックが設定される。
- 複数のトランザクションが互いに終了待ちになることをデッドロックという。デッドロックが発生すると、片方のトランザクションが強制的に終了される（？）
- デッドロックの原因には、データの更新を行うのに共有ロックを取得している場合や、データを更新する順序（排他ロックの取得順序）が一定でないことがある。デッドロックを防ぐためには、これらが起きないようにする。
