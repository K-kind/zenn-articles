---
title: "supbase-jsでAuthのセッションがすぐに切れる問題の対処"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "javascript"]
published: true
---
# 発生した問題

前提環境: @supabase/supabase-jsのv1系（1.33.3で確認）

supabase-jsは、Authのセッションのトークンの有効期限（デフォルトは1時間）が切れても、自動でトークンをリフレッシュしてくれる。

しかし、v1系では、セッションの取得 `supabase.auth.session()` が同期的であり、トークンのリフレッシュを待たない。

そのため、`supabase.auth.session()` の戻り値がnullかどうかでログイン状態を判定している場合、ページ遷移後初回にはトークンリフレッシュが間に合わすログアウト状態となり、ページをリロードするとログイン状態が戻るというおかしな挙動になり得る。

関連Issue: https://github.com/supabase/supabase/discussions/5272

# 解決策

@supabase/supabase-jsのv2から、セッションの取得方法が変わり、トークンの期限が切れている場合はリフレッシュを待ってくれるようになったため、v2に上げることで解決する。

```js
// Before
const session = supabase.auth.session()

// After
const { data: { session } } = await supabase.auth.getSession()
```

（非同期処理に変わったため、機械的には置き換えられないが）

https://supabase.com/docs/reference/javascript/upgrade-guide

# 余談

当初トークンのリフレッシュがされない（ように見えた）のは、 `createClient` 時の `autoRefreshToken` というオプションが設定されていないためかと思ったが、これは最初からtrueになっているようで、無関係だった。

https://supabase.com/docs/reference/javascript/v1/initializing
