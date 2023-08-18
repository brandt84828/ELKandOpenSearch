# ElaticSearch

## 文件目錄結構
bin:script

config:elasticsearch.yml 集群配置文件

JDK JAVA運行環境

data: path.data數據文件

lib: java類庫

logs: path.log日誌文件

modules: ES模組

plugins: 已安裝插件

## JVM配置
1. Xmx & Xms設置相同
2. Xmx不要超過機器記憶體的50%
3. 不要超過30GB


## API
* Document CRUD
    * Index : 如果ID不存在則創建新的Document，否則先刪除現有的Document再創建新的(版本會增加)
    * Create : 如果ID已經存在會失敗
    * Update : Document必須已經存在，更新只會對相應字段做增量修改

* Bulk API
    * 支持在一次API調用中對不同索引進行操作
    * 支持四種類型操作
        * Index
        * Create
        * Update
        * Delete
    * 可以在URI中指定index，也可以在請求中的Payload中進行
    * 操作中單條操作失敗並不會影響其他操作
    * 返回結果包括了每一條操作執行的結果
* mget
    * 批量讀取
    * 可以減少網路連接所產生的開銷，提高性能
* msearch
    * 批量查詢
* Search API
    * URI Search
        * 在URL中使用查詢參數
    * Request Body Search
        * 使用Elasticsearch提供的，基於JSON格式更加完備的Query Domain Specific Language (DSL)
* 常見錯誤
    * 無法連接 : 網路故障或集群掛掉
    * 連接無法關閉 : 網路故障或節點出錯
    * 429 : 集群過於繁忙
    * 4xx : Request Body格式有錯
    * 500 : 集群內部錯誤

### URI Search
```
GET /movies/_search?q=2012&df=title&sort=year:desc&from=0&size=10&timeout=1s
{
    "profile":true
}
```
* q指定查詢語句，使用Query String Syntax
* df默認字段，不指定時會對所有字段進行查詢
* Sort排序 / from 和 size用於分頁
* Profile可以查看查詢是如何被執行的
* "Beautiful Mind"表示查詢Beautiful AND Mind且要求前後順序保持一致, 不加雙引號則為Beautiful OR Mind，(Beautiful Mind) 分組(預設是OR)
* 布林操作
    * AND / OR / NOT / && / || / !
        * 必須大寫
        * title:(matrix NOT reloaded)
* 分組
    * +表示must
    * -表示must_not
    * title:(+matrix -reloaded)
* 範圍查詢
    * 區間表示: []閉區間, {}開區間
        * year:{2019 TO 2018]
        * year:[* TO 2018]
* 算術符號
    * year:>2010
    * year:(>2010 && <=2018>)
    * year:(+>2010 +<2018>)

### Request Body Search
* 將查詢語句透過HTTP Request Body發送給Elasticsearch
* Query DSL
```
POST /movies,404_idx/_search?ignore_unavailable=true
{
    "profile": true,
    "query": {
        "match_all": {}
    }
}
```
* 分頁
    * From從0開始，默認返回10個結果
    * 獲取靠後分頁成本較高
```
POST /kibana_sample_data_ecommerce/_search
{
    "from": 10,
    "size": 20,
    "query": {
        "match_all": {}
    }
}
```
* 排序
    * 最好在數字型與日期型字段上排序
    * 因為對於多值類型或分析過的字段排序，系統會選一個值，無法得知該值
```
GET sample/_search
{
    "sort":[{"order_date":"desc"}],
    "from":10,
    "size":5,
    "query": {
        "match_all": {}
    }
}
```
* _source filtering
    * 如果_source沒有儲存，那就只返回匹配的Document metadata
    * _source支持使用通佩符，_source["name*","desc*"]
```
GET sample/_search
{
    "_source":["order_date","order_date","category.keyword"],
    "from":10,
    "size":5.
    "query": {
        "match_all": {}
    }
}
```
* 腳本字段
    * 舉例: 訂單中有不同的匯率，需要結合匯率對訂單價格進行排序
```
GET sample/_search
{
    "script_fields": {
        "new_field": {
            "script": {
                "lang": "painless",
                "source": "doc['order_date'].value+'hello'"
            }
        }
    },
    "from": 10,
    "size": 5,
    "query": {
        "match_all": {}
    }
}
```
* Match
```
GET /comments/_doc/_search
{
    "query":{
        "match": {
            "comment": {
                "query": "Last Chrismas",
                "operator": "AND"
            }
        }
    }
}
```

## 工具
Cerebro : ElasticSearch管理工具，可以透過Web頁面對ElasticSearch進行操作，支持index的建立、刪除和修改集群配置等。

## Document
* Elasticsearch是面向Document，Document是所有可搜尋數據的最小單位
    * 日誌文件中的日誌項
    * 一本電影的具體資訊 / 一張唱片的詳細資訊
    * MP3播放器裡的一首歌 / 一篇PDF中的具體文件

* Document會被序列化成JSON格式，保存在Elasticsearch中
* 每個Document都有一個Unique ID
    * 可以自己指定ID
    * 透過Elasticserach自動生成

### Document Metadata
Metadata : 用於標註Document的相關資訊

* _index: Document所屬的index name
* _type: Document所屬的類型名稱
* _id: Document Unique ID
* _source: Document原始Json數據
* _all: 整合所有字段內容到該字段(已廢除)
* _version: Document的版本資訊
* _score: 相關性評分

## Index
Index是Document的容器，是一個Document的結合，Indexing則是指保存一個Document到Elasticsearch的過程

* Index體現了邏輯空間的概念: 每個索引都有自己的Mapping定義，用於定義包含的Document的字段名稱和字段類型
* Shard體現了物理空間的概念: 索引中的數據分散在Shard上
* 索引的Mapping與Settings
    * Mapping定義Document字段的類型
    * Setting定義不同的數據分布

### 倒排索引
* 倒排索引的兩核心組成
    * 單詞詞典(Term Dictionary) : 記錄所有Document的單詞，紀錄單詞到倒排列表的關聯關係
        * 單詞詞典一般比較大，可以透過B+樹或Hash拉鍊法實現，以滿足高性能的插入與查詢
    * 倒排列表(Posting List) : 紀錄單詞對應的Document結合，由倒排索引向組成
        * 倒排索引項 (Posting)
            * Document ID
            * 詞頻 TF : 該單詞在Document中出現的次數，用於相關性評分
            * 位置(Position) : 單詞在Document中分詞的位置，用於語句搜尋(phrase query)
            * 偏移(Offset) : 記錄單詞的開始結束位置，實現高亮顯示
* Elasticsearch的倒排索引
    * Elasticsearch的JSON Document中的每個字段都有自己的倒排索引
    * 可以指定對某些字段不做索引
        * 優點 : 節省儲存空間
        * 缺點 : 字段無法被搜索

## 分布式系統
分布式系統的可用性與擴展性
* 高可用性
    * 服務可用性 : 允許有節點停止服務
    * 數據可用性 : 部分節點丟失，不會丟失數據
* 可擴展性
    * 請求量提升 / 數據的不斷增長 (將數據分布到所有節點上)

分布式特性
* Elasticsearch的分布式架構好處
    * 儲存的水平擴容
    * 提高系統的可用性，部分節點停止服務，整個集群的服務不受影響
* Elasicsearch的分布式架構
    * 不同的集群透過不同的名字來區分，可透過配置文件或者命令修改
    * 一個集群可以有一個或著多個節點

## 節點
* 節點是一個Elasticsearch的實例
    * 本質上就是一個Java進程
    * 一台機器上可以運行多個Elasticsearch進程，但是生產環境一班建議一台機器上只運行一個Elasticsearch實例
* 每一個節點都有名字，透過配置文件配置或啟動時指定
* 每一個節點在啟動之後會分配一個UID，保存在data目錄下

Master-eligible nodes 和 Master Node
* 每個節點啟動後，默認就是一個Master eligible Node
    * 可以設置node.master: false禁止
* Master-eligible節點可以參加選主流程，成為Master Node
* 當第一個節點啟動時，它會將自己選舉為Master Node
* 每個節點上都保存了集群的狀態，只有Master Node才能修改集群的狀態資訊
    * 集群狀態(Cluster State)，維護了一個集群中必要的資訊
        * 所有的節點資訊
        * 所有的索引和其相關的Mapping與Setting資訊
        * 分片的路由資訊
    * 任意節點都能修改資訊會導致數據的不一致性

* Data Node
    * 可以保存數據的節點叫做Data Node。負責保存分片數據，在數據擴展上起到了重要的作用
* Coordinating Node
    * 負責接受Client的請求，將請求分發到合適的節點，最終把結果匯集到一起
    * 每個節點默認都起到了Coordinating Node的職責
* Hot & Warm Node
    * 不同硬體配置的Data Node，用來實現Hot & Warm架構，降低集群部署的成本
* Machine Learning Node
    * 負責跑機器學習Job，用來做異常檢測
* Tribe Node
    * (5.3開始使用Cross Cluster Search) Tribe Node連接到不同的Elasticsearch集群，並且支持將這些集群當成一個單獨的集群處理

## 分片 (Primary Shard & Replica Shard)
* Primary Shard : 用以解決數據水平擴展的問題。透過主分片可以將數據分布到集群內的所有Node之上
    * 一個分片是一個運行的Lucene的實例
    * 主分片數在索引創建時指定，後續不允許修改，除非Reindex
* Replica : 用以解決數據高可用的問題。分片是主分片的拷貝
    * 副本分片數可以動態調整
    * 增加副本數可以在一定程度上提高服務的可用性(讀取的吞吐)

分片的設定
* 對於生產環境中分片的設定需要提前做好容量規劃
    * 分片數設置過小
        * 導致後續無法增加節點實現水平擴展
        * 單個分片的數據量太大，導致數據重分配耗時
    * 分片數設置過大，(7.0開始默認主分片設置成1，解決了over-sharding的問題)
        * 影響搜索結果的相關性評分，影響統計結果的準確性
        * 單個節點上過多的分片會導致資源浪費，同時也會影響性能

集群的健康狀況
* GET _cluster/health
* Green : 主分片與副本都正常分配
* Yellow : 主分片全部正常分配，有副本分片未能正常分配
* Red : 有主分片未能分配
    * 例如當Server Disk容量超過85%時，去創建一個新的索引

## Analysis 與 Analyzer
* Analysis : 文本分析是把全文本轉換成一系列單詞(term/token)的過程，也叫分詞
* Analysis是透過Analyzer來實現的
    * 可使用Elasticsearch內置的分析器或按需訂製化分析器
* 除了在數據寫入時轉換詞條，匹配Query語句時也需要用相同的分析器對查詢語句進行分析

### Analyzer
* 分詞器是專門處理分詞的組件，Analyzer由三部分組成
    * Character Filters : 針對原始文本處理，例如去除html
    * Tokenizer : 按照規則切分為單詞
    * Token Filter : 將切分的單詞進行加工，小寫、刪除stopwords、增加同義詞等
* 使用_analyzer API
    * 直接指定Analyzer進行測試
    * 指定索引的字段進行測試
    * 自定義分詞器進行測試
* Standard Analyzer
    * 默認分詞器
    * 按詞切分
    * 小寫處理
    * Stopword默認關閉
* Simple Analyzer
    * 按照非字母切分，非字母的都被去除
    * 小寫處理
* Whitespace Analyzer
    * 按照空格切分
* Stop Analyzer
    * 相比Simple Analyzer多了stop filter
* Keyword Analyzer
    * 不分詞，直接將輸入當成一個term輸出
* Pattern Analyzer
    * 透過正則表達式進行分詞
    * 默認是\W+，非字符的符號進行分詞
    * 小寫處理
* Language Analyzer
* ICU Analyzer
    * 需要安裝plugin
    * 提供了Unicode的支持，更好的支持亞洲語言

## 查詢相關度
衡量的兩種方式
* Precision
* Recall

## Mapping
* Mapping類似資料庫中的schema的定義，作用如下
    * 定義索引中的字段名稱
    * 定義字段的數據類型，例如字串 / 數字 / Boolean
    * 字段 / 倒排索引的相關配置 / (Analyzed or Not Analyzed, Analyzer)
* Mapping會把JSON Document映射成Lucene所需要的扁平格式
* 一個Mapping屬於一個索引的Type
    * 每個Document都屬於一個Type
    * 一個Type有一個Mapping定義
    * 7.0開始不需要在Mapping定義中指定type資訊

### 字段的數據類型
* 簡單類型
    * Text / Keyword
    * Date
    * Integer / Floating
    * Boolean
    * IPv4 & IPv6
* 複雜類型
    * 對象類型 / 嵌套類型
* 特殊類型
    * geo_point & geo_shape / percolator

### Dynamic Mapping
* 在寫入Document的時候，如果index不存在會自動創建
* Dynamic Mapping的機制使我們無需手動定義Mappings
* 但是有時候會推算的不對，例如地理位置資訊
* 當類型如果設置不對時，會導致一些功能無法正常運行，例如Range查詢

### 更改Mapping的字段類型
* 兩種情況
    * 新增加字段
        * Dynamic設為true時，一旦有新增字段的Document寫入，Mapping也同時被更新
        * Dynamic設為false，Mapping不會被更新，新增字段的數據無法被索引，但是資訊會出現在_source中
        * Dynamic設置成Strict，Document寫入失敗
    * 對已有字段，一旦已經有數據寫入，就不再支持修改字段定義
        * Lucene實現的倒排索引，一旦生成後，就不允許修改
    * 如果希望改變字段類型，必須Reindex API，重建索引
* 原因
    * 如果修改了字段的數據類型，會導致已被索引的屬於無法被搜尋
    * 但是如果是增加新的字段，就不會有這樣的影響

### 控制Dynamic Mappings
|     | true  | false | strict |
| ---- | ---- | ---- | ---- |
|  Document可以索引  | YES | YES | NO |
| 字段可以索引  | YES | NO | NO |
| Mapping被更新  | YES | NO | NO |
```
PUT movies
{
    "mappings": {
        "_doc":{
            "dynamic":"false"
        }
    }
}
```
* 當dynamic被設置成false的時候，存在新增字段的數據寫入，該數據可以被索引，但是新增字段被丟棄
* 當設置成strict模式的時候，數據寫入直接出錯

### 定義Mapping
自定義Mapping的一些建議
* 可以參考API手冊，純手寫
* 為了減少工作量及減低出錯機率，可以依照以下步驟
    * 創建一個臨時的index，寫入一些樣本數據
    * 透過訪問Mapping API獲得該臨時文件的Dynamic Mapping定義
    * 修改後使用，使用該配置創建索引
    * 刪除臨時索引

控制當前字段是否被索引
* Index : 控制當前字段是否被索引。默認為true，如果設置成false，該自斷不可被搜尋
```
PUT users
{
    "mappings": {
        "firstName": {
            "type": "text"
        },
        "lastName": {
            "type": "text",
            "index": false
        }
    }
}
```

Index Options
* 四種不同級別的Index Options配置，可以控制倒排索引記錄的內容
    * docs : 記錄doc id
    * freqs : 記錄doc id和term frequencies
    * positions : 記錄doc id / term frequencies / term position
    * offsets : 記錄doc id / term frequencies / term position / character offects
* Text類型默認記錄positions，其他默認為docs
* 記錄內容越多，占用儲存空間越大

null_value
```
GET users/_search?q=mobile:NULL

PUT users
{
    "mappings": {
        "properties": {
            "mobile": {
                "type": "keyword",
                "null_value": "NULL"
            }
        }
    }
}
```
* 需要對Null值實現搜尋
* 只有Keyword類型支持設定Null_Value

copy_to
```
GET users/_search?q=fullName:(F A)

PUT users
{
    "mappings": {
        "firstName": {
            "type": "text"
            "copy_to": "fullName"
        },
        "lastName": {
            "type": "text",
            "copy_to": "fullName"
        }
    }
}
```
* _all在7中被copy_to所替代
* 滿足一些特定的搜尋需求
* copy_to將字段的數值拷貝到目標字段，實現類似_all的作用
* copy_to的目標字段不出現在_source中

數組類型
* Elasticsearch中部提供專門的數組類型，但是任何字段都可以包含多個相同類型的數值
```
PUT users/_doc/1
{
    "name": "twobirds",
    "interests": ["reading", "music"]
}
```

## Reference
[極客時間](https://time.geekbang.org/course/detail/100030501-102655)