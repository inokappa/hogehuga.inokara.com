---
title: Sensu の InfluxDB Extenstion を使う（可視化ツールを試してみる）
date: 2014-04-23 22:18 JST
tags: Sensu, InfluxDB, AWS, Grafana, Tasseo
---

<H2>はじめに</H2>

`Sensu` の監視メトリクスを `Extension` を利用して `InfluxDB` に登録することが出来たので登録出来たメトリクスを見てみる。`InfluxDB` 自身でもグラフ表示が可能で以下のようにクエリの結果を表示することが出来るが無理をさせてしまうとブラウザが重くなってしまったりで使い勝手はイマイチな印象。

![](images/2014042303.png)

***

<H2>見る</H2>

<H3>influxdb-cli の準備</H3>

以下で紹介されていたので試してみた。可視化は `GUI` だけではない...（と思う

 * [InfluxDBとGrafanaとfluentdで、twitterデータのリアルタイム集計・可視化](http://qiita.com/ixixi/items/a56ea15b582c7f014a57)
 * [FGRibreau/influxdb-cli](https://github.com/FGRibreau/influxdb-cli)

インストールには `nodejs` が必要になるので `nodejs` をインストールするところから始める。

~~~~
sudo yum install -y nodejs npm --enablerepo=epel
~~~~

そして `npm` コマンドを利用してインストールを進める。

~~~~
sudo npm install influxdb-cli -g
~~~~

インストールが完了すると以下のようにアクセスする。

~~~~
influxdb-cli -d ${データベース}
~~~~

`influxdb-cli` のコマンドオプションは下記の通り。

![](images/2014042401.png)

ちょっと使ってみる。

<H3>influxdb-cli を使ってみる</H3>

`Sensu` のメトリクスデータを突っ込んでいるデータベースにアクセスしてみる。

~~~~
influxdb-cli -d sensu
~~~~

以下のように表示される。

![](images/2014042402.png)

適当にクエリを投げてみる。既にメモリの使用量についても `sensu` からメトリクスを送っている状態なので以下のような感じでクエリを投げる。

~~~~
select * from memory_metrics limit 5
~~~~

以下のように表示される。

![](images/2014042403.png)

ちょっと `where` 句を使って条件を絞ってみましょか。

~~~~
select * from memory_metrics group by time(1m) where metric =~ /.*ec2.internal.memory.used$/ limit 5
~~~~

以下のような結果が出力される。

![](images/2014042404.png)

おお。ポイントとしては...

 * `group by time(1m)` でメトリクスデータから `1` 分毎を抽出
 * `where metric =~` で正規表現を使って `metric` から該当する条件で抽出

上記のように `group by time($interval)` や `where` を利用して条件を絞った結果を得られることが出来る。

また、おもろいなあと思った機能が上記のようなクエリを登録しておくことが出来る `Continuous Queries` という機能。`RDBMS` で言うところのトリガー、ビューとかかな。

 * [Continuous Queries](http://influxdb.org/docs/query_language/continuous_queries.html)

以下のように `into ${Query Name}` を使って登録する。

~~~~
select * from memory_metrics group by time(1m) where metric = 'ip-xxx-xxx-x-xxx.ec2.internal.memory.used' into memory_metrics.used.1m
~~~~

登録した内容は `list continuous queries` で確認することが出来るし、削除は `drop continuous query ${id}` で削除することが出来る。登録と `list continuous queries` の流れは下記の通り。

![](images/2014042405.png)

尚、イジった感じだと下記のような点に注意。

 * `group by time` は必要
 * `where = metric` は正規表現は使えないっぽい

登録したクエリでデータの抽出をやってみる。

~~~~
select * from memory_metrics.used.1m limit 5
~~~~

以下のような結果が得られる。おお。

![](images/2014042406.png)

ついでにブラウザでも登録したクエリを試してみると以下が表示された。

![](images/2014042407.png)

驚いてばかりだったけど疲れてきたので `Grafana` とか `Tasseo` での可視化はまたこんど。

***

<H2>お疲れ様でした</H2>

 * `InfluxDB` まだわからん

***
