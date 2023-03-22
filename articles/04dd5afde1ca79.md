---
title: "Prisma+MySQLで日時をJSTで保存する方法"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["prisma", "mysql", "nodejs", "typescript"]
published: false
publication_name: gibjapan
---

Prismaを使うと、日時がUTCで保存されてしまいます。そのため、日時をJSTで保存する方法を調べました。

## 状況

Prismaのスキーマが次のような場合を考えます。

```ts
```

以下のコードでデータを登録すると、日時がUTCで保存されてしまいます。

```ts
await prisma.task.create({
  dueDate: new Date("2023-03-23"),
  startAt: new Date(),
})
```

データベース

## 解決方法

Prismaは保存するときにUTCに変換し、取得するときはUTCとして解釈します。JSTとして保存するためには、登録するときに+9hし、取得するときに-9hすればよいです。

具体的には、次のようにミドルウェアを使います。

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

しかし、タイムゾーンを変更しても`default(now())`や`@updatedAt`を設定したカラムは変化しませんでした。

```
```

ミドルウェアを経由していないかつ、Prismaが明示的に値をしているしているからだと考えられます。そこで、`default(now())`や`@updatedAt`の代わりに次のようにします。

```ts diff
model Task {
-  createdAt DateTime @default(now())
-  updatedAt DateTime @updatedAt
+  createdAt DateTime @default(dbgenerated("NOW(3)"))
+  updatedAt DateTime @default(dbgenerated("NOW(3) ON UPDATE NOW(3)"))
}
```

これで、登録日時や更新日時もJSTで保存できるようになりました。

## この問題についてのIssue

次のIssueで議論されていますが、現時点（2023-03-22）では解決されていません。

[Improve Timezone Support for Existing MySQL Databases configured with a Non-UTC Timezone · Issue #5051 · prisma/prisma](https://github.com/prisma/prisma/issues/5051#issuecomment-1279790199)

## そもそもデータベースにJSTで保存するべきなのか？

:::details MySQLの日付型のおさらい

MySQLには日時を表す型が、DATETIME型とTIMESTAMP型の2つがあります。

DATETIME型は、日時を表す文字列として保存され、タイムゾーンによって取得結果が変わることはありません。

TIMESTAMP型は、保存するときにUTCに変換され、取得するときに現在のタイムゾーンに変換されます。しかし、TIMESTAMP型には2038年問題があります。

:::
MySQLのDATETIME型はタイムゾーンの情報を持ちません（`"2023-03-22 15:14:30"`のような文字列です）。そのため、1つのタイムゾーンに揃えて保存する必要があります。

アプリケーションが複数のタイムゾーンを使う場合は、データベースにはUTCで保存するのが無難だと思います[^1]。

[^1]: 揃っていればなんでもいい気はします

アプリケーションが日本だけで使われる場合は、UTCかJSTで保存すると思います。JSTで保存する場合には、次のメリットがあります。

- データベースの日時を確認する場合に読みやすい
- SQLを直接書いて検索する場合に、タイムゾーンをずらさなくて良い（`where createdAt > "2023-01-01"`のように書ける）

他には、既存のデータベースがJSTで保存されている場合があります。特に、MySQLのタイムゾーンをUTC以外にした場合は、`CURRENT_TIMESTAMP`などで自動挿入される値がそのタイムゾーンの値になります。

以上のことを考えると、**データベースにJSTで保存する強い理由はなく、Prismaのサポート状況を考えるとUTCで保存するのが無難**だという結論になりました。

## Prismaはどうなっているべきなのか？

MySQLとデータをやりとりするライブラリには、「DATETIME型のカラムにどのタイムゾーンで保存するのか（取得するときにどのタイムゾーンとして解釈するのか）」という設定が必要です[^2]。先述の通り、MySQLのDATETIME型はタイムゾーンの情報をもたないからです。

[^2]: 文字列でやりとりする場合は不要です

例えば、mysql2には`timezone`オプションがあり、この値によってDATETIME型のカラムへの書き込み・読み取りの結果を変えられます。

```ts
```

## まとめ

MySQLに日時を保存するとき、UTC以外のタイムゾーンで保存する強い理由は見つかりませんでした。そのため、現在のPrismaのサポート状況を考えると、UTCで保存するのが無難です。

既存のデータがJSTで保存されている、どうしてもJSTで保存したい場合などの理由でJSTに保存する場合は、

- ミドルウェアで日時を調整する
- `default(now())`や`@updatedAt`を使うとUTCで保存されてしまうので、代わりに`@dbgenerated("NOW()")`や`@dbgenerated("NOW() ON UPDATE NOW()")`を使う

の2つの対応が必要です。

一般的な話をすると、MySQLとデータをやりとりするライブラリには「どのタイムゾーンで保存するのか・どのタイムゾーンとして解釈するのか」というオプションが必要です。しかし、Prismaにはこのオプションがありません。

いろいろなDBMSを扱っている都合大変だとは思いますが、Prismaにtimezoneオプションが追加されるのを待っています。

## 参考

- [MySQL :: MySQL 8.0 リファレンスマニュアル :: 5.1.15 MySQL Server でのタイムゾーンのサポート](https://dev.mysql.com/doc/refman/8.0/ja/time-zone-support.html)
