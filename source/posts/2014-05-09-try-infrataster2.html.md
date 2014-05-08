---
title: Infrataster 補講
date: 2014-05-09 06:55 JST
tags: Infrataster, Serverspec, Ruby
---

<H2>はじめに</H2>

`Infrataster` の `mysql_query` リソースを試してみた。

***
***

<H2>参考</H2>

`mysql_query` リソースはプラグインとして提供されている。尚、引き続き、以下を参考にさせて頂く。

 * [ryotarai/infrataster](https://github.com/ryotarai/infrataster)
 * [ryotarai/infrataster-plugin-mysql](https://github.com/ryotarai/infrataster-plugin-mysql)

***
***

<H2>memo</H2>

以下、メモ。

***

<H3>環境</H3>

 * `CentOS release 6.5 on AWS EC2`
 * `ruby 2.1.1p76 (2014-02-24 revision 45161) [x86_64-linux]`
 * `mysql-server-5.1.73-3.el6_5.x86_64`

***

<H3>インストール</H3>

以下のように `Gemfile` を用意。

~~~
source 'https://rubygems.org'

gem 'infrataster'
gem 'infrataster-plugin-mysql'
~~~

`bundle install` して終わり。

***

<H3>spec_helper</H3>

`spec_helper.rb` は以下の通りに用意。

`infrataster-plugin-mysql` を `require` する必要があるようだ。

~~~~
require 'infrataster/rspec'
require 'infrataster-plugin-mysql'

Infrataster::Server.define(
  :db,
  'localhost',
  mysql: {user: 'root', password: 'your_password'},
)

RSpec.configure do |config|
  config.treat_symbols_as_metadata_keys_with_true_values = true
  config.run_all_when_everything_filtered = true
  config.filter_run :focus

  config.order = 'random'
end
~~~~

ポイントは `mysql: {user: 'root', password: 'your_password'},` かな。

***

<H3>sample をそのまま</H3>

`spec` ファイルはサンプルをそのまま利用させて頂く。

~~~~
describe server(:db) do
  describe mysql_query('SHOW STATUS') do
    it 'returns positive uptime' do
      row = results.find {|r| r['Variable_name'] == 'Uptime' }
      expect(row['Value'].to_i).to be > 0
    end
  end
end
~~~~

`spec` ファイルを作成後、以下のようにテストを実行する。

~~~~
bundle exec rspec
~~~~

以下のように結果が表示された。

![](images/2014050810.png)

***
***

<H2>引き続き...</H2>

クエリを変えたりして試していく。

***
***
