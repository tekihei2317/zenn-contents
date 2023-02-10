---
title: "ReactでtRPCのミドルウェアっぽいものを作る"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "trpc", "typescript"]
published: true
publication_name: "gibjapan"
---

ReactとtRPCを使ってアプリを開発しています。サーバーサイドの認可（の一部）でtRPCのミドルウェアを使っており、フロントエンドでも同じようにできる気がしたのでやってみました。

## tRPCのミドルウェアとは

tRPC（@trpc/server）のミドルウェアは、プロシージャが実行される前に処理を実行するためのもののことです。tRPCのプロシージャは、MVCでいうとシングルアクションのコントローラーです。ミドルウェアは、認可やロギングなどに使われます。

[Middlewares | tRPC](https://trpc.io/docs/middlewares)

tRPCのミドルウェアの特徴は、次のミドルウェアに値（コンテキスト）を渡せることです。コンテキストはミドルウェアを通っていき、最終的にプロシージャに渡されます。型の情報が次のミドルウェアやプロシージャに引き継がれるので扱いやすいです。

```ts
// サンプルより引用
const t = initTRPC.context<Context>().create();
export const middleware = t.middleware;
export const publicProcedure = t.procedure;

const isAuthed = middleware(({ ctx, next }) => {
  // ctx.user は User | undefined
  if (!ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }

  return next({
    ctx: {
      // ctx.user は User
      user: ctx.user,
    },
  });
});

const protectedProcedure = publicProcedure.use(isAuthed);
// ctx.user は User
protectedProcedure.query(({ ctx }) => ctx.user);
```

## Reactでもやってみる

Reactでコンポーネントの表示の前に何かを実行するためには、例えばpropsやchildrenにコンポーネントを渡す方法があります。しかし、このやり方でミドルウェアからコンポーネントに値を渡すためには、render propをする必要があります。

そうすると、複数のミドルウェアを使うと可読性が落ちたり、型を書くのが面倒だったりします。

```tsx
// propsやchildrenにコンポーネントを渡す
<ProtectedRoute page={Profile} />
<RequireAuth><Profile /></RequireAuth>

// 値を渡すためにはrender propで書く必要がある
<ProtectedRoute component={(user) => <Profile user={user} />} />
```

そのため、ReactでもtRPC風にミドルウェアを書けるライブラリを作ってみました。以下のように使えます。

```tsx:src/middleware.tsx
import { Navigate } from "react-router-dom";
import { useAuthContext } from "./authentication";
import { initMiddleware } from "./lib/react-middleware";

// ミドルウェアに渡すデータ（コンテキスト）を取得するカスタムフック
export function useMiddlewareContext() {
  // この例では、Reactのコンテキストからデータを取得している
  const authContext = useAuthContext();

  return { userState: authContext.userState };
}

type Context = ReturnType<typeof useMiddlewareContext>;

const { MiddlewareComponent, createMiddleware } = initMiddleware<Context>();

const Loading = () => <div>ローディング中...</div>;
const Unauthorized = () => <div>権限がありません</div>;

// ミドルウェアを作成する
const ensureLoggedIn = createMiddleware(({ ctx, next }) => {
  if (ctx.userState.isLoading) return <Loading />;
  if (!ctx.userState.user) return <Navigate to="/login" replace />;

  return next({ user: ctx.userState.user });
});

// ミドルウェアコンポーネント（tRPCのprocedureに対応）を作成する
export const AuthMiddleware = MiddlewareComponent.use(ensureLoggedIn);
export const AdminMiddleware = AuthMiddleware.use(({ ctx, next }) => {
  if (ctx.user.type !== "admin") return <Unauthorized />;

  return next({ user: ctx.user });
});
```

```tsx:src/pages/Admin.tsx
import { AdminMiddleware, useMiddlewareContext } from "../middleware";
import { InferHandlerProps } from "../lib/react-middleware";

export const AdminContents = ({ ctx }: InferHandlerProps<typeof AdminMiddleware>) => {
  return (
    <div>
      <p>管理者ページ</p>
      <p>{ctx.user.userName}</p>
    </div>
  );
};

export const Admin = () => {
  return <AdminMiddleware ctx={useMiddlewareContext()} component={AdminContents} />;
};
```

## ライブラリの実装の詳細

まず、用語を整理します。

| 用語                       | 定義                                                                                             |
| -------------------------- | ------------------------------------------------------------------------------------------------ |
| コンテキスト               | ミドルウェアを通っていくデータ                                                                   |
| ミドルウェア               | コンポーネントを表示するか、次のミドルウェアを実行するもの                                           |
| ミドルウェアコンポーネント | ミドルウェアを持つコンポーネント。表示されるときに、持っているミドルウェアを順番に実行していく。 |

コアの部分の実装は以下です。ミドルウェアコンポーネントは、最初に受け取るコンテキストの型（`TInputContext`）と、全てのミドルウェアを通過した後のコンテキストの型（`TOutputContext`）をもっています。`use`では、引数のミドルウェアの型をもとに`TOutputContext`を変化させています。

```ts
function createMiddlewareComponent<TInputContext, TOutputContext = TInputContext>(
  middlewares?: AnyMiddlewareFunction[]
): MiddlewareComponent<TInputContext, TOutputContext> {
  const MiddlewareComponent = ({ ctx, component }: MiddlewareComponentProps<TInputContext, TOutputContext>) => {
    let index = 0;

    const next = (ctx: unknown) => {
      const middleware = MiddlewareComponent._middlewares[index++];

      if (middleware) return middleware({ ctx, next });
      return component({ ctx: ctx as TOutputContext });
    };

    return next(ctx);
  };

  MiddlewareComponent._middlewares = middlewares ? middlewares : [];

  MiddlewareComponent.use = function <TNewContext>(fn: MiddlewareFunction<TOutputContext, TNewContext>) {
    return createMiddlewareComponent<TInputContext, TNewContext>([...MiddlewareComponent._middlewares, fn]);
  };

  return MiddlewareComponent;
}

export function initMiddleware<Context>() {
  return {
    MiddlewareComponent: createMiddlewareComponent<Context>(),
    createMiddleware: createMiddlewareFactory<Context>(),
  };
}
```

全ての実装はこちらにあります。

https://github.com/tekihei2317/react-middleware-poc/blob/main/examples/react-router/src/lib/react-middleware.tsx

## 使用例

以下のような認証・認可の仕様のアプリケーションを実装するとします。

- / 誰でも閲覧可能
- /login ログインしていないユーザーのみ閲覧可能
- /profile ログインしているユーザーが閲覧可能
- /admin 管理者のみ閲覧可能

まずは、それぞれの処理に応じたミドルウェアと、ミドルウェアコンポーネントを作成します。

```tsx:src/middleware.tsx
const ensureLoggedIn = createMiddleware(({ ctx, next }) => {
  if (ctx.userState.isLoading) return <Loading />;
  if (!ctx.userState.user) return <Navigate to="/login" replace />;

  return next({ user: ctx.userState.user });
});

const ensureNotLoggedIn = createMiddleware(({ ctx, next }) => {
  if (ctx.userState.isLoading) return <Loading />;
  if (ctx.userState.user !== undefined)
    return <Navigate to={ctx.userState.user.type == "admin" ? "/admin" : "/profile"} replace />;

  return next({ user: ctx.userState.user });
});

// ログイン中のユーザーのみアクセスできる
export const AuthMiddleware = MiddlewareComponent.use(ensureLoggedIn);

// ログイン中のユーザーのアクセスを禁止する
export const RestrictAuthenticatedUser = MiddlewareComponent.use(ensureNotLoggedIn);

// 管理者のみアクセスできる
export const AdminMiddleware = AuthMiddleware.use(({ ctx, next }) => {
  if (ctx.user.type !== "admin") return <Unauthorized />;

  return next({ user: ctx.user });
});
```

次に、各ページを作成してルーティングに登録します。ルーティングライブラリはReact Routerを使っています。

```tsx:src/router.tsx
import { createBrowserRouter, RouterProvider } from "react-router-dom";
import { AppLayout } from "./AppLayout";
import { Admin } from "./pages/Admin";
import { Login } from "./pages/Login";
import { Profile } from "./pages/Profile";
import { Top } from "./pages/Top";

const router = createBrowserRouter([
  {
    path: "",
    element: (
      <AppLayout>
        <Top />
      </AppLayout>
    ),
  },
  {
    path: "/login",
    element: (
      <AppLayout>
        <Login />
      </AppLayout>
    ),
  },
  {
    path: "/profile",
    element: (
      <AppLayout>
        <Profile />
      </AppLayout>
    ),
  },
  {
    path: "/admin",
    element: (
      <AppLayout>
        <Admin />
      </AppLayout>
    ),
  },
]);

export const AppRoutes = () => <RouterProvider router={router} />;
```

最後に、ページにミドルウェアを適用します。`/admin`ページの例です。

```tsx:src/pages/Admin.tsx
import { AdminMiddleware, useMiddlewareContext } from "../middleware";
import { InferHandlerProps } from "../lib/react-middleware";

export const AdminContents = ({ ctx }: InferHandlerProps<typeof AdminMiddleware>) => {
  return (
    <div>
      <p>管理者ページ</p>
      <p>{ctx.user.userName}</p>
    </div>
  );
};

export const Admin = () => {
  return <AdminMiddleware ctx={useMiddlewareContext()} component={AdminContents} />;
};
```

これで動くようになりました。しかし、ページ側にコンポーネントを2つ書くのが少し冗長なので、ユーティリティ作って解消します。

```ts:src/middleware.tsx
import { initMiddleware, MiddlewareComponent as IMiddlewareComponent } from "./lib/react-middleware";

// ミドルウェアコンポーネントの出力と、ページコンポーネントの入力が合うようにしている
interface PageComponent<TContext> {
  ({ ctx }: { ctx: TContext }): JSX.Element;
  Middleware: IMiddlewareComponent<Context, TContext>;
}

export function RenderPage<TContext>({ page: Page }: { page: PageComponent<TContext> }) {
  return <Page.Middleware ctx={useMiddlewareContext()} component={Page} />;
}
```

これでページ側をシンプルに書けるようになりました。

```tsx:src/pages/Admin.tsx
import { AdminMiddleware } from "../middleware";
import { InferHandlerProps } from "../lib/react-middleware";

export const Admin = ({ ctx }: InferHandlerProps<typeof AdminMiddleware>) => {
  return (
    <div>
      <p>管理者ページ</p>
      <p>{ctx.user.userName}</p>
    </div>
  );
};

Admin.Middleware = AdminMiddleware;
```

```tsx:src/router.tsx
import { RenderPage } from "./middleware";

const router = createBrowserRouter([
  {
    path: "/admin",
    element: (
      <AppLayout>
        <RenderPage page={Admin} />
      </AppLayout>
    ),
  },
]);
```

他のページにも同様にミドルウェアを設定し、ルーティングを修正します。完成版のコードはこちらにあります。

https://github.com/tekihei2317/react-middleware-poc/tree/main/examples/react-router

## 感想

tRPCのミドルウェアの型のつけ方が気になっていたので、理解できてよかったです。TypeScriptの勉強にもなりました。

実用性は、静的に型を解決することにメリットがある場合はありそうです。例えば、ユーザーにロールが複数ある場合は、型を絞り込めなさそうなので使いにくいかなと思いました。

## 参考

- [Middlewares | tRPC](https://trpc.io/docs/middlewares)
- [【Next.js】アクセスコントロールパターン](https://zenn.dev/aiji42/articles/450ce962cc225a)
- [react-router/examples/auth at main · remix-run/react-router](https://github.com/remix-run/react-router/tree/main/examples/auth)
