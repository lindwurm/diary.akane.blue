---
title: メディアの配信にCloudFrontを採用しました
date: 2020-10-01 06:50:00
tags: 
- メンテナンス
cover_img: /images/aws_logo.png
summary: メディアの配信に使用するCDNをAmazon CloudFrontに移行しました。
---

タイトルの通りで、mstdn.maud.io のメディアファイルの配信に Amazon CloudFront を採用しました。

## 今度は何？

前回は [メディアの保存先を Amazon S3 に移行した](/2020/05/06/migrate-spaces-to-s3/) わけですが、今度はこれらのメディアが実際に皆さんの元へと配信される経路である [CDN](https://ja.wikipedia.org/wiki/%E3%82%B3%E3%83%B3%E3%83%86%E3%83%B3%E3%83%84%E3%83%87%E3%83%AA%E3%83%90%E3%83%AA%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AF) を Amazon の提供する [CloudFront](https://aws.amazon.com/jp/cloudfront/) に移行しました。

## 作業内容

* AWS Certificate Manager (ACM) でドメインに対するSSL/TLS証明書の発行をリクエスト
    * CloudFront に適用できる証明書は `us-east-1` （米国東部/バージニア北部）の ACM で発行されたものに限るらしいので注意
        * https://docs.aws.amazon.com/ja_jp/acm/latest/userguide/acm-regions.html
    * CAA レコードを設定しておかないと発行が通らないので気をつけましょう（2敗）
        * https://docs.aws.amazon.com/ja_jp/acm/latest/userguide/setup-caa.html
* CloudFront で `Create Distribution` → `Web` を選ぶ
    * `Origin Domain Name` はエンドポイント(`バケット名.s3-ap-northeast-1.amazonaws.com`)で
    * `Restrict Bucket Access` は有効にしておくとバケットのオリジンURLを直で叩けなくできそうなので設定
    * `Viewer Protocol Policy` は `Redirect HTTP to HTTPS` に
    * `Price Class` は一応 `US, Canada, Europe, Asia, Middle East And Africa` 
        * もしこれらに該当しない地域からご利用頂いているユーザの方はご一報ください。`All Edge Locations (Best Performance)` にするかしばらく悩みます
    * `SSL Certificate` でさっき ACM から作った証明書を使うように
* CNAME レコードをさっき生やした `hogehoge.cloudfront.net` なドメインに向ける

## 何が嬉しい？

今までだと S3 から一旦外（インターネット）に出て Cloudflare にキャッシュされて、またインターネットを通して皆さんの元へ届けられていたところ、S3 から CloudFront 間が AWS 内で完結したことで、初めてファイルがキャッシュされるまでのタイムロスが軽減される…といいですね（目的の半分）。

見栄えの問題としては、カスタム404ページがいい感じに機能するようになりました。存在しないパスを開いても常に https://s3-mstdn.maud.io/index.html の中身が表示されます。

あとは ACM で証明書が発行されているので発行者が `Amazon` になっていてかっこいいかもしれない（？）。

## 費用面の話

Cloudflare では通信量が月に2TB近くても無料で利用できたんですが、CloudFront では幾らかかるのか気になって夜も8時間しか眠れません。

とはいえ、旧構成では S3 → Cloudflare 間の最初にキャッシュされるまでの通信量が月に 150-200GB だったので、今回そこの通信料が無くなるのと合わせてどれくらいの負担になるかですね…

減らせるものは減らしておこうと考えた結果、一時的に連合リレーへの接続を取りやめています。余裕があれば戻すかも。

## 追記: S3 へのオリジンアクセスを拒否して CloudFront 経由のみに限定する

悪意のある第三者から S3 のオリジンに死ぬほどアクセスされて青天井…なんてことは絶対に起きてほしくないものです。

しかし前の構成では静的サイトホスティングを有効にしてCloudflareにキャッシュさせていたため、多くのファイルが公開状態にありました。

今回、CloudFront Origin Access Identity (OAI) と Mastodon で用いる IAM ユーザにのみアクセスを許可するバケットポリシーを書いて、それ以外のパブリックアクセスはブロックするように設定しました。

<script src="https://gist.github.com/lindwurm/95b2ef27f42e0ec38c44a098b0c75495.js"></script>

ブロックパブリックアクセスは2つ目の項目、 **任意のアクセスコントロールリスト (ACL) を介して許可されたバケットとオブジェクトへのパブリックアクセスをブロックする** のみ **オン** にしています。

![](/images/s3_blockaccess.png)

> ちなみにこれを知るまでの少しの間、以下のような最悪バケットポリシーを書いていました。CloudFront OAI と Mastodon で用いる IAM ユーザ （と、サーバIP）以外からの GET アクションを全てブロックする意図で書かれましたが、この書き方に行き着くまでの時間をだいぶ無駄にした気がします。

<script src="https://gist.github.com/lindwurm/fe5220bc4acae46ea52569a534138683.js"></script>