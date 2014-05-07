---
title: Chef の私的なチートシート 2014 夏
date: 2014-05-05 08:09 JST
tags: Chef
---

<H2>はじめに</H2>

`Chef` を操作する各種コマンドについてうろ覚えだったりして毎回ググるのが面倒なので纏めてみる。

***
***

<H2>関係者の導入について</H2>

`Chef` そのものや必須（だと思う）ツール各種の導入について。

***

<H3>Chef</H3>

一番簡単なのは `Omunibus` インストーラーを利用する。

~~~~
curl -L https://www.opscode.com/chef/install.sh | bash
~~~~

というか、この方法以外ではやった記憶が無いけど、出来るとすれば、各ディストリビューションのパッケージインストールコマンドでインストールが出来そう。

ちなみに、この方法でインストールすると組み込みの `Ruby` が以下のパスにインストールされる。

~~~~
/opt/chef/embedded/bin/
~~~~

***

<H3>knife-solo</H3>

簡単。

~~~~
gem install knife-solo --no-ri --no-rdoc -V
~~~~

***

<H3>Berkshelf</H3>

簡単。

~~~~
gem install berkshelf --no-ri --no-rdoc -V
~~~~

だけど EC2 インスタンスの `t1.micro` だとコンパイル途中で止まることがあるので、インストールの際は環境のリソースについて注意すること。

***
***

<H2>chef-solo を実行出来る環境の準備</H2>

`chef-solo` を実行して `Cookbook` を適用する為に必要な最低限の設定ファイル各種。

***

<H3>solo.rb</H3>

`cookbooks` や `site-cookbooks` のパス等の各種パス、ログの出力レベルを設定する。

~~~~
file_cache_path "/path/to/chef-solo"
cookbook_path ["/path/to/cookbooks", "/path/to/site-cookbooks"]
role_path "/path/to/roles"
log_level :info
~~~~

詳細については下記を見ること。

 * [solo.rb](http://docs.opscode.com/config_rb_solo.html)

上記の例はホントに最低限。

***

<H3>run_list</H3>

`JSON` 形式で書く。配列の最後はカンマ（ケツカンマと言うらしい）不要なので注意。

~~~~
{
  "run_list": [
    "recipe[apache2]",
    "recipe[mysql]"
  ]
}
~~~~

`nodes/${HOSTNAME}.json` というファイル名で保存することが多い。

尚、`run_list` 以下が参考になる。

 * [Chef-client/Chef-soloのランリスト(run_list)書式大全](http://qiita.com/sawanoboly/items/e954bd3a4e156944bbb1)

***

<H3>chef-solo の実行</H3>

以下のように実行する。

~~~~
chef-solo -c solo.rb -j nodes/localhost.json
~~~~

尚、`chef-solo` については下記を読む。

 * [chef-solo](http://docs.opscode.com/chef_solo.html)

***
***

<H2>Berkshelf</H2>

`Berkshelf` の使い方。`Berkshelf` は `cookbook` の `gem` みたいなもの。

***

<H3>Berksfile の書き方</H3>

以下のように書く。

~~~~
source "https://api.berkshelf.com"

cookbook "mysql"
cookbook "nginx", "~> 2.6"
~~~~

詳細は下記を読むこと。

 * [BERKSHELF / GETTING STARTED](http://berkshelf.com/)

尚、`Berksfile` 内に `metadata` の記述がある場合にはカレントディレクトリに `metadata.rb` ファイルを探しに行こうとするので注意する。（もちろん、`metadata.rb` が存在しない場合には `berks vendor` コマンドは失敗する）

***

<H3>berks cookbook</H3>

`Cookbook` を作成する。

~~~~
berks cookbook ${COOKBOOK_NAME} ${COOKBOOK_PATH}
~~~~

上記を実行すると `${COOKBOOK_PATH}` に `Cookbook` が作成される。

***

<H3>berks vendor</H3>

`Berksfile` に書いた `Cookbook` をインストールする。

~~~~
berks vendor ${COOKBOOK_PATH}
~~~~

上記を実行すると `${COOKBOOK_PATH}` に `Berksfile` に記載された `Cookbook` がインストールされる。

***

<H3>他の Cookbook から利用する</H3>

他の `Cookbook` から利用する場合には二つのステップが必要。

 * レシピに `include_recipe '${COOKBOOK_NAME}'` を追加
 * `run_list` に `include_recipe` した `Cookbook` を追加する

以下のような感じ。

~~~~
include_recipe "docker"
#
docker_image 'centos'
#
docker_container 'centos' do
  command 'echo "a"'
  detach true
end
~~~~

前回の記事で利用したレシピ。以下は `run_list`。

~~~~
{
  "run_list": [
    "recipe[docker]",
    "recipe[docker-test]"
  ]
}
~~~~

***
***

<H2>その他</H2>

その他 `Chef` を利用する上で覚えておくと良いこと等。

***

<H3>chef-apply</H3>

`chef-solo` よりも限定的。引数にレシピファイルを指定することでレシピに書かれている内容を実行する。

~~~~
chef-apply default.rb
~~~~

また、レシピに書かれている内容（`DSL`）を引数として渡すことでその内容を実行してくれる。

~~~~
chef-apply -W -e 'package "httpd" do action :install end'
~~~~

上記のように `-e` オプションに続いてレシピの `DSL` を記載する。`-W` は `--why-run` オプション。実行すると以下のように結果が表示される。

![](images/2014050501.png)

おお。

尚、以下がとても参考になった。

 * [chef-solo,chef-shellより更にミニマムなchef-apply](http://qiita.com/sawanoboly/items/770fe24058fab5ab613d)

用途としては、ちょっとしたレシピに使えそう。上記の参考サイトでは `ohai` を実行したり直接 `Ruby` のコードを実行させたりしていて地味に便利そう。

***

<H3>Handlers</H3>

`Cookbook` を適用する前、適用した後に何らかの処理を加えることが出来るのが `Handlers` 。詳細については以下がとても詳しい。

 * [ChefのReport, Exeptionハンドラを使う](http://qiita.com/sawanoboly/items/29a488b0d3a781bd1d62)
 * [About Handlers](http://docs.opscode.com/essentials_handlers.html)

[@sawanoboly](https://twitter.com/sawanoboly) さんににマジ感謝。

`Handlers` には以下の三種類が存在する。

 * `exception`（例外が発生した際になんかやる）
 * `report`（`Cookbook` が適用が正常に終了した際になんかやる）
 * `start`（`chef-client` が走る前になんかやる）

各々はレシピから呼び出すか `client.rb` 又は `solo.rb` に記述して利用することが出来る。例えば、`solo.rb` に以下の記述しておいて...

~~~~
require 'chef/handler/json_file'
report_handlers << Chef::Handler::JsonFile.new(:path => "/var/chef/reports")
exception_handlers << Chef::Handler::JsonFile.new(:path => "/var/chef/reports")
~~~~

`chef-solo` を実行すると...

~~~~
chef-solo -c solo.rb -j nodes/localhost.json
~~~~

`/var/chef/reports/` 以下に `chef-run-report-YYYYMMDDhhmmss.json` という `JSON` ファイルが出力されている。

 * [Chef Handlers json_file Handler example](https://gist.github.com/inokappa/6c4b95367277abadd460)

上記は出力例のほんの一部。数行のレシピなのに大量の `JSON` ファイルが出力される。

尚、以下のようなサードパーティ製の `Handler` も存在する。

 * [Graphite](https://github.com/imeyer/chef-handler-graphite/wiki)
 * [HipChat](https://github.com/mojotech/hipchat/blob/master/lib/hipchat/chef.rb)
 * [SNS](http://onddo.github.io/chef-handler-sns/)
 * [ZooKeeper](http://onddo.github.io/chef-handler-zookeeper/)

ほうほう。上記以外にもいくつか存在する。`Graphite` あたりをちょっと試してみたい。

***

<H3>ohai で ノードの情報を取得する</H3>

前述の `chef-apply` + `Ohai` でノードの情報を簡単に取り出すには以下のように実行する。

~~~~
chef-apply -e "puts node[:hostname]"
~~~~

プラットフォームや `OS` の条件分岐でちょいと確認したい時に便利。以下は属性の一部。

 * `node['platform']` はプラットフォームを出力する
 * `node['platform_version']` はプラットフォームのバージョンを出力する

その他の属性情報については以下を参照。

 * [Automatic Attributes](http://docs.opscode.com/ohai.html#automatic-attributes)

***
***

<H2>さいごに</H2>

 * 自分で使いそうなネタを纏めてみた
 * `Handlers` は面白そうなので `Graphite` と絡めて試してみたい

***
***
