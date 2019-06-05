---
title: メディアをオブジェクトストレージに移行しました
date: 2019-06-05 19:38:28
tags:
- メンテナンス
cover_img: /images/spaces_on_digitalocean.png
---

タイトルの通りで、先日ようやく mstdn.maud.io サービス開始以来、約2年強のメディアファイルを引っさげて無事にオブジェクトストレージへ移行したので、その記録です。

## はじめに

この記事は 2019-05-18 に開催された [東海道らぐ横浜の集い 2019春の巻](https://tokaidolug.connpass.com/event/128671/) での発表『[mstdn-maud-io を支えるストレージの話](https://speakerdeck.com/lindwurm/tokaidolug-yokohama-201905)』で使用したスライドと口頭で補った内容、また移行から約1ヶ月が経過した現在の話から成るものです。

## 2周年ありがとう

mstdn.maud.io は 2019-04-12 を以て無事にサービス開始から2周年を迎えることができました。  
3年目になった mstdn.maud.io をこれからもよろしくお願いします。

## 2年経過しての悩み

Webサービスというものは、運用開始から2年も経過すると様々な問題や悩みが出てきます（よね？）。  
このサーバなら何かと言えば、やはり最初に挙がってくるのはストレージ容量でしょう。[2018年1月にディスクフルに陥った回](https://diary.akane.blue/2018/01/16/server-disk-resized/) などは記憶に新しいかと思います。このときは250GBのディスクを使い切って500GBのものに切り替えたんですが、2周年を迎えた時点では320GBが使用済みという状況でした。

![移行前に70%=350GBを超えた回もありました](/images/filesystem_alert.jpg)

一般的なMastodonインスタンスの運用において、一番ストレージを食うのはメディアでしょうから、この調子では数年内に再び今のディスクも使い切ってしまうのではないか、という心配をするのは当然でしょう。ディスクを再び使い切ったときに取れる対策はあるでしょうか？

まず、前回と同じくより大きなサイズのディスクでクローンをして切り替える方法はありますが、次のような理由から避けたいと思うはずです:

- まずコピーに時間がかかる。前回は200GBで約1時間、次に500GB使い切った時は…？
- 作業の手間。これは私事ですが、前回は無職で現在は学生、来年度には（きっと）社会人です。
- ディスクの月額費用。次の容量のお値段は…考えたくないですね。

この方法は当時の知識に限りがあり、サービス復旧が優先であった前回の一度限りとすべきでしょう。

## 再考: オブジェクトストレージ

オブジェクトストレージへの切替は、前回も一応案としては出ていましたが、触ったことがないことを選外の主な理由としていました。今回はそこまで急ぎではないので、やるべきことをしっかりまとめて取り組む余裕があります。

オブジェクトストレージの利点とか優位性についてはここでは省略します（補足: スライドに書いてあるリストはIaaS型事業者から借りて利用する場合の金銭面に焦点を当てた話であり、それぞれの原理とは別の話です）。

結果として容量の心配をする必要がほぼなくなり、他の方法よりも安価に解決できるという点でオブジェクトストレージへの切替を検討することになりました。

## 事業者選定

一応雰囲気だけの試算があるんですが、これについては

- 目安として200GB程度の使用を想定
- 月毎の外向きの転送量は把握していない
    - 当時サーバ自体の移行も別件で検討しておりrsyncによる同期がちょくちょく走っていたので、 `eth0` の `TX bytes` とかを雑に見るだけでは把握ができなかった

という雑なやつだったのであんまり金額は参考にしないで良いと思います。


### Amazon S3

> https://aws.amazon.com/jp/s3/

- GAFAのA（どっちの？）
- 有名
- たぶん強い
- 東京リージョンがある
- 転送量を把握してないので実際の請求額に不安があり保留
    - リクエスト単位が初めてだとわかりにくすぎる

**200GB利用時: $5 (ストレージ分) + リクエスト料金**

面倒なので [料金 - Amazon S3](https://aws.amazon.com/jp/s3/pricing/) をみて。

### Google Cloud Storage

> https://cloud.google.com/storage/

- GAFAのG
- たぶん強い
- 東京リージョンとかある
- なんかあったときに自分のGoogleアカウントを巻き込みたくないなーというだけの理由で外されたはず（ひどい）

**200GB利用時: $5.2 (ストレージ分) + 下り転送量課金 ($0.12/GB)**

### Wasabi

> https://wasabi.com/

- S3互換を謳っている
- 安い
    - 保存が $5.99/TB で転送量課金無し
    -  最初1TBまではこの額固定で、以降は従量課金($0.0059/GB)
        > Wasabi has a minimum monthly charge associated with 1 TB of storage ($5.99/month). If you store less than 1 TB of active storage in your account, your total charge will still be $5.99/month.
        > https://wasabi.com/pricing/pricing-faqs
- とにかく安いからか利用しているインスタンスはちらほら見かける
- リージョンがアメリカの東西とヨーロッパの3箇所しかなく、どれも遠い
    - CDN噛ませば読み込みはまだマシになるかもしれないけどアップロードがつらそう

**200GB利用時: $5.99**

### ConoHa オブジェクトストレージ

> https://www.conoha.jp/objectstorage/

- ご存知GMOグループのVPS屋さんですね
- OpenStack Swiftっぽい
- 転送量課金なし
- 450円/100GB
    - Wasabi見た後だと高く感じてしまう…

**200GB利用時: 900円**

### Spaces on DigitalOcean

> https://www.digitalocean.com/products/spaces/

- VPSとか提供してるクラウド事業者として知られてる方だと思う
- S3互換
    - Cephっぽい
        > Spaces is built with Ceph, just like block storage.
        > https://www.digitalocean.com/docs/spaces/overview/#high-availability
- 提供開始は2017年くらいと後発め
- WebUIにファイルブラウザがあるっぽい
- CDNが追加料金無しで利用できる
    - PoP(Points of Presence, エッジサーバのある配信拠点)がbetaだけど東京にもある
        > Asia: Hong Kong (beta), Manila (beta), Seoul (beta), Singapore (beta), Tokyo (beta)
        > https://www.digitalocean.com/docs/spaces/overview/#regional-availability
- サブドメインの割当はSpaces側でできる
    - ドメインのDNS丸ごとDigitalOceanで管理するようにするとLet's Encryptが使える
    - 自前でSSL証明書持ち込むならそれでもよい
- そこそこ安い
    - $5 で保存が250GBと外向き転送量（CDN含む）1TB/monthまで
    - 以後従量課金(保存 $0.02/GB, 転送 $0.01/GB)
- 最寄りのリージョンはシンガポール(`SGP1`)かなあ
    - 他にはSan Francisco, New York, Amsterdam, Frankfurt
- 最近周りで使い始めたインスタンスが増え始めた
    - サンプル数: 2

**200GB利用時: $5 + 下り1TB超過時の転送料 $0.01/GB (CDNを含む)**

### おまけ: Scaleway

> https://www.scaleway.com/en/object-storage/

全部終わった後に知ったので選択肢になかったんですが、比較として書いておく価値はあるかもしれないと思ったので。

- ARMやAtomを採用したVPS/専用サーバのホスティングで知られるとこ
- S3互換
- betaが取れて正式サービスとして始まったのは2019年の2月？
    - https://blog.scaleway.com/2019/object-storage-general-availability/
- そこそこ安い
    - €5 で保存が500GBと外向き転送量500GB/monthまで
    - 以後従量課金(保存 €0.01/GB, 転送 €0.02/GB)
    - 価格的にはSpacesとタメ張ってるようでいて、ストレージとネットワークに対する優先度が逆っぽいのが興味深い
- リージョンはパリ（フランス）とアムステルダム（オランダ）の2箇所
    - 日本からは遠いかなあ

### 最終的に

Amazon S3とGoogle Cloud Storageは転送量の試算を諦めて早々に選択肢から外れました。ストレージ代だけで転送料含めたWasabiやSpacesとお値段並ぶんじゃ転送料分まるまる高くつくようなもんだしなあ…のお気持ちになってしまったので。  
確かに可用性とかの面では明らかにこれらを選んだほうが強いとは思うのだけれども、サーバ運用費について寄付を受け付けない方針を堅持しながらだと懐には限度があるのでゆるしてほしいです。

ConoHaは個人鯖とか、100GB以内で済む自信があるなら何かと都合は良いのかなあとは思います。200GBとか300GBとか増えていくと他との差がめちゃめちゃ開いてしまうので苦しいかなあという感じで除外。

んでWasabiとSpacesみたいな格安オブジェクトストレージ頂上決戦みたいな感じになってしまったんですが、太平洋超えるよりはシンガポールのほうがマシかなあという感じで **Spaces on DigitalOcean** になりました。

![](/images/spaces_on_digitalocean.png)

## 移行の準備

### アカウント作成

DigitalOceanのアカウントを作成します。わたしのときは招待リンク経由で60日間有効な100ドル分のクレジットがもらえたんですが、その [2週間後に変更](https://www.digitalocean.com/docs/platform/release-notes/#may-2019) があって、現在は30日間有効な50ドル分のクレジットになったようです。また、[GitHub Student Developer Pack](https://education.github.com/pack) でも同額のクレジットを得られるので（併用不可っぽい）、大学生の方は活用するとよさそう。

> ちなみに招待リンクは https://m.do.co/c/2a16162bd95a です。

### Projectを作る

DigitalOceanで提供してるVPSとかマネージドDBとかの他サービスを横断的にまとめる単位っぽいですね。ここにしか使わないのであんまり意味はないんですが、作らないと始まらないので適当に１つ作っておきます。

### Spaceを作成

S3で言うところのバケットですね。DOはこういうときにサービス名で呼びたがるらしい。サーバ作成も "Create Droplets" だし。

![](/images/create_space.png)

最初にリージョンを選びます。最寄りはシンガポールですね。  
次にCDNを使ったり使わなかったりを決めます。わたしは使うことにしました。独自ドメインがあるならここでサブドメインの割当を設定できます。  
あとは最後に名前と、どのProjectに属するのかを決めておわりです。簡単ですね。

## メディアファイルを転送する

今までのメディアファイルもオブジェクトストレージに全部アップロードしてやることになるんですが、100GB超えの転送はなかなかにしんどいので、できる限り転送前に減らしておきたいですよね。

### メディアを減らす: 基本編

まずはご存知 `tootctl media remove` です。Mastodonインスタンスを運用したことがある方ならおわかりかと思うんですが、メディアを減らすと言えばまずは外のインスタンスから飛んできた投稿についてくるメディアのキャッシュをまずは減らしますよね？というところです。

デフォルトでは直近7日分を残してそれ以前を削除しますが、 `--days=N` オプションで日数を指定できるので有効活用していきましょう。削除が走るので、実行前には `--dry-run` で確認をおすすめします。

ここはDockerで運用しているので、実際に叩くとこうですね:

```bash
$ docker-compose run --rm web bundle exec bin/tootctl media remove --days=1
>Removed 321752 media attachments (approx. 87.6GB)
```

**>Removed 321752 media attachments (approx. 87.6GB)**

🤔 🤔 🤔 🤔 🤔 🤔 🤔 🤔 🤔 🤔 (X-Files Theme BGM)

なぜか87.6GBも減りましたね（？？？）。cronで定期的に削除が走るようにはしていたはずだったんですが、crontabの記述に不備があったんですかね…

### 減らす: 応用編？

今のでもだいぶ減ったんですが、もう少し減らせるところがないか見てみます。例えば投稿内にWebサイトへのリンクが貼られていたとき、タイムライン上にカードが表示されますよね？カードに表示される画像もサーバ上に保存されているわけですが、特に連合経由で流れてきたものに関しては無駄に感じるかもしれません。

![こういうの](/images/preview_card_sample.png)

今回は `find` コマンドを活用して、一定期間（ここでは7日）よりも古いものを削除してみます（確認がしたいだけであれば、`-delete` を付けずに実行すると対象をひたすら列挙するだけになります）。

> 参考: https://angristan.xyz/moving-mastodon-media-files-to-wasabi-object-storage/  
> (ただし、記事中のコマンド通り `-mtime 7` だと最終更新が "7日前" に該当する24時間分のファイルを削除するだけなので、 `+7` で "7日前より過去" を指定すべきじゃないかなと思います)


```bash
$ find public/system/preview_cards/ -mtime +7 -type f -delete
```

うちは `public/system/preview_cards/` が40GBくらいあって、30GBくらいがこれで減りました。

ところで `preview_cards` の画像の再取得タイミングについて把握をしないままに実行したので、あんまり手放しで勧められる方法ではないです。ご注意ください。

## 同期: rcloneのセットアップ

メディアを十分に減らしたところでオブジェクトストレージ側に転送していきましょう。

S3やその互換向けのツールはいくらでもあると思うんですが、今回は rsync っぽいノリでクラウドストレージを扱えるらしい [rclone](https://rclone.org/) というのを使ってみました。

`rclone config` って叩いたときの対話型のセットアップがすごくやさしいので見て欲しい。

```bash
$ rclone config

No remotes found - make a new one
n) New remote
s) Set configuration password
q) Quit config

n/s/q> n

name> spaces

Type of storage to configure.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
 1 / A stackable unification remote, which can appear to merge the contents of several remotes
   \ "union"
 2 / Alias for a existing remote
   \ "alias"
 3 / Amazon Drive
   \ "amazon cloud drive"
 4 / Amazon S3 Compliant Storage Provider (AWS, Alibaba, Ceph, Digital Ocean, Dreamhost, IBM COS, Minio, etc)
   \ "s3"
(略)
27 / http Connection
   \ "http"

Storage> s3

** See help for s3 backend at: https://rclone.org/s3/ **
Choose your S3 provider.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
 1 / Amazon Web Services (AWS) S3
   \ "AWS"
 2 / Alibaba Cloud Object Storage System (OSS) formerly Aliyun
   \ "Alibaba"
 3 / Ceph Object Storage
   \ "Ceph"
 4 / Digital Ocean Spaces
   \ "DigitalOcean"
 5 / Dreamhost DreamObjects
   \ "Dreamhost"
 6 / IBM COS S3
   \ "IBMCOS"
 7 / Minio Object Storage
   \ "Minio"
 8 / Netease Object Storage (NOS)
   \ "Netease"
 9 / Wasabi Object Storage
   \ "Wasabi"
10 / Any other S3 compatible provider
   \ "Other"

provider> 4
```

種別で「S3またはその互換」を選んだら該当する事業者がばーっとリストで出てきて、じゃあDigitalOcean、っつったらその後で「endpoint指定する時DOなら以下のリージョンがあるよ？何番？」って聞いてくれて入力が最低限で済むんですよ。やさしさの塊。

```bash
Endpoint for S3 API.
Required when using an S3 clone.
Enter a string value. Press Enter for the default ("").
Choose a number from below, or type in your own value
 1 / Digital Ocean Spaces New York 3
   \ "nyc3.digitaloceanspaces.com"
 2 / Digital Ocean Spaces Amsterdam 3
   \ "ams3.digitaloceanspaces.com"
 3 / Digital Ocean Spaces Singapore 1
   \ "sgp1.digitaloceanspaces.com"

endpoint> 3
```

それで最終的なconfigは `spaces` という名前でremoteが定義された形になります。

```
[spaces]
type = s3
provider = DigitalOcean
env_auth = false
access_key_id = ********************
secret_access_key = ****************************************
endpoint = sgp1.digitaloceanspaces.com
acl = public-read
```

`rclone lsd <remote>:` ってやるとバケットの一覧が見れるので、`spaces:` って指定したときにちゃんと自分の作ったバケットが出てくればオッケーです。

```bash
$ rclone lsd spaces:
2019-05-02 08:53:04        media-mstdn-maud-io
```

あとはこんな感じで同期を走らせます。80GB弱で24時間ちょっとですかね。

```bash
$ rclone sync ~/mastodon/public/system spaces:media-mstdn-maud-io -P
```

## 切替作業

### 切替こわくないよ

だいたいのファイルを同期できたら、今後のアップロード先とかの設定をオブジェクトストレージ側に向けていく切替をやります。

ここで、無停止に近いかたちでやろうとすると、切替直前とかの投稿分がオブジェクトストレージ側にまだ同期されていなかったりとかで間に合いませんが、そのへんは割り切りましょう。仕方のないことです。消えるわけではないので、事前にサーバの利用者からは許しを得ましょう。

同期がだいたいできてから早めにやるに越したことはないです。時間が開けば開くだけ「ローカルにあってオブジェクトストレージにない」メディアが増えるので、当然あとでもう一度同期させるときの量が増えますよね。上で言及した分の対象が広がるので、利用者的にも嬉しくないです。

最初はわたしもサービスを停止してから完全に同期を完了させて切り替えるかなあ、というつもりだったんですが、切替自体は設定ファイルの変更とその反映なので、アップデートに伴う瞬断くらいの時間で終わると思います。

### 作業

やることは簡単で、S3を使用する場合に必要な項目が `.env.production` に用意されているので、これを埋めてそれを反映させたら終わりです。

```
# S3 (optional)
S3_ENABLED=true                                     # コメント解除
S3_BUCKET=media-mstdn-maud-io                       # バケット名
AWS_ACCESS_KEY_ID=**********                        # アクセストークン
AWS_SECRET_ACCESS_KEY=**********                    # シークレットキー
# S3_REGION=                                        # Spacesでは不要
S3_PROTOCOL=https                                   # https
S3_ENDPOINT=https://sgp1.digitaloceanspaces.com     # エンドポイントを指定
S3_ALIAS_HOST=media-mstdn.maud.io                   # CDNを利用した場合のホスト名
```

保存したらDockerなら `docker-compose up -d` とかでコンテナを再起動すれば設定が反映されると思います。

### 切り替えたら

完了したら、まずはメディアの配信元URLが無事に切り替わっているかどうかを確認します。既にアップロードされていて、オブジェクトストレージにも同期が済んでいるものが良いでしょう。自分のプロフィールを開いてアイコンの画像を別タブで開くとかが簡単ですかね。

そして、新規に画像を添付した投稿が可能かどうか、またその画像がオブジェクトストレージから配信されているかも同様に確認します。

無事に切り替わっていた場合、**最後の同期から切替を行うまでの間にサーバに保存されたメディア** が閲覧できなくなります。これは挙動自体は間違っていなくて、

- APIは既に移行先の新しいURLを返すようになっている
- が、案内された先であるオブジェクトストレージ上にはまだ該当するファイルが存在しない

という状況が発生しているためです（よね？）。また、この期間に該当するメディアを初めて取得しようとした **他のインスタンス** も同じ理由でそのメディアを取得することができません。

インスタンス自身だけでなく、外部のインスタンスに対してもこのような影響があるため、移行に関して内外への周知は重要です。

### 最後のコピー

切替を確認できたら、前回の同期から切替までの間に保存されたメディアをオブジェクトストレージにアップロードしていきます。切替後は「ローカルにはなくてオブジェクトストレージにある」メディアが発生していて、これらは消してはいけないので、同期というよりはコピーですね。

```bash
$ rclone copy ~/mastodon/public/system spaces:media-mstdn-maud-io -P
```

rcloneはファイルに対する変更の有無だったり、コピーの要/不要をサイズ・更新日時・md5sumで確認して判断するようですが、これが全部に対してチェックが走るのかめちゃくちゃに時間がかかるので覚悟しておきましょう。この回でのアップロード量は300MBもなかったんですが、チェックを含めた完了までには6-7時間がかかりました。

こうしてサービス開始から2年ちょっとの分、約84GB、68万件のファイルが無事に移行完了しました。

## おまけ: CDN導入とか

mstdn.maud.io では今まで全てのサービスを1台のマシンで提供していました。メディアも同じサーバに保存されており、画像にアクセスが集中すると露骨にMastodon側に影響が出がち（その最たる例が所謂オイゲン砲とか呼ばれるそれ）で、動画の読み込みにもかなり難がありました。

そのため元々負荷軽減のためにCDNの導入を勧められることがあったのですが、ちょうどオブジェクトストレージへの移行にあたって

- 保存されているオリジンサーバが遠い
- ユーザに最適な経路でメディアが降ってくると嬉しい
- 別ドメインで配信されるようになるのでキャッシュの適用範囲に悩まなくて良さそう
- DO側でCDNは提供されているとはいえ、転送量にカウントされるのでそれ頼みで終わらせるわけにもいかない

などの理由が重なり、（ようやく）オブジェクトストレージへの移行と同タイミングでのCDNの導入に踏み切りました。

### 設定編: Cloudflare

わたしは元々 `maud.io` のDNSをCloudflareで管理していたので、そのまま同社のCDNも利用することにしました。

Edge Certificatesは単体とサブドメインに対してshared/universalのを発行する分には無料なのでいい話ですね。共用は嫌だとか `media.mstdn.maud.io` みたいな既存のに当てはまらないホスト名を使いたいなら月10ドルくらいから発行させてくれるみたいですが、わたしは妥協してワイルドカード証明書で事足りるように `media-mstdn.maud.io` にすることを決めました。

![](/images/cloudflare_universal_ssl.png)

サーバ用の証明書も無料でくれるので活用していきましょう。少し順序が遡ってこのへんはバケット作成前の話になるんですが、Origin Certificates に15年くらいのを作って

![](/images/cloudflare_origin_cert.png)

DigitalOcean側のサブドメイン設定で使います。

![](/images/do_add_custom_subdomain.png)

いいですね。

![](/images/do_subdomain.png)

CNAMEで `media-mstdn.maud.io` を `<バケット名>.sgp1.cdn.digitaloceanspaces.com` に向けて設定しました。

![](/images/enable_cloudflare_cdn.png)

この時点で `media-mstdn.maud.io` はDigitalOceanのエッジサーバを指すわけですが、 `Status` 欄の雲マークをオレンジの **DNS and HTTP Proxy (CDN)** に切り替えることでCloudflareのCDNを利用するようになります。

### Page Rules

Cloudflareではキャッシュに関するルールをURLベースで設定でき、3つまで無料で利用できます。

重要なこととして、Cloudflareはデフォルトでは `.mp4` のファイルをキャッシュしません。

> Which file extensions does Cloudflare cache for static content? – Cloudflare Support  
> https://support.cloudflare.com/hc/en-us/articles/200172516-What-file-extensions-does-CloudFlare-cache-for-static-content-

ので、ここでPage Rulesを使ってキャッシュ対象にしていくわけなんですが、ちょうどメディアだけが別のサブドメインになったので、細かいことは考えずに当該ドメイン以下を全部キャッシュするという選択ができますね。

```
URLmatch: media-mstdn.maud.io/*
Settings: Cache Everything
```

そんな感じで雑に対応したので、 `.mov` とか `.webm` とかが来ても大丈夫でしょう。来るのか知らんけど。

## その後

- 動画の読み込みは劇的に改善されました。
    - サンプル: https://mstdn.maud.io/@hota/102110123034825165
- かわいいおばけの画像が添付されたトゥートに対して [@Gargron@mastodon.social](https://mastodon.social/@gargron) によるブーストが観測されましたが、露骨に重くなることはありませんでした。
- 今回の移行によるコストの削減は現状なく、むしろオブジェクトストレージの分だけ増えていますが、将来的にサーバを移行する際にはローカルのメディアを保持する必要がなくなり、より小さいディスクで契約できることでしょう。
- CloudflareによるCDNの効果は絶大で、月に1.5TB程度がCached Bandwidthとして計上される見通しです。これが無かったらDigitalOceanの転送量枠を超えて追加の従量課金が始まるところでしたが、5月分の請求は $5 のみだったのでばっちりキャッシュされているようで何よりですね。

という感じで、もっと早くやっておけばよかったと思う程度には満足しています。