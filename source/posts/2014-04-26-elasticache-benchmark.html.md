---
title: ElastiCache for memcached の分散処理をどうするか悩む
date: 2014-04-26 06:33 JST
tags: AWS, ElastiCache, RDS, memcached, twemproxy
---

<H2>はじめに</H2>

ちょいウンチク（というか個人的なメモ）。

 * `ElastiCache` は `memcahced` と `Redis` に対応したキャッシュサービス
 * `Redis` の場合にはレプリケーションが可能
 * `memcached` もクラスタ内にノードを複数持てるが...しかしキャッシュの同期等は出来ない

以下の図のような事がデフォルトでは出来ないことになる...。

ということで `memcached` を使う場合、可用性を考慮して複数のノードで分散処理を考えた場合には何か別な方法を考えにゃあかんことになる。

***
***

<H2>参考</H2>

ググると以下のように既に同じ疑問にぶち当たって試行錯誤されている方の事例がヒットする。

 * [TwemproxyからElastiCacheに分散(同じキーは同じElastiCacheへ)してみる](http://blog.suz-lab.com/2012/12/twemproxyelasticacheelasticache.html)
 * [ElastiCacheとELBとtwemproxy](http://d.conma.me/entry/2013/01/22/195331)

どうやら `twemproxy` というツールを使うことで分散処理を実現出来そうだ。

 * [twitter/twemproxy](https://github.com/twitter/twemproxy)

また、`RDS` の `MySQL 5.6` の `memcached` プラグインを使い `RDS` の `Multi AZ` を組み合わせることで可用性の向上が期待出来そう。

 * [Amazon RDS MySQL 5.6と新機能を試してみた](http://dev.classmethod.jp/cloud/aws/amazon-rds-mysql56-new-features/)

ということで、上記の参考にならい `twemproxy` の設定と簡単なベンチマークを取ってみたい。既に上記の参考等で同様のベンチマークを取られていたりするが手を動かしたい性分なので自分でもやってみる。

***
***

<H2>設定</H2>

***

<H3>もろもろ割愛</H3>

`ElastiCache` の `memcached` 設定自体は割愛するが構築する `ElastiCache` の `memcached` は以下のような構成。

 * `cache.t1.micro`
 * `memcached 1.4.14`
 * 2 nodes

以下、今回のキモになりそうな部分だけ手順等を記載。

***

<H3>twemproxy(nutcracker)</H3>

`twemproxy` は `twitter` でも利用されていたのされるツールだが、現在は `nutcracker` という名前のようだ。インストールはコンパイルが必要となるので注意。今回利用した環境は `EC2` インスタンス（`t1.micro` からの `Amazon Linux`）。

~~~~
wget https://twemproxy.googlecode.com/files/nutcracker-0.3.0.tar.gz
tar zxvf utcracker-0.3.0.tar.gz
cd utcracker-0.3.0
./configure
make
sudo make install
~~~~

ビルドにあたっては開発ツールが必要になるので適宜インストールしておくのをお忘れなく...

以下は一応ヘルプ表示。

~~~~
This is nutcracker-0.3.0

Usage: nutcracker [-?hVdDt] [-v verbosity level] [-o output file]
                  [-c conf file] [-s stats port] [-a stats addr]
                  [-i stats interval] [-p pid file] [-m mbuf size]

Options:
  -h, --help             : this help
  -V, --version          : show version and exit
  -t, --test-conf        : test configuration for syntax errors and exit
  -d, --daemonize        : run as a daemon
  -D, --describe-stats   : print stats description and exit
  -v, --verbosity=N      : set logging level (default: 5, min: 0, max: 11)
  -o, --output=S         : set logging file (default: stderr)
  -c, --conf-file=S      : set configuration file (default: conf/nutcracker.yml)
  -s, --stats-port=N     : set stats monitoring port (default: 22222)
  -a, --stats-addr=S     : set stats monitoring ip (default: 0.0.0.0)
  -i, --stats-interval=N : set stats aggregation interval in msec (default: 30000 msec)
  -p, --pid-file=S       : set pid file (default: off)
  -m, --mbuf-size=N      : set size of mbuf chunk in bytes (default: 16384 bytes)
~~~~

設定は `YAML` 形式で書き、ファイル名を `nutcracker.yml` として以下のように設定した。

~~~~
hoge:
  listen: 127.0.0.1:11211
  hash: fnv1a_64
  distribution: ketama
  timeout: 100
  auto_eject_hosts: true
  server_retry_timeout: 2000
  server_failure_limit: 1
  servers:
   - 192.168.xxx.1:11211:1
   - 192.168.xxx.2:11211:1
~~~~

各種設定についてはざっくり下記の通り。

 * `listen` は `nutcracker` が `Listen` する `IP` や `Port` を指定する
 * `hash` はハッシュ関数の種類を指定する
 * `distribution` はキー配布の種類を指定する
 * `auto_eject_hosts` はノードが停止した場合に停止したノードを切り離すかを指定する
 * `server_retry_timeout` は `auto_eject_host` が `true` の場合にノード（サーバー）への再接続の時間を指定（デフォルトは `30000` ミリ秒）
 * `server_failure_limit` は `auto_eject_host` が `true` の場合にノード（サーバー）への接続失敗の上限（デフォルトは `2`）
 * `servers` には `memcached` のノードを指定する（追記：ノードは `IP` アドレスで指定する必要がある）

設定後に以下のようにして `nutcracker` を起動する。

~~~~
nutcracker -d -c nutcracker.yml
~~~~

`-d` はデーモンモード、`-c` で設定ファイルを指定する。起動すると `11211` 番ポートと `22222` 番ポートが `Listen` する。`22222` 番ポートは `nutcracker` のステータスが確認することが出来るので...起動したことは下記のようにして確認。

~~~~
curl -s localhost:22222 | jq .
~~~~

以下のような結果が `JSON` で返ってくる。

~~~~
{
  "hoge": {
    "192.168.xxx.53": {
      "out_queue_bytes": 0,
      "out_queue": 0,
      "in_queue_bytes": 0,
      "in_queue": 0,
      "response_bytes": 16,
      "server_eof": 0,
      "server_err": 0,
      "server_timedout": 0,
      "server_connections": 1,
      "server_ejected_at": 0,
      "requests": 2,
      "request_bytes": 42,
      "responses": 2
    },
    "192.168.xxx.164": {
      "out_queue_bytes": 0,
      "out_queue": 0,
      "in_queue_bytes": 0,
      "in_queue": 0,
      "response_bytes": 0,
      "server_eof": 0,
      "server_err": 0,
      "server_timedout": 0,
      "server_connections": 0,
      "server_ejected_at": 0,
      "requests": 0,
      "request_bytes": 0,
      "responses": 0
    },
    "fragments": 0,
    "forward_error": 0,
    "server_ejects": 0,
    "client_connections": 2,
    "client_err": 0,
    "client_eof": 0
  },
  "timestamp": 1398523199,
  "uptime": 213,
  "version": "0.3.0",
  "source": "ip-192-168-xxx-138",
  "service": "nutcracker"
}
~~~~

おお、動いているようだ。尚、構成下記のようなイメージとなる。

![](images/2014042701.png)

***

<H3>memslap の導入</H3>

`memcached` のベンチマークには `memslap` というツールを使う。`memslap` は `libmemcached` に同梱されている。

~~~~
sudo yum install libmemcached.x86_64
~~~~

簡単な使い方は下記の通り。

~~~~
memslap -s 127.0.0.1:11211
~~~~

***
***

<H2>ベンチマーク</H2>

いきなりで恐縮だが、とりあえず以下のコマンドにてベンチマークを取ってみた。

~~~~
memslap -s ${memcached_host}
~~~~

`${memcached_host}` には `nutcracker.yml` の `listen` で指定されたホストと `ElastiCache` のノード一台を指定してベンチマークを取得した。

***

<H3>twemproxy 経由</H3>

以下の通り `3` 回テスト。

 1. 5.108 sec
 1. 4.885 sec
 1. 6.067 sec

***

<H3>ElastiCache for memcached 1 node</H3>

以下の通り `3` 回テスト。

 1. 4.139 sec
 1. 5.472 sec
 1. 3.923 sec

***

細かいパラメータの指定は行っていないが、上記の通り `twemproxy` 経由は若干オーバーヘッドがあるようだ。

***
***

<H2>twemproxy の動きを見てみる</H2>

***

<H3>今更だけど</H3>

`twemproxy` って何してんだろうってことで...`debug` モードを有効にしてコンパイルしてみる。

~~~~
CFLAGS="-ggdb3 -O0" ./configure --enable-debug=full
make
sudo make install
~~~~

デバッグモードを最高にしてログを `/tmp/debug.log` に出力して起動する場合には下記のように実行する。

~~~~
nutcracker -d -c nutcracker.yml -v 11 -o /tmp/debug.log
~~~~

***

<H3>ログ見てみる</H3>

以下のように `set` と `get` のリクエストを投げた場合...

~~~~
echo -e "set test 0 0 3\r\n123\r" | nc localhost 11211
~~~~

からの...

~~~~
echo "get test" | nc -C localhost 11211 
~~~~

を実行すると以下のように結果は表示される。

~~~
VALUE test 0 3
123
END
~~~

ログは以下の `gist` のようなログが出力される。

 * [(twemproxy) nutcracker の冗長モードで起動した場合のログ](https://gist.github.com/inokappa/11331404)

結構な量のログが流れる中で `set` した際に...

~~~~
nc_server.c:644 key 'test' on dist 0 maps to server 'xxx.xxx.x.164:11211:1'
~~~~

上記のようなログが記録される。また、`get` の場合にも...

~~~~
nc_server.c:644 key 'test' on dist 0 maps to server 'xxx.xxx.x.164:11211:1'
~~~~

上記のようなログが流れる。`twemproxy` は該当の `key` がどのノードに記録されているかを記憶しているのかな...。ということは、該当の `key` がストアされているノードが停止した場合にはどうなるのかしらということでノードを停止してみたところ...

~~~~
$ echo "get test" | nc -C localhost 11211
SERVER_ERROR Connection timed out
~~~~

上記のようにエラーとなった。また、`key` がストアされていないノードへの問い合わせが発生した場合には下記のような出力となった。

~~~~
$ echo "get test" | nc -C localhost 11211
END
~~~~

***

<H3>自分なりの理解</H3>

 * `twemproxy` は何らかのアルゴリズムで複数のノードの中から選択して一台のノードにデータを `set` している
 * そして...キーがどのノードに `set` されたかを管理している
 * `get` のリクエストがあった場合にはデータが記録されているノードから `get` してクライアントに返している
 * あくまでも `twemproxy` はキャッシュクラスタへのリクエストをルーティングしているだけでノード間のデータ同期等の面倒までは見ていない（当たり前と言えば、当たり前だが...）

***
***

<H2>最後に</H2>

 * `twemproxy` を利用すれば `ElastiCache for memcached` クラスタでデータの分散処理は行える
 * 但し、データの可用性、冗長性は期待できないので注意する
 * 可用性、冗長性の考慮が必要であれば `ElastiCache for Redis` 又は `RDS for MySQL 5.6` + `memcached Plugin` + `Multi AZ` を選択する必要があると思う
 * `RDS for MySQL 5.6` + `memcached Plugin` の構成については別途で簡単なベンチマークをとってみたい

***
***
