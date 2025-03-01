---
title: "react-router(v7) x remix-auth(v4)でOAuth2認証する"
emoji: "🕸️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["reactrouter", "remix", "OAuth2"]
published: true
---

:::message
リフレッシュトークンやトークンの取り消しについての説明はしていません。🙇
:::

# はじめに

最近 remix-auth を使おうと思って調査していたところ、v3 から v4 へのアップデートにより使用方法が大きく変更されていることを知りました。インターネット上の記事やチュートリアルの多くが v3 向けに書かれているため、その情報をもとに実装しようとしてもうまく機能しません。

このような経験をされた方のために、v4 での実装方法についてまとめました。

:::message alert
OAuth2は本来認可の仕組みであり、アクセストークンを発行できるかどうかで認証を行うため、利用者側は怖い。今回は社内アプリケーションであり、他の要件も考慮した上でこの方法を採用しましたが、本番環境で使用する際はご注意ください。
:::

# 使用したライブラリ

- [remix-auth](https://github.com/sergiodxa/remix-auth)
- [remix-auth-oauth2](https://github.com/sergiodxa/remix-auth-oauth2)

# 実装方法
React RouterにおけるSessionとCookieの管理は、[公式ドキュメント](https://reactrouter.com/explanation/sessions-and-cookies)を確認してください。

## 1. authenticator の設定

以下のように `authenticator` を設定します：

```typescript:auth.server.ts
import { Authenticator } from "remix-auth";
import { OAuth2Strategy } from "remix-auth-oauth2";
import { commitSession, getSession } from "./session.ts";

// 取得するユーザー情報の型(適宜修正してください。)
type User = {
  id: number;
  name: string;
  roleType: number;
  mailAddress: string;
  userId?: string;
};

// セッションに保存するユーザー情報の型
export type SessionUser = User & {
  accessToken: string;
  refreshToken: string;
};

// 認証を行うインスタンスを作成
export const authenticator = new Authenticator<SessionUser>();

// Strategyの登録
authenticator.use(
  // OAuth2を利用する
  new OAuth2Strategy(
    {
      authorizationEndpoint: "<OAUTH_END_POINT>",
      tokenEndpoint: "<TOKEN_END_POIND>",
      clientId: "your_app_client_id",
      clientSecret: "your_app_client_secret",
      redirectURI: "redirect_uri",
    },
    async ({ tokens }) => {
      // ユーザ情報を取得(適宜修正してください。)
      const userResponse = await fetch(
        "<GET_USER_END_POIND>",
        {
          headers: {
            Authorization: `Bearer ${tokens.accessToken()}`,
          },
        },
      );

      if (!userResponse.ok) {
        throw new Error("Failed to fetch user data");
      }

      const user: SessionUser = await userResponse.json();

      // トークン情報をユーザーオブジェクトに含めて返す
      return {
        ...user,
        accessToken: tokens.accessToken() || "",
        refreshToken: tokens.refreshToken() || "",
      };
    },
  ),
  // 任意ですが、複数ストラテジーを登録する場合は必要です。
  "provider_name", 
);
```

## 2. ログインページの実装

v4 からは自分で `getSession` を呼び出してセッションの状態を確認する必要があります：

```typescript:login.tsx
export const action = async ({ request }: Route.ActionArgs) => {
  try {
    // 認証
    // redirect_uriにリダイレクトします。
    return await authenticator.authenticate("provider_name", request);
  } catch (error) {
    if (error instanceof Response) {
      throw error;
    }

    logger.error(error);
    return {
      error: "認証エラーが発生しました。",
    };
  }
};

export const loader = async ({ request }: Route.LoaderArgs) => {
  const session = await getSession(request.headers.get("Cookie"));
  // セッションがある場合はトップページにリダイレクト
  if (session.has("user")) {
    return redirect("/");
  }

  return null;
};

...
```

## 3. コールバックページの実装

認証後のリダイレクト先は以下のように実装します。v4 では自分でセッションを保存する必要があります：

```typescript:auth/callback.tsx
import { authenticator, saveSession } from "@/.server/auth";
import { redirect } from "react-router";
import type { Route } from "./+types/callback.ts";

export async function loader({ request }: Route.LoaderArgs) {
  // 認証
  // login.tsx でリダイレクトされた場合、ここでアクセストークンの取得処理などが行われます。
  const user = await authenticator.authenticate("provider_name", request);
  // セッションを保存 - v4では自分で実装します
  const headers = await saveSession(request, user);
  // Set-Cookieをつけて"/"にリダイレクト
  return redirect("/", { headers });
}

export default function CallbackPage() {
  ...
}
```
`login.tsx`と`callback.tsx`の両方で`authenticator.authenticate()`を呼び出してます。そのため無限にリダイレクトされるじゃんと思われるかもしれません。
実際そんなことは起きなくて、ログイン画面からリダイレクトされてきたコールバックURL には `state` パラメータと `code` パラメータが付与されます。remix-auth-oauth2 はこの stateパラメータの有無を確認し、存在すれば認証サーバーからの正規のコールバックと判断してアクセストークン取得処理を行い、なければ直接アクセスされたと判断して認証プロセスを開始します。このような状態管理により、コールバック URLに何度アクセスしても適切に処理が振り分けられ、無限にリダイレクトされることはありません。
https://github.com/sergiodxa/remix-auth-oauth2/blob/0015b9efbd4a8d17fe35e997561555c536d529ee/src/index.ts#L69-L103

## 4. ログアウト機能の実装

v4 では、ログアウト処理も自分で実装する必要があります。

```typescript:logout.tsx
import { destroySession, getSession } from "@/.server/session";
import { Button } from "@/components/ui/button";
import { Form, redirect } from "react-router";
import type { Route } from "./+types/logout.ts";

export async function action({ request }: Route.LoaderArgs) {
  const session = await getSession(request.headers.get("Cookie"));
  return redirect("/login", {
    headers: {
      // セッションをクリア
      "Set-Cookie": await destroySession(session),
    },
  });
}

// 直接アクセスされた場合はトップページにリダイレクト
export function loader() {
  return redirect("/");
}

export default function LogoutPage() {
  ...
}
```

以上です。
少しでも誰かの役に立てれば嬉しいです。


# 参考
https://zenn.dev/kyrice2525/articles/update-remix-auth
