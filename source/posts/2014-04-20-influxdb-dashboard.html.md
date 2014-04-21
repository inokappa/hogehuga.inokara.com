---
title: InfluxDB を引き続き触ってみるぜよ（Grafana とか使ってみる）
date: 2014-04-20 07:48 JST
tags: influxdb,fluentd,よくわからん
---

<H2>kibana のようなダッシュボード</H2>

人はなぜか `kibana` のようなダッシュボードが好きである。普段は「マネジメントコンソールからポチポチするのはかったりー」と嘆きながら `AWS CLI` なんかバシバシ書いている `DevOps` な人も `kibana` のような黒い画面に折れ線グラフがバババーって表示されているダッシュボードは大好きなはず...

ということで `Grafana` をちょっとだけ触ってみるのと `Tasseo` というダッシュボードもあるようなのでそちらも触ってみる。

***

<H2>Grafana の存在</H2>

<H3>はじめに</H3>

`kibana` を `fork` して作られた `Grafana` というダッシュボードが `InfluxDB` を標準でサポートしているので触ってみる。

<H3>インストール</H3>

は簡単。以下からダウンロードする。

 * [Grafana Download](http://grafana.org/download/)

ダウンロードして `Web` サーバーの `DocumentRoot` に展開するだけ。

<H3>設定</H3>

以下等を参考にさせて頂いて `config.js` を修正する。

 * [Grafana Download](http://grafana.org/docs/)
 * [Grafana on InfluxDB をちょっとだけ触ってみた](http://qiita.com/sonots/items/8fbc92ff1c3e57ee7de7)

以下のように修正した。

```diff
--- config.sample.js    2014-04-20 13:12:19.116976452 +0900
+++ config.js   2014-04-20 19:42:50.471148947 +0900
@@ -21,8 +21,8 @@
      * Basic authentication requires special HTTP headers to be configured
      * in nginx or apache for cross origin domain sharing to work (CORS).
      * Check install documentation on github
-     */
     graphiteUrl: "http://"+window.location.hostname+":8080",
+     */

     /**
      * Multiple graphite servers? Comment out graphiteUrl and replace with
@@ -33,6 +33,17 @@
      *  }
      */

+
+    datasources: {
+      influx: {
+        default: true,
+        type: 'influxdb',
+        url: "http://"+window.location.hostname+":8086/db/hogehuga",
+        username: 'root',
+        password: 'root',
+      }
+    },
+
     default_route: '/dashboard/file/default.json',

     /**
```

<H3>InfluxDB へのデータ登録</H3>

`InfluxDB` へのデータ登録は以下のように `HTTP API` で登録してみる。

```
#!/bin/bash

T=`ruby -e 'puts Time.now.to_i * 1000'`
V=`echo $RANDOM`
cat << EOT > data
[
  {
    "name": "ahoaho",
    "columns": ["time", "value"],
    "points": [
      [$T, $V]
    ]
  }
]
EOT
curl -v --dump-header - -X POST 'http://localhost:8086/db/hogehuga/series?u=root&p=root' -d @data
```

やっぱりレスポンスボディに何かしらの情報が欲しいところだが上記のスクリプトを何度か叩く。実行後に`InfluxDB` のダッシュボードにて...

![](images/2014042005.png)

上記のように `select * from ahoaho` というようなクエリを投げると一応グラフ化してくれる。まあ、これでも悪くないけど。

<H3>Grafana での可視化</H3>

セットアップした `Grafana` でも `InfluxDB` に登録したデータを見てみる。

![](images/2014042101.png)

上図の通り操作することで `InfluxDB` に登録したデータをグラフとして表示してくれる。

おおって感じ。

***

<H2>Tasseo ってのもなうでヤングっぽい</H2>

`Tasseo` というなんと呼んでいいのか解らないダッシュボードも `InfluxDB` をバックエンドとして利用出来るようなのでちょっとだけ触ってみた。

 * [obfuscurity/tasseo](https://github.com/obfuscurity/tasseo)

<H3>セットアップ</H3>

ざっくり以下のような感じ。

```
git clone https://github.com/obfuscurity/tasseo.git
cd tasseo
bundle install
cd dashboards
cat << EOT > hoge.js
var metrics =
[
  {
    target: "value",
    series: "ahoaho"
  }
];
export INFLUXDB_URL=http://${your_host}:8086/db/hogehuga
export INFLUXDB_AUTH=root:root
foreman start
```

<H3>確認</H3>

`foreman start` を実行してブラウザで `http://${your_host}:5000` にアクセスすると以下のような超シンプルページが表示される。

![](images/2014042102.png)

`hoge` にアクセスしてみると...

![](images/2014042103.png)

上記のように `InfluxDB` から `ahoaho` の `value` を取得してシンプルなグラフとして表示している。ああ、かなりシンプル。

***

<H2>ということで</H2>

 * もう遅いので寝ます
 * 見た感じあきらかに用途が異なりそうなので `Grafana` と `Tasseo` を比較するつもりはありません
 * `InfluxDB` と各種ダッシュボードとの連携は試せたけど今のところは `Grafana` が良さそう
