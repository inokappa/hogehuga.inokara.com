---
title: RabbitMQ の HTTP API でクラスタ、ノードの各種稼働ステータスメトリクスを監視してみる
date: 2014-04-28 00:21 JST
tags: RabbitMQ, Sensu, InfluxDB, Jq, よくわからん
---

<H2>はじめに</H2>

最近、随所に `RabbitMQ` を見かけるようになった気がするかっぱです、こんばんわ...

`RabbitMQ` も当然アプリケーションの一つでサーバーマシンのリソースは多少なりにも使っているわけで...ということで `RabbitMQ` の各種稼働ステータスを `RabbitMQ` 自身が提供する `Management HTTP API` で監視してみようという算段です。

今回は `Sensu` で利用している `RabbitMQ` の以下の項目を監視してみる為に色々とやってみようと思いますです。

 * メモリ使用量
 * コネクション数
 * チャンネル数

***
***

<H2>参考</H2>

 * [RabbitMQ Management HTTP API](http://hg.rabbitmq.com/rabbitmq-management/raw-file/3646dee55e02/priv/www-api/help.html)

***
***

<H2>やってみましょか</H2>

以下の全ての操作は `RabbitMQ` が稼働しているホスト上にて実行した。また、認証情報（下記の `-u` オプションで指定している部分）は環境に応じて読み替えて頂きたいのとデフォルトのパスワードは変更しましょう。

ちなみに `RabbitMQ` の管理用のダッシュボードは以下のように状態を示しております。

![](images/2014042907.png)

<H3>メモリ使用量を取得</H3>

こんな感じで。

~~~~
curl -s -u guest:guest localhost:15672/api/nodes | jq '.[].mem_used'
~~~~

結果は以下の通り。

![](images/2014042901.png)

***

<H3>コネクション数を取得</H3>

こんな感じで。

~~~~
curl -s -u guest:guest localhost:15672/api/connections | jq -r '.|length'
~~~~

結果は以下の通り。

![](images/2014042902.png)

***

<H3>チャンネル数を取得</H3>

こんな感じで。

~~~~
curl -s -u guest:guest localhost:15672/api/channels | jq -r '.|length'
~~~~

結果は以下の通り。

![](images/2014042903.png)

***
***

<H2>InfluxDB にメトリクスを放り込む</H2>

せっかくなので簡単な `bash` を使って取得出来る値を `InfluxDB` に突っ込んでみましょ。

<H3>こんな bash</H3>

~~~~
#!/bin/bash

T=`ruby -e 'puts Time.now.to_i * 1000'`
V1=`curl -s -u guest:guest localhost:15672/api/nodes | jq '.[].mem_used'`
V2=`curl -s -u guest:guest localhost:15672/api/connections | jq -r '.|length'`
V3=`curl -s -u guest:guest localhost:15672/api/channels | jq -r '.|length'`
cat << EOT > data
[
  {
    "name": "rabbitmq_metrics",
    "columns": ["time", "mem_used", "connection", "channels"],
    "points": [
      [$T, $V1, $V2, $V3]
    ]
  }
]
EOT
curl --dump-header - -X POST 'http://localhost:8086/db/rabbitmq/series?u=root&p=root' -d @data
~~~~

も少し工夫が出来ますが、とりあえず上記のスクリプトを `cron` 使ってグルグル回します。

<H3>InfluxDB の DataInterface</H3>

暫くすると `InfluxDB` にメトリクスが蓄積されてきたのが以下の図です。

![](images/2014042904.png)

おお、お手軽ですな。

***
***

<H2>tasseo で可視化</H2>

<H3>tasseo ってなんすか？</H3>

以下のような特徴があります。

 * `InfluxDB` をメトリクスストアとしてサポートするアプリケーション
 * 他にも `Graphite` もサポートしているようです
 * `Sinatra` で実装されていてグラフの表示には `D3.js` が利用されています
 * しきい値を設けることでしきい値を超えるような値を検知した場合にはグラフの色を変えて表示出来るようです

以下は `github` のリポジトリです。

 * [obfuscurity/tasseo](https://github.com/obfuscurity/tasseo)

個人的な印象としては...

 * かなりお手軽（以下の `tasseo` セットアップをご覧下さい）
 * `JavaScript` で設定を書ける（自分は `JavaScript` はよく知りません）
 * でも良くわからない

<H3>tasseo セットアップ</H3>

既に `Ruby` の `1.9.3` 以上がインストールされている環境でセットアップを行いました。

~~~~
git clone https://github.com/obfuscurity/tasseo.git
cd tasseo
bundle install
export INFLUXDB_URL=http://${influxdb_host}:8086/db/rabbitmq
export INFLUXDB_AUTH=${influxdb_user}:${influxdb_pass}
~~~~

次に `tasseo/dashboards` 以下に設定ファイルを作成します。設定ファイルは `JavaScript` で記述して `.js` という拡張子で保存します。今回は `InfluxDB` を利用するので以下のように設定します。

~~~~
var metrics =
[
  {
    target: "mem_used",
    series: "rabbitmq_metrics"
  },
  {
    target: "connection",
    series: "rabbitmq_metrics"
  },
  {
    target: "channels",
    series: "rabbitmq_metrics"
  }
];
~~~~

`target` には `InfluxDB` のカラムを指定します。`series` にはその名の通り `InfluxDB` の `series` を指定します。また、上記以外にも以下のような設定が可能です。

 * period - Range (in minutes) of data to query from Graphite. (optional, defaults to _5_)
 * refresh - Refresh interval for charts, in milliseconds. (optional, defaults to _2000_)
 * theme - Default theme for dashboard. Currently the only option is `dark`. (optional)
 * padnulls - Determines whether to pad null values or not. (optional, defaults to _true_)
 * title - Dictates whether the dashboard title is shown or not. (optional, defaults to _true_)
 * toolbar - Dictates whether the toolbar is shown or not. (optional, defaults to _true_)
 * normalColor - Set normal graph color. (optional, defaults to `#afdab1`)
 * criticalColor - Set `critical` graph color. (optional, defaults to `#d59295`)
 * warningColor - Set `warning` graph color. (optional, defaults to `#f5cb56`)
 * interpolation - Line smoothing method supported by D3. (optional, defaults to _step-after_)
 * renderer - Rendering method supported by D3. (optional, defaults to _area_)
 * stroke - Dictates whether stroke outline is shown or not. (optional, defaults to _true_)

<H3>tasseo 起動</H3>

設定が終わったら以下のようにして `tasseo` を起動します。

~~~~
foreman start
~~~~

又は以下のように起動することも出来るようです。

~~~~
bundle exec rackup -I lib -p 5000 -s thin
~~~~

起動してブラウザから `http://${tasseo_host}:5000` にアクセスすると...以下のような画面が表示されます。

![](images/2014042905.png)

おお。`rabbitmq-metrics` のリンクをクリックすると...

![](images/2014042906.png)

上図のように `InfluxDB` のメトリクスを取得出来ていますが...グラフ...描かれておりません。ガビーン。本来であれば以下のような図が表示されるようです...

![](https://raw.githubusercontent.com/obfuscurity/tasseo/master/lib/tasseo/public/i/tasseo.png)

とりあえず `3` つのメトリクスを一つの画面で確認出来ることは出来てます...。

***
***

<H2>さいごに</H2>

 * `RabbitMQ` の `HTTP API` を利用して `RabbitMQ` の各種ステータスを取得出来ますね
 * しかもお手軽に
 * `InfluxDB` に放り込むのも簡単です
 * `tasseo` は `kibana` や `Grafana` のようなメトリクスデータを表示する `dashboard` ツールの一つ
 * 結構お手軽や
 * でも、まだちゃんと動いてまへん

***
***
