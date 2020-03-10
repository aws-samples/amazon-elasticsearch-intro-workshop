# Lab 4: Amazon ES の応用的な使い方

ハンズオンの冒頭で述べたように，Amazon ES のユースケースは大きくわけて，ログ分析と全文検索の 2 つがあります．すでにログ分析については Lab 2 で触れました．この Lab 4 では，全文検索を中心に，いくつかの応用的な使い方についてみていきたいと思います

## Section 1: Amazon ES を用いた日本語全文検索

TBW

## Section 2: カスタム日本語辞書と同義語辞書を用いた，より高度な日本語全文検索

TBW

## Section 3: SQL を用いたログ分析

ここまで _search API を直接叩く形の検索クエリの書き方についてみてきました．しかし Elasticsearch のクエリは，JSON の入れ子で記述をする必要があり，書くのに手間がかかってしまいます．こうした問題をカバーするための方法がいくつかあります．例えば Python には，[Elasticsearch DSL](https://elasticsearch-dsl.readthedocs.io/en/latest/) という高レベルのクエリ記述ライブラリや，低レベルのクライアントである [Python Elasticsearch Client](https://elasticsearch-py.readthedocs.io/en/master/) があり，これらを用いることで比較的簡単にクエリを記述・実行することができます．

このセクションでは，上記方法ではない第三の道として，SQL によるクエリ記述を試してみましょう．Open Distro の SQL 機能を用いることで，慣れ親しんだ SQL を用いて Elasticsearch のクエリを発行することができます．

### \_opendistro/\_sql API によるクエリの実行

ここでも，先ほどと同様に Dev tools を用いて，API を実行していきましょう．

1. 画面左側の![kibana_devtools](../images/kibana_devtools.png)マークをクリックして，Dev tools のメニューを開きます

2. 下の **"Console"** に以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．これは，**"workshop-log-*"** に適合するすべての index に対して，status ごとのレコード数と平均気温を集計するというものです

   ```json
   POST _opendistro/_sql
   {
     "query": """
   select
     status
     , count(*) as cnt
     , avg(currentTemperature) as avgTemperature
   from
     workshop-log*
   group by
     status
   """
   }
   ```
   1. これを実行すると，以下のような結果が得られるでしょう（実際の集計結果は，実行環境ごとに異なった値となる点にご注意ください）．"aggregations" 以下に集計結果が表示されているのがみて取れるかと思います

      ```json
      {
        "took" : 12,
        "timed_out" : false,
        "_shards" : {
          "total" : 35,
          "successful" : 35,
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
                "doc_count" : 96930,
                "cnt" : {
                  "value" : 96930
                },
                "avgTemperature" : {
                  "value" : 79.99918497885072
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

4. 今度はクエリだけではなく，集計結果も SQL の実行結果のような形で表示してみましょう．以下のように `format=csv` というクエリパラメタをつけた形で同じクエリを実行します

   ```json
   POST _opendistro/_sql?format=csv
   {
     "query": """
   select
     status
     , count(*) as cnt
     , avg(currentTemperature) as avgTemperature
   from
     workshop-log*
   group by
     status
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
     status
     , count(*) as cnt
     , avg(currentTemperature) as avgTemperature
   from
     workshop-log*
   group by
     status
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

この Lab では，日本全文検索，カスタム辞書の適用，そして SQL API の使用という，Elasticsearch による検索の応用的な側面にフォーカスを当てました．以上で Elasticsearch の Workshop は全て完了です．[こちらの手順](../cleanup/README.md)に沿って，忘れずに後片付けを行なってください．リソースが残ったままだと，課金が発生し続けます．