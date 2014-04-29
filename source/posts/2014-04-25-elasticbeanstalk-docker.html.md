---
title: Elastic Beanstalk で Docker が使えるようになったそうなので試してみる
date: 2014-04-25 00:43 JST
tags: AWS, Elastic Beanstalk, Docker
image_src: ![](images/2014042501.png)
---

<H2>はじめに</H2>

 * 私自身 `Elastic Beanstalk` は初めて
 * タイトル通り `Elastic Beanstalk` で `Docker` が使えるようになったそうなので試してみる

***
***

<H2>追記</H2>

<H3>複数の Port を晒すことが出来るのか？</H3>

ドキュメントを読むと現時点ではコンテナで `Listen` している複数のポートを晒すのは非対応のようだ。

 * [Dockerfile and Dockerrun.aws.json](http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_docker_image.html)

>You can specify multiple container ports, but AWS Elastic Beanstalk uses only the first one to connect your container to the host's reverse proxy and route requests from the public Internet.

ああ、残念。`EXPOSE` させている最初のポートが `ELB` の `80` ポートにバインドされるようだ。

<H3>後始末の手順</H3>

 1. `Elastic Beanstalk` で生成されたインスタンスを `Terminate`
 1. `Elastic Beanstalk` で生成された `Security Group` を `Delete`
 1. 最後にアプリケーション環境を `Terminate`

って感じ。

***
***

<H2>とりあえずザックリ使う</H2>

<H3>Elastic Beanstalk とは</H3>

初 `Elastic Beanstalk` なのでちょっとだけ何者かを調べたが一言で言うと...

 * `PaaS`

`Heroku` や `Engine Yard` 等（共に使用経験ほぼ無し）と同じ土俵な製品なのかなという位の理解しか無い。すいません。

***

<H3>参考</H3>

 * [【AWS発表】AWS Elastic Beanstalk for Docker](http://aws.typepad.com/aws_japan/2014/04/aws-elastic-beanstalk-for-docker.html)
 * [AWS Elastic BeanstalkでDockerコンテナをデプロイしてみた](http://dev.classmethod.jp/tool/docker/aws-elastic-beanstalk-for-docker-1/)
 * [AWS Elastic Beanstalk（初心者向け 超速マスター編）JAWSUG大阪](http://www.slideshare.net/shimy_net/aws-elastic-beanstalk-23314834)

クラスメソッドさん仕事が速い！

***

<H3>やってみよー</H3>

マネジメントコンソールから `Elastic Beanstalk` を選んでプルダウンから `Docker` を選択して... `Launch Now` をクリックするだけ...

![](images/2014042502.png)

以下のような画面に切り替わる。

![](images/2014042503.png)

そして暫くすると以下のような画面になれば準備完了！！

![](images/2014042504.png)

この間に以下のようなことが行われているようだ。

 * `ELB` の作成
 * `S3` のバケット生成
 * `Security Group` の生成（後になって環境を `Terminate` 出来ないよーとなる原因）
 * `Auto Scaling` のグループ作成
 * `Cloud Watch` の設定
 * `EC2` インスタンスの作成

かなりのお仕事を裏でやっててスゴイなあと思いつつ `Security Group` を消さないと環境を `Terminate` 出来なかったりするのは要注意（私的見解）。

***

<H3>docker build</H3>

ウンチクはこれくらいにして、さっそく `docker build` してみる。参考にさせて頂いた資料を斜め読みすると従来の `Dockerfile` を直接利用出来る以外に以下から設定出来るとのこと。

 * `Dockerfile`
 * `Docker.aws.json` という `JSON` 形式のファイル
 * `Dockerfile` or `Docker.aws.json` と `ADD` する設定ファイル各種

そのまま `Dockerfile` 使えるのが嬉しいですな。ということで既に作成済である以下の `Dockerfile` を使ってみる。

 * [dockerfiles / InfluxDB / Dockerfile](https://github.com/inokappa/dockerfiles/blob/master/InfluxDB/Dockerfile)

![](images/2014042505.png)

上図の `Upload and Deploy` をクリック！

![](images/2014042506.png)

ローカル `PC` から `Dockerfile` をアップロード。同時に `Version lable` を記入する。この `Verion Label` をいい加減に設定すると怒られる。（何度か試すうちに同じ `label` とかになると...）アップロードして記入した後で `Deploy` をクリックすると...

![](images/2014042507.png)

おお、始まりましたな。暫くすると...

![](images/2014042508.png)

おやっと様でした。右上のリンクをクリックすると...

![](images/2014042509.png)

早速ログイン！といきたいところだったがログインが出来なかったので何か設定が足りないようだ...上記の画面が見れるということは`InfluxDB` の `8083` ポートが `ELB` で `80` ポートにバインドされているっぽい。では、その他のポートはどこで設定するかな調べなきゃ←イマココ

***
***

<H2>おまとめ申し上げます</H2>

 * 普通に `EC2` インスタンスでも `Docker` は扱うことが出来るけどボタン一発は嬉しい
 * 既存の `Dockerfile` が一応、それなりに動くのも更に嬉しい
 * でも、細かい設定とかどうすれば良いのかは引き続き調べよう

***
***
