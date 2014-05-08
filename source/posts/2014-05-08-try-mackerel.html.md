---
title: mackerel をとりあえず試してみる
date: 2014-05-08 23:33 JST
tags: mackerel, Monitoring
---

<H2>はじめに</H2>

トライアルを申し込んでいた、はてな謹製の監視ツール `mackerel` のベータ版がリリースされたとのことで早速試してみた。

ちなみに以下のような環境で試してみた。

 * `EC2` インスタンス（`CentOS i6.5`）
 * `Docker` コンテナ（`CentOS 6.4`）
 * `Docker` コンテナ（`Debian Wheezy`）

***
***

<H2>参考</H2>

 * [Mackerel](https://mackerel.io/)
 * [Mackerelをベータ公開しました](http://blog-ja.mackerel.io/)
 * [特徴](https://mackerel.io/guide/features)
 * [用語集](http://help-ja.mackerel.io/entry/glossary)
 * [ユーザ定義のメトリックを投稿する](http://help-ja.mackerel.io/entry/advanced/custom-metrics)
 * [YAPC::Asia 2013ではてなのサーバ管理ツールの話のはなしをしました](http://yuuki.hatenablog.com/entry/2013/09/21/154911)

***
***

<H2>インプレッション</H2>

とりあえずインプレッション等...

***

<H3>第一印象</H3>

 * とっても簡単
 * `Sensu` 等と同じエージェント型の監視ツール
 * アカウント作成からオーガニゼーションの登録、ログイン、エージェントのインストール、監視開始まで `5` 分も掛からない
 * エージェントをインストールした後に暫くしたら `Hosts` にホストが登録されてた

***

<H3>サービスとロール</H3>

 * ホストを登録しただけでは監視は開始されない
 * サービスというホストのグループを作成し、その中でロールという役割の単位を作成し、そのロールにホストを紐付ける
 * 一つのホストは複数のサービスやロールに所属することが出来る

***
***

<H2>スクリーンショット</H2>

とりあえずスクリーンショットをば...

***

<H3>ログイン</H3>

![](images/2014050802.png)

***

<H3>ダッシュボード</H3>

![](images/2014050804.png)

***

<H3>Services （1）</H3>

![](images/2014050805.png)

***

<H3>Services （2）Role 毎のメトリクス</H3>

![](images/2014050806.png)

***

<H3>Host の一覧</H3>

![](images/2014050807.png)

***

<H3>Host 毎のメトリクス（1）</H3>

![](images/2014050801.png)

***

<H3>Host 毎のメトリクス（2）</H3>

![](images/2014050808.png)

***
***

<H2>気づいたこと</H2>

気づいた事など。

<H3>エージェントのインストール</H3>

 * `RPM` と `deb` パッケージ、そして実行形式のバイナリが配布されている
 * パッケージをダウンロードして `API` キーを設定してサービスを起動するだけ
 * `Install the Agent` を見ながらインストールすればホント簡単

***

<H3>ホスト毎の監視項目</H3>

以下の項目が監視対象となっている。

 * `load`
 * `memory`
 * `disk`
 * `interface`

以下のようにユーザー定義の監視項目も追加出来るようだ。

***

<H3>エージェントの設定ファイル</H3>

基本的に `API` キー以外の設定は不要のようだが、設定ファイル（`/etc/mackerel-agent/mackerel-agent.conf`）は下記の通り。

~~~~
# pidfile = "/var/run/mackerel-agent.pid"
# root = "/var/lib/mackerel-agent"
# verbose = false

# Configuration for Sensu Plugins
# Sensu checks (ref. http://sensuapp.org/docs/0.12/adding_a_check)
# Currently, metric type command can be used
# [sensu.checks.vmstat]
# command = "ruby /etc/sensu/plugins/system/vmstat-metrics.rb"
# type = "metric"
# [sensu.checks.curl]
# command = "ruby /etc/sensu/plugins/http/metrics-curl.rb"
# type = "metric"
~~~~

`Sensu` の文字列がチラッと...これはユーザー定義のメトリクスを `Sensu` プラグインの互換フォーマットで送ることが出来るらしい。ほうほう。ちなみに `Sensu` プラグイン互換のフォーマットって以下のような感じ。

~~~~
absinthe.local.load_avg.one 0.89  1365270842
absinthe.local.load_avg.five  1.01  1365270842
absinthe.local.load_avg.fifteen 1.06  1365270842
~~~~

みんな大好き `Sensu` の `Metrics` プラグインが使えるというのは嬉しいかも。

***

<H3>API</H3>

`API` も実装されていて、現時点（`v0`）では以下のようなことが出来るようだ。

 * ホスト情報の登録
 * ホスト情報の取得
 * ホスト情報の更新
 * ホストのステータスの更新
 * メトリックの投稿
 * ホストの一覧

ホスト一覧は以下のように取得出来る。

~~~~
curl -s -H "X-Api-Key:your_api_key" https://mackerel.io/api/v0/hosts.json
~~~~

また、ホストの情報は以下のように取得出来る。

~~~~
curl -s -H "X-Api-Key:your_api_key" https://mackerel.io/api/v0/hosts/${hostid}
~~~~

`X-Api-Key` は必須。`${hostid}` はダッシュボードからは確認することが出来ないようなので、ホスト一覧から確認するか、エージェントログに記録されているようなので確認する。尚、レスポンスは `JSON` で返ってくるのでレスポンスは `Jq` 等でよしなに。

以下はホスト情報のレスポンス。（一部）

![](images/2014050809.png)

***
***

<H2>引き続き...</H2>

触っていきたいと思う。

***
***
