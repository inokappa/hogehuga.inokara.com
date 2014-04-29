---
title: 「Jenkins と cron」という資料を作ってプチ勉強会で発表しようと思ってます
date: 2014-04-29 18:03 JST
tags: Jenkins, 勉強会
---

<H2>はじめに</H2>

管理しているサーバーの `cron` 設定がカオスになりがちだったので「こりゃあかん」ということで、次のプチ勉強会で「Jenkins と cron」というタイトルで喋ろうかと思って資料を作ってみた。ちなみに初めての `Keynote` の資料となる。

***
***

<H2>資料と補足</H2>

<H3>資料</H3>

<script async class="speakerdeck-embed" data-id="97188b00b1650131b593068d4df30270" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

***

<H3>補足</H3>

 * `Jenkins` 側で `cron` ジョブの結果をチェックしてアラートを飛ばす場合にはチェック用のジョブを作る必要がありそう
 * `XML` をバイナリエンコードする場合には `hexdump` とか使うと良い

また、`XML` フォーマットは以下の通りとなる。全ての項目が必須となる。

~~~~
<run>
  <log encoding="hexBinary">$(hexdump -v -e '1/1 "%02x"' $LOG)</log>
  <result>${RESULT}</result>
  <duration>${ELAPSED_MS}</duration>
</run>
~~~~

ちなみに `cron` ジョブの結果を `POST` する `bash` スクリプトを以下にアップした。

 * [inokappa/my-jenkins-wrapper](https://github.com/inokappa/my-jenkins-wrapper)

このスクリプトをシェル関数として利用すればすぐに `cron` の結果を `Jenkins` 先生に飛ばせるはず。

***
***

<H2>さいごに</H2>

 * `Keynote` は使ってるだけでなんかカッコいい
 * `LT` 等の発表資料を作る時のポイントってなんだろなー

***
***
