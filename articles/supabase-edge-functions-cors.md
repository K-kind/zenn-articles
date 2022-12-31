---
title: "SupabaseのEdge Functions(Beta)のCORSエラー対応"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase"]
published: true
---

SupabaseのEdge Functions(Beta)をブラウザから呼び出すと、CORSエラーが発生してハマったため、その対応方法メモ。

結論としては、以下のexampleに挙げられている方法で対応できる。

https://github.com/supabase/supabase/blob/master/examples/edge-functions/supabase/functions/browser-with-cors/index.ts

# 問題の例

`supabase functions new hello` で以下のような関数を作成、デプロイする。

```ts
import { serve } from "https://deno.land/std@0.131.0/http/server.ts"

serve(async (req) => {
  const { name } = await req.json()
  const data = { message: `Hello ${name}!` }

  return new Response(JSON.stringify(data), {
    headers: { "Content-Type": "application/json" }
  })
})
```

JSのクライアント側で以下のように呼び出す。

```ts
await supabase.functions.invoke("hello", { name: 'World' })
```

この時、以下のようなエラーが発生して呼び出しが失敗した。

```log
Access to fetch at 'https://sample.functions.supabase.co/hello' from origin 'http://localhost:3000' has been blocked by CORS policy:
Response to preflight request doesn't pass access control check:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
If an opaque response serves your needs, set the request's mode to 'no-cors' to fetch the resource with CORS disabled.
```

# 問題の対応

## 不十分な対応

Access-Control-Allow系のレスポンスヘッダーを追加した。

```ts
  return new Response(JSON.stringify(data), {
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type"
    }
  })
```

ヘッダー内容は十分そうだが、ブラウザ側では先ほどと同じエラーが出てしまった。

### 調査内容

調査した所、サーバー側のログに以下のエラーが出ていた。

```log
SyntaxError: Unexpected end of JSON input
```

これは、preflightリクエストでも `await req.json()` を含む全てのコードが実行されていて、preflightリクエストにはボディが含まれていないため、parseが失敗していることが原因だった。

（これまでCORS系の処理はライブラリに任せてしまっていたので、preflightリクエストを自分で捌くという意識が抜けてしまっていた）

## 成功した対応

Access-Control-Allow系のヘッダー追加に加えて、OPTIONSリクエストの場合はすぐに200レスポンスを返す分岐を追加した。

→ 正常に呼び出せるようになることを確認。

```ts
import { serve } from "https://deno.land/std@0.131.0/http/server.ts"

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type"
}

serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders })
  }

  const { name } = await req.json()
  const data = { message: `Hello ${name}!` }

  return new Response(JSON.stringify(data), {
    headers: {
      "Content-Type": "application/json",
      ...corsHeaders,
    }
  })
})
```

# 処理を共通化する例

全ての関数でOPTIONSリクエストを捌く処理を実装するのは冗長な気がするので、共通化する例。

_shared/handleWithCors.ts（_で始まるフォルダはEdge Functionsと認識されない）

```ts
export const handleWithCors = (
  handler: (req: Request) => Promise<Response>
) => {
  return (req: Request) => {
    if (req.method === "OPTIONS") {
      return new Response("ok", { headers: corsHeaders })
    }

    return handler(req)
  }
}
```

```ts
import { handleWithCors } from "../_shared/handleWithCors.ts";

serve(
  handleWithCors(async (req) => {
    // 通常のリクエストを捌く処理
  }
)
```
