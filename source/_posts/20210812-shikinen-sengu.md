---
title: サーバーの式年遷宮を執り行いました
date: 2021-08-12 19:00:00
tags:
- メンテナンス
cover_img: /images/cloud_db.png
summary: 稼働から4年経ったので立て直す回
---

タイトルの通りで、 mstdn.maud.io のサーバーを立て直して移設しました。

## 式年遷宮とは？

> 遷宮（せんぐう）とは、神社の本殿の造営または修理の際に、神体を従前とは異なる本殿に移すことである。
> 定期的な遷宮を式年遷宮（しきねんせんぐう）と言う。「式年」とは「定められた年」の意。
>
> https://ja.wikipedia.org/wiki/%E9%81%B7%E5%AE%AE

というわけで、我らが mstdn.maud.io もサービス提供開始から4年経って闇っぽい部分(後述)とか積もってきたので収容ホストを作り直して身軽になろう、ということで取り掛かりました。

たぶん今後も遅くても4年を目処にやることになりそうなので、実質遷宮じゃん、ということでそう呼んでいます。1回目なのに式年とは…とか言わない。

## :don: が抱えるサーバー構成の闇

今までのサーバーは [2017年8月](https://mstdn.maud.io/@hota/4395394) に立てたもの（初年ゆえか、試行錯誤の果てに、ここまでに3回くらい引っ越しています）で、その後 [ディスク容量を拡張したり](https://diary.akane.blue/2018/01/16/server-disk-resized/) して今に至ります。

当時のLTSな Ubuntu Server というとつまり…そういうことです。2021年にもなって **Ubuntu 16.04.x** が動いていました。もう既に闇ですね。よいこのみんなはサポート切れる前にきちんと計画的な式年遷宮をしようね。遅くても4年経って2つ次のLTSまで出てきたら変え時じゃないですか？

また、Ubuntu といえば `do-release-upgrade` があると思いますが、これによるアップグレードをしなかった理由もいくらかあります。ひとつがソフトウェア構成で、確か TLS 1.3 対応のために OpenSSL や Nginx が PPA 経由でインストールされたものであったこと、ふたつめがオブジェクトストレージに移行した後だったのでその分必要なくなったディスクサイズを小さくして作り直したいと思っていた、などです。

## 構成の見直し: データベース

さて、サーバーの引っ越しは確定しました。次は何を持っていき、何を持っていかないかです。メディアは前回外に出しました。今度はデータベースかなあ、というわけでマネージドDBの利用を検討します。

今回はサーバーと同じさくらのクラウドの [アプライアンス「データベース」](https://manual.sakura.ad.jp/cloud/appliance/database/about.html) を利用します。サーバーと同じルーター/スイッチの下に配置でき、接続元ネットワークをローカルに制限可能なので便利です。

PostgreSQL は 12 系のみが利用可能となっていますが、実際に接続したところ 12.1 でした。

## 式年遷宮（本番）

そうこうしている間にテスト用サーバーのほうでもDBの移行ができたので、本番環境もやっていきましょう。

事前の告知は十分に行います。

<iframe src="https://mstdn.maud.io/@hota/106674187597916491/embed" class="mastodon-embed" style="max-width: 100%; border: 0" width="600" height="400" allowfullscreen="allowfullscreen"></iframe><script src="https://mstdn.maud.io/embed.js" async="async"></script>

道路工事の看板風なのはわたしがそうしたかったからです。一度はやってみたいよね。まあまあウケが良くてよかったです。

サービス停止後にまずは [メンテナンスページの表示](https://mstdn.maud.io/mente.html) に切り替えます。雑。もっといい方法が知りたい。

```nginx.conf
location / {
  allow xxx.xxx.xxx.xxx; # 自宅IPに制限
  deny all;
}

error_page 403 =503 /mente.html;
location = /mente.html {
  internal;
  allow all;
}
```

データベースは普通に `pg_dump` します。しました。85GBあったDBは45分くらいで47GBのファイルを吐きました。思ったより短かったですね。

タイムラインが壊れると悲しいので Redis もダンプを取ろうとしたんですが、よく考えたら Docker 環境では `volumes` で [指定されたパス](https://github.com/mastodon/mastodon/blob/main/docker-compose.yml#L23) に `dump.rds` が残っているので、これを新しいサーバーでも同じように置き直すだけでよかったです。楽ですね。

データベースのリストアに一番時間がかかりましたが、全体としては短く済んで良かったと思います。4時間9分でメンテ明けです。