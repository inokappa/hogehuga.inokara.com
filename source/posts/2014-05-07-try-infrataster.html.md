---
title: Infrataster を試す
date: 2014-05-07 23:18 JST
tags: Infrataster, Serverspec, Ruby
---

<H2>はじめに</H2>

`Web` アプリケーションや `Web` サーバーをブラウザとかから見た状態をテストするツールがあったらいいよなーって思っていたら、`RSpec` の記法でテスト出来る `Infrataster` ツールが既に出回っていたので大慌てで試してみた。

 * [ryotarai/infrataster](https://github.com/ryotarai/infrataster)

また、良いタイミングでメンテナンス作業前後でプロセスの状態やサービスの稼働状態をどうやって確認しようかなって考えていたところだったので `Serverspec` と併用して実運用で取り入れられたら嬉しい。

***
***

<H2>参考</H2>

 * [ryotarai/infrataster](https://github.com/ryotarai/infrataster)
 * [Infrataster - Infra Behavior Testing Framework #oedo04](https://speakerdeck.com/ryotarai/infrataster-infra-behavior-testing-framework-number-oedo04)
 * [InfratasterでNginxのルーティングのテスト書いてる](http://apehuci-kitaitimakoto.sqale.jp/apehuci/?date=20140505)

***
***

<H2>試す</H2>

試した環境は下記の通り。

 * `MacOS X 10.9 Mavericks`
 * `Ruby 2.0.0p247 (2013-06-27 revision 41674) [x86_64-darwin13.0.0]`

<H3>準備</H3>

`Infrataster` を下記の通りインストールする。

~~~~
gem install infrataster --no-ri --no-rdoc -V
~~~~

インストールが終わったら適当にディレクトリをこさえて `rspec --init` を実行する。

~~~~
rspec --init
~~~~

***

<H3>とりあえず試す</H3>

`Example` に倣って試してみる。

 * [Example](https://github.com/ryotarai/infrataster#example)

まず、以下のように `spec_helper.rb` を書いた。

~~~~
require 'infrataster/rspec'

Infrataster::Server.define(
  :app,
  'kome.inokara.com'
)

RSpec.configure do |config|
  config.treat_symbols_as_metadata_keys_with_true_values = true
  config.run_all_when_everything_filtered = true
  config.filter_run :focus

  config.order = 'random'
end
~~~~

次に `spec_helper.rb` と同じディレクトリに以下のような `spec` ファイルを用意した。

~~~~
require 'spec_helper'

describe server(:app) do
  describe http('http://kome.inokara.com') do
    it "responds content including 'Middleman'" do
      expect(response.body).to include('Middleman')
    end
    it "responds as 'text/html'" do
      expect(response.headers['content-type']).to match(%r{^txt/html})
    end
    it "responds as '200'" do
      expect(response.status).to be 200
    end
  end
end
~~~~

あえて `expect(response.headers['content-type']).to match(%r{^txt/html})` をタイポしてテストを実行してみる。

~~~~
rspec
~~~~

以下のように結果が出力された。

![](images/2014050702.png)

以下の通りにタイポを直してあらためて... 

~~~~
require 'spec_helper'

describe server(:app) do
  describe http('http://kome.inokara.com') do
    it "responds content including 'Middleman'" do
      expect(response.body).to include('Middleman')
    end
    it "responds as 'text/html'" do
      expect(response.headers['content-type']).to match(%r{^text/html})
    end
    it "responds as '200'" do
      expect(response.status).to be 200
    end
  end
end
~~~~

`rspec` を実行すると...

![](images/2014050703.png)

おお、テストオッケー。

尚、上記のテストは `http://kome.inokara.com` に対して下記のテストを行っていることになる。

 * レスポンスボディ内に `Middleman` という文字列が含まれているか？
 * レスポンスヘッダの `content-type` は `text/html` が含まれているか？
 * レスポンスコード（ステータス）は `200` であるか？

なるほど、なるほど。

***
***

<H2>memo</H2>

`README` を読んだ上でのメモ。（自分の独断と偏見を含んでいるので実際にはちゃんと `README` を読みましょう）

***

<H3>spec_helper.rb</H3>

`spec_helper.rb` にテストの対象となるホストの情報等を記載する。

~~~~
Infrataster::Server.define(
  # spec ファイル内で利用するホストの名前
  :app,
  # ホストの IP アドレスを指定（FQDN でもイケた） 
  'xxx.xxx.xxx.xxx',
)
~~~~

また、`MySQL` のクエリの結果等もテスト出来るので `MySQL` の接続情報 `Infrataster::Server.define` 内に下記のように設定出来る。

~~~~
Infrataster::Server.define(
  #
  mysql: {user: 'app', password: 'app'},
)
~~~~

***

<H3>Resource</H3>

`http` resource では以下のオプションを利用出来る。

 * `method`
 * `params`
 * `header`

以下のようにして使うようだ。

~~~~
require 'spec_helper'

describe server(:app) do
  describe http(
    'http://kome.inokara.com',
    method: :post,
    params: {'kome' => 'tabetai'},
    headers: {'USER' => 'onigiri'}          
  ) do
    it "responds content including 'Gochisosama'" do
      expect(response.body).to include('Gochisosama')
    end
~~~~

***

その他、`capybara` や `mysql_query` という resource が使えるようなので追って試す。

***
***

<H2>とりあえず</H2>

`Infrataster` についてチョー簡単に触れてみたが、従来は `curl` とかを駆使してやっていたことが `Ruby` と `RSpec` に置き換わることで以下のようなメリットがあると考えている。

 * プロキシや `rewrite` 等の `Web` サーバーの振る舞いから、アプリケーションの細かい挙動についてテストが出来るようになる
 * `Serverspec` と併用することでインフラからアプリケーションのレイヤーまで類似したフレームワークてテスト出来る
 * インフラのテストがよりプログラマバルになるし、アプリケーションエンジニアと共通言語でコミュニケーション出来るようになる
 * `rewrite` ルールが増えてくると更新したら別のパスが使えなくなったりのデグレードが良くあるのでチェックとして使えそう

また、ドキュメントによるとデータベースのクエリの返り値もテスト出来るとのことなので、データベースを含めた `Web` アプリケーション全体の挙動についてテスト出来ると嬉しいなあと妄想している。

***
***
