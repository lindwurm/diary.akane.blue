---
title: メディアをAmazon S3に移行しました
date: 2020-05-06 12:35:00
tags: 
- メンテナンス
cover_img: /images/aws_logo.png
summary: 1年前にオブジェクトストレージに移行したメディアを今度はAmazon S3に移行しました。
---

タイトルの通りで、mstdn.maud.io のメディアファイルを今度は Amazon S3 へ移行したので、その記録です。

## 3周年ありがとう

[mstdn.maud.io](https://mstdn.maud.io/about) は 2020-04-12 を以て、無事にサービス開始から3周年を迎えることができました。皆様からのこれまでの応援に感謝するとともに、4年目になった mstdn.maud.io を引き続きよろしくお願いします。

## あれ、1年前にもオブジェクトストレージに移行してましたよね？

はい（[メディアをオブジェクトストレージに移行しました | 茜の鯖缶日誌](https://diary.akane.blue/2019/06/05/move-media-to-object-storage/)）。

前回はサーバのローカルストレージから [Spaces on DigitalOcean](https://www.digitalocean.com/products/spaces/) （以下 "DO Spaces"）に移行しました。最初こそ画像へのアクセス集中を回避できたことで快適に使えていたものの、当サーバで利用していたシンガポールリージョンにおいて秋冬頃からちょこちょこ障害が発生するようになったり、（公式にインシデントとしては上がってこないものの）どうにもアップロード/ダウンロードともに不安定なことが多発するなど、細かな不満が積もり始めていました。

4月末からいよいよアップロードの成功率が極端に下がるようになったこともあり、この度 DO Spaces から他のオブジェクトストレージへの移行を本格的に検討するようになりました。

## 1年ぶりの再検討

$5 前後の格安オブジェクトストレージはあれから更に増えたりした（[Scaleway](https://www.scaleway.com/en/object-storage/)のプランが変わったり、[Vultr](https://www.vultr.com/products/object-storage/)が新たに生えたりした）んですが、再び茨の道を選ぶのは避けておこうという前提はありました。これらと比べても一番近いのが今まで使っていたDigitalOceanのシンガポールリージョン(SGP1)だったので、どうしても今以上に遠くなってしまいます。

そこそこアツい話題としては単価がぶっちぎりで安い [Wasabi](https://wasabi.com/) が [NTT Comと組んで東京リージョンを設置した](https://www.ntt.com/about-us/press-releases/news/article/2019/0930_2.html)んですが、まだEnterprise Cloudの顧客向けにしか公開されてないので今回は選外です。[サポートKB曰く later in 2020 っつってる](https://wasabi-support.zendesk.com/hc/en-us/articles/360039372392-How-do-I-access-the-Wasabi-Tokyo-ap-northeast-1-storage-region-)ので気長に待ってればそのうち降ってくるとは思いますが、USの各リージョンもお値段なりみたいな感じの不安定さと聞いてるので、今のところ人柱の予定はないです…。

毎年再検討して引っ越して…というのはあんまりやりたいとは思わず、これ以上の選択肢は思いつかないという気持ちから、多少の費用は覚悟の上で [Amazon Simple Storage Service](https://aws.amazon.com/jp/s3/) (以下 S3) に移行することにしました。

> 転送が終わったくらいのタイミングでBackblaze（毎年HDD故障率レポートを公開していることで知られる）が自社クラウドストレージである [B2](https://www.backblaze.com/b2/cloud-storage.html) において、[S3互換API（パブリックベータ）の提供を開始する](https://www.backblaze.com/blog/backblaze-b2-s3-compatible-api/)ことをアナウンスしました。ぐぬぬ…

## メディアの転送

AWSに関しては死ぬほど情報が転がってるので登録からバケット作成の手順についてここでわざわざ書く必要はないでしょう。強いて言うならバケット名は使用するドメインに揃えるくらいかしら。

今回は別のサブドメインを切り直しました。 `s3-mstdn.maud.io` です。同じサブドメインで掛け変えると浸透待ちで新しいメディアにアクセスできないとかあったりして面倒な気がしたので…

### 容量削減

ここも前回とあまり変わりませんね。 `tootctl media remove` を活用して転送前に可能な限り減らしておきます。`--days=1` で 53GB ほど減りました。

`preview_cards` も tootctl から削除できますが、これは S3 に転送しなければ済む話なので今回は触らないでおきます。削除もめちゃ時間かかるので…（具体的にはファイル数がメディアの10倍、サイズは同じく50GB強といったところ）。

### いざ転送

前回同様に対話型セットアップが便利な [rclone](https://rclone.org/) を使います。これも使い方は前回を見てください。

今回は `preview_cards` 以外を転送していくんですが、手元で `--exclude` がうまく行かなかったので今回は [Filter rule](https://rclone.org/filtering/#filter-from-read-filtering-patterns-from-a-file) を雑に書いて使用しました。

```txt
+ accounts/**
+ custom_emojis/**
+ media_attachments/**
+ site_uploads/**
- preview_cards/**
```

そんで `--filter-from <filter-file>` でこう。

```bash
$ rclone copy spaces:media-mstdn-maud-io s3:s3-mstdn.maud.io -P --filter-from s3SyncFilter.txt
```

ひとまず151.6GBを転送しましたが、58時間ほどかかりました…

### 切り替え

うーん、ここも前回と一緒ですね。`.env.production` の項目を書き直してサービス再起動で反映。おわり。

試しに画像を添付してみましたが、大丈夫そうですね。

最大で1日前くらいまでの画像が見えない状態になっていましたが、一晩 `rclone copy` 回してたらだいたい直ったと思います（執筆現在もまだ回してますが、消えてるものがあったら教えて下さい）。

## S3の前にCloudflareを噛ませる

同じAWSで固めてCloudFront使うとか、適当なVPS用意してnginxでキャッシュさせるとか、いろいろな方法がありますが、今回は ~~金も手間もケチって~~ Cloudflareを使います。前回同様にPage ruleを書いておきましょう。

他に DO Spaces と比べての注意点としては、Cloudflareで生成したOrigin certificateが使えないので、SSL/TLS encryption modeを `Full` 以下にすることになるあたりでしょうか。

更にS3側で `static site hosting` を有効にするとエンドポイントがHTTPしか提供されないので、Cf側を `Flexible` まで落とすことになるのと、`Always Use HTTPS` を有効にしたほうが良いと思います（後者、Page rulesで使おうとすると他のルールが書けなくなるのでゾーン全体への適用になります…）。

DNSのほうは CNAME に `{bucket}.s3-website-{region}.amazonaws.com.` (static site hostingのエンドポイント)を指定する感じになりました。

## その後

- メディアのアップロードは大幅に改善されたように思います。
    - コケることが無くなったのはもちろん、経路的に近くなったことで速度もそこそこには
- 読み込みも気持ち早くなったのではないでしょうか
    - 感想お待ちしております
- 実運用コストについてはある程度続けてからといった感じ。
    - とりあえず移行にあたっての費用は20-30ドルくらいは覚悟してるとこです。
        - `PutObject` はもちろんのこと、rcloneがcheck走らすときに `ListBucket` をゴリゴリ使ったぽくて少し震えています。

## 追記: 更にその後 (2020/07/10)

- 移行費用ですが、およそ35ドルになりました。
- その次、6月分は29ドルほどでした。
    - 30日だったことを考えると他の月も30ドルくらいで済むんじゃないかなと思っています
- [WebARENA](https://web.arena.ne.jp/)がWasabiの取り扱いを始めました。
    - https://web.arena.ne.jp/wasabi/
    - 現状、個人で東京リージョンが利用できる唯一の方法ですね
    - [年内は1TB分無料のキャンペーン](https://web.arena.ne.jp/campaign/2020/cam_wasabi/)も実施中
    - 通常料金は 1TBまで税込834円/月 、安さのためにサポート/SLA無しとのこと
        - 本家が $5.99 なのを考えるとアレだけど、東京リージョンのパフォーマンスと可用性次第かなあ
        - 今突撃して人柱になるか、本家サイトから東京リージョン扱えるようになる頃まで評判とか様子見するかは自由
- 特にここから変える理由はないので、当サーバーはAmazon S3で続投予定です。
