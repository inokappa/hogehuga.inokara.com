---
title: cloudwatch-agent というツールを作ったので cloudwatch-agent-chef というのも作った
date: 2014-05-23 10:42 JST
tags: CloudWatch, AWS, Chef
---

<H2>はじめに</H2>

 * `CloudWatch` のエージェントっぽいのを `bash` で作った
 * どうやってインスタンスに配るんですか？という問いに対して、じゃあ `Chef` でというノリだけで作った

***
***

<H2>cloudwatch-agent について</H2>

せっかくなので `cloudwatch-agent` についてもちょっと。

***

<H3>そもそも何？</H3>

 * [inokappa/cloudwatch-agent](https://github.com/inokappa/cloudwatch-agent)
 * 単にインスタンス内の `Load Average` とか `Memory` 使用率、`Disk` 使用率を `CloudWatch` にポストする
 * 初回実行時に `Alarm` の設定までしてしまう
 * 通知については `SNS` の `Topic ARN` を指定してメールで通知する（事前に `Topic ARN` を作っておく必要がある）
 * `aws cli` の `Python` 版を利用する

***

<H3>なんで作った？</H3>

 * 当初は `Sensu` をやりたかったけどチーム内を説得しきれなかった為
 * 引き継ぎし易く、且つインフラチーム内でもメンテナンスし易い `bash` でとりあえず作った

***

<H3>Sensu ぽっく</H3>

簡単に監視項目を追加出来るようにそれぞれの監視項目をシェル関数として定義してメインのスクリプトから呼ぶだけのアーキテクチャ（笑）で実装した。例えば、特定のプロセス数を監視したい場合には `plugins` ディレクトリ以下に `get_` から始まる以下のようなシェル関数を置くだけ。

~~~~
function get_count_process_middleman() {
  v=`ps aux | grep middleman | grep -v grep`
  t="1"
  u="Count"
}
~~~~

尚、上記で設定される変数は下記の通り。

 * `v` => 監視結果
 * `t` => しきい値
 * `u` => `CloudWatch` にポストする際に利用する `Unit` を指定。

一応、監視そのものを行うスクリプトには以下のようなルールを設けている。

 * 関数名とファイル名（`.sh` を除く）は同じにする
 * 必ず `get_` から始める

このルールに従って監視スクリプトを置けば監視項目の追加が出来る。

***

<H3>課題</H3>

ノリと勢いで作ったものだけに上記にあるように課題も多い。

 * エラー処理が全く無い
 * 動作ログ自体の出力が無い

など、など。今後、誰も使わないかもしれないが気が向いたら改善していきたい。

***
***

<H2>aws cli について</H2>

`cloudwatch-agent` で使っている `aws cli` についてちょっと。 

<H3>put_metrics_data</H3>

メトリクスデータを `CloudWatch` に投げるコマンドで `cloudwatch-agent` 内では以下のように利用している。

~~~~
function put_metrics_data() {
  aws --region ${REGION} cloudwatch put-metric-data \
    --namespace "${NAMESPACE}" \
    --metric-data file:///tmp/${1}.json
}
~~~~

`--namespace` を引数で指定する以外は `--metric-data` で以下のような `JSON` ファイル化したメトリクスデータを引数として与える。

~~~~javascript
[
  {
    "MetricName": Metrics Name,
    "Value": Metrics Value,
    "Unit": Unit,
    "Dimensions": [
      {
        "Name": "Instanceid",
        "Value": Instance_id
      },
      {
        "Name": "Hostname",
        "Value": hostname
      }
    ]
  }
]
~~~~

詳細については[こちら](http://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-data.html)で。

***

<H3>put-metric-alarm</H3>

このコマンドは初回のみしか利用されないがスクリプト内で以下のように利用されている。

~~~~
  aws --region ${REGION} cloudwatch put-metric-alarm \
    --actions-enabled \
    --alarm-name ${NAMESPACE}-${hostname}-${1} \
    --alarm-actions ${ARN} \
    --ok-actions ${ARN} \
    --metric-name ${1} \
    --namespace ${NAMESPACE} \
    --statistic Average \
    --period 300 \
    --evaluation-periods 1 \
    --threshold ${t} \
    --unit ${u} \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --dimensions Name=Instanceid,Value=${Instance_id} Name=Hostname,Value=${hostname}
~~~~

詳しくは[こちら](http://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-data.html)を見て頂くとして、この設定の場合に以下の条件で `Alarm` が設定される。

 * しきい値は `--threshold ${t}` で指定
 * `--period` で指定された `300` 秒の間にしきい値が `--evaluation-periods` で指定された回数（`1`）を超えた場合に...
 * `--alarm-actions` で指定された `SNS` の `Topic ARN` を利用して通知する

日本語がオカシイかもしれないけどこんな感じ。

***
***

<H2>cloudwatch-agent-chef について</H2>

で `Chef` で配布すっかーと作ったのが `cloudwatch-agent-chef`...これまた `Sensu` のマネ。

***

<H3>なにすんの？</H3>

 * `cloudwatch-agent` をインスタンスに配布する
 * `Cron` も設定されるの収束させて `5` 分後に監視が開始される
 * 初回のポストの際には `Alarm` の設定も行われる
 * 監視項目の追加は `LWRP` の独自リソースで設定する
 * 使い方は [README.md]() を...

***

<H3>Cookbook</H3>

 * [cloudwatch-agent-chef](https://github.com/inokappa/cloudwatch-agent-chef)

***

<H3>イケてないとこ</H3>

 * `AWS` の `API` キーは `Encrypt` な `data_bags` で管理したかった（現状は `Attribute`）

***

<H3>ここでも Sensu っぽく</H3>

監視項目を追加したい場合には `LWRP` の独自リソース（`cloudwatch_agent_chef_create_script`）を下記のように書いて収束させる。

~~~~ruby
cloudwatch_agent_chef_create_script "get_hoge-huga.sh" do
  action :create
  metrics_name "hoge-huga"
  get_metrics_command "v=`ps aux| grep hoge | wc -l`"
  unit "Count"
  threshold "1"
  bin_path "#{BINPATH}"
end
~~~~

上記をレシピに書いて収束を実行させると...以下のような内容の `#{BINPATH}/plugins/get_hoge-huga.sh` が作成される。

~~~~
function get_hoge-huga() {
  v=`ps aux| grep hoge | wc -l`
  t="1"
  u="Count"
}
~~~~

***
***

<H2>まとまってないけど</H2>

 * `CloudWatch` とシェルで簡単にインスタンスリソースを監視することが出来た
 * `Sensu` のエージェント型のアーキテクチャやプラグイン機構等自分なりの認識の中で色々と真似てみた
 * また `Cookbook` を作るにあたって `LWRP` を利用してみたがこちらは簡単に利用することが出来た
 * 気になったのが `LWRP` で定義したリソース名にハイフンが使えないっぽいのはホント？
 * 車輪の再発明だったが勉強になった

***
***
