---
title: db_repack を活用してMastodonのDB肥大化を解消する (Docker編)
date: 2020-01-26 04:50:00
tags:
- メンテナンス
cover_img: /images/docker.png
---

2020-01-26 深夜0時半頃から2時半頃(JST)にかけての約2時間、mstdn.maud.io ではサービス停止を伴うメンテナンスを実施しました。ご協力ありがとうございました。

<!-- more -->

## 経緯

mstdn.maud.io (以下、当サーバー)は稼働開始からまもなく3年を迎えようとしています。1800 人を超えるユーザーの皆さんに支えられて、総投稿数は 355 万トゥートを突破しました（数字はいずれも執筆時点）。

年月と共に積み重ねていくものが多いのは人だけではなく、データを抱えるストレージも同様です。昨年はメディアにディスクを圧迫され、[オブジェクトストレージに移行](https://diary.akane.blue/2019/06/05/move-media-to-object-storage/)しました。

他にストレージをじわじわと食い潰し始めていたのがデータベースで、気づいたときには70GBを超えていました。

### pg_repack との出会い

正月早々に、Mastodon の開発者であり mastodon.social を運用する Eugen 氏によって次のような投稿がなされました。

* [https://mastodon.social/@Gargron/103406143264729242](https://mastodon.social/@Gargron/103406143264729242)

  > To other Mastodon sysadmins: Check what pg_repack can do for you, especially if your server has been around for a really long time. PostgreSQL indexes get bloated over time and VACUUM does nothing about it. pg_repack can re-build indexes without downtime.

* [https://mastodon.social/@Gargron/103406939008008311](https://mastodon.social/@Gargron/103406939008008311)

  > I shaved off 40 GB from the mastodon.social database just by using pg_repack on indexes tonight.

なんと、`pg_repack` を使うとダウンタイムなしでインデックスの再構築を行うことが出来るとのことで、特に長期間運用されているサーバーにこそ効果的であり、実際に mastodon.social ではそれで 40 GB もの容量削減に成功したというのです。

また、これを受けて [おさ](https://mstdn.nere9.help/@osapon) 氏が [データベースの断片化率を計算する方法](https://gist.github.com/osapon/8abd82ac942a0ccdce7e02b0540b15db) を公開しており、当サーバーで試算したところ 8% を超える値が出ました。6GBくらいは減るのではないかという見通しが立ったこと、あわよくばパフォーマンスの改善にもつながって欲しいという思いから、試してみることにしました。

## Docker 環境で、どう使う？

Mastodonインスタンスで pg_repack を使用している日本語の記事としては、 [【Mastodon】pg_repackでインスタンス無停止のDB不要領域削除 - Qiita](https://qiita.com/west2538/items/a82827ece65469c8c2be) などがあります。しかし、当サーバーは上の例とは異なり、Dockerを利用してMastodonサーバーを運用しています。このとき、様々な問題に直面しました。

### 堅牢さが仇

ここで `docker-compose.yml` の冒頭、 `db` コンテナについての記載を見てみましょう。

```yml
version: '3'
services:

  db:
    restart: always
    image: postgres:9.6-alpine
    networks:
      - internal_network
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
# 後略
```

`db` コンテナは `internal_network` のみに接続しており、外部との接続を絶っています。運用上は当然の対応であり、ここを安易に変えるべきではないでしょう。

しかし、 `pg_repack` をインストールする上では大きな障壁となりがちです。

まず、 `docker-compose exec db /bin/bash` などで直接コンテナに入ってインストールを行う場合、

* `make` に必要なパッケージを取得できない
* `pg_repack` のダウンロードができない

などがあります。後者は `docker cp` などでホストからコピーしてくるなどの対処が考えられますが、前者を解決できないことには事が進みません。

### 戦闘回避

外部と疎通しない以上は、あらかじめインストールされた状態の Docker イメージを用意するのが得策でしょう。では、PostgreSQL 9.6の公式イメージをベースとした Dockerfile を書き…ま…

…せんでした。ちょうどいいところに [hartmut-co-uk/pg-repack-docker](https://github.com/hartmut-co-uk/pg-repack-docker) という、 PostgreSQL 10 の公式イメージをベースに `pg_repack` のインストールを加えたものが公開されていたので、これを fork して PostgreSQL 9.6 ベースのものを用意してしまいましょう。たった1行の改変で済んでしまいました。よかったですね。

こうしてできたものが [lindwurm/pg-repack-docker:9.6](https://github.com/lindwurm/pg-repack-docker/tree/9.6) になります。Docker Hub だと [lindwurm/pg_repack](https://hub.docker.com/r/lindwurm/pg_repack) です。このために登録しました。

## やっていく

`docker-compose.yml` で `db` コンテナの参照イメージを変更します。先ほど Docker Hub に上げたイメージを使います。

```diff
version: '3'
services:

  db:
    restart: always
-    image: postgres:9.6-alpine
+    image: lindwurm/pg_repack:9.6
    networks:
      - internal_network
```

`postgres:9.6-alpine` (Alpine Linux) と `postgres:9.6` (Debian) の切り替えはstagingサーバーでテストした感じだと PostgreSQL のバージョンさえ間違えていなければ大丈夫だったので、後者をベースにした今回のイメージもきっと大丈夫です。

* [https://mstdn.nere9.help/@osapon/103407047435265861](https://mstdn.nere9.help/@osapon/103407047435265861)

  > pg_repackのロック無しというのは厳密には違って、一瞬だけロックは掛かるんだけど、最後のデータを差し替える瞬間だけロックがかかる。なので、jpみたいな大忙しサーバだと、一通り処理が終わったタイミングとかでDBへアクセスするようなやつを全部止めてpg_repackがロックを取れるようにしてやる必要か有るんじゃないかと思っている。昔ふぁぼるっくでやったら、データベースが常に大忙しだったので、いつまでもpg_repackがロックを獲得できなくて大変なことになった。

という投稿があり、止めるタイミングに迷ってもアレなので今回は実行前にサービスを停止し、作業用にDBだけを起こして入りました。

```
(host)$ docker-compose up -d db
```

```
(host)$ docker-compose exec db bash
```

`pg_repack` をDBに登録

```
(db)# psql -U postgres
```

```
(psql)> CREATE EXTENSION pg_repack;
```

```
(psql)> \q
```

実行します（ `.env.production.sample` の通り、 `DB_USER` / `DB_NAME` ともに `postgres` として。適宜読み替えてください）

```
(db)# pg_repack -d postgres -a -U postgres
```

当サーバーではこの完了待ちで約2時間止めっぱなしにしてしまいましたが、よほどのことがない限り無停止目指して稼働させてても問題ない気がします。

完了を確認したところで他のコンテナも起こしてサービス復旧、丸2時間の停止時間となりました。

## 得られた成果

完了後に再度断片化率を計算したところ、 0.06 % に落ち着きました。目的は達成されたと言ってよいでしょう。それ以上に成果が出たのがストレージ使用量で、実行前は 79 GB あったのが 49 GB まで下がり、なんと約 30 GB も削減されました。なんということでしょう！

おまけ: `/pghero/space` から確認したところ、最もサイズが削減された項目は `status_stats` で、 -8.5 GB ほどでした。