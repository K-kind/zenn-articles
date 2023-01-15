---
title: "Supabaseでログインなしのリアルタイムアプリを作ってみた"
emoji: "🥚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "react", "typescript", "個人開発"]
published: true
---

個人開発した「ログインなし」「リアルタイム」のアプリをSupabaseで実現するにあたり必要だった工夫についてご紹介します。

# 作ったアプリ

アジャイル開発の見積もり手法として使われる「プランニングポーカー」を、オンラインで行うためのシンプルなアプリを作りました。使ってみていただけると嬉しいです🙌

https://planning-poker.k-kind.dev

![デモ](https://planning-poker.k-kind.dev/images/demo.gif)

# リポジトリ

https://github.com/K-kind/planning-poker

ディレクトリ構成等は、 https://github.com/alan2207/bulletproof-react を参考にしています。

# 使った技術

- React（create-react-app、react-router）
- Supabase (DB、リアルタイム通信)
- Mantine（UIライブラリ）

それぞれ、新しめの技術を使ってみたいという理由で選択しました。

# アプリの特性

ログインは無しで、部屋のURLを知っている人は誰でも参加できる形にしました。

また、各プレイヤーによるカードの変更はリアルタイムに反映する必要があります。

これらの特性は、2023年1月時点でのSupabaseとは相性が良くなかったようで、以下の点で苦労しました。

# 苦労した点1. データのアクセス制御

Supabaseでは、RLS（Row Level Security）で、行のアクセスを制御することができます。

```sql
-- ログインユーザーが所有する部屋のみ、取得を許可する例
create policy "Individuals can view their own rooms."
  on rooms for select
  using ( auth.uid() = user_id );
```

しかし、RLSは基本的に上記のようにログインユーザーがいることを前提として、その行のアクセスを制限するもののようです。

Firestoreのgetとlistの制御のように、IDを指定した１つのデータに対するアクセスのみを許可する、ということはできないようでした。

そのため、IDをuuidにして予測されにくくしたとしても、以下のように全部屋の情報が取得・変更されることを防ぐことはできず、

```ts
supabase.from("rooms").select("*");
// => 全部屋の情報を取得

supabase.from("rooms").delete().not("id", "is", null);
// => idがnullでない部屋（つまり全部屋）を削除する
// => そもそもidを知らなくても一括で操作できてしまう
```

「ログインしなくても部屋のIDさえ分かれば、部屋のデータにアクセスできる」という制御は、RLSだけで行うことはできませんでした。

## 工夫した点

### 匿名ログイン

そこで今回は、ユーザーが認証情報を入力することなく、裏で匿名でログインされているという形にして、まずRLSが機能するようにしました。

Firebaseでは匿名認証という機能がありますが、Supabaseでは2023年1月時点ではないようで、Issueで望む声が上がっているようです。

https://github.com/supabase/gotrue/issues/68

今回は、Issueで提案されているように、ランダムなメールアドレスとパスワードを生成してログインするという形で一旦実装しました。

### 匿名ユーザーと部屋を紐付ける

そして以下のように、匿名ユーザーと部屋を多対多で紐付けました。

![](/images/planning-poker-app/er-diagram.png)

このroom_usersテーブルのデータは、部屋に入る前に自動で作成されるようにしました。

これにより、room_usersを持っている、つまり、部屋のIDを知っている人だけが、部屋の情報を取得、変更できるようにRLSを設定することができました。

```sql
create policy "Allow select access" on public.rooms
  for select to authenticated using (
    auth.uid() in (
      select user_id from room_users
      where room_id = rooms.id
    )
  );

-- updateやdeleteも同様
```

このRLSによって、全ての部屋の情報を取得しようとしても、許可された行のみが返るようになります。

```ts
supabase.from("rooms").select("*"); // => 自分が一度入室した部屋の情報のみ取得
```

（room_users自体も他の人には見えないようにして、room_idが部外者に知られることはないようにしています）


# 苦労した点2. リアルタイム通信

DBの変更をリアルタイムに購読する機能については、まだ使いづらい点が多い印象でした。

詳しくは以下の記事に書かせていただきました。

https://zenn.dev/k_kind/articles/supabase-realtime-postgres

本来はroomsとplayersは別テーブルに分けるべきですが、そうすると購読の処理がとても煩雑になりそうだったため、roomsの中にJSON型でplayersを持たせるようにしました。

それはそれで、他のプレイヤーまで古いデータで更新してしまう不具合を生んだりしました。

# Supabaseに対する感想

#### TypeScriptとの相性が良さそう

DBの型を自動生成することもできて、クエリの型付けも優秀なので、その点はFirestoreより優れていると思いました。

https://zenn.dev/k_kind/articles/supabase-type-generate

#### Edge Functionsについて

任意の関数を実行するAPIを簡単に作成・デプロイできるEdge Functionsも利用しました。

コンソール上のログが見やすかったり、Betaですが完成度が高い印象でした。

注意点として、無料プランで試した限り、初回の呼び出し時に0.8秒ほど起動時間がかかり、5分ほど呼び出しが無いとまたコールドスタート状態になるようです。

また実行環境はDenoのみのため、学習コストが若干必要でした。

# 最後に

おかしな点や、無駄なことをやっている部分がありましたら、ご指摘いただけますと幸いです🙇‍♂️
