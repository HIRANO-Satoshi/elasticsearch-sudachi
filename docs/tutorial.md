# Elasticsearch用Sudachiプラグイン チュートリアル

Elasticsearch プラグインは 5.6, 6.8 の最新バージョンと7系の各マイナーバージョンをサポートしています。

以下では Elasticsearch 7.5.0 で Sudachi をつかう手順をしめします。
 
まずプラグインをインストールします。 

```
$ cd elasticsearch-7.7.0
$ bin/elasticsearch-plugin install https://github.com/WorksApplications/elasticsearch-sudachi/releases/download/v7.7.0-2.0.2/analysis-sudachi-7.7.0-2.0.2.zip
```

パッケージには辞書が含まれていません。https://github.com/WorksApplications/SudachiDict から最新の辞書を取得し、 `$ES_HOME/sudachi` の下に置きます。 3つの辞書のうち以下では core 辞書を利用します。

```
$ wget https://object-storage.tyo2.conoha.io/v1/nc_2520839e1f9641b08211a5c85243124a/sudachi/sudachi-dictionary-20191030-core.zip
$ unzip sudachi-dictionary-20191030-core.zip
$ sudo mkdir /etc/elasticsearch/sudachi
$ sudo cp sudachi-dictionary-20191030/system_core.dic /etc/elasticsearch/sudachi
```

配置後、Elasticsearch を再起動します。

設定ファイルを作成します。 

```json:analysis_sudachi.json
{
    "settings" : {
        "analysis" : {
            "filter" : {
                "romaji_readingform" : {
                    "type" : "sudachi_readingform",
                    "use_romaji" : true
                },
                "katakana_readingform" : {
                    "type" : "sudachi_readingform",
                    "use_romaji" : false
                }
            },
            "analyzer" : {
                "sudachi_baseform_analyzer" : {
                    "filter" : [ "sudachi_baseform" ],
                    "type" : "custom",
                    "tokenizer" : "sudachi_tokenizer"
                },
                "sudachi_normalizedform_analyzer" : {
                    "filter" : [ "sudachi_normalizedform" ],
                    "type" : "custom",
                    "tokenizer" : "sudachi_tokenizer"
                },
                "sudachi_readingform_analyzer" : {
                    "filter" : [ "katakana_readingform" ],
                    "type" : "custom",
                    "tokenizer" : "sudachi_tokenizer"
                },
                "sudachi_romaji_analyzer" : {
                    "filter" : [ "romaji_readingform" ],
                    "type" : "custom",
                    "tokenizer" : "sudachi_tokenizer"
                },
                "sudachi_analyzer": {
                    "filter": [],
                    "tokenizer": "sudachi_tokenizer",
                    "type": "custom"
                }
            },
            "tokenizer" : {
                "sudachi_tokenizer": {
                    "type": "sudachi_tokenizer",
                    "mode": "search",
                    "resources_path": "/etc/elasticsearch/config/sudachi"
                }
            }
        }
    }
}
```

インデックスを作成します。 

```
$ curl -X PUT 'localhost:9200/test_sudachi' -H 'Content-Type: application/json' -d @analysis_sudachi.json
{"acknowledged":true,"shards_acknowledged":true,"index":"test_sudachi"}
```

解析してみます。 

```
$ curl -X GET "localhost:9200/test_sudachi/_analyze?pretty" -H 'Content-Type: application/json' -d'{"analyzer":"sudachi_analyzer", "text" : "関西国際空港"}'
{
  "tokens" : [
    {
      "token" : "関西国際空港",
      "start_offset" : 0,
      "end_offset" : 6,
      "type" : "word",
      "position" : 0,
      "positionLength" : 3
    },
    {
      "token" : "関西",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "国際",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "word",
      "position" : 1
    },
    {
      "token" : "空港",
      "start_offset" : 4,
      "end_offset" : 6,
      "type" : "word",
      "position" : 2
    }
  ]
}
```

`search mode` が指定されているでA単位とC単位の両方が出力されます。

動詞、形容詞を終止形で出力してみます。 

```
$ curl -X GET "localhost:9200/test_sudachi/_analyze?pretty" -H 'Content-Type: application/json' -d'{"analyzer":"sudachi_baseform_analyzer", "text" : "おおきく"}'
{
  "tokens" : [
    {
      "token" : "おおきい",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

表記を正規化して出力してみます。 

```
$ curl -X GET "localhost:9200/test_sudachi/_analyze?pretty" -H 'Content-Type: application/json' -d'{"analyzer":"sudachi_normalizedform_analyzer", "text" : "おおきく"}'
{
  "tokens" : [
    {
      "token" : "大きい",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

読みを出力してみます。 

```
$ curl -X GET "localhost:9200/test_sudachi/_analyze?pretty" -H 'Content-Type: application/json' -d'{"analyzer":"sudachi_readingform_analyzer", "text" : "おおきく"}'
{
  "tokens" : [
    {
      "token" : "オオキク",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

読みをローマ字 (Microsoft IME 風) で出力してみます。

```
$ curl -X GET "localhost:9200/test_sudachi/_analyze?pretty" -H 'Content-Type: application/json' -d'{"analyzer":"sudachi_romaji_analyzer", "text" : "おおきく"}'
{
  "tokens" : [
    {
      "token" : "ookiku",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

そのほか、品詞によるトークンの除外、ストップワードなどが利用できます。

こちらもご参照ください: [Elasticsearchのための新しい形態素解析器 「Sudachi」 - Qiita](https://qiita.com/sorami/items/99604ef105f13d2d472b) （Elastic stack Advent Calendar 2017）
