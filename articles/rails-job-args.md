---
title: "RailsのActive Jobの引数に注意"
emoji: "🦉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rails", "Ruby"]
published: true
---

RailsのActive Jobの引数に、モデルオブジェクトの配列を渡すと、無駄に大量のSQLが発行される場合があるため注意が必要。

## 前提

- Ruby 3.1.3
- Rails 7.0.4
- キューのアダプターとしてSidekiqなどを利用している

# 悪い例

複数のUserオブジェクトを用いて何か処理をしたいジョブを以下のように実装する。

```rb
class SampleJob < ApplicationJob
  queue_as :default

  def perform(users)
    # ...
  end
end
```

これを以下のように呼ぶと、エラーが発生する。

```rb
users = User.all
SampleJob.perform_later(users)
# => エラー: Failed enqueuing SampleJob to Sidekiq(default):
#           ActiveJob::SerializationError (Unsupported argument type: ActiveRecord::Relation)
```

#### エラーとなる理由

Jobをperform_laterで呼ぶ時、引数は一度シリアライズされてRedis等に格納され、それがジョブサーバーでデシリアライズされてperformメソッドに渡される。

そのため、User.allのようなActiveRecord::Relationのオブジェクトは、シリアライズができず、エラーになる。

## 最悪な回避方法

上記エラーを回避するため、ActiveRecord::Relationを無理やり配列に変換すると、実行自体はできるようになる。

```rb
users = User.all.to_a
SampleJob.perform_later(users)
```

（ここまで露骨な変換は普通ないかもしれないが、 `User.all.each_slice(1000)` や `User.find_in_batches` で分割処理しようとして無意識に配列に変換してしまうことはありそう。）

#### 裏で起きていること

この時、Userオブジェクトの配列が、以下のようにシリアライズされる。

```log
[#<GlobalID:0x0000000106045630 @uri=#<URI::GID gid://アプリ名/User/1>>, #<GlobalID:...]
```

ジョブサーバー側では、これをデシリアライズする時に、配列内のオブジェクトの数だけ、SELECT文を発行して、Userオブジェクトの配列を復元している。

```log
SELECT "users".* FROM "users" WHERE "users"."id" = ? LIMIT ?  [["id", 1], ["LIMIT", 1]]
SELECT "users".* FROM "users" WHERE "users"."id" = ? LIMIT ?  [["id", 2], ["LIMIT", 1]]
SELECT "users".* FROM "users" WHERE "users"."id" = ? LIMIT ?  [["id", 3], ["LIMIT", 1]]
SELECT "users".* FROM "users" WHERE "users"."id" = ? LIMIT ?  [["id", 4], ["LIMIT", 1]]
...
```

#### 問題点

DBの負荷が増え、処理速度も低下してしまう。

# 良い例

上記の問題を解消するため、JobはUserのIDの配列を受け取り、明示的にUserを取得し直す形に変更する。

```rb
def perform(user_ids)
  users = User.where(id: user_ids)
  # ...
end
```

```rb
user_ids = User.all.pluck(:id)
SampleJob.perform_later(user_ids)
```

この方法であれば、ジョブサーバー側では1回のSQLで目的のusersを復元することができる。

```log
SELECT "users".* FROM "users" WHERE "users"."id" IN (?, ?, ?, ?, ?, ?,...
```

# まとめ

Active Jobの引数は、一度シリアライズされることを意識し、大量のモデルオブジェクトの配列を渡すことがないように注意する。

Action Mailerのメールをdeliver_laterで送信する際の引数も同様（同じ仕組みが使われるため）。
