# Lab 4: Amazon ES の応用的な使い方

ハンズオンの冒頭で述べたように，Amazon ES のユースケースは大きくわけて，ログ分析と全文検索の 2 つがあります．すでにログ分析については Lab 2 で触れました．この Lab 4 では，全文検索を中心に，いくつかの応用的な使い方についてみていきたいと思います．

## Section 1: Amazon ES を用いた全文検索

ここまで主にログ分析を中心に，Amazon ES の機能について触れてきました．しかし Elasticsearch という名前の通り，もともと Elasticsearch は全文検索を行うためのプロダクトとして開発されてきました．そこでこのセクションでは，Amazon ES を用いて全文検索を試してみたいと思います．

### 全文検索用の index 作成

まずは，全文検索を行うための index を新たに作成しましょう．

1. 画面左側の![kibana_devtools](../images/kibana_devtools.png)マークをクリックして，Dev tools のメニューを開きます

2. 下の **"Console"** に以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．ここでは，シンプルに "content" という 1 フィールドのみが存在する，mydocs という index を作成しました．Lab 2 では，データ挿入時に Amazon ES 側でフィールドのマッピングを自動認識する形で index を作成しましたが，ここでは明示的にテキストの解析を行うために，前もってマッピングを指定して index を作成しています

   ```json
   PUT mydocs
   {
     "mappings" : {
       "properties" : {
         "content" : {
           "type" : "text",
           "analyzer": "standard"
         }
       }
     }
   }
   ```

3. 続いて以下のコマンドを実行して，作成した index に対してドキュメントを 2 件追加します

   ```json
   POST mydocs/_bulk
   {"index":{"_index":"mydocs","_type":"_doc"}}
   {"content":"Amazon Redshift is a high speed enterprise grade data warehouse service."}
   {"index":{"_index":"mydocs","_type":"_doc"}}
   {"content":"Amazon Web Services offers various kind of analytics services."}
   ```

上の手順 2 で index を作成した際に，`"analyzer": "standard"` と設定しました．Elasticsearch では analyzer を指定することで，自動でテキストフィールドを解析して，後から検索しやすい形にします．Standard analyzer は Amazon ES のデフォルト analyzer で，検索に役立つさまざまな設定を行うことができます．詳しくは[こちら](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html)をご覧ください．

### 全文検索クエリの実行

それでは，上で作った index に対して実際に検索クエリを投げてみましょう．

1. Dev tools の Console に対して，以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．`_search` API を叩くことで，検索クエリを実行することができます．クエリパラメタとして，"content" フィールドが "講座" にマッチするものを取得するようなクエリ条件を指定しています

   ```json
   GET mydocs/_search?q=content:"redshift"
   ```

2. 想定通り，以下のような結果が得られたかと思います

   ```json
   {
     "took" : 4,
     "timed_out" : false,
     "_shards" : {
       "total" : 5,
       "successful" : 5,
       "skipped" : 0,
       "failed" : 0
     },
     "hits" : {
       "total" : {
         "value" : 1,
         "relation" : "eq"
       },
       "max_score" : 0.2876821,
       "hits" : [
         {
           "_index" : "mydocs",
           "_type" : "_doc",
           "_id" : "YKXvBXEBdQ_VtJWAeY85",
           "_score" : 0.2876821,
           "_source" : {
             "content" : "Amazon Redshift is a high speed enterprise grade data warehouse service."
           }
         }
       ]
     }
   }
   
   ```

3. ここで "Redshift" ではなく，"redshift" で検索しても正しくドキュメントにヒットしたことに気づいたでしょうか．この背景を確認するために，以下のクエリを実行して，どのように文章が解析されたのかを確認してみましょう

   ```json
   GET mydocs/_analyze
   {
     "analyzer": "standard", 
     "text": "Amazon Redshift is a high speed enterprise grade data warehouse service."
   }
   ```

6. すると以下のように， 全ての単語が小文字に変換されていることに気づくでしょう．これは standard analyzer の中に [Lower Case Token Filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lowercase-tokenfilter.html) と呼ばれるフィルターが含まれており，ここで全ての単語を小文字に変換しているのです．ここで使用した standard analyzer 以外にも，さまざまな built-in の analyzer がありますので，詳しく知りたい方は[こちら](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html)を参照してください．

   ```json
   {
     "tokens" : [
       {
         "token" : "amazon",
         "start_offset" : 0,
         "end_offset" : 6,
         "type" : "<ALPHANUM>",
         "position" : 0
       },
       {
         "token" : "redshift",
         "start_offset" : 7,
         "end_offset" : 15,
         "type" : "<ALPHANUM>",
         "position" : 1
       },
       {
         "token" : "is",
         "start_offset" : 16,
         "end_offset" : 18,
         "type" : "<ALPHANUM>",
         "position" : 2
       },
       {
         "token" : "a",
         "start_offset" : 19,
         "end_offset" : 20,
         "type" : "<ALPHANUM>",
         "position" : 3
       },
       {
         "token" : "high",
         "start_offset" : 21,
         "end_offset" : 25,
         "type" : "<ALPHANUM>",
         "position" : 4
       },
       {
         "token" : "speed",
         "start_offset" : 26,
         "end_offset" : 31,
         "type" : "<ALPHANUM>",
         "position" : 5
       },
       {
         "token" : "enterprise",
         "start_offset" : 32,
         "end_offset" : 42,
         "type" : "<ALPHANUM>",
         "position" : 6
       },
       {
         "token" : "grade",
         "start_offset" : 43,
         "end_offset" : 48,
         "type" : "<ALPHANUM>",
         "position" : 7
       },
       {
         "token" : "data",
         "start_offset" : 49,
         "end_offset" : 53,
         "type" : "<ALPHANUM>",
         "position" : 8
       },
       {
         "token" : "warehouse",
         "start_offset" : 54,
         "end_offset" : 63,
         "type" : "<ALPHANUM>",
         "position" : 9
       },
       {
         "token" : "service",
         "start_offset" : 64,
         "end_offset" : 71,
         "type" : "<ALPHANUM>",
         "position" : 10
       }
     ]
   }
   ```

### Synonym の設定

続いて類義語（Synonym）の設定を行いたいと思います．ユーザーが検索を行う際に，実際に本文に含まれている単語にピッタリ一致するキーワードで検索してくれるとは限りません．似たような意味ではあるが，異なる表現のキーワードを使う場合があるでしょう．類義語を設定しておくことで，そのようなときでもきちんと検索結果を返せるようになります．

1. Dev tools の Console に対して，以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．先ほど作成した index を削除してしまいます

   ```json
   DELETE mydocs
   ```

2. 続いて新しい index を作成します．今度は，末尾に "my_synonym" という新しい類義語の設定を加えています．ここでは "amazon web services"，"aws"，"cloud" の 3 つを，同じ単語と見なすようにしています．"redshift"，"rs"，"dwh" も同様です

   ```json
   PUT mydocs
   {
     "mappings" : {
       "properties" : {
         "content" : {
           "type" : "text",
           "analyzer": "my_analyzer"
         }
       }
     },
     "settings": {
       "index": {
         "analysis": {
           "analyzer": {
             "my_analyzer": {
               "type": "custom",
               "tokenizer": "standard",
               "filter": [
                 "my_synonym",
                 "lowercase",
                 "stop"
               ]
             }
           },
           "filter": {
             "my_synonym" : {
               "type": "synonym",
               "synonyms": [
                 "amazon web services,aws,cloud",
                 "redshift,rs,dwh"
               ]
             }
             
           }
         }
       }
     }
   }
   ```
   
3. 先ほどと同様にデータを追加します

   ```json
   POST mydocs/_bulk
   {"index":{"_index":"mydocs","_type":"_doc"}}
   {"content":"Amazon Redshift is a high speed enterprise grade data warehouse service."}
   {"index":{"_index":"mydocs","_type":"_doc"}}
   {"content":"Amazon Web Services offers various kind of analytics services."}
   ```

4. 同様に検索クエリを実行してください．ただし今回は，文章には含まれていない単語で検索を行います

   ```json
   GET mydocs/_search?q=content:"aws"
   ```

5. 文章に単語が踏まれていないにも関わらず，問題なく検索結果が取得できていることを確認できるでしょう！

   ```json
   {
     "took" : 4,
     "timed_out" : false,
     "_shards" : {
       "total" : 5,
       "successful" : 5,
       "skipped" : 0,
       "failed" : 0
     },
     "hits" : {
       "total" : {
         "value" : 1,
         "relation" : "eq"
       },
       "max_score" : 0.8630463,
       "hits" : [
         {
           "_index" : "mydocs",
           "_type" : "_doc",
           "_id" : "maUaBnEBdQ_VtJWA-48R",
           "_score" : 0.8630463,
           "_source" : {
             "content" : "Amazon Web Services offers various kind of analytics services."
           }
         }
       ]
     }
   }
   ```

6. 最後に，元のドキュメントがどのように解析されているかを確認するために，以下のコマンドを実行します

   ```json
   GET mydocs/_analyze
   {
     "analyzer": "my_analyzer", 
     "text": "Amazon Web Services offers various kind of analytics services."
   }
   ```

7. 今度は先ほどと違い，"is"，"of" といった単語が含まれていないのがみて取れるでしょう．実はこれ，先ほど index を再作成する際に "stop" という "filter" を追加したためです．この [Stop Tken Filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-stop-tokenfilter.html) は，検索の役に立ちづらい助詞や前置詞を，解析する際にあらかじめ除外しておくというものです．これにより，"of" という単語で検索しても，ドキュメントがヒットしないようになります

   ```json
   {
     "tokens" : [
       {
         "token" : "amazon",
         "start_offset" : 0,
         "end_offset" : 6,
         "type" : "<ALPHANUM>",
         "position" : 0
       },
       {
         "token" : "web",
         "start_offset" : 7,
         "end_offset" : 10,
         "type" : "<ALPHANUM>",
         "position" : 1
       },
       {
         "token" : "services",
         "start_offset" : 11,
         "end_offset" : 19,
         "type" : "<ALPHANUM>",
         "position" : 2
       },
       {
         "token" : "offers",
         "start_offset" : 20,
         "end_offset" : 26,
         "type" : "<ALPHANUM>",
         "position" : 3
       },
       {
         "token" : "various",
         "start_offset" : 27,
         "end_offset" : 34,
         "type" : "<ALPHANUM>",
         "position" : 4
       },
       {
         "token" : "kind",
         "start_offset" : 35,
         "end_offset" : 39,
         "type" : "<ALPHANUM>",
         "position" : 5
       },
       {
         "token" : "analytics",
         "start_offset" : 43,
         "end_offset" : 52,
         "type" : "<ALPHANUM>",
         "position" : 7
       },
       {
         "token" : "services",
         "start_offset" : 53,
         "end_offset" : 61,
         "type" : "<ALPHANUM>",
         "position" : 8
       }
     ]
   }
   ```

このセクションでは，全文検索における同義語やフィルタの利用についてみてきました．しかし本セクションで振られたのはごく一部で， Amazon ES ではより細かくさまざまな全文検索の設定を行うことができます．[Elasticsearch のドキュメント](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-kuromoji-tokenizer.html) に詳細が書かれていますので，ぜひご一読ください．

## Section 2: SQL を用いたログ分析

ここまで _search API を直接叩く形の検索クエリの書き方についてみてきました．しかし Elasticsearch のクエリは，JSON の入れ子で記述をする必要があり，書くのに手間がかかってしまいます．こうした問題をカバーするための方法がいくつかあります．例えば Python には，[Elasticsearch DSL](https://elasticsearch-dsl.readthedocs.io/en/latest/) という高レベルのクエリ記述ライブラリや，低レベルのクライアントである [Python Elasticsearch Client](https://elasticsearch-py.readthedocs.io/en/master/) があり，これらを用いることで比較的簡単にクエリを記述・実行することができます．

このセクションでは，上記方法ではない第三の道として，SQL によるクエリ記述を試してみましょう．Open Distro の SQL 機能を用いることで，慣れ親しんだ SQL を用いて Elasticsearch のクエリを発行することができます．

### \_opendistro/\_sql API によるクエリの実行

ここでも，先ほどと同様に Dev tools を用いて，API を実行していきましょう．

1. 画面左側の![kibana_devtools](../images/kibana_devtools.png)マークをクリックして，Dev tools のメニューを開きます

2. 下の **"Console"** に以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．これは，**"workshop-log-*"** に適合するすべての index に対して，status ごとのレコード数と平均気温を集計するというものです．Lab 2 の Section 2 で解説したように，グルーピング処理を行う場合は keyword 型のフィールドを使用する必要がありますので，ここでは status.keyword を用いています

   ```json
   POST _opendistro/_sql
   {
     "query": """
   select
     status.keyword
     , count(*) as cnt
     , avg(currentTemperature) as avgTemperature
   from
     workshop-log*
   group by
     status.keyword
   """
   }
   ```
3. これを実行すると，以下のような結果が得られるでしょう（実際の集計結果は，実行環境ごとに異なった値となる点にご注意ください）．"aggregations" 以下に集計結果が表示されているのがみて取れるかと思います

   ```json
   {
     "schema": [
       {
         "name": "status.keyword",
         "type": "double"
       },
       {
         "name": "cnt",
         "alias": "cnt",
         "type": "double"
       },
       {
         "name": "avgTemperature",
         "alias": "avgTemperature",
         "type": "double"
       }
     ],
     "total": 3,
     "datarows": [
       [
         "OK",
         86425,
         79.99312698871854
       ],
       [
         "WARN",
         7673,
         79.42864590121204
       ],
       [
         "FAIL",
         1940,
         80.93762886597938
       ]
     ],
     "size": 3,
     "status": 200
   }
   ```
   
4. 今度はクエリだけではなく，集計結果も SQL の実行結果のような形で表示してみましょう．以下のように `format=csv` というクエリパラメタをつけた形で同じクエリを実行します

   ```json
   POST _opendistro/_sql?format=csv
   {
     "query": """
   select
     status.keyword
     , count(*) as cnt
     , avg(currentTemperature) as avgTemperature
   from
     workshop-log*
   group by
     status.keyword
   """
   }
   ```

5. 実行結果として，以下のようなみやすい csv の値が表示されるのが確認できるかと思います．フォーマットとしては，`csv` 以外に `jdbc`, `raw` といったフォーマットもサポートされています．詳しくは [Open Distro のドキュメント](https://opendistro.github.io/for-elasticsearch-docs/docs/sql/protocol/)を参照してください

   ```csv
   status.keyword,cnt,avgTemperature
   OK,96930.0,79.99918497885072
   WARN,8595.0,79.84409540430482
   FAIL,2162.0,80.45189639222941
   ```

### _sql API のクエリを Elasticsearch のクエリに変換する

SQL でクエリをかけることがわかりましたが，ではこの内容を実際に Elasticsearch のクエリに直すとどのような形になるのでしょうか．Open Distro には SQL で書かれたクエリと等しい Elasticsearch クエリを返す `_opendistro/_sql/_explain` API を備えています．

1. 以下の SQL を実行してください

   ```json
   POST _opendistro/_sql/_explain
   {
     "query": """
   select
     status.keyword
     , count(*) as cnt
     , avg(currentTemperature) as avgTemperature
   from
     workshop-log*
   group by
     status.keyword
   """
   }
   ```

2. すると，次のような結果が返されます．この結果自体が，`_search` API に投げるクエリそのものです．

   ```json
   {
     "from" : 0,
     "size" : 0,
     "_source" : {
       "includes" : [
         "status.keyword",
         "COUNT",
         "AVG"
       ],
       "excludes" : [ ]
     },
     "stored_fields" : "status.keyword",
     "aggregations" : {
       "status#keyword" : {
         "terms" : {
           "field" : "status.keyword",
           "size" : 200,
           "min_doc_count" : 1,
           "shard_min_doc_count" : 0,
           "show_term_doc_count_error" : false,
           "order" : [
             {
               "_count" : "desc"
             },
             {
               "_key" : "asc"
             }
           ]
         },
         "aggregations" : {
           "cnt" : {
             "value_count" : {
               "field" : "_index"
             }
           },
           "avgTemperature" : {
             "avg" : {
               "field" : "currentTemperature"
             }
           }
         }
       }
     }
   }
   ```

3. そこで，上記の結果を貼り付けて，以下のように `_search` API を叩いてください

   ```json
   POST _search
   {
     "from" : 0,
     "size" : 0,
     "_source" : {
       "includes" : [
         "status",
         "COUNT",
         "AVG"
       ],
       "excludes" : [ ]
     },
     "stored_fields" : "status",
     "aggregations" : {
       "status.keyword" : {
         "terms" : {
           "field" : "status.keyword",
           "size" : 200,
           "min_doc_count" : 1,
           "shard_min_doc_count" : 0,
           "show_term_doc_count_error" : false,
           "order" : [
             {
               "_count" : "desc"
             },
             {
               "_key" : "asc"
             }
           ]
         },
         "aggregations" : {
           "cnt" : {
             "value_count" : {
               "field" : "_index"
             }
           },
           "avgTemperature" : {
             "avg" : {
               "field" : "currentTemperature"
             }
           }
         }
       }
     }
   }
   ```

4. すると，`_opendistro/_sql` の結果と同じものが得られます．なお通常の `_search` API は，`csv` や `raw` のような結果表示には対応していません

   ```json
   {
     "took" : 109,
     "timed_out" : false,
     "_shards" : {
       "total" : 134,
       "successful" : 134,
       "skipped" : 0,
       "failed" : 0
     },
     "hits" : {
       "total" : {
         "value" : 10000,
         "relation" : "gte"
       },
       "max_score" : null,
       "hits" : [ ]
     },
     "aggregations" : {
       "status.keyword" : {
         "doc_count_error_upper_bound" : 0,
         "sum_other_doc_count" : 0,
         "buckets" : [
           {
             "key" : "OK",
             "doc_count" : 96932,
             "cnt" : {
               "value" : 96932
             },
             "avgTemperature" : {
               "value" : 79.99918499566706
             }
           },
           {
             "key" : "WARN",
             "doc_count" : 8595,
             "cnt" : {
               "value" : 8595
             },
             "avgTemperature" : {
               "value" : 79.84409540430482
             }
           },
           {
             "key" : "FAIL",
             "doc_count" : 2162,
             "cnt" : {
               "value" : 2162
             },
             "avgTemperature" : {
               "value" : 80.45189639222941
             }
           }
         ]
       }
     }
   }
   ```

以上で `_sql` APIによるクエリの実行はおしまいです．注意点として，この API はすべての標準 SQL コマンドに対応しているわけではありません．例えば集計関数としては，現状`avg()`, `count()`, `max()`, `min()`, `sum()` にのみ対応しています．詳細については [Open Distro のドキュメント](https://opendistro.github.io/for-elasticsearch-docs/docs/sql/operations/)を確認してください．

## まとめ

この Lab では，全文検索とそのカスタマイズ，そして SQL API の使用という，Elasticsearch による検索の応用的な側面にフォーカスを当てました．以上で Elasticsearch の Workshop は全て完了です．[こちらの手順](../cleanup/README.md)に沿って，忘れずに後片付けを行なってください．リソースが残ったままだと，課金が発生し続けます．