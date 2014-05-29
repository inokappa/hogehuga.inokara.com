---
title: ちょっとだけ解った気になる Graphite と Grafana 
date: 2014-05-27 13:12 JST
tags: Grafana, Graphite, Sensu, Docker, Elasticsearch
---

<H2>はじめに</H2>

 * `Sensu` のメトリクスを蓄積して可視化する為のツールとして `Graphite` がほぼデファクトになっているのでマジメに触っておく（`Graphite` はメトリクスを蓄積する為のデータベースで `graphite webapp` が `Graphite` に蓄積されたメトリクスを可視化するものと考えると良いかも）
 * 上記の通り、`Graphite` では `graphite webapp` という可視化ツールがデフォっぽいけど `Grafana` というみんな大好き `Kibana` をフォークしたツールもあるので触っておく
 * 今回は `Graphite` に関してはこちらの `Dockerfile` を利用して環境を構築した

***
***

<H2>参考</H2>

 * [The Carbon Daemons](http://graphite.readthedocs.org/en/latest/carbon-daemons.html#the-carbon-daemons)
 * [Configuring Carbon](http://graphite.readthedocs.org/en/latest/config-carbon.html#configuring-carbon)
 * [Feeding In Your Data](http://graphite.readthedocs.org/en/latest/feeding-carbon.html#feeding-in-your-data)
 * [Grafana Document](http://grafana.org/docs/)

***
***

<H2>Graphite</H2>

![](images/2014052704.png)

***

<H3>アーキテクチャ</H3>

`Graphite` は以下のような `3` つのコンポーネントで構成されている。

 * `carbon` - 時系列データを収集する（`Twisted` というネットワーク・プログラミング用のフレームワークで実装されている）
 * `whisper` - 収集した時系列データを蓄積するシンプルなデータベース
 * `graphite webapp` - `Django` の `Web` アプリケーションで実装され、オンデマンドでグラフを描画するのに `Cairo` が利用されている

***

<H3>セットアップ</H3>

それほど難しくない。ディストリビュージョンによってはパッケージが用意されているのでそちらを利用しても良いかも。

***

<H3>Carbon の種類</H3>

`Carbon` の種類という言い方が正しいのか解らないが `Carbon` には以下のような種類のデーモンが利用出来る。

 * [carbon-cache.py](http://graphite.readthedocs.org/en/latest/carbon-daemons.html#carbon-cache-py)
 * [carbon-relay.py](http://graphite.readthedocs.org/en/latest/carbon-daemons.html#carbon-relay-py)
 * [carbon-aggregator.py](http://graphite.readthedocs.org/en/latest/carbon-daemons.html#carbon-aggregator-py)

全ての設定は `carbon.conf` にて設定が可能で `[]` で区切られたセクションで個々の設定を行う。以下、各々の特徴を簡単にピックアップ。（ドキュメントの意訳と実際に試してみた結果）

-

<H4>carbon-cache</H4>

 * 色々なプロトコルでメトリクスを受信してディスクに書き込む
 * メトリクスを受信するとメモリにバッファして `whisper` ライブラリにより適当な間隔でディスクにフラッシュする

また...

>The [cache] section tells carbon-cache.py what ports (2003/2004/7002), protocols (newline delimited, pickle) and transports (TCP/UDP) to listen on.

とあるので `Firewall` 等では `2003` と `2004` そして `7002` ポートは開放しておく必要があるので注意。また、それぞれの用途については `carbon.conf` に色々とコメントが書かれているのでそちらも参考になる。

~~~~
LINE_RECEIVER_INTERFACE = 0.0.0.0
LINE_RECEIVER_PORT = 2003

# Set this to True to enable the UDP listener. By default this is off
# because it is very common to run multiple carbon daemons and managing
# another (rarely used) port for every carbon instance is not fun.
ENABLE_UDP_LISTENER = False
UDP_RECEIVER_INTERFACE = 0.0.0.0
UDP_RECEIVER_PORT = 2003

PICKLE_RECEIVER_INTERFACE = 0.0.0.0
PICKLE_RECEIVER_PORT = 2004

# Set to false to disable logging of successful connections
LOG_LISTENER_CONNECTIONS = True

# Per security concerns outlined in Bug #817247 the pickle receiver
# will use a more secure and slightly less efficient unpickler.
# Set this to True to revert to the old-fashioned insecure unpickler.
USE_INSECURE_UNPICKLER = False

CACHE_QUERY_INTERFACE = 0.0.0.0
CACHE_QUERY_PORT = 7002
~~~~

上記は `carbon.conf` からポートに関する部分を抜粋したもの。`2003` ポート自体がメトリクスを受け入れる為のポートのようで、デフォルトでは `UDP` はオフになっていることが解る。また、`2004` ポートは `pickle` という `Python` 独自のデータストリーム方式でのメトリクスを待ち受けるようだ。`pickle` については[こちら](http://docs.python.jp/3.3/library/pickle.html#module-pickle)や[こちら](http://graphite.readthedocs.org/en/latest/feeding-carbon.html#the-pickle-protocol)が参考になりそう。

-

<H4>carbon-relay</H4>

 * メトリクスのレプリケーションとシャーディング（冗長化とスケーリング）を提供する
 * `RELAY_METHOD = rules` となっている場合に複数のホストで稼働している `carbon-cache` にメトリクスをリレーすることが出来る
 * `RELAT_METHOD = consistent-hashing` の場合には複数の `carbon-cache` ホストでのシャーディングを定義出来る
 * また、`graphite-web` からのデータ取り出しに際しては `local_settings.py` の `CARBONLINK_HOSTS` で定義した複数の `carbon-cache` ホストから読み込む

尚、`Graphite` のクラスタリングについては以下の記事に詳細に紹介されていた。

 * [Clustering Graphite](http://bitprophet.org/blog/2013/03/07/graphite/)
 * [Graphite cluster setup blueprint](http://www.slideshare.net/AnatolijDobrosynets/graphite-cluster-setup-blueprint)

また、自分がなんやかんや言うよりも [@y_uuki](http://twitter.com/y_uuk1) さんの以下の資料で一目瞭然！

<script async class="speakerdeck-embed" data-slide="26" data-id="0c5a65f0418f0131e7fd7a757649ab26" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

解りやすい。

-

<H4>carbon-aggregator</H4>

 * `carbon-cache` より前で動作する
 * `whisper` に書き込む前にメトリクスをバッファリングすることで `whisper` への負荷を減らす
 * `granular reporting` が不要な場合にも有用で `whisper` のサイズ削減にも貢献する（※`granular reporting` の意味がイマイチ解りませんでした...）

***

<H3>storage-schemas.conf</H3>

`storage-schemas.conf` では...

 * メトリクス収集の周期と期間を定義している
 * メトリクスのパス毎に指定可能
 * 定義された内容は `whisper` によって解釈される

ということで、ざっくり言うとメトリクスの保存ポリシーを定義している。

尚、設定については以下の三つを設定する必要がある。

 * 設定名（という言い方が正しいか解らないけど）`[]` で囲む
 * `pattern=` は正規表現での設定が可能
 * `retentions=` は `frequency:history` といフォーマットで記載

以下は例。

~~~~
[hoge_collection]
pattern = hoge.huga$
retentions = 10s:14d
~~~~

`hoge_collection` という名前で `hoge.huga` で終わるパスのメトリクスは `10` 秒毎に取得し `14` 日間保持することを示している。

また、周期や保存の期間については以下のような単位で設定が可能。

 * `s` - 秒
 * `m` - 分
 * `h` - 時間
 * `d` - 日
 * `y` - 年

***

<H3>Carbon を操作して解るメトリクスのフォーマット</H3>

そろそろうんちくも疲れてきたので...[こちら](http://graphite.readthedocs.org/en/latest/feeding-carbon.html)を参考にして、以下のように `nc` を利用して `Graphite` のホストに対してメトリクスを送ってみる。

~~~~
PORT=2003
SERVER=172.17.0.2
echo "`hostname -s`.hoge.huga.test `echo $RANDOM` `date +%s`" | nc ${SERVER} ${PORT}
~~~~

適当に何度か叩いていると...

![](images/2014053001.png)

おおー。（※[こちら](http://graphite.readthedocs.org/en/latest/feeding-carbon.html#the-plaintext-protocol)によると `nc` で `-q0` オプションを指定しない場合には `TCP` 接続を開きっぱなしになる`nc` のバージョンもあるようなので注意する。）

ということで上記にあるように `Graphite` が解釈出来るメトリクスの基本的なフォーマットは以下の通りとなる。

~~~~
${metric path} ${metric value} ${metric timestamp}
~~~~

`${metric path}` については `.` ドットを区切りとして階層構造を持たせることが出来る。どの位の階層を持てるかを試してみたのが以下の図。

![](images/2014053002.png)

結構イケそう...（実際の限界については情報を見つけることが出来なかった）

***
***

<H2>Grafana</H2>

![](images/2014052705.png)

<H3>Kibana の従兄弟</H3>

と勝手に命名。

見ての通り `Kibana` にとてもよく似ているのはもちろん、グラフの設定情報については `Kibana` 同様に `Elasticsearch` に保存するような設定になっているのでちゃんと使おうとすると `Elasticsearch` が必要になる。

![](images/2014053003.png)

上記のようにエラーが表示される。まあ、ちょっと試す分には全然問題ないかと...

***

<H3>使い方</H3>

使い方はとっても簡単。一時間位使ってみた記事を以下に書いたのでそちらを見て頂くといかに簡単かが解るかも...

 * [Graphite と Grafana を 1 時間位使ってみたメモ](http://inokara.hateblo.jp/entry/2014/05/28/230025)

ポイントとしては `Grafana` の `config.js` にちゃんと `graphite-web` の設定を行えれば `Graphite` のメトリクスを以下のように自動で取得してくれているので、あとは必要に応じてメトリクスを選択するだけ。

![](images/2014053004.png)

グラフにメトリクスを追加したい場合でも `Add Query` をクリックするだけ。後は自動的に取得されている `Graphite` のメトリクスを選択していく。

***

<H3>基本的な使い方以外にも...</H3>

[ドキュメント](http://grafana.org/docs/)を見ると基本的な使い方以外にも以下のような機能が実装されている。

 * [Templated Dashboards](http://grafana.org/docs/features/templated_dashboards/)
 * [Scripted Dashboards](http://grafana.org/docs/features/scripted_dashboards/)
 * [Playlist](http://grafana.org/docs/features/playlist/)

個人的に気になったのが [Scripted Dashboards](http://grafana.org/docs/features/scripted_dashboards/) と[Playlist](http://grafana.org/docs/features/playlist/) の二つ。ざっくり言うと `Scripted Dashboard` は `JavaScript` を使って動的にダッシュボードを生成することが出来る機能、`Playlist` は指定した期間のグラフを保存しておいて後からも見ることが出来る機能のようだ。

***
***

<H2>まとめ</H2>

この記事を通して痛感したこと、解ったこと。

***

<H3>痛感したこと</H3>

 * `Graphite` とか `Grafana` 関係無しに英語力足りてねえ
 * `Python` 力が全く無い...

***

<H3>解ったこと</H3>

 * `Graphite` も `Grafana` もさらっと使う分にはとても簡単
 * `Graphite` については `2006` 年くらいから続いているプロジェクトでかなり歴史はある（＝実績もある？）
 * `Graphite` は `3` つのコンポーネント（`Carbon` と `whisper` と `graphite-web`）がある
 * `Carbon` は `3` つのデーモン（`carbon-cache` と `carbon-relay` と `carbon-aggregator`）がある
 * `carbon-cache` はメトリクス収集、`carbon-relay` は冗長化とシャーディング、`carbon-aggregator` はバッファ
 * メトリクスデータの基本フォーマットは `${metric path} ${metric value} ${metric timestamp}`
 * `Grafana` は `Elasticsearch` も必要な場合があるが基本的には操作は簡単！

***

<H3>次やること</H3>

 * `Graphite` の `Render URL API` や `whisper` について調べる
 * `Grafana` の `Scripted Dashboards` と `Playlist` について調べる

***
***
