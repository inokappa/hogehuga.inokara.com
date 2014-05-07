---
title: Chef Handlers + Graphite で Chef の実行結果を可視化する
date: 2014-05-06 09:26 JST
tags: Chef, Graphite
---

<H2>はじめに</H2>

前回の記事で `Chef` には `Handlers` と呼ばれるレポート機能が備わっていることを知った。レポートの出力先としては `JSON` ファイルや `IRC` 等があり、みんな大好き、私も大好き（になろうと頑張る）`Graphite` に出力出来るようなので試してみたいと思う。尚、`Fluentd` にも出力することが出来るようなので（以下、参考リンク）活用の機会が広がるような気がしている。

***
***

<H2>参考</H2>

 * [Chef 活用ガイド 〜コードではじめる構成管理〜](http://ascii.asciimw.jp/books/books/detail/978-4-04-891985-2.shtml)（`135` ページから `145` ページの「5.7.1 ハンドラ(Chef::Handler)」）
 * [About Handlers — Chef Docs](http://docs.opscode.com/handlers.html)
 * [ChefのReport, Exeptionハンドラを使う](http://qiita.com/sawanoboly/items/29a488b0d3a781bd1d62)
 * [はかどるChefの小ネタ集](http://www.slideshare.net/YukihikoSawanobori/chef-miscs/13)（`13` ページから `18` ページあたりで `Handlers` について言及されている）
 * [Chef のログを Fluentd に流す](http://blog.ryotarai.info/blog/2014/02/14/send-chef-logs-to-fluentd/)
 * [Graphite Docs » Overview](http://graphite.readthedocs.org/en/latest/overview.html)
 * [Install graphite on a CentOS/RHEL server](http://www.linuxsysadmintutorials.com/install-graphite-on-a-centosrhel-server/)
 * [CentOSにRPMでGraphite+Diamondをインストールする](http://qiita.com/takakiku/items/4dbee4739801cb8f60a2)

***
***

<H2>登場人物</H2>

登場人物（`Chef Handlers` と `Graphite`）について。

***

<H3>Chef Handlers</H3>

 * `Chef Client` のレポーティング機能
 * `Start Handlers` と `Report Handlers` と `Exception Handlers` の三種類
 * `node` の情報、経過時間、例外が発生した際の各種情報等を取得可能
 * `JSON` ファイル以外にも書き出し先は選べるし自作も可能
 * `Chef::Handler` を継承して独自のハンドラを作成することも可能

***

<H3>Graphite</H3>

 * `Python` で実装されている
 * 時系列データの保存
 * 時系列データをオンデマンドでグラフに描画してくれる
 * `carbon` と `whisper` そして `graphite webapp` という三つのコンポーネントで構成されている
 * `carbon` は `Twisted` というデーモンツールで時系列データを収集する役目
 * `whisper` は時系列データを保存しておくデータベース 
 * `graphite webapp` は `whisper` からデータを取り出してグラフを描画する

`Graphite` について [@y_uuk1](https://twitter.com/y_uuk1) さんの以下の資料の図が個人的には一番解りやすかったので拝借させて頂きます...

<script async class="speakerdeck-embed" data-slide="24" data-id="0c5a65f0418f0131e7fd7a757649ab26" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

上の一枚だけで概要は掴めるかと。

***
***

<H2>準備</H2>

`Chef Handlers` と `Graphite` を設定。 利用した環境は下記の通り。

 * `CentOS 6` の `AMI`
 * `m3.medium`
 * `chef-11.12.4-1.el6.x86_64`（オムニバスインストーラーからインストール）

***

<H3>Chef Handlers</H3>

既に `Chef` はセットアップされた状態から、`chef-handler-graphite` を`gem` を使ってインストールする。

 * [imeyer/chef-handler-graphite](https://github.com/imeyer/chef-handler-graphite)

インストールは下記のように行った。

~~~~
/opt/chef/embedded/bin/gem install chef-handler-graphite --no-ri --no-rdoc -V
~~~~

これだけ...簡単。

***

<H3>Graphite</H3>

設定は以下のページに書いた。

 * [Graphite Setup for CentOS](https://gist.github.com/inokappa/2be2ad0af6527b6749df)

Graphite-web の初期設定を行う場合に以下を実行する。

~~~~
/usr/lib/python2.6/site-packages/graphite/manage.py syncdb
~~~~

実行すると以下のようにユーザー名やパスワードを問われる。

![](images/2014050601.png)

詳細の出力については以下の通り。

 * [graphite-web の初期設定出力](https://gist.github.com/inokappa/cc24f85c4e05c8790d06)

設定後、ブラウザで `Graphite` ホストにアクセスすると...

![](images/2014050602.png)

これで `Graphite` は使える状態になった。

***
***

<H2>やってみる</H2>

`Chef Handlers` は `Cookbook` 内に設定するか、`client.rb` や `solo.rb` に設定すれば利用出来るとのことなので、`solo.rb` に設定してみる。

***

<H3>solo.rb に...</H3>

`solo.rb` に以下を追加する。

~~~~
require 'chef-handler-graphite'
graphite_handler = GraphiteReporting.new
graphite_handler.metric_key = "kappatest.chef.#{Chef::Config.node_name}"
graphite_handler.graphite_host = "localhost"
graphite_handler.graphite_port = "2003"
report_handlers << graphite_handler
exception_handlers << graphite_handler
~~~~

***

<H3>Cookbook を適用</H3>

テストに以下の `Cookbook` を選んだ。

 * [elasticsearch](http://community.opscode.com/cookbooks/elasticsearch)

`Berksfile` に以下を追加して...

~~~~
source "https://api.berkshelf.com"

cookbook "java"
cookbook "elasticsearch"
~~~~

`berks vendor` を実行する。

~~~~
/opt/chef/embedded/bin/berks vendor cookbooks
~~~~

以下のように `run_list` を用意する。

~~~~
{
  "name": "elasticsearch-cookbook-test",
  "run_list": [
    "recipe[java]",
    "recipe[elasticsearch]"
  ],

  "java": {
    "install_flavor": "openjdk",
    "jdk_version": "7"
  },

  "elasticsearch": {
    "cluster" : { "name" : "elasticsearch_test_chef" }
  }
}
~~~~

`run_list` を用意したら以下のようにして `Cookbook` を適用する。

~~~~
chef-solo -c solo.rb -j nodes/localhost.json
~~~~

ちなみに、`-W` オプションつけて何度か `why-run` で試してもみた。`Cookbook` 適用後に以下のように表示された。

![](images/2014050603.png)

おお、ちなみに `Graphite` の方は...

![](images/2014050604.png)

おお。

***

<H3>どんな値が取れているのか？</H3>

以下のような値が `Graphite` で確認することが出来る。

 * `all_resources` は `run_list` に定義されたレシピの各種リソース（の数）
 * `elapsed_time` は `start_time` と `end_time` の差を表現
 * `fail` は例外を含んでいる（数）
 * `success` は例外を含んでいない（数）
 * `updated_resources` は `all_resources` のうちで `Chef::Runner` によって変更されたリソース（の数）

詳細については以下を参考になる。

 * [run_status Object](http://docs.opscode.com/handlers.html#run-status-object)

上記を参考にしながら `Graphite` のグラフを眺めているとより理解が深まる...（かも）。

***
***

<H2>最後に</H2>

 * 以前から興味があって試したいと思っていた `Chef Handlers` をやっと試せた
 * コンバージェンスの結果はを詳細に追うことはあまり無かったが、今後は `Handlers` を活用していきたい
 * 今回は単純に `Graphite` に飛ばすだけだったが、`fluentd` を介して `MongoDB` 等に保存するという事例もある
 * ちなみに `Graphite` の使い方もなんとなく解ってきたのは良かった

***
***
