---
title: "個人開発アプリをRemix + Cloudflare D1に移行してみた"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["remix", "cloudflare", "d1", "sqlc"]
published: false
publication_name: "bs_kansai"
---

以前、Type Challenges Judgeという、type-challengesのオンラインジャッジを作りました。

type-challengesの問題の回答を提出すると正誤判定が行われ、自分がどれくらい正解したかや、他の人の回答が確認できるアプリです。

https://github.com/tekihei2317/type-challenges-judge


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

公式のマイグレーションガイドを参考にして移行しました。途中でつまることもありましたが、移行の差分自体は小さかったため1日程度でできました。

[Migrating from React Router | Remix](https://remix.run/docs/en/main/guides/migrating-react-router-app)

https://github.com/tekihei2317/type-challenges-judge/pull/16

Cloudflareにデプロイする場合は、必要なパッケージやファイルの中身に違う部分があります。そのため、create-remixでCloudflare Pagesで作ったプロジェクトを適宜参照しました。

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

マイグレーションガイドでは`root.tsx`に`<Script />`が書かれていないのですが、これがないとReactが動かないので注意が必要です（1時間くらい溶かしました）。

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

次に、既存のコードをRemixで動かすようにしました。Remixのコードは`app/`ディレクトリに入れるので、まずは`src/`ディレクトリの中身を`app/`ディレクトリに移動しました。

それから、ページコンポーネントのファイル名をRemixの規約に合わせて変更しました。Remixでは、Next.jsなどのフレームワークと同じようにファイルシステムベースのルーティングを採用しています。

[Route File Naming (v2) | Remix](https://remix.run/docs/en/main/file-conventions/route-files-v2)

その後、主に次の2つの変更をするとローカル環境で動作するようになりました。

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

Type Challenges Judgeには、主にユーザー・問題・提出という3つのエンティティがあります。ある問題を解いたかや、難易度ごとにどれくらい解いたかを取得したいため、問題の挑戦結果（正解したかどうか）も保存するようにしています。

### シードの作成

次に、初期データを登録するスクリプトを書きました。少し詰まったのは、D1にはWorkerからアクセスできないため、スクリプトからD1にどのようにアクセスするかです。

[cloudflare/wildebeest](https://github.com/cloudflare/wildebeest/tree/main)のコードを見てみると、Workerを`unstable_dev`でスクリプトから起動していたので、その方法でやってみました。

https://github.com/cloudflare/wildebeest/blob/20efb7f0eb504462be869b91102307991d991c2f/frontend/mock-db/run.mjs#L15-L28

### データベースアクセスを書き換える

Firestoreからデータを取得する処理を、D1から取得する処理に書き換えました。データベースアクセスには、sqlc（sqlc-gen-ts-d1）を使ってみました。

- https://github.com/sqlc-dev/sqlc
- https://github.com/orisano/sqlc-gen-ts-d1

sqlcは、テーブル定義のDDLとSQLから、データベースアクセス用のコードや型を生成するライブラリです。sqlc-gen-ts-d1は、Cloudflare D1用のプラグインです。

普段はPrismaを使うことが多いのですが、Cloudflare Workersで動かすことはできない[^2]ようなので、sqlcを使ってみることにしました。

[^2]: https://twitter.com/ogawa0071/status/1658788985307807746

### 認証について

### データの移行

最後に、FirestoreのデータをD1に移行しました。次の手順で行いました。

- `firebase-admin`でFirestoreからデータを取得して、JSONファイルに書き出す
- シードと同じ方法でローカルのD1に書き込む
- ローカルのDBのダンプを、`wrangler d1 execute`で本番DBに書き込む


データの件数は少なかったため、問題なく移行できました。

```bash
exported 34 users # ユーザー
exported 343 submissions # 提出
exported 232 problem results # 挑戦結果
```

## 感想

## 参考

- [Migrating from React Router | Remix](https://remix.run/docs/en/main/guides/migrating-react-router-app)
- [Route File Naming (v2) | Remix](https://remix.run/docs/en/main/file-conventions/route-files-v2)
- [Deploy a Remix site · Cloudflare Pages docs](https://developers.cloudflare.com/pages/framework-guides/deploy-a-remix-site/)

## メモ

- まだ書いていない
  - カラムをスネークケースにする必要がある
  - 認証で詰まった
- 感想
  - Remix レンダリング・データ取得の方法が決まっていて考えることが少ない
  - D1 db.batch トランザクションが欲しい db.transaction((tx) => {})
  - sqlc SQLを直接書くことはパフォーマンスに自覚的になることにつながって良さそうだなと思った。SQLを書いてコード生成する手間はあるので、複雑なRead以外はKyselyで書くのもいいかもしれないと思いました。
- これから
  - 判定処理がまだ移行できていないので移行する（tscをCloudflare Workersで動かすのに手こずっている）
  - ランキング機能があるとユーザーが増えそうだと思ったので、実装する
