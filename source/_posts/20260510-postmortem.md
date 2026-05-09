---
title: サーバーのディスクサイズを拡張しました（8年ぶり2回目）
date: 2026-05-10 05:00:00
tags:
- メンテナンス
cover_img: /images/postmortem/discord_alert.png
summary: ついに8年とかいう数字が出てくるようになってしまった、時の流れが怖いです。
---

タイトルの通りで、 mstdn.maud.io のサーバーのディスクサイズを拡張しました（8年ぶり2回目）。

前回は[こちら](/2018/01/16/server-disk-resized)。時の流れが怖いです。

## 経緯

DBやメディアを外出ししてスリム化した mstdn.maud.io も、流石にディスクサイズをケチりすぎて数ヶ月おきにアラートが頻発するようになっていました。**40GBはやりすぎ。**

> ![](/images/postmortem/discord_alert.png)  
> しきい値反復横跳び。最悪

並行して logrotate などの設定も詰めていたのですが、ログ以外の部分も稼働時間に合わせて徐々に肥大化してて、発報する頃にはだいたい20GBくらい食ってるので根本的に足りてないですね。ということで、増やします。40GB の次は 100GB です。

## 実行

コンパネでの操作は前回と同じなので割愛します。15分くらいで終わるので楽ですね。

それでは、またパーティションサイズの拡張をコンパネから…

> ![](/images/postmortem/failed.png)
> oh

…なんか前回もコケてたし成功率0すぎる（サンプル:2）

### 格闘

`fdisk` を叩くと、GPTテーブルのサイズ不一致で怒られてしまいました。

```
GPT PMBR size mismatch (83886079 != 209715199) will be corrected by write.
The backup GPT table is not on the end of the device. This problem will be corrected by write.
This disk is currently in use - repartitioning is probably a bad idea.
It's recommended to umount all file systems, and swapoff all swap
partitions on this disk.
```

とりあえずLLMにでも対処方法を聞きましょう。エラーメッセージみたいな定型文には強いでしょう（適当）

> ![](/images/postmortem/copilot_1.png)

> ![](/images/postmortem/copilot_2.png)  
> MSアカウントがAkane Aikawaなのでサーバーに強い藍川茜概念がCopilotの中にある図

Copilotはrootfsが乗っているなら直接操作するのは危険、アンマウントする必要があるからレスキュー用のISOイメージなどから起動して修復するしか…みたいな回答をしてきましたが、他のメンバーが貼ってくれたVPSのマニュアルに `growpart` コマンドによるパーティション拡張方法の記載がありました。

うーん、やはり持つべきものは複数人体制、AIよりも人間の温かみのある知恵ですね（？）

https://manual.sakura.ad.jp/vps/server/disk-expansion/ubuntu-24.04.html#ubuntu-24-04-growpart

> ![](/images/postmortem/discord_gg.png)  
> gdisk のこと失念してるのは普通にジジイだったかも

### growpart 最高！

何が最高かはGeminiにまとめさせました。

> `growpart` は `fdisk` の「人間がやるとミスりやすい計算と操作」を安全に自動化してくれます。
> 
> - **開始セクタの自動維持**
> `growpart` は現在の開始位置を正確に読み取り、そこを動かさずに末尾だけを広げます。人間が「開始セクタは何番だったっけ…」とメモを取る必要がありません。
> - **カーネルへの強制通知**
> `fdisk` で書き換えた後、カーネルが「パーティションが変わったよ」という通知をうまく受け取れず、再起動が必要になることがよくあります。 `growpart` は内部で `partx` や `sfdisk` を使い、実行中のシステムに「今すぐ認識しろ」と安全に促します。 
> - **GPT/MBR の差分を吸収**
> GPT形式の場合、ディスクの末尾にバックアップのパーティションテーブルがあります。ディスクサイズを広げるとこのバックアップ位置も動かす必要がありますが、`growpart` はこれも自動で修正します。

これの最後が活きた形ですね。

```
CHANGED: partition=2 start=4096 old: size=83881951 end=83886046 new: size=209711071 end=209715166
```

ログの全体を残してなかったけどここだけ伝わればいいでしょう。無事に修正してくれました。

あとは `resize2fs` を叩くところは前回と同じです。

```
$ lsblk /dev/vda
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    253:0    0  100G  0 disk 
├─vda1 253:1    0    1M  0 part 
└─vda2 253:2    0  100G  0 part /

$ sudo resize2fs /dev/vda2 
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/vda2 is mounted on /; on-line resizing required
old_desc_blocks = 3, new_desc_blocks = 7
The filesystem on /dev/vda2 is now 26213883 (4k) blocks long.

$ df -h /dev/vda2
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        99G  8.8G   86G  10% /
```

### 公式ドキュメントを読もう2026

ここまで書いてきましたが、なんと `growpart` を使用したパーティション拡張はさくらのクラウドのマニュアルにも記載があります。最初から読んでいればこんなことには…。

https://manual.sakura.ad.jp/cloud/storage/modifydisk/resize-partition.html#growpart
