---
title: EC2 インスタンスに Puppet で Sensu Server をセットアップする
date: 2014-05-03 23:38 JST
tags: AWS, Sensu, Puppet, Chef, Serverspec, RabbitMQ
---

<H2>はじめに</H2>

 * 自分自身 `Sensu` の環境を `RPM` パッケージからインストールした事は何度かあったが `Chef` や `Puppet` を利用してのインストールはほぼ皆無なので試してみた
 * `Chef` での導入については下記の参考サイトにて詳しく解説されている
 * ということで `Puppet` を使って導入を試みてみる

***
***

<H2>参考</H2>

<H3>Chef でのセットアップ</H3>

 * [Sensu Serverをインストールする手順メモ](http://d.hatena.ne.jp/rx7/20140502/p1)
 * [Sensu導入と初期設定について](http://blog.kenjiskywalker.org/blog/2014/05/02/newbie-sensu/)

***

<H3>Puppet まわり</H3>

 * [sensu/sensu-puppet](https://github.com/sensu/sensu-puppet)
 * [[Puppet Users] Puppet fails to run if ruby1.8 is not installed.](http://grokbase.com/t/gg/puppet-users/143yjsa5qq/puppet-fails-to-run-if-ruby1-8-is-not-installed)
 * [yum repo and package dependencies with puppet](http://rene.bz/yum-repo-and-package-dependencies-puppet/)
 * [Docs: Type Reference](http://docs.puppetlabs.com/references/latest/type.html)
 * [入門Puppet - Automate Your Infrastructure](http://tatsu-zine.com/books/puppet)
 * [puppet の言語構造](http://mizzy.org/blog/2007/03/19/1/)
 * [andytinycat/puppet-omnibus](https://github.com/andytinycat/puppet-omnibus)

`andytinycat/puppet-omnibus` は今回は触れていないが、既に稼働している `Ruby` の環境で `Puppet` の動作に必要な `Ruby` の関連パッケージをビルドしてくれるっぽい。機会があったら試してみたい。

***
***

<H2>設定</H2>

<H3>環境</H3>

 * `t1.micro`
 * `Market Place` の `CentOS`（`ami-eb6b0182`）

`Amazon Linux` ではなく `CentOS` を選んだ理由は以下の「困った点」に書いた。

***

<H3>Puppet を動かすまで</H3>

結局、`Puppet` で全部は出来ないので手動で以下の手順を行う。

~~~~
yum -y update openssl
yum -y update
yum install -y gcc-c++ patch readline readline-devel zlib zlib-devel libyaml-devel libffi-devel openssl-devel make bzip2 autoconf automake libtool bison iconv-devel curl wget vim
/etc/init.d/iptables stop
chkconfig iptables off
rpm --import https://yum.puppetlabs.com/RPM-GPG-KEY-puppetlabs
wget https://yum.puppetlabs.com/el/6/products/x86_64/puppetlabs-release-6-10.noarch.rpm
rpm -ivh puppetlabs-release-6-10.noarch.rpm
yum install puppet
~~~~

ここまでの手順で `AMI` 化しておくと良いかも。

***

<H3>Sensu のセットアップに必要な Puppet モジュールのインストール</H3>

`Chef` で言うところの `Community Cookbook` にあたる（？）モジュールを幾つか以下のようにインストールしておく。

~~~~
sudo puppet module install example42/redis
sudo puppet module install puppetlabs/rabbitmq
sudo puppet module install sensu/sensu
~~~~

上記の操作で以下のモジュールをインストールすることになる。

 * [example42/redis](https://forge.puppetlabs.com/example42/redis)
 * [puppetlabs/rabbitmq](https://forge.puppetlabs.com/puppetlabs/rabbitmq)
 * [sensu/sensu](https://forge.puppetlabs.com/sensu/sensu)

このようなモジュールは `Puppet Forge` と呼ばれるサイトにて管理されている。

***

<H3>Sensu のセットアップ</H3>

ここまで問題無く準備が整えば後は簡単。以下のマニフェスト（`Chef` で言うとこのレシピ）を `/etc/puppet/manifests/site.pp` として保存する。

<script src="https://gist.github.com/inokappa/206bbc5276ef6b90d81c.js"></script>

尚、`${your_password}` については環境に応じて適宜設定すること。保存したら以下のように `puppet apply` を利用してマニフェストを適用する。

~~~~
cd /etc/puppet
puppey apply manifests/site.pp
~~~~

暫くすると以下のように `Puppet` によるプロビジョニングが完了する。途中、ところどころで `Warning` や `Error` が出てしまっているが静観。

![](images/2014050402.png)

詳細な動作ログについては以下の通り。

 * [Puppet を使って Sensu Server をプロビジョニングした際のログ](https://gist.github.com/inokappa/40846327dbfaff19b298)

プロビジョニングは成功するものの `Sensu` 関連が全く起動していない模様。

一応、`/var/log/sensu/` 以下に `sensu-client.log` が出力されていたので見てみると...

~~~~
{"timestamp":"2014-05-04T00:50:43.590293+0000","level":"error","message":"[amqp] Detected TCP connection failure"}
{"timestamp":"2014-05-04T00:50:43.590639+0000","level":"fatal","message":"rabbitmq connection error","error":"failed to connect"}
~~~~

むむ。どうやら `RabbitMQ` への接続に失敗しているようだ。今回は特に `SSL` の設定をいれていないので証明書や鍵ファイルの置き場については特に気にしなくてよさそうなのでおそらく `RabbitMQ` に `Sensu` 用の設定が入っていないと思われる。

~~~~
curl -u guest:guest localhost:15672/api/vhosts
~~~~

で `RabbitMQ` の `Virtual Host` の一覧を取得出来るので確認すると...

~~~~
[{"name":"/","tracing":false}]
~~~~

ああ、なるほど。ということで、もう一度マニフェストを適用してみると...

~~~~
cd /etc/puppet
puppet apply manifests/site.pp
~~~

以下のようなログが出力された。

 * [Puppet を使って Sensu Server をプロビジョニングした際のログ（2）](https://gist.github.com/inokappa/38440fb4149a176d610f)

ついでに `RabbitMQ` の `Virtual Host` も...

~~~~
[{"name":"/","tracing":false},{"recv_oct":759,"recv_oct_details":{"rate":0.0},"send_oct":1113,"send_oct_details":{"rate":0.0},"name":"/sensu","tracing":false}]
~~~~

ちゃんと追加されているが...なぜか `Sensu Server` 等のプロセスが上がらないので、もう一度マニフェストを適用すると無事に `Sensu Server` 等のプロセスが起動した。

<H3>設定の確認とダッシュボードへのアクセス</H3>

ちゃんとマニフェストが適用されているかは `Serverspec` を使って確認したいので以下のように `serverspec` をインストールする。

~~~~
sudo gem install serverspec --no-ri --no-rdoc -V
~~~~

今回は `localhost` で `Serverspec` を動かす為、初期設定の `sercespec-init` では `localhost` を選択して準備して以下のような `spec` を書いた。

<script src="https://gist.github.com/inokappa/f6cbab790a26d0269908.js"></script>

`rspec` を走らせると...

![](images/2014050403.png)

グッジョブ！

そして、ブラウザで `Sensu Server` ホストの `8080` にアクセスしてみると...（認証情報はデフォルト）

![](images/2014050404.png)

おお、やっと拝めた。

ということで、`Puppet` を使って `Sensu Server` を構築してみたが、構築してみて幾つか困った（困るであろう）点があったので以下に纏めた。

***
***

<H2>困った点</H2>

<H3>何が困ったのか？</H3>

 * 素の状態から `Puppet` を `yum` でインストールしようとすると `Ruby 1.8.7` がインストールされてしまう...（原因は以下の「Ruby 1.8.7 に依存したパッケージ」に書いた）
 * ちょっと回避させて `Puppet` パッケージの導入は成功するものの結局は上手く動かない
 * `sensu` 自体は組み込みの `Ruby` で動くので問題ない
 * でも、`Puppet` が動いている環境で最新の `Ruby` を動かすことが容易に出来ない...ということになる
 * `Amazon Linux` は起動直後は既に `ruby 2.0.0p451` がインストールされている状態で結果として色々と面倒だったので `CentOS` の `AMI` を利用した

***

<H3>インストールされる Ruby 関連のパッケージ</H3>

`CentOS` にて `Puppet` をパッケージからインストールした場合に以下のような `Ruby` 関連のパッケージが一緒にインストールされる。

![](images/2014050401.png)

`Chef` のように `Chef` に同梱されている組み込みの `Ruby` というのが存在しないのは若干、残念。

***

<H3>Ruby 1.8.7 に依存したパッケージ</H3>

以下のリンクによると...

 * [[Puppet Users] Puppet fails to run if ruby1.8 is not installed.](http://grokbase.com/t/gg/puppet-users/143yjsa5qq/puppet-fails-to-run-if-ruby1-8-is-not-installed#2014040536beqvqz7gbloqcpwumhp2swrm)

以下のパッケージが `Ruby 1.8.7` に依存したライブラリでコンパイルされているとのこと。

 * ruby-augeas
 * ruby-shadow

ガビーンである。対策としては上記のパッケージを環境に応じた `Ruby` でコンパイルし直せとのこと...ちょっとハードルが上がる。

***
***

<H2>最後に</H2>

 * `Puppet` を使ってチョーシンプルな `Sensu Server` を構築してみた
 * 当初は `Amazon Linux` で頑張っていたけど挫折して `CentOS` の `AMI` を利用した
 * 以前にも `Docker` コンテナを使って同じことをやってみたがその時に比べるとちょっとスマートに出来た
 * 但し、`SSL` の設定は行っていない
 * そして、一番の懸念点が `Puppet` が古い `Ruby` に依存しているところ...こちらの回避策らしいプロジェクトがあるようなのでそちらを試してみたいかな...

***
***
