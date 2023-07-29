---
title: "個人開発アプリをRemix + Cloudflare D1に移行してみた"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["remix", "cloudflare", "d1"]
published: false
publication_name: "bs_kansai"
---

以前、Type Challenges Judgeという、type-challengesのオンラインジャッジを作りました。

https://zenn.dev/gibjapan/articles/23bebcd855dc86

このアプリをRemix + Cloudflare（Pages、D1）に移行したので、やったことについて書こうと思います。

## 技術スタックについて

Type Challenges Judgeは、React + React Router + Firebase（Authentication、Firestore、Hosting、Functions）という技術スタックで開発しました。

今回はこのアプリをRemix + Cloudflare（Pages、D1、Workers）でリニューアルしてみました。なぜこの構成にしたかというと、RemixやCloudflare（特にD1）を使ってみたかったというのが一番の理由です。

Type Challenges Judgeはアプリの規模があまり大きくなく、新しい技術を試すのにちょうどいい規模でした。また、ルーティングにReact Routerを使っているため、Remixへの移行のハードルも小さいと予想できました。

## 移行の手順

次の2つの手順で行いました。

- React RouterからRemixに移行して、Cloudflare Pagesにデプロイする
- FirestoreからCloudflare D1に移行する
- (Cloud FunctionsからCloudflare Workersに移行する)←未定

D1にアクセスするためには、Remixのloader/actionを使う必要がありそうだったので、先にRemixに移行して、それからデータベースを変更することにしました。

## React RouterからRemixに移行する

公式のマイグレーションガイドを参考にして移行しました。

[Migrating from React Router | Remix](https://remix.run/docs/en/main/guides/migrating-react-router-app)

Cloudflareにデプロイする場合は、必要なパッケージやファイルの中身に違う部分があります。そのため、create-remixでCloudflare Pagesで作ったプロジェクトを適宜参照しました。

途中でつまることもありましたが、移行の差分自体は小さかったです。

https://github.com/tekihei2317/type-challenges-judge/pull/16

### 既存のプロジェクトにRemixを追加して動かす

まずは必要なパッケージをインストールしました。

```bash
yarn add @remix-run/react @remix-run/cloudflare @remix-run/cloudflare-pages
yarn add -D @remix-run/dev wrangler cloudflare/workers-types
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

:::details Remixのルーティングの規約について

Remixのルーティング(v2)の特長は、URLの区切りをディレクトリではなく`.`で表現することと、ファイル名でレイアウトが決まることです。具体的には次のようになります。

| URL           | ファイル名                       | レイアウト                                |
| ------------- | -------------------------------- | ----------------------------------------- |
| `/`           | `_index.tsx`                     |                                           |
| `/articles`   | `articles.tsx`                   |                                           |
| `/articles`   | `articles._index.tsx`            | `articles.tsx`                            |
| `/articles/1` | `articles.$articleId.tsx`        | `articles.tsx`                            |
| `/articles/1` | `articles.$articleId._index.tsx` | `articles.$articleId.tsx`、`articles.tsx` |


あるルートファイルのレイアウトは、そのファイルのプレフィックスとなるようなルートファイルです。具体的には、`A.B.C.tsx`のレイアウトは、`A.B.tsx`と`A.tsx`です（それらが存在すれば）。

挙動としては、`A.B.C.tsx`が`A.B.tsx`の`<Outlet />`に挿入され、`A.B.tsx`が`A.tsx`の`<Outlet />`に挿入されます。

:::

ページコンポーネントのファイル名を変更したあと、動かすために主に次の2つの変更が必要でした。

- コンポーネントをデフォルトエクスポートに変更する
- `react-router-dom`を`@remix-run/react`に変更する

これで、ローカルで一通り動作することを確認できました。

### Cloudflare Pagesにデプロイする

Cloudflare Pagesへのデプロイは、CloudflareのコンソールのWorkers & Pagesからアプリを作成して、GitHubのリポジトリと連携するとできました。

[Deploy a Remix site · Cloudflare Pages docs](https://developers.cloudflare.com/pages/framework-guides/deploy-a-remix-site/)

## FirestoreからCloudflare D1に移行する

WIP

## 参考

- [Migrating from React Router | Remix](https://remix.run/docs/en/main/guides/migrating-react-router-app)
- [Route File Naming (v2) | Remix](https://remix.run/docs/en/main/file-conventions/route-files-v2)
- [Deploy a Remix site · Cloudflare Pages docs](https://developers.cloudflare.com/pages/framework-guides/deploy-a-remix-site/)

## メモ

- 移行の手順
  - React Router → Remix
  - Firestore → Cloudflare D1
- つまったところ
  - Firebaseにはクライアント側からアクセスする必要がある
  - root.tsxに<Script />をつける
- 機能を追加する（ランキング機能）
- その他
  - データベースアクセスのライブラリについて
