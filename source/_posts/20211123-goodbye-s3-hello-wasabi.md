---
title: メディアをWasabiに移行しました
date: 2021-11-23 06:35:00
tags: 
- メンテナンス
cover_img: /images/wasabi.png
summary: 今度はS3からWasabiに移行しました…
---

タイトルの通りで、mstdn.maud.io のメディアファイルを今度は Wasabi (+ Cloudflare) へ移行したので、その記録です。

## あれ、去年オブジェクトストレージ事業者引っ越したばかりですよね？

はい（[メディアをAmazon S3に移行しました | 茜の鯖缶日誌](https://diary.akane.blue/2020/05/06/migrate-spaces-to-s3/)）。

Amazon S3 + CloudFront 構成はほぼ無敵と言った感じで、丸一年（S3+Cloudflareの期間含めると1年半）続けて使用感において不満はありませんでした。使用においては。

### 何がよくなかった？

問題は（やっぱり）金銭的コストで、とりあえず1年やってみたんですが CloudFront は高い…。あんまり連合してない個人サーバならまだ勧められるレベルで済むかもですが、弊サーバの場合は外向きの配信量が月に 1TB 以上あるのでメディアだけで**毎月 $110-150** の請求が来ていました。連合リレーに参加するのをやめてこれなので、参加していた場合のコストは考えたくもないですね…

CloudFront の代わりに別のサーバーを用意して Nginx や Varnish でキャッシュするとか、そこに Cloudflare 挟むとかも一瞬考えましたが、個人的には S3 のオリジンアクセスを絶対に無くしたかったのでオブジェクトを公開可能に落とすのが許容できませんでした。こんなことで神経質になるくらいなら転送料であんまり心配しなくていい他の事業者に乗り換えるほうが金銭的にも精神的にも楽になれるのでは、という感じで今回に至ります。

## もはや毎年恒例、再検討

### [Backblaze B2](https://www.backblaze.com/b2/cloud-storage.html)

- 保存 $0.005/GB, 転送料 $0.01/GB
- S3互換API を提供するようになった
- B2 <-> Cloudflare 間では [転送料課金無し](https://www.cloudflare.com/ja-jp/bandwidth-alliance/)
- [独自ドメイン非サポートらしい](https://scrapbox.io/heguro/Backblaze_B2_%E3%81%AB%E7%8B%AC%E8%87%AA%E3%83%89%E3%83%A1%E3%82%A4%E3%83%B3%E8%A8%AD%E5%AE%9A%E3%81%8C%E3%81%AA%E3%81%84)
- 採用事例が少なすぎて情報に乏しい
  - なくはない: [MastodonでBackblaze B2を使う](https://scrapbox.io/heguro/Mastodon%E3%81%A7Backblaze_B2%E3%82%92%E4%BD%BF%E3%81%86)
- ところでどこにあるの？

### Wasabi

- `ap-northeast-*` は $6.99/TB (ほかは $5.99/TB), 転送料なし
- 法人オンリーだった東京リージョンが個人も [WebARENA](https://web.arena.ne.jp/wasabi/) から申し込めるようになったり
  - 良くも悪くも、為替による請求額の変動がない
  - SLAやサポートなし
- 年明けてからいつの間にか本家サイトからも東京リージョン触れるようになり
- [東京に APAC オフィスが生えたり](https://wasabi.com/press-releases/wasabi-expands-global-presence-announces-apac-headquarters-in-japan/)
- 東京に続き、大阪リージョン (`ap-northeast-2`) が [開設された](https://wasabi.com/press-releases/wasabi-technologies-accelerates-apac-operations-and-growth-with-new-storage-region-in-osaka/)
- USしか使えなかった頃からちょこちょこ採用例はある
  - 国内リージョンでの話はあんまり聞かないしTwitterでも見ない…
- 実はテスト環境 `staging.mstdn.maud.io` が使ってる
- **テスト環境で大きな問題は起きてなかったので採用**

### 選外

#### GCP, Azure

いや使ったところで誤差でしょ

#### その他安いやつ

- DigitalOcean Spaces
  - もうシンガポールはこりごりだよ～
- Linode
  - お前も [最寄りシンガポール](https://www.linode.com/docs/products/storage/object-storage/) なのか
- Vultr
  - まだ New Jersey しかない…
- Scaleway
  - Paris, Amsterdam, Warsaw
  - 全部遠い

そんな感じで Wasabi にしました。

## メディアの転送

バケット名を使用するドメインに揃えておくと CNAME 向けるだけで済んでお得。ドメインは前回と同じで、 `s3-mstdn.maud.io` です。 *前回はご迷惑おかけしました…*

### 容量削減

まず `tootctl media remove-orphans` でどこからも参照されずゴミになっているメディアを消して回ります。

```bash
$ tootctl media remove-orphans

Removed 104716 orphans (approx. 23.1 GB)
```

... なんで 23.1 GB も減るんですか。


あとはいつも通り `tootctl media remove` を活用して転送前に可能な限り減らしておきます。


```bash
$ tootctl media remove 

Removed 449271 media attachments (approx. 157 GB)
```

そうはならんやろ（なっとるやろがい）。

`preview_cards` も tootctl から削除できますが、今回も転送しないため触らないでおきます。削除もめちゃ時間かかるので（二回目）。

### いざ転送

前回や前々回同様に対話型セットアップが便利な [rclone](https://rclone.org/) を使います。これも使い方は前々回くらいを見てください。

`rclone setup` では番号をぽちぽち選ぶだけで Wasabi の東京リージョンを設定でき、あの長ったらしいエンドポイントを入力する手間が [省けます](https://github.com/rclone/rclone/commit/839c20bb350adba8de897a777b11f6ab02206c30) 。 *数ヶ月前にPR投げといて正解でしたね！*

今回も [Filter rule](https://rclone.org/filtering/#filter-from-read-filtering-patterns-from-a-file) をめちゃくちゃ雑に書いて使用しました。

前回から変わった点として、リモートサーバーからのキャッシュ分は `cache/media_attachments/` みたいに分かれるようになったのでフィルターもぐちゃぐちゃになってきます。

#### 転送しないやつ

- `index.html` と `error.html`
  - 書き直したので
- キャッシュした添付メディア
- リンクのプレビューカード用画像

#### 転送するやつ

- ローカルのアカウント画像
  - アイコンとバナー
- リモートのアカウント画像のキャッシュ
  - アイコンとバナー
- ローカルのカスタム絵文字
- リモートのカスタム絵文字のキャッシュ
- ローカルの添付メディア
- サイト用画像

#### 結果

```txt
- *.html
- cache/media_attachments/**
- cache/preview_cards/**
- preview_cards/**
+ accounts/**
+ cache/accounts/**
+ cache/custom_emojis/**
+ custom_emojis/**
+ media_attachments/**
+ site_uploads/**
```

そんで `--filter-from <filter-file>` でこう。

```bash
$ rclone copy s3:s3-mstdn.maud.io wasabi:s3-mstdn.maud.io -P --filter-from s3filter.txt --s3-no-head

Transferred:      211.988 GiB / 211.988 GiB, 100%, 273.
Checks:           1243974 / 1243974, 100%
Transferred:      1241144 / 1241144, 100%
Elapsed time:  12h29m19.5s
```

12時間半で212GiBくらいを転送しました。速いですね。

## 切り替え

今回はドメインが同じなのもあって、差分が増えないうちに見える方から切り替えてしまいます。適当な告知をして、もう使わないDNSレコードを削除し、 CNAME を新しいエンドポイントに向けたらいつものオレンジの雲を有効にしておわり。SSL/TLS encryption modeは `Full` で動きましたね。

`index.html` にサービス名を入れているので、浸透の確認に使えました。

アップロードする側はいつも通りで、 `.env.production` の項目を書き直してサービス再起動で反映。おわり。今回はついでに Mastodon のアップデートも行いました。

## その後

### 使用感

- 東京にあるおかげか、極端にメディアのアップロードが遅いということは無いと思います
  - 気持ち遅くなったかもしれないというか、なってても文句言えない
- ダウンロードは（今のところ）遜色ないように見えます

後は、かつての Spaces や Wasabi の USリージョンみたいに障害が頻発しなければよいですね…

### コスト

- AWS から出ていくときにかかった費用は ~~後で決まったら書きます~~ 60ドルちょいでした。

| service | data | price |
|---|---|---:|
| Wasabi (Tokyo) | ~1TB | 6.99 USD |
| Cloudflare Pro | - | 20.00 USD |

*せーの、* **予測可能なコスト、最高～～～！**

### 追記

[もう少しだけ続くんじゃ…](/2021/12/02/welcome-back-to-s3/)