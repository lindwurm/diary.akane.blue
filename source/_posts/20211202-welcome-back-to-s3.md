---
title: メディアをS3に戻しました
date: 2021-12-02 04:00:00
tags: 
- メンテナンス
cover_img: /images/aws_logo.png
summary: Wasabiに移行して10日、AWSに出戻りすることになりました
---

助けてくれ。

## tl;dr

- CloudFront の転送料が劇的に安くなる見通しが経った
- わずか10日で Wasabi+Cloudflare 構成に別れを告げて S3+CloudFront 構成に出戻りした
- 前回の引っ越しは無駄だったかもしれん

## 何？

![cloudfront-new-free-tier](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2021/11/24/free_cw_1tb_3.png)

> *Amazon CloudFront からのデータ転送は、(50 GB から) 1 か月あたり **最大 1 TB** のデータで **無料** になり、サインアップ後 12 か月間の制限もなくなりました。*
> *また、HTTP および HTTPS の無料リクエスト数を 2,000,000 から **10,000,000** に増やし、1 か月あたり 2,000,000 回の CloudFront 関数の無料起動に対する 12 か月の制限を撤廃しました。この拡張は、中国の CloudFront POP からのデータ転送には適用されません。*
> *この変更は **2021 年 12 月 1 日から適用** され、自動的に有効になります。この変更により、世界中の数百万人の AWS をご利用いただいているお客様には、この 2 つのカテゴリのデータ転送に対する料金が AWS の月額請求に記載されなくなります。これらの割り当てのいずれか、または両方を超えて使用するお客様には、全体のデータ転送料金が引き下げられます。*
> - **[AWS 無料利用枠のデータ転送量の拡大 — リージョンから 100 GB、Amazon CloudFront から 1 TB / 月 | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/aws-free-tier-data-transfer-expansion-100-gb-from-regions-and-1-tb-from-amazon-cloudfront-per-month/)**

**あの！！！！！！！！！！！！！！！！！！！**

[前回の移行](/2021/11/23/goodbye-s3-hello-wasabi/) 直後にとんでもない発表をされてて震えています。さっき知りました。

データ転送量の無料枠が 50GB から **1TB** になると弊サーバにおけるメディアの転送量の9割方、月によっては10割賄えるので、一番の問題だった CloudFront のコストがめちゃくちゃに下がります。えっもうこれしかなくない？ないじゃん。ほんまありがとう AWS 。 *できたらもうちょっと早く言ってくれ。*

ついでにその他のサービスもAWSリージョンから外のインターネット向きの転送が **100GB/月** まで無料になりました。前回 S3 から Wasabi に転送したときにファイルのダウンロード元として CloudFront を噛ませるオプションを使い忘れていた（終わってから知った）ので、必要なかった転送を安くなる前に行ってしまってダブルで痛い。勘弁してくれ。

他の大多数の Mastodon サーバなら 1TB は十分な転送量だと思うので、（保存にかけるコストと天秤ではあるものの）かなり優秀な選択肢になったのではないでしょうか。

弊サーバの場合は月によって1TB超えたり超えなかったりするので正確な予測は難しいですが、雑な試算でも10～50USD/月くらいの見込みでいるので、まあこのくらいならより安定した構成を取りたいですね、みたいなところで出戻りを決断しました。

## 作業

幸いにも S3 のバケットは消してないし、 CloudFront のディストリビューションもそのまま残してあったので、`CAA` と `CNAME` レコードを前のに戻して Mastodon の `.env.production` に書いてた設定を戻すだけ（コメントアウトして残しててよかった）で切り替え作業は終わり。

あとは10日前とは逆向きで `rclone copy` を走らせておけばそのうち差分のコピーが済みます。今度は `--fast-list` オプションを試してみました。時間かかるしメモリ食うけどAPIリクエストが控えめになるっぽい？

## おわりに

- 移行作業は急ぎすぎないほうがいい
- 予測可能なコストを取ったつもりが、予測不可能な変更のおかげで要らんコストかけたことになって死んだ
  - まあ今月だけでも旧料金比で十分取り返せそうなのが救い
- *たすけてくれ*

### 広告

- またしても何も知らず無駄骨を折った hota への投げ銭先は [こちら](https://github.com/sponsors/lindwurm) です。
  - *サーバ費用ではなく個人への支援として扱われます*

## 追記: 付録

Q. 実際いくら溶かしたんですか？

A. 出戻り前の時点で以下の通り。

~~rcloneのオプション次第でもうちょっと詰めれたはず…と思ったけど、それ以前にそもそも移行しなかったら発生してないんだよな。~~

| 名目 | 費用(USD) |
|:---:|---:|
| `ListBucket` | 37.49 |
| `GetObject` | 22.51 |
| `GET` | 9.53 |
| `HeadObject` | 1.10 | 
| Cloudflare Pro | 20.00 |
| Wasabi | 6.99 |
| **合計** | **97.62** |
