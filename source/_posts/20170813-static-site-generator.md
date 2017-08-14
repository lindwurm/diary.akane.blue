---
title: はじめまして
date: 2017-08-13T19:04:00.000Z
tags: hexo
thumbnail: /images/deploying-hexo-with-netlify.png
---
[1年半くらい前](http://dev.maud.io/entry/2016/03/15/hatenablog) に様々なブログプラットフォームを試した末 [はてなブログPro](http://hatenablog.com/guide/pro) に登録する選択をしたけれど、やっぱり静的サイトジェネレータを使ったブログも書きたくなったので初投稿です。

[Hexo](https://hexo.io/) を使うのは1年半振りということになりますね。[Jekyll](https://jekyllrb.com/) や [Hugo](https://gohugo.io/)、[Gatsby](https://www.gatsbyjs.org/) とかを差し置いてこいつなのは、単にちゃんと使っていた時期があったのと、Material Designっぽいテーマで良さげなのがあったから程度です。

せっかくなので1年半前の記事で挙げていた欠点（だと勘違いしていたとこ）とかをどう乗り越えたか書き残しておこうと思います。

### 同じタイトルの記事を書けなかったり、大量の .md が散らばりそうで厳しい

* よく見たら `_config.yml` でファイル名の規則変えれるようになってて、記事名に影響しないように日付ベースでファイル名つけれるっぽい。
* この記事の例で言えば、以下のようにしとけば `/source/_posts/20170813-static-site-generator.md` にしといても [https://diary.akane.blue/2017/08/13/static-site-generator](https://diary.akane.blue/2017/08/13/static-site-generator) みたいな感じでアクセスできますね

```
url: https://diary.akane.blue
root: /
permalink: :year/:month/:day/:title/
new_post_name: :year:month:day-:title.md
```

### 生成物でGitHubが無駄に緑化されるのつらい

* GitHubにはソース部分だけ置いて、[Netlify](https://www.netlify.com/) でdeployするようにしたら幸せになれました。

### オンラインエディタ欲しい

* これはHexoに限らず他の静的サイトジェネレータにも対応してるっぽいんだけど、[Netlify CMS](https://www.netlifycms.org/) でなんとかなりそうです。
  * 参考: [Netlify CMSを使ってHexoに記事を投稿する - unsweets.log](https://blog.unsweets.net/2017/03/write-blog-with-netlify-cms.html)
