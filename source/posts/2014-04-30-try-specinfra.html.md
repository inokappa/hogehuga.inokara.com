---
title: Specinfra を試してみる
date: 2014-04-30 01:37 JST
tags: Specinfra, Serverspec, Docker
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
 * [swipely/docker-api](https://github.com/swipely/docker-api)

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

準備も整ったところで `backend` のメソッド達を呼び出してみる。

~~~~
backend.methods
~~~~

以下のようにズラズラーと `backend` で利用出来るメソッド達が表示される。

![](images/2014050101.png)

おお。

では、メソッドの中から `run_command` なんか試してみたりする。

~~~~
backend.run_command('pwd')
~~~~

以下のように `pwd` コマンドの実行結果を含んだ結果が出力された。

![](images/2014050102.png)

ついでに、先ほどの `backend.methods` から `commands` というメソッドを利用して、さらに...

~~~~
backend.commands.check_running('httpd')
~~~~

としてみると...以下のように `backend.check_running('httpd')` を実行する際に裏で実行される `OS` のコマンドが表示された。

![](images/2014050103.png)

ああ、これが各 `OS` のコマンドを抽象化しているということなのかな。

***
***

<H2>私的応用</H2>

横領ではなくて応用。まだまだちゃんと `Specinfra` の事は知らないけど `pry` から触っていて色々とやりたくなってきたので簡単に試してみた。

***

<H3>ミニマムな私的 Serverspec</H3>

適当に構築した `Docker` コンテナホストに対して `Serverspec` 的なことを実行してみる。

以下のスクリプトは見ての通り、

 * `httpd`
 * `sshd`
 * `ntpd`

それぞれのプロセスが起動しているかをチェックするもの。

~~~~
#!/usr/bin/env ruby

require 'specinfra'
require 'net/ssh'

include SpecInfra::Helper::Ssh
include SpecInfra::Helper::DetectOS

SpecInfra.configuration.ssh = Net::SSH.start('localhost', 'sshuser', {:port => '49156', :password => 'sshuser'})

PROCESSES=['httpd','sshd','ntpd']
PROCESSES.each do |process|
  c = backend.check_running("#{process}")
  if "#{c}" == "true"
    puts "#{process} is running"
  else
    puts "#{process} is not running"
  end
end
~~~~

実行結果は下記の通り。

![](images/2014050104.png)

おお、やっていることはまさに `Serverspec` そのもの！`Serverspec` も裏側ではこのようなことが行われている...という認識を持てた。

***

<H3>Backend として docker が使えるようなので...</H3>

`Specinfra` は `Backend` として以下のような環境が利用出来る。（他にもある）

 * `ssh`
 * `lxc`
 * `docker`
 * `shellscript`

ということで...試してみた。例のごとく `pry` ではじめてみる。

~~~~
require 'specinfra'
require 'docker'
~~~~

を実行すると...

![](images/2014050301.png)

ふむ。次にヘルパーメソッド（と呼ぶのかな？）を `include` する。

~~~~
include SpecInfra::Helper::Docker
include SpecInfra::Helper::RedHat
~~~~

を実行すると...

![](images/2014050302.png)

次に `Docker API` の `URL` を指定してからコンテナイメージを指定する。

~~~~
Docker.url = 'http://127.0.0.1:4243'
SpecInfra.configuration.docker_image = 'centos'
~~~~

![](images/2014050303.png)

この状態から `backend.methods` を叩くと...

![](images/2014050304.png)

ほうほう。では `run_command` あたりを。

~~~~
backend.run_command ('ls')
~~~~

以下のように `centos` コンテナ内で `ls` を実行した結果が返ってきた。

![](images/2014050305.png)

おお、今度は `install ('httpd')` あたりを。

~~~~
backend.install ('httpd')
~~~~

以下のようにコンテナ内で `Apache` を `yum install httpd` でインストールした結果が返ってきた。

![](images/2014050306.png)

ファンタスティック！

上記のように `Specinfra` 内で抽象化されたコマンドで `Docker` コンテナ内に `Apache` をインストールすることが出来るのを確認した。

***
***

<H2>さいごに</H2>

 * `OS` のコマンドの違いを抽象化するという考え方にとても共感出来るし、`Specinfra` はその方法論の一つを具現化したものだと思う
 * `Specinfra` を触ることで `Serverspec` が裏側でどんなことをやっているかが少し解った
 * 自分の `Ruby` 力の低さから用語の使い方や認識に誤りがあると思われるが...も少し触ってみる

***
***
