---
title: Sensu の導入を諦めて CloudWatch にした件
date: 2014-05-09 20:12 JST
tags: AWS, CloudWatch, Sensu, Monitoring
---

<H2>悔しい</H2>

タイトル通り、とあるプロジェクトに密かに導入を進めていた `Sensu` の導入を諦めて、とりあえず `CloudWatch` で当面は凌ぐことにしたので言い訳代わりのメモ。

***
***

<H2>Sensu の導入を諦めた理由</H2>

なぜ `Sensu` の導入を諦めたか。以下の理由が挙げられる。

 * 自分の社内調整と社内プレゼン不足だった
 * 社内検証が大幅に遅れてしまい、社内での運用基準みたいなものを満たせそうになかった
 * 今後、面倒を見れないことが確定したから
 * プロジェクト自体のサーバーが少ない
 * 幸いにも `AWS` 環境ということで `CloudWatch` もある

ということで、導入を諦めたのは `Sensu` がどうこうではなくて自分の進め方や時間的な制約によるものである。機会があれば、改めて `Sensu` は導入したいと考えているが、まずは `CloudWatch` で進めることにした。

***
***

<H2>CloudWatch 入門</H2>

`CloudWatch` について各種情報へのリンクを掲載。既に公式ドキュメントやスライド、実際に利用されている方のメモなど充実している。

<H3>CloudWatch とは</H3>

以下の資料は必読。

 * [Amazon CloudWatch](http://aws.amazon.com/jp/cloudwatch/)
 * [CloudWatch によるインスタンスのモニタリング](http://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/using-cloudwatch.html)
 * [Amazon CloudWatch](https://aws.amazon.com/jp/cloudwatch/?nc1=h_l2_dm)
 * [CloudWatchの使い方](http://www.slideshare.net/ShinsukeYokota/cloudwatch-29076381)

以下のような特徴を持つ。

 * CPU 使用率、データ転送、ディスク使用のアクティビティ、インスタンスのステータスを監視
 * 5 分間隔にて監視を行う
 * 結果は 2 週間保存される
 * CloudWatch のコンソールからメトリクス毎、又はインスタンス ID 毎に監視メトリクスを確認出来る
 * Alarm を設定すると各しきい値に応じて通知を出すことも出来る

<H3>CloudWatch カスタムメトリクス</H3>

以下はカスタムメトリクスについて書かれた資料の一部。

 * [はじめてのCloudWatch（AWS） 〜カスタムメトリクスを作って無料枠でいろいろ監視する〜](http://qiita.com/hilotter/items/5d5c7cddb5e580bd7aa5)
 * [CloudWatch編～Custom Metricsパート①～](http://recipe.kc-cloud.jp/archives/4651)
 * [AWSのCloudWatchでカスタムメトリックスを使用する](http://d.hatena.ne.jp/ke-16/20130311/1363025895)

独自のメトリクスを `API` を介して `CloudWatch` で監視することも出来る。
例えば、`CloudWatch` 単体の場合、通常のサーバーリソース監視で取得出来るような...

 * Load Average
 * Memory の使用量
 * 各種アプリケーションサービスの稼働監視（Port の死活やプロセスの状況） 

等が取得出来ない。そこで、`API` ツールと組み合わせて独自のスクリプトでこれらを監視することが出来る。

***
***

<H2>設定とかスクリプトとか</H2>

***

<H3>公式スクリプト</H3>

以下は `Amazon` が公式に配布している `CloudWatch` カスタムメトリクス用スクリプト。

 * [Amazon CloudWatch Monitoring Scripts for Linux](http://aws.amazon.com/code/8720044071969977)

上記のサイトを見ると以下の項目が取得出来るとのこと。

 * Memory Utilization (%)
 * Memory Used (MB)
 * Memory Available (MB)
 * Swap Utilization (%)
 * Swap Used (MB)
 * Disk Space Utilization (%)
 * Disk Space Used (GB)
 * Disk Space Available (GB)

尚、上記のスクリプトは `Perl` で書かれている。

今回、`Amazon` の公式スクリプトを使うことも検討したが、今後のメンテナンスや `Load Average` や `HTTP` のステータス監視をまとめて監視したかったので簡単な `bash` スクリプトを各種サイトを参考にこさえることにした。

***

<H3>クライアントツールの準備</H3>

CloudWatch のクライアントツールをダウンロードして展開。

~~~~
wget http://ec2-downloads.s3.amazonaws.com/CloudWatch-2010-08-01.zip
unzip CloudWatch-2010-08-01.zip
~~~~

CloudWatch のクライアントツールを /opt/aws 以下に移動。

~~~~
sudo cp CloudWatch-1.0.20.0 /opt/aws/
sudo ln -nsf /opt/aws/CloudWatch-1.0.20.0 /opt/aws/CloudWatch
~~~~

credential ファイルを修正する。

~~~~
cd /opt/aws/
sudo cp credential-file-path.template  /opt/aws/CloudWatch/credentials
~~~~

`/opt/aws/CloudWatch/credentials` を修正する。

~~~~
AWSAccessKeyId=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AWSSecretKey=yyyyyyyyyyyyyyyyyyyyyyyyyyyy
~~~~

***

<H3>bash スクリプト</H3>

以下の通り。

<script src="https://gist.github.com/inokappa/ea084cbcacdd0e203e43.js"></script>

今回は[こちら](http://d.hatena.ne.jp/ke-16/20130311/1363025895)で紹介されていたスクリプトを利用させて頂きつつ（有難うございました！）少し修正を加えた。

***
***

<H2>ということで...</H2>

今後の予定としては...

 * `bash` スクリプトの拡張
 * `Alarm` の設定
 * `API` ツールの詳細な調査

をやる。

***
***
