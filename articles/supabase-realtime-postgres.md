---
title: "Supabase JSのリアルタイムリスナーを使ってみる"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "typescript", "postgresql"]
published: true
---

RDB版のFirebaseと言われる [Supabase](https://supabase.com/) にも、Firestoreと同じようにDBの変更をリアルタイムに購読する機能があります。

https://supabase.com/docs/guides/realtime

この機能をJSクライアントから利用する例と、使ってみた所感をまとめます。

先にまとめると、2023年1月時点では、以下の点から、限られた場面でしか使えなさそうという印象でした。

- 購読対象の絞り込み条件を柔軟に指定できない
- 変更があったデータ単体を受け取るしかなく、不便

# 利用方法

## DB側の設定

### publicationの設定

以下のSQLを実行します。

```sql
-- supabase_realtimeというpublicationを作成する
begin;
  drop publication if exists supabase_realtime;
  create publication supabase_realtime;
commit;

-- publicationに、購読対象のテーブルを追加する
alter publication supabase_realtime add table <テーブル名>;
```

または、[Replication](https://app.supabase.com/project/_/database/replication) でGUI上で設定できるようです。

### RLSの設定

購読対象テーブルのRLS (Row Level Security) を有効にします。

```sql
alter table public.<テーブル名> enable row level security;

-- ログインユーザーには全データのSELECTを許可する（本来はusingの中に適切な条件を記載する）
create policy "Allow select access" on public.<テーブル名>
  for select to authenticated using (true);
```

## JSクライアント側

バージョン: @supabase/supabase-js (2.2.3)

### シンプルな購読の例

以下は、RLSで許可された全ての行に対する変更を購読する例です。

```ts
import { createClient } from "@supabase/supabase-js";
const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_KEY);

const channel = supabase
  .channel("table-db-changes") // 任意のチャンネル名
  .on(
    "postgres_changes",
    {
      event: "*", // "INSERT" | "UPDATE" | "DELETE" のように特定イベントだけの購読も可能
      schema: "public",
      table: "<テーブル名>",
    },
    (payload) => {
      console.log(payload);
    }
  )
  .subscribe();

// 購読を終了するとき
supabase.removeChannel(channel);
```

### 購読データの内容

上記の `payload` の中身の例（UPDATEの場合）

```ts
{
  commit_timestamp: "2023-01-09T09:53:37Z",
  errors: null,
  eventType: "UPDATE",
  new: { id: "<ID>", ... }, // 更新後のデータの中身すべて
  old: { id: "<ID>" }, // 更新前データ（主キーのみ）※1
  schema: "public",
  table: "<テーブル名>",
}
```

※1. `alter table <テーブル名> replica identity full;` の設定により、oldにも全ての値を取得できるようです。

### 指定した行のみを購読する例

以下のように `filter` を指定することで、特定の行のみを購読することができます。

```ts
const channel = supabase
  .channel("table-db-changes")
  .on(
    "postgres_changes",
    {
      event: "*",
      schema: "public",
      table: "<テーブル名>",
      filter: `id=eq.${id}`, // あるidの行のみ購読
    },
    (payload) => {
      console.log(payload);
    }
  )
  .subscribe();
```

supabase-jsのv2時点では、このfilterは `eq`, `neq`, `gt`, `gte`, `lt`, `lte` による絞り込みしか対応していないようです。

https://github.com/supabase/realtime-js/issues/185

（`eq` 以外の情報はドキュメントになく、上のIssueで知りました。）

### 購読開始やエラー時の処理

最後の `subscribe` に引数を渡すことで、購読開始やエラー発生時に任意の処理を行うことができます。

```ts
  ...
  .subscribe((status, err) => {
    if (status === "SUBSCRIBED") {
      // 購読開始時に行いたい処理
    }

    if (status === "CHANNEL_ERROR") {
      // 接続切れなどのエラー時に行いたい処理
    }

    if (status === "TIMED_OUT") {
      // 接続タイムアウト時に行いたい処理（エラー時との違いは分からず）
    }
  });
```

# 使ってみた感想

## 購読対象の絞り込み条件を柔軟に指定できない

上述したように、filterによる購読対象の絞り込みは、1つのカラムに対する単純な条件しか指定することができず、実際のユースケースに対応できない場合が多そうです。

## 変更の受け取り方が不便

### データが単体でしか渡ってこない

例えば一覧の変更を購読する場合、Firestoreの `onSnapshot` は、あるドキュメントの変更があった際に、その変更を含む全ての対象ドキュメントが渡ってきます。

しかしSupabaseでは、変更があった行のみが1つずつ渡ってくるため、クライアント側で一覧データの一部を差し替えるか、再フェッチする必要があります。

### 初回のフェッチが別途必要

Firestoreの `onSnapshot` は、変更がなくても購読開始時に一度対象データを取得してくれますが、Supabaseでは変更があったデータ単体しか取得できないため、初期表示のためのフェッチが別途必要になります。

### RDBなのに正規化を崩したくなる

1対多で紐付く子データの変更も一緒に購読したい場合、親データの購読と別に子データの購読も行う必要があり、その処理は複雑になるため、正規化を崩して、JSON型で子データを親テーブルに含めるような設計を検討したくなりそうです。

## まとめ

やはりFirestoreと比べてしまうと、Supabaseのリアルタイム処理はまだ完成度が低いという印象でした。

今後の進化に期待です。🙏
