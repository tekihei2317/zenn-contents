---
title: "Prisma+MySQLで日時をJSTで保存したくなったときに読む記事"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["prisma", "mysql", "nodejs", "typescript"]
published: true
publication_name: gibjapan
---

Prismaで日時を保存すると、UTCに変換されて保存されてしまいます。そのため、日時をJSTで保存する方法を調べました。

ソースコードはこちらにあります。

https://github.com/tekihei2317/prisma-timezone-middleware

## バージョン

- prisma 4.11.0
- MySQL 8.0.31

## 結論

- Prismaには、MySQLのDATETIME型をどのタイムゾーンで扱うのかのオプションがないので、UTCで保存するのが無難
- 既存のデータがあるなどの理由でどうしてもJSTで保存したい場合は、Prismaのミドルウェアを使って変換する

## 状況

Prismaのスキーマが次のような場合を考えます。

```ts
model Task {
  id        Int      @id @default(autoincrement())
  dueDate   DateTime @db.Date
  startAt   DateTime
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

以下のコードでデータを登録すると、日時がUTCで保存されてしまいます。

```ts
// 2023-03-22 16:08に実行
await prisma.task.create({
  data: {
    dueDate: new Date("2023-03-22"),
    startAt: new Date(),
  },
});
```

```text
+----+------------+-------------------------+-------------------------+-------------------------+
| id | dueDate    | startAt                 | createdAt               | updatedat               |
+----+------------+-------------------------+-------------------------+-------------------------+
|  1 | 2023-03-22 | 2023-03-22 07:08:26.953 | 2023-03-22 07:08:27.058 | 2023-03-22 07:08:27.058 |
+----+------------+-------------------------+-------------------------+-------------------------+
```

## 解決方法

Prismaは保存するときにUTCに変換し、取得するときはUTCとして解釈します。JSTとして保存するためには、登録するときに+9hし、取得するときに-9hすればよいです。

具体的には、Prismaのミドルウェアの機能を使います。

:::details 実装

```ts
import { Prisma } from '@prisma/client'
import { PrismaClient } from '@prisma/client'

// eslint-disable-next-line @typescript-eslint/no-explicit-any
function setOffsetTime(object: any, offsetTime: number) {
  if (object === null || typeof object !== 'object') return

  for (const key of Object.keys(object)) {
    const value = object[key]
    if (value instanceof Date) {
      object[key] = new Date(value.getTime() + offsetTime)
    } else if (value !== null && typeof value === 'object') {
      setOffsetTime(value, offsetTime)
    }
  }
}

export const timezoneMiddleware: Prisma.Middleware = async (params, next) => {
  const offsetTime = 9 * 60 * 60 * 1000

  setOffsetTime(params.args, offsetTime)
  const result = await next(params)
  setOffsetTime(result, -offsetTime)

  return result
}

const prisma = new PrismaClient()
prisma.$use(timezoneMiddleware)

export { prisma }
```

:::

また、MySQLのタイムゾーンもJSTに変更します。なぜかというと、`default(now())`は`CURRENT_TIMESTAMP(3)`に変換されるからです。`CURRENT_TIMESTAMP`は、MySQLのタイムゾーンに応じた値を返します。

ローカルではdocker-composeを使っていたため、次のようにしてタイムゾーンを設定しました。

```yml
db:
  image: mysql:8.0
  ports:
    - 3306:3306
  environment:
    - TZ=Asia/Tokyo
```

:::details タイムゾーンが変わっていることの確認
```text
mysql> show variables like '%zone%';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| system_time_zone | JST    |
| time_zone        | SYSTEM |
+------------------+--------+

mysql> select now();
+---------------------+
| now()               |
+---------------------+
| 2023-03-22 16:14:24 |
+---------------------+
```
:::

この状態で登録してみると、`default(now())`や`@updatedAt`を設定したカラムはUTCのままでした。

```text
+----+------------+-------------------------+-------------------------+-------------------------+
| id | dueDate    | startAt                 | createdAt               | updatedAt               |
+----+------------+-------------------------+-------------------------+-------------------------+
|  1 | 2023-03-22 | 2023-03-22 07:08:26.953 | 2023-03-22 07:08:27.058 | 2023-03-22 07:08:27.058 |
|  2 | 2023-03-22 | 2023-03-22 16:17:13.337 | 2023-03-22 07:17:13.432 | 2023-03-22 07:17:13.432 |
+----+------------+-------------------------+-------------------------+-------------------------+
```

理由は、ミドルウェアを経由していないかつ、Prismaが明示的に値を指定しているからだと考えられます。そこで、`default(now())`や`@updatedAt`の代わりに次のようにします。

```ts diff
model Task {
-  createdAt DateTime @default(now())
-  updatedAt DateTime @updatedAt
+  createdAt DateTime @default(dbgenerated("NOW(3)"))
+  updatedAt DateTime @default(dbgenerated("NOW(3) ON UPDATE NOW(3)"))
}
```

これで、登録日時や更新日時もJSTで保存できるようになりました。

```text
+----+------------+-------------------------+-------------------------+-------------------------+
| id | dueDate    | startAt                 | createdAt               | updatedAt               |
+----+------------+-------------------------+-------------------------+-------------------------+
|  1 | 2023-03-22 | 2023-03-22 07:08:26.953 | 2023-03-22 07:08:27.058 | 2023-03-22 07:08:27.058 |
|  2 | 2023-03-22 | 2023-03-22 16:17:13.337 | 2023-03-22 07:17:13.432 | 2023-03-22 07:17:13.432 |
|  3 | 2023-03-22 | 2023-03-22 16:18:28.011 | 2023-03-22 16:18:28.110 | 2023-03-22 16:18:28.110 |
+----+------------+-------------------------+-------------------------+-------------------------+
```

## この問題についてのIssue

次のIssueで議論されていますが、現時点（2023-03-22）では解決されていません。

[Improve Timezone Support for Existing MySQL Databases configured with a Non-UTC Timezone · Issue #5051 · prisma/prisma](https://github.com/prisma/prisma/issues/5051#issuecomment-1279790199)

## そもそもデータベースにJSTで保存するべきなのか？

:::details MySQLの日付型のおさらい

MySQLには日時を表す型が、DATETIME型とTIMESTAMP型の2つがあります。

DATETIME型は、日時を表す文字列として保存され、タイムゾーンによって取得結果が変わることはありません。

TIMESTAMP型は、保存するときにUTCに変換され、取得するときに現在のタイムゾーンに変換されます。しかし、TIMESTAMP型には2038年問題があります。

[MySQL :: MySQL 8.0 リファレンスマニュアル :: 11.2.2 DATE、DATETIME、および TIMESTAMP 型](https://dev.mysql.com/doc/refman/8.0/ja/datetime.html#:~:text=MySQL%20%E3%81%AF%E3%80%81%20DATE%20%E5%80%A4%E3%82%92,%E5%80%A4%E3%81%AB%E4%BD%BF%E7%94%A8%E3%81%95%E3%82%8C%E3%81%BE%E3%81%99%E3%80%82)

:::

MySQLのDATETIME型はタイムゾーンの情報を持ちません（`"2023-03-22 15:14:30"`のような文字列です）。そのため、1つのタイムゾーンに揃えて保存する必要があります。

アプリケーションが複数のタイムゾーンを使う場合は、データベースにはUTCで保存するのが無難だと思います[^1]。

[^1]: 揃っていればなんでもいい気はします。

アプリケーションが日本だけで使われる場合は、UTCかJSTで保存すると思います。JSTで保存する場合には、次のメリットがあります。

- データベースの日時を直接確認するときに読みやすい
- SQLを直接書いて検索する場合に、タイムゾーンをずらさなくて良い（`where createdAt > "2023-01-01"`のように書ける）

他には、既存のデータベースがJSTで保存されている場合があります。特に、MySQLのタイムゾーンをUTC以外にした場合は、`CURRENT_TIMESTAMP`などで自動挿入される値がそのタイムゾーンの値になります。

以上のことを考えると、**データベースにJSTで保存する強い理由はなく、Prismaのサポート状況を考えるとUTCで保存するのが無難**だという結論になりました。この場合、MySQLのタイムゾーンもUTCにします。

## 他のライブラリはどうなっているのか？

MySQLとデータをやりとりするライブラリには、「DATETIME型のカラムにどのタイムゾーンで保存するのか（取得するときにどのタイムゾーンとして解釈するのか）」という設定が必要です[^2]。先述の通り、MySQLのDATETIME型はタイムゾーンの情報をもたないからです。

[^2]: 文字列でやりとりする場合は不要です。

例えば、mysql2には`timezone`オプションがあり、この値によってDATETIME型のカラムへの書き込み・読み取りの結果を変えられます。

```ts
import mysql from "mysql2";

const connection = await mysql.createConnection({
  host: "localhost",
  user: "root",
  database: "dev",
  password: "secret",
  timezone: "+09:00", // JSTで書き込み・読み取りが行われる
});
```

## まとめ

MySQLに日時を保存するとき、UTC以外のタイムゾーンで保存する強い理由は見つかりませんでした。そのため、現在のPrismaのサポート状況を考えると、UTCで保存するのが無難だといえそうです。

既存のデータがJSTで保存されている、どうしてもJSTで保存したい場合などの理由でJSTで保存する場合は、次のようにします。

- ミドルウェアで日時を調整する
- `default(now())`や`@updatedAt`を使うとUTCで保存されてしまうので、代わりに`@dbgenerated("NOW()")`や`@dbgenerated("NOW() ON UPDATE NOW()")`を使う

一般的に、MySQLとデータをやりとりするライブラリには「日時をどのタイムゾーンで保存するのか・どのタイムゾーンとして解釈するのか」というオプションが必要です。しかし、Prismaにはこのオプションがありません。

いろいろなDBMSを扱っている都合上大変だとは思いますが、Prismaにtimezoneオプションが追加されると嬉しいなと思います。

## 参考

- [Improve Timezone Support for Existing MySQL Databases configured with a Non-UTC Timezone · Issue #5051 · prisma/prisma](https://github.com/prisma/prisma/issues/5051#issuecomment-1279790199)
- [MySQL :: MySQL 8.0 リファレンスマニュアル :: 5.1.15 MySQL Server でのタイムゾーンのサポート](https://dev.mysql.com/doc/refman/8.0/ja/time-zone-support.html)
