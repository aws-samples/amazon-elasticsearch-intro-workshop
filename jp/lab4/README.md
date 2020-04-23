# Lab 4: Amazon ES の応用的な使い方

ハンズオンの冒頭で述べたように，Amazon ES のユースケースは大きくわけて，ログ分析と全文検索の 2 つがあります．すでにログ分析については Lab 2 で触れました．この Lab 4 では，全文検索を中心に，いくつかの応用的な使い方についてみていきたいと思います．

## Section 1: Amazon ES を用いた日本語全文検索

ここまで主にログ分析を中心に，Amazon ES の機能について触れてきました．しかし Elasticsearch という名前の通り，もともと Elasticsearch は全文検索を行うためのプロダクトとして開発されてきました．そこでこのセクションでは，Amazon ES を用いて日本語の全文検索を試してみたいと思います．

### 日本語全文検索用の index 作成

まずは，日本語検索を行うための index を新たに作成しましょう．

1. 画面左側の![kibana_devtools](../images/kibana_devtools.png)マークをクリックして，Dev tools のメニューを開きます

2. 下の **"Console"** に以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．ここでは，シンプルに "content" という 1 フィールドのみが存在する，jpdocs という index を作成しました．Lab 2 では，データ挿入時に Amazon ES 側でフィールドのマッピングを自動認識する形で index を作成しましたが，ここでは明示的に日本語テキストの解析を行うために，前もってマッピングを指定して index を作成しています

   ```json
   PUT jpdocs
   {
     "mappings" : {
       "properties" : {
         "content" : {
           "type" : "text",
           "analyzer": "kuromoji"
         }
       }
     }
   }
   ```

3. 続いて以下のコマンドを実行して，作成した index に対してドキュメントを 2 件追加します

   ```
   POST jpdocs/_bulk
   {"index":{"_index":"jpdocs","_type":"_doc"}}
   {"content":"近所の地銀に口座を持っている"}
   {"index":{"_index":"jpdocs","_type":"_doc"}}
   {"content":"築地銀だこにはしょっちゅう行く"}
   ```

上の手順 2 で index を作成した際に，`"analyzer": "kuromoji"` と設定しました．Elasticsearch では analyzer を指定することで，自動でテキストフィールドを解析して，後から検索しやすい形にします．[Kuromoji](https://github.com/atilika/kuromoji) は Java で書かれたオープンソースの日本語形態素解析エンジンで，Amazon ES で日本語テキスト解析を行う際に使われる analyzer です．kuromoji analyzer には複数の解析処理が含まれています．一覧は[こちら](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-kuromoji-analyzer.html)をご覧ください．

### 日本語検索クエリの実行

それでは，上で作った index に対して実際に検索クエリを投げてみましょう．

1. Dev tools の Console に対して，以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．`_search` API を叩くことで，検索クエリを実行することができます．クエリパラメタとして，"content" フィールドが "講座" にマッチするものを取得するようなクエリ条件を指定しています

   ```json
   GET jpdocs/_search?q=content:"口座"
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
       "max_score" : 0.5753642,
       "hits" : [
         {
           "_index" : "jpdocs",
           "_type" : "_doc",
           "_id" : "c6W25nABdQ_VtJWA5ID1",
           "_score" : 0.5753642,
           "_source" : {
             "content" : "近所の地銀に口座を持っている"
           }
         }
       ]
     }
   }
   ```

3. では次に，Dev tools の Console に対して，以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください

   ```json
   GET jpdocs/_search?q=content:"地銀"
   ```

4. すると，今度は想定と違い，ドキュメントが 2 つとも検索結果として返ってきてしまいました

   ```json
   {
     "took" : 8,
     "timed_out" : false,
     "_shards" : {
       "total" : 5,
       "successful" : 5,
       "skipped" : 0,
       "failed" : 0
     },
     "hits" : {
       "total" : {
         "value" : 2,
         "relation" : "eq"
       },
       "max_score" : 0.5753642,
       "hits" : [
         {
           "_index" : "jpdocs",
           "_type" : "_doc",
           "_id" : "c6W25nABdQ_VtJWA5ID1",
           "_score" : 0.5753642,
           "_source" : {
             "content" : "近所の地銀に口座を持っている"
           }
         },
         {
           "_index" : "jpdocs",
           "_type" : "_doc",
           "_id" : "dKW25nABdQ_VtJWA5ID1",
           "_score" : 0.5753642,
           "_source" : {
             "content" : "築地銀だこにはしょっちゅう行く"
           }
         }
       ]
     }
   }
   ```

5. この理由を確かめるために，格納したドキュメントがどのように解析されたのかを確認してみます．Dev tools  で以下の内容を実行してください．`_analyze` API を叩くことで，指定した analyzer で text を解析した結果を表示させることができます

   ```json
   GET jpdocs/_analyze
   {
     "analyzer": "kuromoji", 
     "text": "築地銀だこにはしょっちゅう行く"
   }
   ```

6. すると，"築地銀だこ" が正しく認識されずに，"築"，"地銀"，"だこ" の 3 つに分割されていたことがわかります．そのため "地銀" で検索された際に，このドキュメントも検索結果に含まれてしまったということです．なお，解析結果に "には" が含まれていないのに気がついた方がいるかもしれません．これは kuromoji analyzer の中に ja_stop token filter というフィルタが含まれているためです．このフィルタは，"で"，"や" といった一般的に検索の役に立たない助詞等を，あらかじめドキュメントを登録する際に取り除いておいてくれます 

   ```json
   {
     "tokens" : [
       {
         "token" : "築",
         "start_offset" : 0,
         "end_offset" : 1,
         "type" : "word",
         "position" : 0
       },
       {
         "token" : "地銀",
         "start_offset" : 1,
         "end_offset" : 3,
         "type" : "word",
         "position" : 1
       },
       {
         "token" : "だこ",
         "start_offset" : 3,
         "end_offset" : 5,
         "type" : "word",
         "position" : 2
       },
       {
         "token" : "しょっちゅう",
         "start_offset" : 7,
         "end_offset" : 13,
         "type" : "word",
         "position" : 5
       },
       {
         "token" : "行く",
         "start_offset" : 13,
         "end_offset" : 15,
         "type" : "word",
         "position" : 6
       }
     ]
   }
   ```

ここまでみてきたように，kuromoji analyzer を指定することで，Amazon ES で日本語全文検索を行うことができるようになります．ただし kuromoji は常に適切な解析を行ってくれるわけではありません．次のセクションでは，適切な解析を行い，良い検索結果を返すためのカスタマイズについてみていきます．

## Section 2: カスタム日本語辞書と類義語の適用

このセクションでは，カスタム日本語辞書と，類義語（Synonym）を設定することで，より適切な検索結果を返せるようにしていきます．

### カスタム日本語辞書の適用

先ほどのように "築地銀だこ" が "築"，"地銀"，"だこ" の 3 つに分割しまっていたのは，"築地銀だこ" が固有名詞であるため，kuromoji がカバーできる一般的な日本語表現ではなかったためです．このような単語をあらかじめ登録しておくことで，analyzer が適切に形態素解析を行ってくれるようになります．

1. Dev tools の Console に対して，以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．先ほど作成した index を削除してしまいます

   ```json
   DELETE jpdocs
   ```

2. 続いて新しい index を作成します．今度は，あらかじめ定義された kuromoji analyzer ではなく，これをカスタマイズしたものを使用します．先ほど "kuromoji" と書かれていた analyzer の値が，今度は "my_analyzer" となっています．この "my_analyzer" の設定が，その下の "settings" 以下に書かれているものです．その中に，`"user_dictionary_rules": ["築地銀だこ,築地 銀だこ,ツキジ ギンダコ,カスタム名詞"]` と書かれているのが，新しく追加されたユーザー辞書です．ここでは "築地銀だこ" が "築地" と "銀だこ" の 2 単語に分割されるように指定しました

   ```json
   PUT jpdocs
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
               "tokenizer": "my_kuromoji_tokenizer",
               "filter": [ "ja_stop" ]
             }
           },
           "tokenizer": {
             "my_kuromoji_tokenizer": {
               "type": "kuromoji_tokenizer",
               "mode": "search",
               "user_dictionary_rules": [
                 "築地銀だこ,築地 銀だこ,ツキジ ギンダコ,カスタム名詞"
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
   POST jpdocs/_bulk
   {"index":{"_index":"jpdocs","_type":"_doc"}}
   {"content":"近所の地銀に口座を持っている"}
   {"index":{"_index":"jpdocs","_type":"_doc"}}
   {"content":"築地銀だこにはしょっちゅう行く"}
   ```

4. 同様に検索クエリを実行してください

   ```json
   GET jpdocs/_search?q=content:"地銀"
   ```

5. 今度は想定通り，検索結果に ""築地銀だこにはしょっちゅう行く" が含まれず，1 件だけ返ってくるでしょう

   ```json
   {
     "took" : 2,
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
       "max_score" : 0.6931472,
       "hits" : [
         {
           "_index" : "jpdocs",
           "_type" : "_doc",
           "_id" : "h6XN5nABdQ_VtJWAXYA2",
           "_score" : 0.6931472,
           "_source" : {
             "content" : "近所の地銀に口座を持っている"
           }
         }
       ]
     }
   }
   ```

6. 最後に，実際にどのように解析が行われたのか，以下のコマンドを実行して確認してみます

   ```json
   GET jpdocs/_analyze
   {
     "analyzer": "my_analyzer", 
     "text": "築地銀だこにはしょっちゅう行く"
   }
   ```

7. 先ほどと異なり，想定通り "築地"，"銀だこ" と分割されているのが確認できます．これで "地銀" にはもう引っかからなくなりました

   ```json
   {
     "tokens" : [
       {
         "token" : "築地",
         "start_offset" : 0,
         "end_offset" : 2,
         "type" : "word",
         "position" : 0
       },
       {
         "token" : "銀だこ",
         "start_offset" : 2,
         "end_offset" : 5,
         "type" : "word",
         "position" : 1
       },
       {
         "token" : "しょっちゅう",
         "start_offset" : 7,
         "end_offset" : 13,
         "type" : "word",
         "position" : 4
       },
       {
         "token" : "行く",
         "start_offset" : 13,
         "end_offset" : 15,
         "type" : "word",
         "position" : 5
       }
     ]
   }
   ```

上記手順の 2 で，"user_dictionary_rules" に指定した形式について補足しておきます．単一文字列の中に，カンマ区切りで以下のような順序で必要な項目を記述します．"形態素解析" および "読み" が複数の単語に分かれる場合には，半角スペースを入れてください．より細かい説明については，[Elasticsearch のドキュメント](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-kuromoji-tokenizer.html)を参照してください．

| 単語       | 形態素解析結果 | 読み            | 品詞         |
| ---------- | -------------- | --------------- | ------------ |
| 築地銀だこ | 築地 銀だこ    | ツキジ ギンダコ | カスタム名詞 |

### 類義語の適用

セクションの最後に，類義語の設定を行いたいと思います．ユーザーが検索を行う際に，実際に本文に含まれている単語にピッタリ一致するキーワードで検索してくれるとは限りません．似たような意味ではあるが，異なる表現のキーワードを使う場合があるでしょう．類義語を設定しておくことで，そのようなときでもきちんと検索結果を返せるようになります．

1. Dev tools の Console に対して，以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．先ほど作成した index を削除してしまいます

   ```json
   DELETE jpdocs
   ```

2. 続いて新しい index を作成します．今度は，末尾に "my_synonym" という新しい類義語の設定を加えています．ここでは "地銀"，"都市銀"，"銀行" の 3 つを，同じ単語と見なすようにしています．"築地銀だこ" と "たこ焼き" も同様です．

   ```json
   PUT jpdocs
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
               "tokenizer": "my_kuromoji_tokenizer",
               "filter": [
                 "my_synonym",
                 "ja_stop"
               ]
             }
           },
           "tokenizer": {
             "my_kuromoji_tokenizer": {
               "type": "kuromoji_tokenizer",
               "mode": "search",
               "user_dictionary_rules": [
                 "築地銀だこ,築地 銀だこ,ツキジ ギンダコ,カスタム名詞"
               ]
             }
           },
           "filter": {
             "my_synonym" : {
               "type": "synonym",
               "synonyms": [
                 "地銀,都市銀,銀行",
                 "築地銀だこ,たこ焼き"
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
   POST jpdocs/_bulk
   {"index":{"_index":"jpdocs","_type":"_doc"}}
   {"content":"近所の地銀に口座を持っている"}
   {"index":{"_index":"jpdocs","_type":"_doc"}}
   {"content":"築地銀だこにはしょっちゅう行く"}
   ```

4. 同様に検索クエリを実行してください．ただし今回は，文章には含まれていない単語で検索を行います

   ```json
   GET jpdocs/_search?q=content:"たこ焼き"
   ```

5. 問題なく検索結果が取得できていることを確認できるでしょう

   ```json
   {
     "took" : 8,
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
       "max_score" : 1.2574185,
       "hits" : [
         {
           "_index" : "jpdocs",
           "_type" : "_doc",
           "_id" : "yqX05nABdQ_VtJWAeoAt",
           "_score" : 1.2574185,
           "_source" : {
             "content" : "築地銀だこにはしょっちゅう行く"
           }
         }
       ]
     }
   }
   ```

6. 最後に，類義語がどのように適用されているかを確認するために，以下のコマンドを実行します

   ```json
   GET jpdocs/_analyze
   {
     "analyzer": "my_analyzer", 
     "text": "築地銀だこにはしょっちゅう行く"
   }
   ```

7. 以下のように，解析結果に，元のドキュメントには含まれていない "たこ焼き" が追加されているのが確認できます．これにより，先ほどのクエリがこのドキュメントにヒットしたわけです

   ```json
   {
     "tokens" : [
       {
         "token" : "築地",
         "start_offset" : 0,
         "end_offset" : 2,
         "type" : "word",
         "position" : 0
       },
       {
         "token" : "たこ焼き",
         "start_offset" : 0,
         "end_offset" : 5,
         "type" : "SYNONYM",
         "position" : 0,
         "positionLength" : 2
       },
       {
         "token" : "銀だこ",
         "start_offset" : 2,
         "end_offset" : 5,
         "type" : "word",
         "position" : 1
       },
       {
         "token" : "しょっちゅう",
         "start_offset" : 7,
         "end_offset" : 13,
         "type" : "word",
         "position" : 4
       },
       {
         "token" : "行く",
         "start_offset" : 13,
         "end_offset" : 15,
         "type" : "word",
         "position" : 5
       }
     ]
   }
   
   ```

このセクションでは，日本語全文検索におけるカスタム辞書や同義語の利用についてみてきました．しかし本セクションで振られたのはごく一部で， Amazon ES ではより細かくさまざまな全文検索の設定を行うことができます．[Elasticsearch のドキュメント](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-kuromoji-tokenizer.html) に詳細が書かれていますので，ぜひご一読ください．

## Section 3: カスタム日本語辞書と類義語をファイルで適用

前のセクションでは，index のマッピングの中に直接カスタム日本語辞書や類義語の指定を行いました．辞書にや類義語に登録する単語の数が少なければ，この形でも問題ありません．しかし単語数が膨大になった場合には，これをマッピング上で管理するのが困難になってきてしまいます．そこでこのセクションでは，カスタム日本語辞書や類義語をファイルに切り出して，それを Amazon ES ドメイン側で読み込む形に変更してみます．

### パッケージを作成して Amazon ES ドメインにアタッチ

カスタム日本語辞書や類義語のファイルを Amazon ES に適用するためには，ファイルを一旦 S3 にアップロードしてから，それを Amazon ES 側にパッケージとして登録します．登録済みパッケージを Amazon ES ドメインにアタッチすることで，パッケージが利用可能となります．それでは順番に実行していきましょう．

1. まず，S3 にアップロードするための[辞書ファイル](./custom_dictionary.csv)と[類義語ファイル](./custom_synonym.csv)をローカルにダウンロードしておいてください
2. 次に AWS マネジメントコンソール上で，画面右上のヘッダー部のリージョン選択にて， **[東京]** となっていることを確認します．もし **[東京]** となっていない場合は，リージョン名をクリックして，**[東京]** に変更してください．続いて AWS マネジメントコンソールの画面左上にある [サービス] から **[S3]** のページを開いてください
3. バケット一覧から，Lab 1 で作成した **"workshop-YYYYMMDD-YOURNAME"** を選択します（YYYYMMDD と YOURNAME は Lab 1 で指定したものに置き換えて考えてください）．続いて画面上部の **[+ フォルダの作成]** ボタンを押して **"package"** と入力し，**[保存]** を押します
4. 作成された **"package"** フォルダをクリックしたら，**[↑ アップロード]** ボタンを押して，先ほどダウンロードした 2 つのファイルを追加してから，右下の **[アップロード]** ボタンを押してください
5. 続いて  AWS マネジメントコンソールの画面左上にある [サービス] から **[Elasticsearch Service]** のページを開いてください
6. 左側メニューの "パッケージ" を選択してから，画面左上の **[インポート]** ボタンを押します．まずはカスタム辞書から登録していきましょう．"名前" に **"custom-dictinoary"**，"パッケージソース" に **"s3://workshop-YYYYMMDD-YOURNAME/package/custom_dictionary.csv"**と入力したら（YYYYMMDD と YOURNAME は Lab 1 で指定したものに置き換えてください），**[インポート]** ボタンを押します
7. 続いて，遷移した先の画面で **[ドメインへの間蓮付け]** を押してから，"workshop-esdomain" を選択して **[関連付け]** をクリックします．下の画面に戻ると，ドメインのステータスが "関連付け中" になっているでしょう．しばらく待つと "利用可能" に変わります．これを利用するためには，画面上側にある ID （F123456789 のような文字列です）が必要なため，これをいったんどこかにメモしておいてください
8. 同様に類義語ファイルも登録します．名前" に **"custom-synonym"**，"パッケージソース" に **"s3://workshop-YYYYMMDD-YOURNAME/package/custom_synonym.csv"**と入力して **[インポート]** ボタンを押します．その後，ドメインへの間蓮付けも行なってください

### パッケージを利用した日本語全文検索の実施

それでは，ドメインに関連付けしたパッケージを用いて，Section 2 と同じ結果が得られるかを確認していきたいと思います．

1. Dev tools の Console に対して，以下の内容をコピーしてから，右側の ▶︎ ボタンを押して，API を実行してください．先ほど作成した index を削除してしまいます

   ```json
   DELETE jpdocs
   ```

2. 続いて新しい index を作成します．"tokenizer" と "filter" で設定する内容が，Section 2 と一部変わっていることを確認してください．パッケージを利用するためには，それぞれ **"analyzers/F0123456789"** という形で，"analyzers/" のあとに先ほどメモしたパッケージ ID をつなげて入力する必要があります．**"F0123456789"** の部分は，それぞれカスタム辞書と類義語のパッケージ ID に置き換えてから，API を実行してください

   ```json
   PUT jpdocs
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
               "tokenizer": "my_kuromoji_tokenizer",
               "filter": [
                 "my_synonym",
                 "ja_stop"
               ]
             }
           },
           "tokenizer": {
             "my_kuromoji_tokenizer": {
               "type": "kuromoji_tokenizer",
               "mode": "search",
               "user_dictionary": "analyzers/F0123456789"
             }
           },
           "filter": {
             "my_synonym" : {
               "type": "synonym",
               "synonyms_path": "analyzers/F0123456789"
             }
             
           }
         }
       }
     }
   }
   ```

3. 先ほどと同様にデータを追加します

   ```json
   POST jpdocs/_bulk
   {"index":{"_index":"jpdocs","_type":"_doc"}}
   {"content":"近所の地銀に口座を持っている"}
   {"index":{"_index":"jpdocs","_type":"_doc"}}
   {"content":"築地銀だこにはしょっちゅう行く"}
   ```

4. まずは検索クエリを実行して，カスタム日本語辞書が適用されてるか確認しましょう

   ```json
   GET jpdocs/_search?q=content:"地銀"
   ```

5. 検索結果に ""築地銀だこにはしょっちゅう行く" が含まれず，1 件だけ返ってきたら，カスタム辞書が適用されているということがわかります

   ```json
   {
     "took" : 2,
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
       "max_score" : 0.6931472,
       "hits" : [
         {
           "_index" : "jpdocs",
           "_type" : "_doc",
           "_id" : "h6XN5nABdQ_VtJWAXYA2",
           "_score" : 0.6931472,
           "_source" : {
             "content" : "近所の地銀に口座を持っている"
           }
         }
       ]
     }
   }
   ```

6. 同様に次の検索クエリを実行して，類義語が適用されているかを確認します

   ```json
   GET jpdocs/_search?q=content:"たこ焼き"
   ```

7. もとの文章に含まれない，以下の検索結果が返ってくることから，ただしく類義語も適用されていることがわかります

   ```json
   {
     "took" : 8,
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
       "max_score" : 1.2574185,
       "hits" : [
         {
           "_index" : "jpdocs",
           "_type" : "_doc",
           "_id" : "yqX05nABdQ_VtJWAeoAt",
           "_score" : 1.2574185,
           "_source" : {
             "content" : "築地銀だこにはしょっちゅう行く"
           }
         }
       ]
     }
   }
   ```

ここまでみてきたように，カスタム辞書や類義語のファイルを切り出して，パッケージとして登録することで，Amazon ES ドメインと別で管理することができます．特に検索エンジンとして Amazon ES を利用する際に，非常に役に立つ機能といえるでしょう．こちらの機能について，より詳しく知りたい方は[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/elasticsearch-service/latest/developerguide/custom-packages.html)をご参照ください．

## Section 4: SQL を用いたログ分析

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

この Lab では，日本全文検索，カスタム辞書の適用，そして SQL API の使用という，Elasticsearch による検索の応用的な側面にフォーカスを当てました．以上で Elasticsearch の Workshop は全て完了です．[こちらの手順](../cleanup/README.md)に沿って，忘れずに後片付けを行なってください．リソースが残ったままだと，課金が発生し続けます．