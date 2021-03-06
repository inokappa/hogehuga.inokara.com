---
title: MongoDB 私的な tips やあれこれ
date: 2014-04-21 18:48 JST
tags: MongoDB
---

<H2>MongoDB は素晴らしい</H2>

と本気で思う...特に使い始めは...。実際に運用を開始してみると色々なネタを提供してくれるし、それを解決することが出来れば自分のスキルとなる...（はず）。ということで `MongoDB` を運用し始めて色々と学んだことなどを自分なりにまとめてみる。

***

<H2>運用監視系</H2>

<H4>show processlist 的なこと</H4>

`MySQL` の `show processlist` 的なことをやる場合には以下を `MongoDB` のコマンドラインインターフェースから以下を実行する。

~~~~
db.currentOp();
~~~~

一時的なモニタリングの場合には以下のように `watch` コマンド等を組み合わせるといいかも。

~~~~
watch 'echo "db.currentOp()" | mongo localhost/db --quiet'
~~~~

レスポンスは `JSON` っぽいフォーマット返ってくるが `jq` でパース出来るのか...？

<H4>コレクションの件数等をリアルタイムに監視</H4>

コレクションの件数等をサクッとリアルタイムに監視したい場合等は以下を `watch` 等と併用して実行する。

~~~~
echo "db.collection.stats()" | /usr/bin/mongo localhost/db --quiet
~~~~

件数以外にも取得可能。実際に叩いてみよう。

***

<H2> ここが困った Mongo さん</H2>

以下、適当に運用と監視してしまって困ったこと等を列挙。

<H4>Capped Collection</H4>

`Capped Collection` は `MongoDB` のデータベースのサイズを予め決めておくことでドキュメントを書き込み続けてもディスクが溢れることが無い（`MongoDB` がよしなに古いドキュメントを上書きしてくれるようだ）というログを保存する時等には夢のような設定だ。

 * [Mongodb Cappedコレクションを試す](http://semind.github.io/blog/2012/03/16/mongodb-cappedkorekusiyonwoshi-su/)

ところが以下のような制約があったり挙動に悩まされた。

 * `Capped Collection` が有効な場合、手動によるドキュメントの `remove` は不可
 * 原因は今のところ不明だが `Capped Collection` の設定が外れてしまいディスク容量を食いつぶしてしましったことがあった

便利だと思ったが...

<H4>Document の削除 = ファイルシステムの開放ではない</H4>

 * コレクションのドキュメントは `remove` で削除はできるが...
 * 削除した分のディスク容量が開放されるわけではないので要注意...

***
