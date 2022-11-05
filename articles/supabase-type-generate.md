---
title: "SupabaseのTS用の型情報を自動生成する"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "typescript"]
published: true
---
# 概要

supabase-js v2から利用できるようになった、型を自動生成する機能を、以下の手順で試してみる。

- Supabase CLIで、TypeScript用の型を生成する
- supabase-js v2で、生成された型を利用する

https://supabase.com/blog/supabase-js-v2

# Supabase CLIの導入

[Supabase CLI](https://github.com/supabase/cli) を使うと、ローカルでコンテナによる開発環境を立ち上げたり、DBのマイグレーションを管理したりできる。

ただ、現状はWIPのようで、変更が入る可能性が高く、ドキュメントも読みづらく、動かすまでが大変だった。

以下は、2022-11-06時点で最新のv1.11.7で型を生成した際の手順。

## インストール

`npm i supabase` でローカルにインストール。（他にbrewなども使えるよう）

※Node v14（npm v6）だと、 `Failed at the supabase@1.11.7 preinstall script.` のエラーで失敗したが、Node v16（npm v8）に上げてみると成功した。

## supabase login

`npx supabase login` で、アクセストークンを入力し、ログインする。（npxは、ローカルにインストールした場合のみ必要。以後は省略する）

この時、アクセストークンの取得方法は以下のように案内がある。

```
You can generate an access token from https://app.supabase.com/account/tokens
```

## supabase init

`supabase init` で、supabase/ フォルダが作成される。

## supabase link

`supabase link --project-ref <プロジェクトのReference ID>` で、プロジェクトをリンクさせる。

Reference IDは、Project Settings画面で確認できる。また、この時DBのパスワードの入力が求められる。

## 型情報の生成

以上で導入は完了し、下記のコマンドで、リンクしたプロジェクトのSchemaからTS用の型を生成できる。

```
supabase gen types typescript --linked > schema.ts
```

## Supabase CLIのその他の機能について

Supabase CLIのmigration管理機能は、GUI上で作成したテーブルのSchemaをSQLとして得られるなど、便利そうだと思ったが、まだ以下のような問題があり、実用はできなさそうだった。

- テーブルの作成順がアルファベット順なので、referenceを設定している場合にエラーが発生する。

# supabase-js v2で、生成された型を使用

createClientの型引数に、生成された型を指定するだけで、各クエリが正確に型付けされる。

```ts
import { createClient } from "@supabase/supabase-js";
import type { Database } from "./schema";

export const supabase = createClient<Database>(
  "SAMPLE_SUPABASE_URL",
  "SAMPLE_SUPABASE_ANON_KEY"
);

// from()内で入力補完が効き、dataは正確に型付けされている
const { data } = await supabase.from("users").select("*");

// insertやupdateに渡すデータも型付けされている
await supabase.from("users").insert({ name: "sample", age: 10 });
```

## より詳しいサンプル

以下のような2つのテーブルを用意し、そこから型を生成した。

users
![](/images/supabase-type-generate/table-users.png)

todos
![](/images/supabase-type-generate/table-todos.png)

型が利用される様子（Open Sandboxして見ないと全てanyになるかも）

@[codesandbox](https://codesandbox.io/embed/silly-rumple-vi028x?fontsize=14&hidenavigation=1&module=%2Fsrc%2Fsample.ts&theme=dark&view=editor)

- NOT NULLかどうかが型に反映される
- joinして子テーブルもまとめて取得する際もちゃんと型に反映される
- Array型やEnum型もしっかり型に反映される

→ とても便利！
