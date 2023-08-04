---
title: "個人開発アプリをRemix + Cloudflare D1に移行してみた"
emoji: "⚡"
type: "tech"
topics: ["remix", "cloudflare", "d1", "sqlc", "blessingsoftware夏のブログリレー"]
published: false
publication_name: "bs_kansai"
---

この記事は『blessing software 夏のブログリレー企画』の5日目の記事です。

昨日はasukaさん([@a_skua](https://twitter.com/a_skua))の「[Flutterを用いたWeb開発の今後について考える](https://zenn.dev/bs_kansai/articles/e5d7ad28baab59)」が公開されました。

次回はKanonさん([@samurai_se](https://twitter.com/samurai_se))の「ブログリレーを主催してみた結果」が公開される予定です！お楽しみに！

## はじめに

以前、Type Challenges Judgeという、type-challengesのオンラインジャッジを作りました。

https://github.com/tekihei2317/type-challenges-judge

Type Challenges Judgeは、type-challengesの問題の回答の正誤判定を行ったり、自分がどれくらい正解したかや、他の人の回答が確認できるアプリです。


このアプリをRemix + Cloudflare（Pages、D1）に移行してみた[^1]ので、やったことについて書こうと思います。

[^1]: まだ判定の処理の移行が完了していません。

## 技術スタックについて

Type Challenges Judgeは、React + React Router + Firebase（Authentication、Firestore、Hosting、Functions）という技術スタックで開発しました。

今回はこのアプリをRemix + Cloudflare（Pages、D1、Workers）でリニューアルしてみました。なぜこの構成にしたかというと、単純にRemixやCloudflare（特にD1）を使ってみたかったというのが一番の理由です。

Type Challenges Judgeはアプリの規模があまり大きくなく、新しい技術を試すのにちょうどいい規模でした。また、ルーティングにReact Routerを使っているため、Remixへの移行のハードルも小さいと予想できました。

## 移行の手順

次の3つの手順で行いました。

- React RouterからRemixに移行して、Cloudflare Pagesにデプロイする
- FirestoreからCloudflare D1に移行する
- （進行中）判定処理をCloud FunctionsからCloudflare Workersに移行する

D1にアクセスするためには、Remixのloader/actionを使う必要がありそうだったので、Remixへの移行を先に行うことにしました。

## React RouterからRemixに移行する

公式のマイグレーションガイドを参考にして移行しました。途中でつまることもありましたが、移行の差分自体は小さく1日程度で完了しました。

[Migrating from React Router | Remix](https://remix.run/docs/en/main/guides/migrating-react-router-app)

https://github.com/tekihei2317/type-challenges-judge/pull/16

Cloudflareにデプロイする場合は、必要なパッケージやファイルの中身にマイグレーションガイドと違いがあります。そのため、create-remixでCloudflare Pagesを選んで作ったプロジェクトも参考にしました。

### 既存のプロジェクトにRemixを追加して動かす

まずは必要なパッケージをインストールしました。

```bash
yarn add @remix-run/react @remix-run/cloudflare @remix-run/cloudflare-pages
yarn add -D @remix-run/dev wrangler @cloudflare/workers-types
```

それから、Remixを動かすのに必要なファイルを追加しました。ファイルの中身は、create-remixで作ったプロジェクトからコピーしました。詳細はPull Requestを参照していただければと思います。

- `server.ts`
- `app/entry.client.tsx`
- `app/entry.server.tsx`
- `app/root.tsx`
- `remix.config.js`

マイグレーションガイドでは`root.tsx`に`<Script />`が書かれていないのですが、これがないとReactが動かないので注意が必要です。

そして、package.jsonのスクリプトを変更して、`yarn run dev`でRemixが動作することを確認しました。

```json diff:package.json
{
-    "dev": "vite",
-    "build": "tsc && vite build",
+    "dev": "remix dev --manual -c \"npm run start\"",
+    "start": "wrangler pages dev --compatibility-date=2023-06-21 ./public",
+    "build": "remix build",
}
```

### 既存のコードを移植する

次に、既存のコードがRemixで動くようにしました。Remixのコードは`app/`ディレクトリに入れるので、まずは`src/`ディレクトリの中身を`app/`ディレクトリに移動しました。

それから、ページコンポーネントのファイル名をRemixの規約に合わせて変更しました。Remixは、Next.jsなどのフレームワークと同じように、ファイルシステムベースのルーティングを採用しています。URLの階層を、ディレクトリではなくファイル名を`.`で区切って表すのが特徴です。

[Route File Naming (v2) | Remix](https://remix.run/docs/en/main/file-conventions/route-files-v2)

その後、主に次の2つの変更をするとローカル環境でアプリケーションが動作するようになりました。

- コンポーネントをデフォルトエクスポートに変更する
- `react-router-dom`を`@remix-run/react`に変更する

### Cloudflare Pagesにデプロイする

Cloudflare Pagesへのデプロイは、CloudflareのコンソールのWorkers & Pagesからアプリを作成して、GitHubのリポジトリと連携するとできました。

[Deploy a Remix site · Cloudflare Pages docs](https://developers.cloudflare.com/pages/framework-guides/deploy-a-remix-site/)

## FirestoreからCloudflare D1に移行する

移行のPull Requestはこちらです。

https://github.com/tekihei2317/type-challenges-judge/pull/17

### データモデルの設計

まず、既存のデータモデルがどうなっているかを把握してから、SQLite用のデータモデルを作成しました。

![](/images/image.png)

Type Challenges Judgeには、主にユーザー・問題・提出という3つのエンティティがあります。問題一覧で解いた問題の色を変えたり、難易度ごとにどれくらい解いたかを取得したいため、問題の挑戦結果（正解したかどうか）も保存するようにしています。

### シードの作成

次に、初期データを登録するスクリプトを書きました。少し詰まったのは、スクリプトからD1へアクセスする方法です。D1にはWorkerからアクセスする必要があるので、スクリプトからWorkerを起動する必要がありました。

[cloudflare/wildebeest](https://github.com/cloudflare/wildebeest/tree/main)を見てみると、`wrangler`の`unstable_dev`を使っていたので、その方法でやってみると出来ました。

https://github.com/cloudflare/wildebeest/blob/20efb7f0eb504462be869b91102307991d991c2f/frontend/mock-db/run.mjs#L15-L28

### データベースアクセスを書き換える

Firestoreからデータを取得する処理を、D1から取得する処理に書き換えました。また、データを取得する場所をRemixのローダーに変更しました。

データベースアクセスには、sqlc（sqlc-gen-ts-d1）を使いました。

- https://github.com/sqlc-dev/sqlc
- https://github.com/orisano/sqlc-gen-ts-d1

sqlcは、テーブル定義のDDLとSQLから、データベースアクセス用のコードや型を生成するライブラリです。sqlc-gen-ts-d1は、sqlcのCloudflare D1用のプラグインです。導入は次の記事を参考にしました。

https://zenn.dev/voluntas/scraps/a032898a9a80ee

普段はPrismaを使うことが多いのですが、Cloudflare Workersで動かすことはできない[^2]ようなので、sqlcを使ってみることにしました。

[^2]: https://twitter.com/ogawa0071/status/1658788985307807746

### sqlcではカラムをスネークケースで定義する必要がある

sqlcを使う場合の注意点は、データベースのカラムをスネークケースで定義する必要があることです。なぜかというと、クエリの実行結果は定義したカラム名で返ってくるのに対して、sqlc側はカラムを小文字に統一して扱うためです。

https://github.com/orisano/sqlc-gen-ts-d1/issues/1

具体的には、`select * from user`の実際の結果の型が`{ userId: string }[]`だとしても、sqlc側でそれに合わせる方法がないということです。

スネークケースからキャメルケースへの変換`sqlc-gen-ts-d1`で行ってくれるため、アプリケーション側では通常通りキャメルケースで扱えます。

### データの移行

最後に、FirestoreのデータをD1に移行しました。次の手順で行いました。

- `firebase-admin`でFirestoreからデータを取得して、JSONファイルに書き出す
- シードと同じ方法でローカルのD1に書き込む
- ローカルのDBをダンプして、`wrangler d1 execute`で本番DBに書き込む


データの件数は少なかったため、問題なく移行できました。

```text
exported 34 users # ユーザー
exported 343 submissions # 提出
exported 232 problem results # 挑戦結果
```

## これからすること

- 判定処理がまだ移行できていないので移行する（Cloudflare Workersでtscを動かすのに詰まっている）
- ランキング機能があると良さそうなので、実装する
- テストを書いてみる

あたりをやる予定です。

## 感想

Remix + Cloudflare D1は、可能性を感じる技術スタックでした。規模が小さい場合はリレーショナルデータベースを無料で運用できるので、個人開発に向いていそうです。

Node.jsのライブラリで動かないものがあることと、現時点ではD1のトランザクションがサポートされていない点は注意が必要です。

sqlcは、SQLを直接書くことがパフォーマンスに自覚的になることにつながって良さそうでした。SQLを書いてコード生成をする手間があるので、複雑なRead以外は[Kysely](https://github.com/kysely-org/kysely)などを使ってみるのもいいのかなと思いました。

## 参考

- [Migrating from React Router | Remix](https://remix.run/docs/en/main/guides/migrating-react-router-app)
- [Route File Naming (v2) | Remix](https://remix.run/docs/en/main/file-conventions/route-files-v2)
- [Deploy a Remix site · Cloudflare Pages docs](https://developers.cloudflare.com/pages/framework-guides/deploy-a-remix-site/)
