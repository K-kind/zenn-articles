---
title: "mise で入れた Node や pnpm が VSCode 拡張で使えない時の対処法"
emoji: "🔗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "nodejs", "vscode"]
published: true
---

# 問題

mise でインストールした Node.js や pnpm が、VSCode の Jest や Playwright 拡張から“だけ”使えない問題が発生しました。

ターミナルでは動くのに、VSCode 拡張側では以下のようなエラーが出てしまいます。

```log
node: not found
Unable to find 'node' executables.
Your Node version is incompatible with "/Users/xxx".

pnpm: not found
pnpm is not accessible.
```

# 解決法 1

[mise 公式ドキュメント](https://mise.jdx.dev/dev-tools/shims.html)にも記載がありますが、`.zprofile` に shims を読み込む設定を追加します。

```bash
echo 'eval "$(mise activate zsh --shims)"' >> ~/.zprofile # これを追加実行する
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc    # これはすでに mise インストール時にやっているはず
```

# 解決法 2

それでも改善しない場合は、`/usr/local/bin` に mise の shims へのシンボリックリンクを作成します。
（特に Playwright 拡張は、私の環境では **この方法でしか解決しませんでした**）

```bash
# node が使えない場合
sudo ln -s "$HOME/.local/share/mise/shims/node" /usr/local/bin/node

# pnpm が使えない場合
sudo ln -s "$HOME/.local/share/mise/shims/pnpm" /usr/local/bin/pnpm
```

# 解説

## エラー原因

mise の通常セットアップでは、`.zshrc` に `eval "$(mise activate zsh)"` が追記され、ターミナルのような **対話シェル** では node / pnpm の PATH が自動で通るようになります。

しかし VSCode の拡張は多くの場合 **非対話シェル** で動作するため、`.zshrc` が読み込まれません。
その結果、mise の PATH が設定されないまま node / pnpm が呼ばれ、上記のエラーにつながります。

## 解決法 1 の解説

`.zprofile` に `eval "$(mise activate zsh --shims)"` を記載する方法です。

### `.zprofile`

`.zprofile` は **ログインシェル** で実行されます。

VSCode 拡張の設定次第では `.zshrc` は読まれなくても `.zprofile` は読み込まれるケースがあり、この差によって挙動が変わることがあります。

### Shims

`mise activate zsh --shims` は `export PATH="$HOME/.local/share/mise/shims:$PATH"` と同義で、 PATH に mise の shims ディレクトリを追加します。

Shims は “緩衝材” のような小さなスクリプトで、`$HOME/.local/share/mise/shims/node` を実行すると、現在のプロジェクト or グローバルに設定された node バージョンへ適切にルーティングしてくれます。

そのため `.zprofile` が実行される環境では、Shims 経由で node が使用できるようになります。

（**対話シェル** では Shims ではなく、グローバルバージョンの実体に直接 PATH が通る方式が使われます）

## 解決法 2 の解説

VSCode 拡張は、zsh ではないシェルで実行されたり、`.zprofile` すら読み込まれないことがあります。
そのような環境でも、多くの場合 `/usr/local/bin` は PATH に含まれているため、ここに node / pnpm があれば実行可能になります。

よって、Shims へのシンボリックリンクを `/usr/local/bin` に作成することで、どのシェル環境からでもグローバル設定の node を実行できるようになります。

```bash
sudo ln -s "$HOME/.local/share/mise/shims/node" /usr/local/bin/node
```

### 備考

これは mise の推奨方法ではないため、環境によっては副作用が出る可能性があります。

また `/usr/local/bin/node` は Homebrew が管理する Node と衝突することがあるため、必要に応じて既存バイナリの整理が必要です。
