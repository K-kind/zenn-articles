---
title: "RailsではDate.todayではなくTime.zone.todayを使おう"
emoji: "🌏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rails", "Ruby"]
published: true
---
# 結論

- `Date.today` は、実行環境のタイムゾーンにおける当日の日付を取得する。

- `Time.zone.today` は、実行環境によらずRailsの `config.time_zone` に設定されたタイムゾーンにおける当日を取得する。

→ 特別な理由がなければ `Time.zone.today` を利用した方が安全

# Date.todayの利用が問題となる例

あるRailsプロジェクトで `config.time_zone` には `Asia/Tokyo` のタイムゾーンが設定されているが、サーバーの環境変数 `TZ` は指定しておらず、タイムゾーンがUTCとなっている。

その場合、当日の日付を取得する意図で `Date.today` を使うと、00:00〜8:59の間は、当日ではなく前日の日付が得られ、意図しない挙動になり得る。

→ 実際に、CI上でTZが指定されておらず、特定の時間帯だけ `Date.today` を利用しているテストが落ちるという現象に遭遇した。

# 改善策

- `config.time_zone` でタイムゾーンを指定する場合、サーバーの環境変数 `TZ` も同じように指定する。
- 仮に `TZ` が指定されていなくても問題ないよう、Railsでは基本的に `Date` や `Time` を使わず、 `TimeWithZone` を使う。

https://qiita.com/jnchito/items/cae89ee43c30f5d6fa2c
