---
title: 第 4 回 Elasticsearch 勉強会に行ってもすぐに帰宅しなければいけませんでしたの巻
date: 2014-04-21 18:37 JST
tags: Elasticsearch, 勉強会
---

<H2>はじめに</H2>

 * Elasticsearch 勉強会に参加したが運用しているシステムに障害が発生してしまい途中で帰宅した
 * と言いながらステッカーはちゃっかり頂いた...有難うございます
 * ので冒頭の `@johtani` さんのお話の途中までの自分用メモ

***

<H2>参加された方のブログ等</H2>

資料も含めて気づいたら更新する。

 * [第4回Elasticsearch勉強会を開催しました。#elasticsearchjp](http://blog.johtani.info/blog/2014/04/21/hold-on-4th-elasticsearch-jp/)
 * [参加レポート:第4回elasticsearch勉強会 #elasticsearchjp](http://dev.classmethod.jp/server-side/4th-elasticsearchjp/)

<H2>アナライズ処理の仕組みとクエリ DSL</H2>

<H4>登壇者</H4>

 * 株式会社シーマーク 大谷　純さん @johtani 

<H4>資料</H4>

 * [アナライズ処理の仕組みとクエリDSL](http://blog.johtani.info/images/entries/20140421/Introduction_analysis_and_query_dsl_for_print.pdf)

<H4>書籍の紹介</H4>

 * `ElasticSearch Server` 日本語版
 * `0.9` 系
 * 売れ行きが良ければ次期バージョンにも対応

<H4>次回勉強会 とか ML</H4>

 * `ML` が活用されない...
 * 次回は `6` 月末位に勉強会を...

<H4>転置インデックスとは</H4>

 * 単語を元にドキュメント ID を付与する

<H4>アナライズ処理</H4>

 * 単語に区切る処理
 * フィールド毎に定義することが出来る（アナライズ処理が行われる）
 * ドキュメント =  レコード
 * `tokenizer` は一つのみ定義

<H4>Char Filter</H4>

 * 複数定義可能
 * 入力文字列を文字列単位で処理

<H4>Tokenizer</H4>

 * 入力文字列をトークン列に分割
 * 英語の場合にはスペース単位で分割していく
 * `keyword tokenizer`
 * `kuromoji_tokenizer`

<H4>TokenFilter</H4>

 * `Tokenizer` に分割された単語に対して処理を行う

<H4>感想</H4>

 * ふわっと `Elasticsearch` を使い始めた関係でこのような全文検索エンジンのバックエンド処理については殆ど知らなかったので良い機会だった
 * 途中退場大変申し訳ございません...m(_ _)m

***

<H2>elasticsearch-hadoopを使ってごにょごにょしてみる</H2>

<H4>登壇者</H4>

 * 株式会社マーズフラッグ R&D部　やまかつ さん　@yamakatu 

<H4>資料</H4>

 * [elasticsearch-hadoopを使ってごにょごにょしてみる](http://www.slideshare.net/yamakatu/elasticsearchhadoop)

***

<H2>CouchbaseとElasticsearchが手を結んだら</H2>

<H4>登壇者</H4>

 * 株式会社アットウェア 佐竹雅央さん @madgaoh 河村康爾さん @ijokarumawak 

***

<H2>未定</H2>

<H4>登壇者</H4>

 * Wantedly, Inc 内田誠悟さん @spesnova 

***

<H2>Elasticsearchのスクリプティング（仮）</H2>

<H4>登壇者</H4>

 * 株式会社富士通ソフトウェアテクノロジーズ 滝田聖己さん @pisatoshi 

<H4>資料</H4>

 * [ElasticsearchのScripting](https://speakerdeck.com/pisatoshi/elasticsearchdescripting)

***

<H2>Elasticsearch 向け多言語解析プラグイン</H2>

<H4>登壇者</H4>

 * ベイシス・テクノロジー株式会社 江口天さん 

***

<H2>感想</H2>

 * を書くつもりだったけど殆どお話を聞けてないので書けない
 * `ElasticSearch Server 日本語版` は電子書籍で購入させて頂いて拝読なう！
 * 次回は `0.9` 系で動いている `Elasticsearch` のバックアップやオプティマイズについて話を聞けたら嬉しいなあ
 * `ElasticSearch Server 日本語版` の感想も書いていければと考えている
 * 主催の @johtani さんやスタッフの皆さん、そして会場をご提供頂いているリクルートさんいつも有難うございます！

***

