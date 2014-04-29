---
title: Sensu の InfluxDB Extension を使ってみた
date: 2014-04-22 18:37 JST
tags: InfluxDB,Sensu,AWS
---

<H2>はじめに</H2>

 * `Sensu` から `InfluxDB` を扱えるようなので試してみる
 * プラグインだとばかり思っていたら `Extension not a handler` と書かれていました...
 * ということで初めて `Extension` を使ってみる
 * 検証の環境は全て `AWS` 上に構築した

***

<H2>Sensu の Extenstion</H2>

ドキュメントは下記。

 * [Sensu extensions](http://sensuapp.org/docs/0.12/extensions)

今回も `Powered by Google 翻訳` だが、`Extension` は下記のようなもののようだ。

 * `Sensu` の `EventMachine` 内の処理で実行される
 * 大量のデータに対して `Handle` や `Mutate` するのにオススメのようだ

***

<H2>設定等</H2>

<H3>事前に...</H3>

 * `InfluxDB` 側で `sensu` というデータベースを作っておく

<H3>github</H3>

ソースコードは以下に公開されている。

 * [lusis/sensu\_influxdb\_handler](https://github.com/lusis/sensu_influxdb_handlear)

ちなみに `sensu` から取り扱う場合には `influxdb` とかではなく `influx` と設定したりするので注意。

<H3>インストール</H3>

プラグインと同様にダウンロードして権限設定したいところだが、`extensions` にダウンロードする。

~~~~
cd /etc/sensu/extensions
sudo wget https://raw.githubusercontent.com/lusis/sensu_influxdb_handler/master/influx.rb
sudo chmod 755 influx.db
~~~~

<H3>設定</H3>

まずは `Metrics` の `Extension` としての設定を `/etc/sensu/conf.d` 以下に `metrics.json` というファイル名で設置する。

~~~~
{
  "handlers": {
    "metrics": {
      "type": "set",
      "handlers": [ "debug", "influx"]
    }
  }
}
~~~~

次に `Extension` 自身の設定（`InfluxDB` のホストや認証情報等）を `/etc/sensu/conf.d/influx.json`に設定する。

~~~~
{
  "influx": {
    "host": "localhost",
    "port": "8086",
    "user": "root",
    "password": "root",
    "database": "sensu"
    //"strip_metric": "somevalue"
  }
}
~~~~

<H3>メトリクスを採りたい監視項目の設定</H3>

今回は `Load Average` のメトリクスを採りたいので以下のプラグインを利用する。

 * load-metrics.rb

`/etc/sensu/plugins/` 以下にダウンロードして権限を付ける。

~~~~
wget -O /etc/sensu/plugins/load-metrics.rb https://raw.github.com/sensu/sensu-community-plugins/master/plugins/system/load-metrics.rb
chmod 755 /etc/sensu/plugins/load-metrics.rb
~~~~

そして `/etc/sensu/conf.d` 以下に `check_load.json` というファイル名で以下を作成する。

~~~~
{
  "checks": {
    "load_metrics": {
      "type": "metric",
      "handlers": ["influx"],
      "command": "/etc/sensu/plugins/load-metrics.rb",
      "subscribers": [
        "test"
      ],
      "interval": 10
    }
  }
}
~~~~

設定が終わったら `sensu-server` を再起動する。

~~~~
/etc/init.d/sensu-server restart
~~~~

再起動すると以下のようなログが出力される。

~~~~
{"timestamp":"2014-04-23T07:13:49.954990+0900","level":"info","message":"loaded extension","type":"handler","name":"influx","description":"outputs metrics to InfluxDB"}
~~~~

また、正常にメトリクスを送っている場合には下記のようなログも出力される。

~~~~
{"timestamp":"2014-04-23T07:14:54.290269+0900","level":"info","message":"handler extension output","extension":{"type":"extension","name":"influx"},"output":"InfluxDB: Handler finished"}
~~~~

***

<H2>InfluxDB を見てみる</H2>

<H3>Data Interface から</H3>

早速、`InfluxDB` の `Data Interface` からクエリを投げてみる。`series` は `sensu` で設定した `check_load.json` の `load_metrics` が定義されている。

また、下記の値が `InfluxDB` に登録されることになる。

 * host
 * metric
 * value

以下のようなクエリを投げることでメトリクスが取得出来る。

以下のようなクエリを投げることでメトリクスが取得出来る。

<img src="../../../images/2014042301.png" />

実行すると...以下のようにグラフが表示された！

<img src="../../../images/2014042302.png" />

おお。

***

<H2>todo</H2>

 * 後で `Grafana` や `Tasseo` で見てみる
 * `influxdb-cli` という CLI ツールがあるようなので試してみる
 * もすこし `InfluxDB` に入ったデータをちゃんと見てみる

***
