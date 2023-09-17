---
title: "next/fontのweightを指定すると遅くなる!?"
emoji: "⛵️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['nextjs', 'lighthouse']
published: true
---

# 結論から

Next.jsの [next/font](https://nextjs.org/docs/app/building-your-application/optimizing/fonts) でWebフォントを使う時は、そのフォントが [Variable fonts](https://fonts.google.com/variablefonts) であれば、 `weight` を指定しない方がCSSサイズを削減できるようです。

# 前提

- Next.js 13.4.19
- App Router使用

# 実験

Next.jsアプリを立ち上げ、今回は [Noto Sans JP](https://fonts.google.com/noto/specimen/Noto+Sans+JP) を以下のように使用します。

```ts:app/layout.tsx
import { Noto_Sans_JP } from 'next/font/google'

const notojp = Noto_Sans_JP({
  weight: ['200', '400', '700'], // 使用するfont-weightを指定する
  subsets: ['latin']
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ja">
      <body className={notojp.className}>{children}</body>
    </html>
  )
}
```

```ts:app/page.tsx
export default function Home() {
  const text =
    "人類社会のすべての構成員の固有の尊厳と平等で譲ることのできない権利とを承認することは";

  return (
    <main>
      <div>Font-weight: 200</div>
      <p style={{ fontWeight: 200 }}>{text}</p>

      <hr />

      <div>Font-weight: 400（デフォルト）</div>
      <p>{text}</p>

      <hr />

      <div>Font-weight: 700</div>
      <p style={{ fontWeight: 700 }}>{text}</p>
    </main>
  );
}
```

これで、以下のように各font-weightで表示することができます。

![](/images/next-font-weight/page.png)

### 実はweightは不要

ですが、実は `Noto Sans JP` は [Variable fonts](https://fonts.google.com/variablefonts) のため、上記の `weight: ['200', '400', '700']` の指定はしなくても、全く同じように表示できます。

```ts:app/layout.tsx
const notojp = Noto_Sans_JP({
  subsets: ['latin']
});
```

##### Variable fontsとは

> 可変フォント (Variable fonts) は幅、太さ、スタイルごとに個別のフォントファイルを用意するのではなく、書体のさまざまなバリエーションを 1 つのファイルに組み込むことができる OpenType フォント仕様の進化版です

https://developer.mozilla.org/ja/docs/Web/CSS/CSS_fonts/Variable_fonts_guide

## Lighthouseスコアへの影響

weight指定ありが99点、なしが100点でした。指定なしの方が描画が0.3秒速くなっています。

##### weight指定ありの場合

![](/images/next-font-weight/score-with-weight-1.png)

##### weight指定なしの場合

![](/images/next-font-weight/score-without-weight-1.png)

### CSSファイルサイズの違い

この違いの原因は、フォント用のCSSファイルサイズがweight指定ありだと3倍になっているためです。

##### weight指定ありの場合

![](/images/next-font-weight/score-with-weight-2.png)

:::details cssファイルの中身の例
200, 400, 700それぞれの指定がある
![](/images/next-font-weight/css-with-weight.png)
:::

##### weight指定なしの場合

![](/images/next-font-weight/score-without-weight-2.png)

:::details cssファイルの中身の例
100から900がまとめられている
![](/images/next-font-weight/css-without-weight.png)
:::

# Variable fontsでない場合

ちなみに、例えば [Noto Serif JP](https://fonts.google.com/noto/specimen/Noto+Serif+JP) のようにVariable fontsでない場合は、 `weight` を指定しないと型エラーを出してくれるようです。

![](/images/next-font-weight/serif-error.png)

# まとめ

以上のように、weightの指定1つでパフォーマンスを改善できる場合があるため、見直してみると良いかもしれません💡
