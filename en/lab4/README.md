# Lab 4: Amazon ES Advanced Usage

As mentioned at the beginning of this hands-on, there are two major use cases for Amazon ES: log analysis and full-text search. For the log analysis had already mentioned in Lab 2. In this Lab 4, let’s take a look for some of the more advanced usage, focusing on full-text search.

## Section 1: Full-Text Search Using Amazon ES

So far, you have been trying on Amazon ES features, mainly log analysis. However, as the name Elasticsearch indicates, Elasticsearch is originally developed as a product for full-text search. In this section, let’s try full-text search using Amazon ES.

### Create an index for full-text search

First, you will create a new index to perform full-text search.

1. Click ![kibana_devtools](../images/kibana_devtools.png) icon on the left of the screen to open the Dev tools menu.

2. Copy the following block of codes to the **"Console"** below, and click ▶ button on the right to execute the API. In this example, an index called mydocs which has only one field called "content" will be created. In Lab 2, the index that automatically recognizes field mappings on Amazon ES when inserting data has created, but here the index designated with mapping in advance to clearly analyze texts will be created.

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

3. Execute the following command to add two documents to the index you have created.

   ```json
   POST mydocs/_bulk
   {"index":{"_index":"mydocs","_type":"_doc"}}
   {"content":"Amazon Redshift is a high speed enterprise grade data warehouse service."}
   {"index":{"_index":"mydocs","_type":"_doc"}}
   {"content":"Amazon Web Services offers various kinds of analytics services."}
   ```

When creating the index in step 2 above, `"analyzer": "standard"` has been set. Elasticsearch automatically analyzes text fields by specifying analyzer to make them easier to search later. Standard analyzer is the default analyzer in Amazon ES and provides a variety of settings to help you search. For more information, see [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html).

### Executing a full-text search query

Now, you will actually execute a search query against the index created above.

1. Copy the following codes to the Dev Tools Console, and click ▶︎ button on the right to execute the API. You can execute the search query by calling `_search` API. As a query parameter, the “content” field is specified query conditions that matches such as “course”.

   ```json
   GET mydocs/_search?q=content:"redshift"
   ```

2. As expected, the following results may be obtained.

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

3. Did you notice that the document matched correctly when searching for “redshift” instead of “Redshift”? To confirm this background, execute the following query to see how the sentence is analyzed.

   ```json
   GET mydocs/_analyze
   {
     "analyzer": "standard",
     "text": "Amazon Redshift is a high speed enterprise grade data warehouse service."
   }
   ```

6. You will notice that all words are converted to lowercase as follows. This includes a filter called [Lower Case Token Filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lowercase-tokenfilter.html) into the standard analyzer, which convert here all words to lowercase. In addition to the standard analyzer used here, there are a variety of built-in analyzer. For more information, see [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html) for more information.

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

### Synonym Settings

Next, you will set up synonyms. When a user performs a search, it is not always possible to search for keywords that match the words in the text body. You might use keywords with similar meanings but different expressions. By setting synonyms, you will be able to receive search results properly in such cases.

1. Copy the following codes to the Dev Tools Console, and click ▶︎ button on the right to execute the API. The index you have created is deleted.

   ```json
   DELETE mydocs
   ```

2. Then, create a new index. Add a new synonym setting to the end of "my_synonym". The three words "amazon web services", "aws", and "cloud" are considered the same here. The same applies to "redshift", "rs", and "dwh".

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

3. Add the data as before

   ```json
   POST mydocs/_bulk
   {"index":{"_index":"mydocs","_type":"_doc"}}
   {"content":"Amazon Redshift is a high speed enterprise grade data warehouse service."}
   {"index":{"_index":"mydocs","_type":"_doc"}}
   {"content":"Amazon Web Services offers various kinds of analytics services."}
   ```

4. Execute the search query in the same way. This time, however, you will search for words that are not included in the sentence.

   ```json
   GET mydocs/_search?q=content:"aws"
   ```

5. Even though there are no words in the sentence, you will be able to make sure that the search results are obtained without any problems!

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
             "content" : "Amazon Web Services offers various kinds of analytics services."
           }
         }
       ]
     }
   }
   ```

6. At last, execute the following command to see how the original document is analyzed.

   ```json
   GET mydocs/_analyze
   {
     "analyzer": "my_analyzer",
     "text": "Amazon Web Services offers various kinds of analytics services."
   }
   ```

7. This time, unlike the previous one, you can see that the words "is" and "of" are not included. Actually, this is because "filter" called "stop" has added when recreating the index earlier. This [Stop Tken Filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-stop-tokenfilter.html) allows you to exclude particles and prepositions that are difficult to search before analyzing them. This prevents the document from being match when searching for the word "of".

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

In this section, you have tried the usage of synonyms and filters in full-text search. However, only a few part of full-text search features has been tried in this section. Other than the setting you have made in this section, Amazon ES allows you to configure a variety of full-text search settings. Please read the[Elasticsearch documentation](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-kuromoji-tokenizer.html) for more details.

## Section 2: Log Analysis Using SQL

So far, you have learned how to write a search query that directly executes _search API. However, Elasticsearch query needs to be described by nesting JSON, which is troublesome to write. There are several ways to solve this problem. For example, Python has a high-level query description library called[Elasticsearch DSL](https://elasticsearch-dsl.readthedocs.io/en/latest/) and a low-level client called [Python Elasticsearch Client](https://elasticsearch-py.readthedocs.io/en/master/), those allows you to write and execute queries relatively easily.

In this section, you will try to write a query using SQL as a tertiary method not mentioned before. Using the SQL function of Open Distro, you can issue Elasticsearch queries using familiar SQL.

### Executing queries with \_opendistro/\_sql API

Again, you will execute the API using Dev tools as before.

1. Click ![kibana_devtools](../images/kibana_devtools.png) icon on the left of the screen to open the Dev tools menu.

2. Copy the following block of codes to the **"Console"** below, and click ▶ button on the right to execute the API. This means that for all indexes that fit **"workshop-log-*"**, the number of records per status and the average temperature is aggregated. As described in Section 2 of Lab 2, when performing the grouping processing, the field of keyword type is required. Here we use status.keyword.

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
3. Executing the code above, you will get the result as follows (Note that the actual aggregate results will be different for each execution environment). You can see the aggregate results displayed in "aggregations".

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

4. Now let's display not only the query but also the aggregate result in the form of the SQL execution result. Execute the same query with the query parameter called `format=csv` as follows.

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

5. As a result of execution, you can confirm that the following simple csv value is displayed. In addition to `csv`, other formats such as `jdbc` and `raw` are also supported. See the [Open Distro document](https://opendistro.github.io/for-elasticsearch-docs/docs/sql/protocol/) for more information.

   ```csv
   status.keyword,cnt,avgTemperature
   OK,96930.0,79.99918497885072
   WARN,8595.0,79.84409540430482
   FAIL,2162.0,80.45189639222941
   ```

### Convert _sql API query to Elasticsearch query

Now, you know that you can execute queries in SQL. Then let's see what would be when this query is actually revised into an Elasticsearch query. Open Distro has `_opendistro/_sql/_explain` API that returns an Elasticsearch query to be equal to a query written in SQL.

1. Execute the following SQL.

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

2. The following result is returned. The result itself is the query to be sent to the `_search` API.

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

3. Paste the above result, and execute `_search` API as shown below.

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

4. You can now receive the result as same as `_opendistro/_sql`. Note that the normal `_search` API does not support to display the results such as `csv` and `raw`.

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

This has completed the query execution by `_sql` API. Note that this API does not support all standard SQL commands. For example, aggregate functions are currently supported only `avg ()`, `count ()`, `max ()`, `min ()`, and`sum ()`.Please see the [Open Distro document](https://opendistro.github.io/for-elasticsearch-docs/docs/sql/operations/) for more details.

## Summary

In this lab focused on the advanced aspects of Elasticsearch search: full-text search, customization, and SQL API usage. All of the Elasticsearch Workshop is now complete. Please do not forget clean up by following [these steps](../cleanup/README.md). If you keep the resource you used in this workshop, you will be continuously charged.