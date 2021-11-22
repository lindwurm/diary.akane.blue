---
title: GitHub ActionsでMastodonのdocker buildを自動化する
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

特に自分たち以外で使うこともないだろう[^2] ということでプライベートリポジトリにしようと思っていたのですが、今流行りの GitHub Packages は Free だとプライベートリポジトリで利用可能なストレージが [500MB まで](https://github.com/features/packages) で、前述の tootsuite/mastodon を見る限り無理そうだったので、かと言ってこれだけのために自前でサーバーを立てようとはあんまり思えずにいたのですが、さくらのクラウドのLabプロダクトとして [コンテナレジストリ](https://manual.sakura.ad.jp/cloud/appliance/container-registry/index.html) が提供されていたので、ここに立てました（非公開）。

[^2]: favicon や apple-touch-icon など弊サーバーに特有のアセットが含まれるため他所で使われることがない

## GitHub Actions

やるなら [akane-blue/mastodon](https://github.com/akane-blue/mastodon) にpushされたら自動でビルドして Docker Hub に上げてくれると便利ですよね。今なら [GitHub Actions](https://github.co.jp/features/actions) を使うのが簡単そうなので試してみました。

> 省略された試行錯誤の記録は https://wiki.maud.io/ja/poem/github-actions を、最新のは https://github.com/akane-blue/mastodon/blob/hota/master/.github/workflows/push_docker_image.yml を参照してください。

`.github/workflows/push_docker_image.yml` を生やします。

```yaml
name: Push our docker image
on:
  push:
    branches:
      - hota/**
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  push_to_hub:
    name: Push Docker image to Docker Registry
    runs-on: ubuntu-latest
    steps:
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      -
        name: Login to Docker Registry
        uses: docker/login-action@v1
        with:
          registry: ${{ secrets.REGISTRY_SERVER }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_TOKEN }}
      -
        name: Checkout akane-blue/mastodon
        uses: actions/checkout@v2
        with:
          repository: akane-blue/mastodon
          path: mastodon
      -
        name: Get latest branch name
        id: set_tag
        run: |
          cd mastodon
          DOCKER_TAG="$(echo ${GITHUB_REF} | sed -e 's/.*\///g' -e 's/-/_/g')"
          # Use Docker `latest` tag convention
          [ "$DOCKER_TAG" == "master" ] && DOCKER_TAG=latest
          echo "::set-output name=docker_tag::${DOCKER_TAG}"
      -
        name: Build and push to Docker Registry
        uses: docker/build-push-action@v2
        with:
          context: mastodon
          push: true
          tags: ${{ secrets.REGISTRY_SERVER }}/${{ secrets.REGISTRY_REPO }}:${{steps.set_tag.outputs.docker_tag}}
```

`REGISTRY_SERVER`, `REGISTRY_USERNAME`, `REGISTRY_TOKEN`, `REGISTRY_REPO` とやたら secrets に突っ込んでいますがゆるして。

### 追記

最近は mastodon/mastodon も GitHub Actions を使うようになったのですが fork 先で元々使ってた身からすると要らないので、他の `.github` 周りも含めて `.gitignore` に突っ込んで消しておきます。

```.gitignore
# This is forked repository, no need to use dependabot
/.github/*
# but we want to use GitHub Actions
!/.github/workflows/push_docker_image.yml
```

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

## 参考

- [GitHub Actionsのワークフロー構文 - GitHub Docs](https://docs.github.com/ja/actions/reference/workflow-syntax-for-github-actions)
- [ワークフローをトリガーするイベント - GitHub Docs](https://docs.github.com/ja/actions/reference/events-that-trigger-workflows)
- [Dockerイメージの公開 - GitHub Docs](https://docs.github.com/ja/actions/guides/publishing-docker-images)
- [環境変数 - GitHub Docs](https://docs.github.com/ja/actions/reference/environment-variables#default-environment-variables)
- [GitHub Actions を使って docker build し Docker Hub に push すると、速くて良い | tbsmcd.net](https://tbsmcd.net/post/github-docker-build-and-push/)
