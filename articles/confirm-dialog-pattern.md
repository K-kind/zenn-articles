---
title: "async/await で直感的に書ける確認ダイアログの実装例（React）"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript", "frontend"]
published: true
---

# はじめに
## 確認ダイアログ、毎回ちょっと面倒ではありませんか？

「削除」や「破棄」などの **取り返しのつかない操作の前に確認ダイアログを挟む** というのは、フロントエンドではおなじみの UI ですよね。

![](/images/confirm-dialog-pattern/capture1.gif)

実装としては次のような形が一般的かと思います。

```tsx
export const ItemDeleteButton = () => {
  const [dialogOpen, setDialogOpen] = useState(false);

  const handleClick = () => {
    setDialogOpen(true);
  };

  const handleCancel = () => {
    setDialogOpen(false);
  };

  const handleConfirm = async () => {
    setDialogOpen(false);
    // アイテム削除処理
  };

  return (
    <>
      <Button variant="destructive" onClick={handleClick}>
        削除
      </Button>
      <ConfirmDialog
        open={dialogOpen}
        description="アイテムを削除しますか？"
        onCancel={handleCancel}
        onConfirm={handleConfirm}
      />
    </>
  );
};
````

ここで、次のような違和感を覚えたことはないでしょうか。

* 確認ダイアログのためだけに `useState` を毎回用意している
* 「削除ボタンを押す → ダイアログが開く → 確認されたら削除」
  という処理の流れが、コード上で分断されてしまう

## `window.confirm` のようなインターフェースで扱いたい

Web 標準の `window.confirm` と同じように、削除ボタン押下時のイベントハンドラ内で、
確認も含めた処理を一直線に書けたら、より直感的だと思いませんか？

```tsx
export const ItemDeleteButton = () => {
  const handleClick = async () => {
    const confirmed = window.confirm("アイテムを削除しますか？");
    if (!confirmed) return;

    // アイテム削除処理
  };

  return (
    <Button variant="destructive" onClick={handleClick}>
      削除
    </Button>
  );
};
```

この記事では、上記のようなインターフェースで、自前の確認ダイアログを表示できるようにする方法を紹介します。

# 実装方法

## 1. ConfirmDialog コンポーネント

まずは、見た目とイベント通知のみを担当する `ConfirmDialog` コンポーネントを用意します。

今回は例として
[shadcn/ui の Alert Dialog](https://ui.shadcn.com/docs/components/alert-dialog) を使用しますが、
MUI など、他のライブラリでも考え方は同じです。

:::details ConfirmDialog.tsx の実装例（参考）

```tsx:components/common/ConfirmDialog.tsx
import { ReactNode } from "react";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";

type Props = {
  open: boolean;
  title?: ReactNode;
  description?: ReactNode;
  cancelButtonText?: string;
  confirmButtonText?: string;
  onCancel: () => void;
  onConfirm: () => void;
};

export const ConfirmDialog = ({
  open,
  title,
  description,
  cancelButtonText,
  confirmButtonText,
  onCancel,
  onConfirm,
}: Props) => {
  return (
    <AlertDialog open={open}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{title || "確認"}</AlertDialogTitle>
          {description && (
            <AlertDialogDescription>{description}</AlertDialogDescription>
          )}
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel onClick={onCancel}>
            {cancelButtonText || "キャンセル"}
          </AlertDialogCancel>
          <AlertDialogAction onClick={onConfirm}>
            {confirmButtonText || "はい"}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
};
```

:::

## 2. Context と Provider で「confirm 関数」を管理する

次に、この確認ダイアログを **どこからでも呼び出せるようにする仕組み** を作ります。

```tsx:providers/ConfirmDialogProvider.tsx
import {
  createContext,
  ReactNode,
  useCallback,
  useMemo,
  useState,
} from "react";
import { ConfirmDialog } from "@/components/common/ConfirmDialog";

type ConfirmOptions = {
  title?: ReactNode;
  description?: ReactNode;
  cancelButtonText?: string;
  confirmButtonText?: string;
};

type ConfirmDialogContextType = {
  confirm: (options: ConfirmOptions) => Promise<boolean>;
};

export const ConfirmDialogContext =
  createContext<ConfirmDialogContextType | null>(null);

type ConfirmState = {
  options: ConfirmOptions;
  resolve: (value: boolean) => void;
};

export const ConfirmDialogProvider = ({
  children,
}: {
  children: ReactNode;
}) => {
  const [confirmState, setConfirmState] = useState<ConfirmState | null>(null);
  const [isOpen, setIsOpen] = useState(false);

  const confirm = useCallback(
    (options: ConfirmOptions) => {
      // confirm が連続で呼ばれた場合、前の呼び出しは false として終了させる
      if (confirmState) {
        confirmState.resolve(false);
      }

      return new Promise<boolean>((resolve) => {
        setConfirmState({ options, resolve });
        setIsOpen(true);
      });
    },
    [confirmState]
  );

  const contextValue = useMemo(() => ({ confirm }), [confirm]);

  const handleCancel = () => {
    confirmState?.resolve(false);
    close();
  };

  const handleConfirm = () => {
    confirmState?.resolve(true);
    close();
  };

  const close = () => {
    setIsOpen(false);
    // アニメーション終了後に state を破棄
    setTimeout(() => {
      setConfirmState(null);
    }, 300);
  };

  return (
    <ConfirmDialogContext value={contextValue}>
      {children}
      {confirmState && (
        <ConfirmDialog
          open={isOpen}
          onCancel={handleCancel}
          onConfirm={handleConfirm}
          title={confirmState.options.title}
          description={confirmState.options.description}
          cancelButtonText={confirmState.options.cancelButtonText}
          confirmButtonText={confirmState.options.confirmButtonText}
        />
      )}
    </ConfirmDialogContext>
  );
};
```

### 実装のキモ

最も重要なのは次の部分です。

```ts
return new Promise<boolean>((resolve) => {
  setConfirmState({ options, resolve });
  setIsOpen(true);
});
```

* `confirm` を呼んだ瞬間に Promise を生成
* `resolve` を state に保持
* ボタン操作で `resolve(true / false)` を呼ぶ

これにより、呼び出し側は

```ts
const confirmed = await confirm(...);
```

という **自然な制御フロー** で処理を書けるようになります。

## 3. Provider をアプリに組み込む

Next.js（App Router）であれば、`app/layout.tsx` などでアプリ全体をラップします。

```tsx
<ConfirmDialogProvider>
  {children}
</ConfirmDialogProvider>
```

これで、どのコンポーネントからでも `confirm` が使える状態になります。

## 4. useConfirmDialog フック

最後に、毎回 Context を直接触らなくて済むよう、カスタムフックを用意します。

```ts:hooks/useConfirm.ts
import { useContext } from "react";
import { ConfirmDialogContext } from "@/providers/ConfirmDialogProvider";

export const useConfirmDialog = () => {
  const context = useContext(ConfirmDialogContext);
  if (!context) {
    throw new Error(
      "useConfirmDialog must be used within a ConfirmDialogProvider"
    );
  }
  return context;
};
```

# 使用例

これで、冒頭で紹介した `window.confirm` と同じ流れで確認ダイアログを利用できるようになります。

```tsx
import { useConfirmDialog } from "@/hooks/useConfirm";

export const ItemDeleteButton = () => {
  const { confirm } = useConfirmDialog();

  const handleClick = async () => {
    const confirmed = await confirm({
      description: "アイテムを削除しますか？",
      // title や confirmButtonText なども必要に応じて指定可能
    });
    if (!confirmed) return;

    // アイテム削除処理
  };

  return <Button onClick={handleClick}>削除</Button>;
};
```

* state 管理なし
* 処理の流れが一直線

かなりスッキリ書けるようになります。

# あとがき

このパターンは React に限らず、
Vue や Svelte など **「Promise を返せる関数 + モーダル」** があるフレームワークであれば応用可能です。

「`window.confirm` のような感覚で書けたら楽なのに……」と思ったことがある方は、
ぜひ一度試してみてください。

もし「このケースだと困りそう」「もっと良い実装がある」などがあれば、
ぜひ教えていただけると嬉しいです 🙏
