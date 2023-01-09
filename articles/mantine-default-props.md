---
title: "Mantineではpropsのデフォルト値を変更できる"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "mantine"]
published: true
---

ReactのUIライブラリーの [Mantine](https://mantine.dev) で、各コンポーネントのpropsのデフォルト値を変えることができて便利でした。

## 方法

MantineProviderのthemeに以下のように記載します。（Mantine v5.10.0時点）

```tsx
import { MantineProvider } from '@mantine/core';

function Demo() {
  return (
    <MantineProvider
      withGlobalStyles
      withNormalizeCSS
      theme={{
        components: {
          Button: {
            defaultProps: {
              // loading時のアイコンを中央に
              loaderPosition: "center"
            }
          }
        }
      }}
    >
      <App />
    </MantineProvider>
  );
}
```

公式ページにしっかり書いてありました。

https://mantine.dev/theming/default-props

v5.10.0時点で、propsの型付けまではされないようですが、themeオブジェクトを利用できたりして、十分実用性がありそうです。

## 利用例

Buttonコンポーネントのloadingアイコンの位置がデフォルトでは左側になっていて、ロード時に一瞬ボタンの幅が変わってしまう点が気になったため、デフォルトの位置を中央に変更してみました。

@[codesandbox](https://codesandbox.io/embed/hardcore-shirley-2iwdn7?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.tsx&theme=dark)

コンポーネントをラップしなくても使いやすくなるので便利ですね。
