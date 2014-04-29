---
title: ElastiCache for memcached の分散処理をどうするか悩む（2）
date: 2014-04-27 07:03 JST
tags: AWS, ElastiCache, RDS, memcached, twemproxy, Sensu
---

<H2>はじめに</H2>

`twemproxy` を使うことで `memcached` クラスタを負荷分散させることは出来そうだが、ノード間に散らばったデータの可用性や冗長性を担保しなければいけない状況ではチト辛そう...ということは解ったのでもう悩んではいませんが...

 * `RDS for MySQL 5.6` + `memcached Plugin` の簡単なベンチマークを取ってみる
 * `twemproxy-web` というツールがあるのでそちらを使ってみる
 * `twemproxy` のメトリクスを `graphite` に飛ばす `sensu` のプラグインもあるみたい！（今回触るのは止めておきます）

尚、ベンチマーク結果は環境等により異なるのであくまでも指標の一つとして見て頂ければ幸いです。

***
***

<H2>RDS for MySQL 5.6 + memcached Plugin のベンチマーク</H2>

<H3>RDS for MySQL 5.6 + memcached Plugin の設定</H3>

以下のようなステップで設定する。

 1. `Options Groups` にて新しいオプショングループを作成する
 1. オプショングループに `Add Option` をクリックして `MEMCACHED` を選択する
 1. `MEMCACHED` オプション自身にもセキュリティグループを指定する必要があるので環境に応じて設定を行う
 1. 作成した `Option Group(s)` を `RDS` のインスタンスに適用する

設定後は以下の様な状態になる（はず）。

![](images/2014042703.png)

個人的な注意点としては下記の通り。

 * 当然だが `VPC` 内の `RDS` に `MEMCACHED` オプションを指定する場合にはセキュリティグループは `VPC` のセキュリティグループを指定する必要がある
 * 設定後はインスタンスの再起動が必要になる（ようだ※要確認）

以下のような環境となる。

![](images/2014042702.png)

<H3>ベンチマーク</H3>

前回と同様に `memslap` を利用してベンチマークを実行するので以下のように実行する。 

~~~~
memslap -s ${memcached_host}
~~~~

結果は下記の通り。尚、`RDS` のインスタンスタイプは `db.m1.small` を利用した。

 1. 168.448 seconds
 1. 198.842 seconds
 1. 195.347 seconds

比較として普通に `VPC` 外に構築した `ElastiCache` に対してもベンチマークを実行してみた結果は下記の通り。尚、インスタンスタイプは `cache.m1.small` とした。

 1. 7.709 seconds
 1. 8.275 seconds
 1. 8.105 seconds

と同じ環境とは言いがたいが素の `ElastiCache` と比較すると `RDS for MySQL 5.6` + `memcached Plugin` は `20` 倍程度の処理時間が必要になるようだ。ムム。

<H3>私的な見解とか</H3>

 * 実装は`mysqld` プロセスの中で `memcached Plugin` がバックエンドとして `innodb` を利用する
 * 単純なキャッシュでの利用を想定すると `ElastiCache` の方が良さそう
 * `MySQL` と `memcached` を別々で運用する場合に `MySQL` と `memcached` 間の問い合わせオーバーヘッド軽減には貢献しそう

***
***

<H2>twemproxy の周辺ツール</H2>

<H3>twemproxy-web</H3>

`twemporxy` を `Ruby` から操作したい場合に `nutcracker` という `gem` がある。その `gem` を利用して `twemproxy` に `Web` インターフェースを提供するのが `twemproxy-web` だ（という認識）。

 * [kontera-technologies/nutcracker-web](https://github.com/kontera-technologies/nutcracker-web)

インストールは下記の通り。但し、`Ruby 1.9` 以上が必要になる。

~~~~
sudo gem install nutcracker-web --no-ri --no-rdoc -V
~~~~

インストール後、下記のように起動する。

~~~~
nutcracker-web --config /path/to/nutcracker.yml --port 22122 -d
~~~~

`--config` に `twemporxy(nutcracker)` の設定ファイルを指定して `--port` には `nutcracker-web` で利用するポートを指定する。また、`-d` でデーモンモードでの起動となる。起動してブラウザでアクセスすると以下のようなページが表示される。

![](images/2014042704.png)

おお...だが...`nodes` 等のプルダウンに何も表示されない等で現時点では利用出来る機能は限定的なのでも少し調べる。

***
***

<H2>参考</H2>

最後になるが、今回の記事を書くにあたり参考にさせて頂いたサイトや資料を自分の `Bookmark` 代わりにメモっておく。

 * [TwemproxyからElastiCacheに分散(同じキーは同じElastiCacheへ)してみる](http://blog.cloudpack.jp/2013/02/aws-news-twemproxy-elasticache-key.html)
 * [ElastiCacheとELBとtwemproxy](http://d.conma.me/entry/2013/01/22/195331)
 * [第4回　memcachedの分散アルゴリズム](http://gihyo.jp/dev/feature/01/memcached/0004?page=3)
 * [Amazon ElastiCache の分散方法](http://understeer.hatenablog.com/entry/2012/01/04/144408)
 * [Amazon RDS MySQL 5.6と新機能を試してみた](http://dev.classmethod.jp/cloud/aws/amazon-rds-mysql56-new-features/)
 * [Memcache Telnet Interface](http://lzone.de/articles/memcached.htm)
 * [Memcached In A shell Using nc And echo](http://www.kutukupret.com/2011/05/05/memcached-in-a-shell-using-nc-and-echo/)
 * [libmemcached付属ツール使用方法](http://l-w-i.net/t/memcached/libmemcached_001.txt)
 * [MySQL 5.6での機能強化点（その2） - NoSQL APIとパフォーマンス・スキーマ](http://thinkit.co.jp/story/2014/01/08/4716)

***
***

<H2>さいごに</H2>

<H3>ElastiCache for memcached の分散構成について</H3>

今回、`ElastiCache for memcached` の分散処理についてちょっとだけ調べてみたが、ストアするデータの可用性、冗長性を考慮するのであれば、そもそも論だが `memcached` よりもレプリケーション機能のある `Redis` を選択するのもアリだと感じた。但し、自動でのフェイルオーバー等の細かい機能の調査を続けたい。

また、`twemproxy` を使えば複数ノードで構成された `memcached` クラスタにおいて、オーバーヘッドは最小限なままで分散構成は取れると思われる。

***

<H3>RDS for MySQL 5.6 + memcached Plugin について</H3>

`RDS for MySQL 5.6` + `memcached Plugin` の組み合わせについてはクエリキャッシュとして、単体の `memcached` を使っている構成の場合には置き換えることを検討しても良いかもしれない。

***
***
