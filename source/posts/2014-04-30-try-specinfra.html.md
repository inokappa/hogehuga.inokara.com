---
title: Specinfra を試してみる
date: 2014-04-30 01:37 JST
tags: Specinfra, Serverspec
---

<H2>はじめに</H2>

`Serverspec` について改めて調査していたら `Specinfra` という `Serverspec` のバックエンドで実行しているコマンド群を抜き出した（という認識）ツールを知った。

かなり今更感が否めないが...[こちら](https://speakerdeck.com/mizzy/specinfra-at-jaws-days-2014)の資料を拝見して...

>OS / ディストリビュージョン毎のコマンドの差異これらを吸収してくれるフレームワーク

という一文（スライドでは二枚分）に「おおっ」と思ったのでとりあえず手を動かすことにした。

***
***

<H2>参考</H2>

 * [specinfra という serverspec/configspec に共通する処理を抜き出した gem をつくった](http://mizzy.org/blog/2013/11/30/1/)
 * [Immutable Infrastructure時代の構成管理ツール基盤SpecInfra](https://speakerdeck.com/mizzy/specinfra-at-jaws-days-2014)
 * [specinfraを使ってみよう](http://qiita.com/sawanoboly/items/d1e7794739d9d31a5316)
 * [「Immutable Infrastructure時代の構成管理ツール基盤SpecInfra」 by 宮下 剛輔氏 #jawsdays – JAWS DAYS 2014 参加レポート Vol.06](http://dev.classmethod.jp/cloud/aws/jawsdays2014-06/)

***
***

<H2>Specinfra とは</H2>

作者の [@gosukenator](https://twitter.com/gosukenator) さんの[こちら](http://mizzy.org/blog/2013/11/30/1/)の記事によると `Serverspec` や `configspec` を作る際に共通する以下の処理を別の `gem` にしたのが `Specinfra` のようだ。

 1. ローカルホストの場合は `exec` リモートホストの場合は `SSH` 等の実行形式を抽象化した `backend` と呼ばれているレイヤー
 1. `OS` ごとに `OS` に適したコマンドを返す `commands` と呼ばれているレイヤー
 1. 他 `properties` や `configuration` というヘルパーメソッド

`3` つ目のヘルパーメソッドについては実行環境に応じた必要な設定を行ってくれる（足りないものがあればエラーとなって注意してくれる）メソッドという認識。（例えば `include SpecInfra::Helper::Lxc` とすれば `lxc` の `gem` の有無をチェックしてくれたりと...もちろん、それだけでは無いと思うが）

***
***

<H2>やってみよー</H2>

[こちら](http://qiita.com/sawanoboly/items/d1e7794739d9d31a5316)に倣って試してみたいと思う。

***

<H3>インストール</H3>

インストールした環境は `MacOS X 10.9 Marvericks` に今更の `Boxen` 経由（！）でインストール。

~~~~
  ruby::gem { "specinfra for 2.0.0-p247":
    gem     => 'specinfra',
    ruby    => '2.0.0-p247'
  }
~~~~

上記のように `specinfra` をインストールするマニフェストを書く。`gem install specinfra` でもぜんぜんオッケー。

***

<H3>pry から利用してみる</H3>

インストールが終わったら早速 `pry` を起動して `Specinfra` を利用してみる。

>チュートリアルはありませんが、serverspecのソースといつものspec_helper.rbを見れば大体つかめます。

[@sawanoboly](https://twitter.com/sawanoboly) さんが上記のように書かれているので、手元の `serverspec` で生成した `spec_helper.rb` を見てみると `SpecInfra` の文字が見える。これが `Serverspec` がバックエンドで `Specinfra` が動いている証。

~~~~
require 'serverspec'

include SpecInfra::Helper::Exec
include SpecInfra::Helper::DetectOS
~~~~

`require 'serverspec'` とあるが `serverspec.rb` 内部で `requier 'specinfra'` とされているのも `serverspec` のソースから読み取れた。ということで、`pry` を起動して以下を実行して `Specinfra` を利用する準備を行う。

![](images/2014043001.png)

準備も整ったところで

