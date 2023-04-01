---
title: "ChatGPT APIに関するQ&A"
emoji: "🌱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["chatgpt", "nodejs"]
published: true
---

今さらながらChatGPTのAPIを試してみて、分かりにくかった点についてまとめます。

:::message
2023/4/1追記
正確にはChatGPT APIではなく、OpenAIのChat Completion APIと呼ぶのが正しそうです。
:::

# 先に結論

- `gpt-3.5-turbo` と `gpt-3.5-turbo-0301` って何が違うの？
  → 一緒
- GPT-4モデルは使えるのか
  → 2023/03/19時点では誰でもは使えない。 [waitlist](https://openai.com/waitlist/gpt-4-api) で予約した人から先行で使える。
- なぜAPIリクエスト時の例に `role: "assistant"` が含まれているの？
  → これまでの会話の文脈を理解させるため

# そもそもChatGPTのAPIとは

OpenAIのAPIのうち、2023年3月1日に公開された「[Chat completions](https://platform.openai.com/docs/guides/chat)」機能を指します。
（これまでも似たようなことを実現できたが、より低コスト・高速になった）

概要や利用方法については以下サイトが参考になりました。

https://saasis.jp/2023/03/06/%E9%81%82%E3%81%AB%E5%85%AC%E9%96%8B%E3%81%95%E3%82%8C%E3%81%9Fchatgpt-api%E3%81%A8%E3%81%AF%EF%BC%9F-%E5%88%A9%E7%94%A8%E3%81%99%E3%82%8B%E3%81%BE%E3%81%A7%E3%81%AE%E6%B5%81%E3%82%8C%E3%80%90/

# APIリクエストの例

例えば、Node.jsでAPIを利用する場合、最小のコードは以下のようになります。

```js
import { Configuration, OpenAIApi } from "openai";

const openai = new OpenAIApi(new Configuration({ apiKey: "<<YOUR API KEY>>" }));

async function ask(content) {
  const response = await openai.createChatCompletion({
    model: "gpt-3.5-turbo",
    messages: [{ role: "user", content  }],
  });

  console.log(response.data);
}

ask("大谷選手について教えて");
```

実行結果

```log
{
  id: 'chatcmpl-6vj...',
  object: 'chat.completion',
  created: 1679218202,
  model: 'gpt-3.5-turbo-0301',
  usage: { prompt_tokens: 21, completion_tokens: 380, total_tokens: 401 },
  choices: [
    {
      message: {
        role: 'assistant',
        content: '大谷翔平選手は、1994年7月5日生まれの日本のプロ野球選手で...（省略）'
      },
      finish_reason: 'stop',
      index: 0
    }
  ]
}
```

# `gpt-3.5-turbo` と `gpt-3.5-turbo-0301` の違いは？

modelには `gpt-3.5-turbo` と `gpt-3.5-turbo-0301` が指定できます。 `gpt-3.5-turbo-0301` は、3月1日時点の `gpt-3.5-turbo` のスナップショットであり、今は `gpt-3.5-turbo` と指定すると `gpt-3.5-turbo-0301` が使われる、ということのようです。

つまり、modelには `gpt-3.5-turbo` を指定しておけば問題なさそうです。

https://note.com/npaka/n/nd34d44628f10

# GPT-4は使えるの？

[GPT-4モデル](https://platform.openai.com/docs/models/gpt-4) は2023/03/19時点では Limited beta であり、誰でもは使えないようです。

`model: "gpt-4"` と指定すると以下のようなエラーが返ってきました。

```log
error: {
  message: 'The model: `gpt-4` does not exist',
  type: 'invalid_request_error',
  param: null,
  code: 'model_not_found'
}
```

[waitlist](https://openai.com/waitlist/gpt-4-api) に登録していれば、キャパシティができた時に使えるようになるようなので、登録してメールを待ちましょう。

https://zenn.dev/ekusiadadus/articles/youtube_chatgpt_4_api

# messagesのroleは何？

roleに指定するのは `system` `assistant` `user` の3つがあります。

### system
`system` は、AIアシスタントの設定を記述でき、以下のようにAIに人格を与えるような面白いことができるようです。
（※gpt-3.5-turbo-0301 時点ではこの設定を重視してくれない場合もあるようです）

https://qiita.com/sakasegawa/items/db2cff79bd14faf2c8e0

### assistantとuser
`assistant` はAIからの回答、 `user` はユーザーからの発話であり、基本的には `user` として聞きたいことを送る形になります。

ではなぜ `assistant` というものがあるかというと、AIからの返答を含むこれまでの会話を全て送ることで、その文脈を踏まえて回答してもらえるようにするためです。

### 文脈について

例えば、上記のコード例で以下のように関連する質問を投げると、2つ目の質問には答えられません。

```js
ask("大谷選手について教えて").then(() => ask("彼の身長は？"));
```

```log
'大谷翔平選手は、1994年7月5日生まれの日本のプロ野球選手で...（省略）'
'申し訳ありませんが、この質問に対する具体的な回答を提供することはできません。何か具体的な情報や文脈を提供していただけますか？'
```

そこで、以下のようにそれまでの会話を保存して送るようにすると、

```js
const messages = [];
async function ask(content) {
  messages.push({ role: "user", content });
  const response = await openai.createChatCompletion({
    model: "gpt-3.5-turbo",
    messages,
  });

  const answer = response.data.choices[0].message;
  messages.push(answer);
  console.log(response.data);
}
```

文脈を踏まえて回答してくれるようになります。

```js
ask("大谷選手について教えて").then(() => ask("彼の身長は？"));
```

```log
'大谷翔平選手は、1994年7月5日生まれの日本のプロ野球選手で...（省略）'
'大谷翔平選手の身長は191cmです。'
```

### 注意点

ただし、これまでの会話を送ると、その分入力トークン数が増えていき、料金が多くかかるため、注意が必要です。

これまでの会話を送らなかった時の "彼の身長は？" の質問の使用トークン数
```log
usage: { prompt_tokens: 15, completion_tokens: 63, total_tokens: 78 },
```

これまでの会話を送った時の "彼の身長は？" の質問の使用トークン数
```log
usage: { prompt_tokens: 314, completion_tokens: 20, total_tokens: 334 },
```

#### トークン・利用料金についての参考
https://zenn.dev/umi_mori/books/chatbot-chatgpt/viewer/how_to_calculate_openai_api_prices

# あとがき

まだ自分もこれからAPIを使い始めるところなので、間違いなどがありましたらご指摘いただけますと幸いです🙏

余談ですが、大谷選手の身長についてGPT-3.5で聞くと、191cmと言ったり、188cmと言ったりと不正確でした（Wikiによると実際は193cm）。

ただ、GPT-4で試すと「約193センチメートル（6フィート4インチ）」と正確に答えてくれたので、本当に進歩がすごいですね。
