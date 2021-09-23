---
title: GitHub Actionsでdocker buildを自動化する
date: 2021-09-23 22:40:00
tags:
- メンテナンス
cover_img: /images/docker.png
summary: アップデートの所要時間短縮を図りました
---

タイトルの通りで、 GitHub Actions を用いて `docker build` を自動化しました。

## 重複するビルド

[前回](https://diary.akane.blue/2021/08/12/shikinen-sengu/) はデータベースサーバーを分離することで、Webサーバー側を身軽にすることに成功しました。

しかしまだ無駄な部分が残っていました。mstdn.maud.io には公開サービスを提供している本番環境のほかに、アップデート前の人柱を担うステージング環境[^1] が存在しています。

[^1]: staging.mstdn.maud.io というのですが、登録は受け付けていないほか、特にフォローもおすすめしません

今までのアップデートの際には、まずステージング環境で `docker-compose build` を実行し、サービスを再起動し、動作に問題ないことを確認したのちに本番環境でも同様に `docker-compose build` し…という手順を踏んでいました。毎回ビルドに時間がかかっており、アップデート開始からデプロイまでの所要時間が読めないのが悩みでした。

独自のカスタムを加えていないプレーンな Mastodon を動かす場合、[tootsuite/mastodon - Docker Hub](https://hub.docker.com/r/tootsuite/mastodon) を利用することができますが、弊サーバーではそこそこ独自の改変が加わっているので自前でやる必要がありました。

## Docker Registry

せっかくなので自分たち用の docker image を置いて docker-compose pull だけで済むようにしたいなあと思い始めました。

特に自分たち以外で使うこともないだろう[^2] ということでプライベートリポジトリにしようと思っていたのですが、これだけのために自前でサーバーを立てようとはあんまり思えず、今流行りの GitHub Packages は Free だとプライベートリポジトリで利用可能なストレージが [500MB まで](https://github.com/features/packages) で、前述の tootsuite/mastodon を見る限り無理そうだったので流れに逆行して [Docker Hub](https://hub.docker.com) になりました。

[^2]: favicon や apple-touch-icon など弊サーバーに特有のアセットが含まれるため他所で使われることがない

## GitHub Actions

やるなら [akane-blue/mastodon](https://github.com/akane-blue/mastodon) にpushされたら自動でビルドして Docker Hub に上げてくれると便利ですよね。今なら [GitHub Actions](https://github.co.jp/features/actions) を使うのが簡単そうなので試してみました。

以下をリポジトリに `.github/workflows/push_docker_image.yml` みたいな名前で作ります。

```yaml
name: Push our docker image
on:
  push:
    branches: [ hota/master ]
  pull_request:
    branches: [ hota/master ]
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  push_to_hub:
    name: Push Docker image to Docker Hub
    runs-on: ubuntu-latest
    steps:
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      -
        name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_TOKEN }}
      -
        name: Checkout Git repository
        uses: actions/checkout@v2
        with:
          repository: akane-blue/mastodon
          path: mastodon
      -
        name: Build and push to Docker Hub
        uses: docker/build-push-action@v2
        with:
          context: mastodon
          push: true
          tags: ${{ secrets.REGISTRY_USERNAME }}/${{ secrets.REGISTRY_REPO }}:${{ secrets.REGISTRY_TAG }}
```

わたしの場合は `hota/master` ブランチだったのでこんな感じですが、適宜読み替えてください。

あとは GitHub リポジトリ上で Settings -> Secrets から `Repository Secrets` に以下の内容を設定する必要があります。

secret | 中身
---|---
`REGISTRY_USERNAME` | Docker Hub のユーザー名
`REGISTRY_TOKEN` | Docker Hub のトークン
`REGISTRY_REPO` | Docker Hub のリポジトリ名
`REGISTRY_TAG` | Docker イメージのタグ名

`actions/checkout@v2` の中でリポジトリがわざわざ指定されてたり、ここの `path` と `docker/build-push-action@v2` の `context` を揃えてるのは最初この Actions を別のリポジトリで管理しようとした名残です。結局、別リポジトリのpushをトリガーにするのがまあまあ面倒臭いことが判明したので akane-blue/mastodon 自身になりましたが…。

## スリム化

無事に docker image のビルドが自動化され、サーバー側はそれらを pull するだけでよくなり、ソースコードをcloneしておく必要がなくなりました。

結果として、現在は以下のファイルだけで構成されています:

*mastodon/*  
　├ **.env.production**  
　├ **docker-compose.yml**  
　├ *public/*  
　│　├ **announcements.json**  
　│　└ *system/*  
　└ *redis/*  
　 　└ **dump.rdb**  

`public/system/` 以下もメディアを外部に置いていれば特に増えないはず（まだ空）なので、すごくスッキリしましたね！
