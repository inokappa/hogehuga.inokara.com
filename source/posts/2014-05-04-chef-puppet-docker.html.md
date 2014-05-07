---
title: Chef とか Puppet から Docker をイジれるようなので試してみる
date: 2014-05-04 17:31 JST
tags: Chef, Puppet, Docker, AWS
---

<H2>はじめに</H2>

みんな大好き、私も大好き、`Docker` のドキュメントを見ていたら（読んでたわけではない）`Chef` や `Puppet` から直接 `Docker` をイジれるようなので試してみる。

***
***

<H2>参考</H2>

`Docker` のドキュメントをそのまま参考にする。

 * [Examples Using Chef](http://docs.docker.io/use/chef/) 
 * [Examples Using Puppet](http://docs.docker.io/use/puppet/) 

以下は各ソースコード。

 * [bflad/chef-docker](https://github.com/bflad/chef-docker)
 * [garethr/garethr-docker](https://github.com/garethr/garethr-docker)

既に `chef-docker` については下記にて紹介されている。

 * [chef-docker cookbook を利用してdockerをインストールする。](http://qiita.com/futoase/items/3bf941985891f7d69a06)

また、`Chef` の `Berkshelf` の使い方については下記が参考になった。

 * [Berkshelf入門](http://qiita.com/DQNEO/items/9adcbd69ecaa62fbef41)

以前に自分でも `Berkshelf` について書いたが、書いた当時よりも `Berkshelf` のバージョンが上がっていたりして書式等も変わっており殆ど役に立たなかったので改めて `Berkshelf` の書き方については自分なりに纏めてみたいと思う。

***
***

<H2>Chef de Docker</H2>

<H3>試した環境</H3>

 * `EC2` インスタンス
 * `t1.micro`
 * `CentOS`

***

<H3>準備（1）</H3>

`Chef` が使える環境を用意する。

 * [EC2 インスタンスに Puppet で Sensu Server をセットアップする](http://hogehuga.inokara.com/2014/05/03/sensu-puppet.html)

上の記事で利用した環境をそのまま利用するが `Chef` そのものに関しては下記のように `Omnibus` インストーラーでインストールした。

~~~~
curl -L https://www.opscode.com/chef/install.sh | bash
~~~~

また、`knife-solo` や `berkshelf` については以下のようにインストールした。

~~~~
/opt/chef/embedded/bin/gem install knife-solo --no-ri --no-rdoc -V
/opt/chef/embedded/bin/gem install berkshelf --no-ri --no-rdoc -V
~~~~

`berkshelf` のインストール途中にコンパイルのプロセスが `kill` されてしまう現象が発生した。（`t1.micro` で十分なメモリと `swap`  領域が不足していた為かと思われる）

***

<H3>準備（2）</H3>

ひと通り役者が揃ったところで `Chef` の場合はもうひと手間。`Chef` のリポジトリを作成する。

~~~~
knife solo init chef-repo-test
~~~~

次に `Berkshelf` を利用して `docker` の `Cookbook` を取得する。

~~~~
cd chef-repo-test
rm -rf cookbooks
vim Berksfile
~~~~

`Berksfile` は以下のように記述する。

~~~~
source "https://api.berkshelf.com/"

cookbook 'docker'
~~~~

記述したら以下を実行する。今回は `Chef` に同梱された `Ruby` を利用している為、`/opt/chef/embedded/bin/berks` となっているが環境に応じて読み替えること。

~~~~
/opt/chef/embedded/bin/berks vendor cookbooks
~~~~

実行後、`cookbooks` 以下には `docker` に関係する `cookbook` がインストールされている。

![](images/2014050408.png)

***

<H3>あともうひと手間</H3>

上記の状態で `docker` パッケージのインストールを行うことは可能だが、別途 `cookbook` を作ってみたいので以下のようにして `cookbook` を作成する。

~~~~
cd /root/chef-repo-test/site-cookbooks
/opt/chef/embedded/bin/berks cookbook docker-test .
~~~~

簡単なレシピを以下のように書く。

<script src="https://gist.github.com/inokappa/f7ad701ee27c25bb806a.js"></script>

今回は `chef-solo` を使って `cookbook` を適用するので `solo.rb` を以下のように用意。

~~~~
file_cache_path "/tmp/chef-solo"
cookbook_path ["/root/chef-repo-test/cookbooks", "/root/chef-repo-test/site-cookbooks"]
role_path "/root/chef-repo-test/roles"
log_level :info
~~~~

そして適用先は `localhost` ということで `nodes/localhost.json` は以下のように用意。

~~~~
{
  "run_list": [
    "recipe[docker]",
    "recipe[docker-test]"
  ]
}
~~~~

***

<H3>プロビジョニング、そろそろいきましょか</H3>

諸々の準備が整ったところで `chef-solo` を実行して `cookbook` を適用する。

~~~~
cd /root/chef-repo-test
chef-solo -c solo.rb -j nodes/localhost.json
~~~~

![](images/2014050409.png)

おお、やってますな。一発目はなぜかコケてしまったが改めて `chef-solo` を実行すると...以下のようなログを出力して正常に終了した。

 * [Chef から Docker コンテナを操作した時のログ](https://gist.github.com/inokappa/4af007ad6eb9a901bcc4)

ちなみに `docker ps -a` とか `docker images` すると...

![](images/2014050410.png)

ファンタスティック！

***

<H3>ということで...</H3>

`Chef` の実行環境を用意するのに若干手間取って（`Berkshelf` の書き方とか使い方とかで）しまったが、簡単にレシピから `Dokcer` を操作することが出来た（出来そう）。また、`Cookbook` の `README` を見ると `docker pull` や `docker run` 以外のことも色々と出来そうなので試してみたい。

尚、`Chef` そのものではないが、以下の点には注意する。

 * `Berkshelf` のインストール（意外に負荷が高い）
 * `Berkshelf` の使い方（`install` ではなくて `vendor` を使う）

***
***

<H2>Puppet de Docker</H2>

<H3>試した環境</H3>

 * `EC2` インスタンス
 * `t1.micro`
 * `CentOS`

***

<H3>準備</H3>

`Puppet` が使える環境を用意する。

 * [EC2 インスタンスに Puppet で Sensu Server をセットアップする](http://hogehuga.inokara.com/2014/05/03/sensu-puppet.html)

上の記事で利用した環境をそのまま利用する。また、合わせてモジュールもインストールする。

~~~~
puppet module install garethr/docker
~~~~

インストールされるモジュールは下記のモジュール。

 * [garethr/garethr-docker](https://github.com/garethr/garethr-docker)

***

<H3>マニフェスト</H3>

以下のようなマニフェストを書いて `/etc/puppet/manifests/site.pp` に保存する。

<script src="https://gist.github.com/inokappa/bd67ac42a0267f5f715d.js"></script>

チョー簡単。

~~~~
include 'docker'
~~~~

で `Docker` のパッケージをインストールする。

~~~~
docker::image { $IMAGES :
  ensure => 'present',
}
~~~~

上記で `docker pull $IMAGE(S)` が実行される。

~~~~
docker::run { $CONTAINER :
  image   => 'centos',
  command => 'echo "a"',
}
~~~~

上記で `docker run -d centos echo "a"` が実行される。

***

<H3>プロビジョニング</H3>

`puppet apply` でマニフェストを適用してみる。尚、上記のマニフェスト一発目だとパッケージのインストールだけで終了してしまうので注意。もう一度、マニフェストを適用するか、パッケージの終了後に `image` と `run` を実行すると `Warning` 等が出ない。

~~~~
cd /etc/puppet
puppet apply manifests/site.pp
~~~~

まずはパッケージのインストール。

![](images/2014050405.png)

次に `docker pull` と `docker run` を一緒に...。

![](images/2014050406.png)

ついでに `docker ps -a` や `docker images` で確認してみる。

![](images/2014050407.png)

***

<H3>ということで...</H3>

マニフェストの書き方を勉強する必要があるが、`Chef` と同様に `Puppet` から簡単に `Docker` コンテナ操作が出来そう。

***
***

<H2>さいごに</H2>

駆け足で `Chef` や `Puppet` のレシピやマニフェストを使って `Docker` の操作を体験した。どちらもコミュニティが提供している `Cookbook` や `Module` を利用することで簡単に操作は出来そうだ。最低限の利用（`pull` して `run`）程度であればどっちも同じだと思われるのでどっちを使うかはお好みで。

但し、`chef-docker` の方が `image` や `container` の操作等が充実している印象を受けた。

 * [Getting Started](https://github.com/bflad/chef-docker#getting-started)

また、上記にてプライベートリポジトリから `pull` して `run` して変更分を `commit` してという流れが書かれていて実際の利用のシチュエーションをイメージ出来ると思われる。

***
***

