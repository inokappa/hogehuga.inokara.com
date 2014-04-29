---
title: Elasticsearch の私的なメモアレコレ
date: 2014-04-24 20:10 JST
tags: Elasticsearch, Shell, Ruby
---

<H2>はじめに</H2>

 * `Elasticsearch` を運用し始めてそれなりに運用っぽいことをし始めたのでメモ
 * あくまで私的なメモなのでもっと良い方法等あれば教えてください！

***
***

<H2>メモ</H2>

基本的にはクエリとか...。

<H4>Elasticsearch のインデックスを削除する</H4>

~~~~
curl -XDELETE 'http://${ES_HOST}:9200/demo/tweet/1?pretty'
~~~~

***

<H4>Elasticsearch へのデータ登録リクエスト</H4>

~~~~
curl -XPUT 'http://${ES_HOST}:9200/demo/tweet/1' -d '{
  "user" : "hoge",
  "post_date" : "2014-04-04T00:01:00",
  "message" : "hogehoge"
}'
~~~~

***

<H4>Elasticsearch へのデータ取得リクエスト</H4>

~~~~
curl -XGET 'http://${ES_HOST}:9200/demo/tweet/1?pretty'
~~~~

***

<H4>Elasticsearch のインデックス内のドキュメントに対する検索</H4>

~~~~
curl -XGET 'http://${ES_HOST}:9200/demo/_search?q=hogehoge&pretty'
~~~~

***

<H4>全ての index 名を取得する（要：jq）</H4>

~~~~
curl -s http://${ES_HOST}:9200/_status | jq -r '.indices|keys[]'
~~~~

`-r` は必須では無いがあったらデータだけが取得出来るのが嬉しい。

***
***

<H2>インデックスのメンテナンススクリプト</H2>

インデックスをメンテナンススクリプトを作ってみた。

<H4>github</H4>

とりあえず以下にアップした。

 * [inokappa/elasticsearch-maintenace](https://github.com/inokappa/elasticsearch-maintenace)

***

<H4>jq でインデックス名の一覧を取得するまでの苦労</H4>

`http://${ES_HOST}:9200/_status` でインデックス名を含む情報を取得することが出来るが、インデックス名だけを取り出す場合に `jq` でどーやればいいのか悩んだが...

結局はインデックス名は `JOSN` 的にはキーとなるので `jq` の `keys` 関数を使うことで解決した。

~~~~
curl -s http://${ES_HOST}:9200/_status | jq -r '.indices|keys[]'
~~~~

ちなみに `length` 関数を使えばインデックスの数を取得することが出来る。ちなみに `Ruby` で書くと以下のように書けると思う。

~~~~ruby
#!/usr/bin/env ruby

require 'net/http'
require 'json'

es = URI.parse('http://localhost:9200/_status')
request = Net::HTTP::Get.new(es.path)
respons = Net::HTTP.start(es.host, es.port) {|http|
  http.request(request)
}
respons_json = JSON.parse(respons.body)
respons_json["indices"].each do |index_name,date|
  puts "#{index_name}"
end
~~~~

インデックスの数を取得したい場合には `res_json["indices"].size` とすれば取得出来る。ちなみに `elasticsearch-ruby` という `gem` もあるようだ。

 * [elasticsearch-ruby](https://github.com/elasticsearch/elasticsearch-ruby)

`curl` と `jq` を使えば一行で書けてしまうが `Ruby` の場合には一行では済まない。用途や周囲の環境に応じて使い分けたい。

***
***

<H2>纏まってないまとめ</H2>

 * `curl` があれば `Elasticsearch` と気軽に会話が出来ますな
 * `jq` があればよりコミュニケーション力が高まるかも
 * ちょっと凝ったことをしようとすれば `Ruby` を使いましょう（頑張って）

***
