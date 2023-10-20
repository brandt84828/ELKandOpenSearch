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

### 節點
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
* 互相ping對方，Node ID低的會成為被選舉的節點
* 其他節點會加入集群但是不承擔Master Node的角色。一旦發現被選中的主節點丟失，就會選舉出新的Master節點
* 每個節點上都保存了集群的狀態，只有Master Node才能修改集群的狀態資訊
    * 集群狀態(Cluster State)，維護了一個集群中必要的資訊
        * 所有的節點資訊
        * 所有的索引和其相關的Mapping與Setting資訊
        * 分片的路由資訊
    * 任意節點都能修改資訊會導致數據的不一致性
    * 處理創建、刪除索引等請求 / 決定分片被分配到哪個節點 / 負責索引的創建與刪除
    * 維護並且更新Cluster State
    * Master Node非常重要，在部屬上需要考慮解決單點的問題
    * 為一個集群設置多個Master節點 / 每個節點只承擔Master的單一角色
* Split-Brain問題，分布式系統的經典網路問題，當網路出現問題，一個節點和其他節點無法連接(假設3個Node，Node1網路斷)
    * Node2和Node3會重新選舉Master
    * Node1自己還是作為Master，組成一個集群，同時更新Cluster State
    * 導致2個Master維護不同的cluster state。當網路恢復時，無法選擇正確的恢復
* 如何避免Split-Brain
    * 限定一個選舉條件，設置quorum(仲裁)，只有在Master eligible節點數大於quorum時，才能進行選舉
        * Quorum = (master total node / 2) + 1
        * 當3個master eligible時，設置discovery.zen.minium_master_nodes為2，即可避免Split-Brain
    * 從7.0開始無須這個配置
        * 移除minimum_master_nodes參數，讓Elasticsearch自己選擇可以形成仲裁的節點
        * 典型的主節點選舉現在只需要很短的時間就可以完成。集群的伸縮變得更安全、更容易並且可能造成丟失數據的系統配置選項更少
        * 節點更清楚記錄他們的狀態，有助於診斷為什麼他們不能加入集群或為什麼無法選舉出主節點

* Data Node
    * 可以保存數據的節點叫做Data Node。負責保存分片數據，在數據擴展上起到了重要的作用(由Master Node決定如何把分片分配到Data Node上)
    * 透過增加Data Node可以解決水平擴展和數據單點的問題
    * 節點啟動後，預設就是Data Node，可以設定node.data: false禁止
* Coordinating Node
    * 負責接受Client的請求，將請求分發到合適的節點，最終把結果匯集到一起
    * 每個節點默認都起到了Coordinating Node的職責
* Hot & Warm Node
    * 不同硬體配置的Data Node，用來實現Hot & Warm架構，降低集群部署的成本
* Machine Learning Node
    * 負責跑機器學習Job，用來做異常檢測
* Tribe Node
    * (5.3開始使用Cross Cluster Search) Tribe Node連接到不同的Elasticsearch集群，並且支持將這些集群當成一個單獨的集群處理

### 分片 (Primary Shard & Replica Shard)
* Primary Shard : 用以解決數據水平擴展的問題。透過主分片可以將數據分布到集群內的所有Node之上
    * 一個分片是一個運行的Lucene的實例
    * 主分片數在索引創建時指定，後續不允許修改，除非Reindex
* Replica : 用以解決數據高可用的問題。分片是主分片的拷貝
    * 副本分片數可以動態調整
    * 增加副本數可以在一定程度上提高服務的可用性(讀取的吞吐)
    * 數據可用性: 透過引入Replica Shard提高數據的可用性。一旦主分片丟失，副本可以Promote成主分片。每個節點上都有完整備份的數據。如果不設置副本，一旦出現結點硬體故障，就有可能造成數據丟失

分片的設定
* 對於生產環境中分片的設定需要提前做好容量規劃
    * 主分片數設置過小
        * 導致後續無法增加節點實現水平擴展
        * 單個分片的數據量太大，導致數據重分配耗時
    * 主分片數設置過大，(7.0開始默認主分片設置成1，解決了over-sharding的問題)
        * 影響搜索結果的相關性評分，影響統計結果的準確性
        * 單個節點上過多的分片會導致資源浪費，同時也會影響性能
    * 副本分片數設置過多，會降低集群整體的寫入效能
* 故障轉移
    * 3個節點共同組成。包含了1個索引、索引設置了3個Primary Shard和1個Replica
    * 節點1是Master節點，節點意外出現故障。集群重新選舉Master節點
    * Node3上的R0提升成P0，集群變黃
    * R0和R1重新分配，集群變綠

集群的健康狀況
* GET _cluster/health
* Green : 主分片與副本都正常分配
* Yellow : 主分片全部正常分配，有副本分片未能正常分配
* Red : 有主分片未能分配
    * 例如當Server Disk容量超過85%時，去創建一個新的索引

### 當Document儲存在Shard上
* Document會儲存在具體的某個主分片和副本分片上: 例如Document1，會儲存在P0和R0分片上
* Document到分片的映射算法
    * 確保Document能均勻分布在所用分片上，充分利用硬體資源，避免部分機器空閒，部分機器繁忙
    * 潛在的算法
        * 隨機 / Round Robin。當查詢Document1，分片數很多需要多次查詢才可能查到
        * 維護Document到分片的映射關係，當Document數據量大的時候，維護成本高
        * 實時計算，透過Document1自動算出需要去哪個分片獲取

#### Document到分片的路由算法
* shard = hash(_routing) % number_of_primary_shards
    * Hash算法確保Document均勻分散到分片中
    * 默認的_routing值是Document id
    * 可以自行制定routing數值，例如用相同國家的商品，都分配到指定的shard
    * 設定Index Settings後，Primary數不能修改的根本原因

### 分片及其生命週期
#### 倒排索引不可變性
* 倒排索引採用Immutable Design，一旦生成不可更改
* 不可變性，帶來的好處:
    * 無需考慮併發寫文件的問題，避免了鎖機制帶來的性能問題
    * 一旦讀入cpu的文件系統緩存便留在那。只又文件系統存有足夠的空間，大部分請求就會直接請求快取，提升了很大的效能
    * 緩存容易生成和維護 / 數據可以被壓縮
* 不可變性帶來的挑戰: 如果需要讓一個新的Document可以被搜尋，需要重建整個索引。

#### Lucene Index
* 在Lucene中，單個倒排索引文件被稱為Segment。Segment是自包含且不可變更的。多個Segments彙總再一起稱為Lucene的Index，其對應的就是ES中的Shard
* 當有新Document寫入時，會生成新的Segment，查詢時會同時查詢所有Segments，並且對結果彙總。Lucene中有一個文件，用來記錄所有Segments訊息，叫做Commit Point
* 刪除的文檔資訊，保存在.del文件中

#### Refresh
* 將Index buffer寫入Segment的過程叫Refresh。Refresh不執行fsync操作
* Refresh頻率: 默認1秒發生一次，可透過index.refresh_internal配置。Refresh後，數據就可以被搜索到了。這也是為什麼Elasticsearch被稱為近實時搜索
* 如果系統有大量的數據寫入，那就會產生很多的Segment
* Index Buffer被占滿時，會觸發Refresh，默認值是JVM的10%

#### Transaction Log
* Segment寫入硬碟的過程相對耗時，借助文件系統緩存，Refresh時先將Segment寫入緩存以開放查詢
* 為了保證數據不會丟失，所以在Index文檔時，同時寫Transaction Log，高版本開始，Transaction默認寫入。每個分片有一個Transaction Log
* 在ES Refresh時，Index Buffer被清空，Transaction log不會清空

#### Flush
* ES Flush & Lucene Commit
    * 調用Refresh，Index Buffer清空並且Refresh
    * 定用fsync，將緩存中的Segments寫入硬碟
    * 清空(刪除)Transaction Log
    * 默認30分鐘調用一次
    * Transaction Log滿(默認為512MB)

#### Merge
* Segment很多，需要定期被合併
    * 減少Segments / 刪除已經刪除的Document
* ES和Lucene會自動進行Merge操作
    * POST my_index/_forcemerge

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

多字段類型
* 多字段特性
    * 廠商名字實現精確匹配
        * 增加一個keyword字段
    * 使用不同的analyzer
        * 不同語言
        * 拼音字段的搜尋
        * 支持為搜尋和索引指定不同的analyzer

Exact Values v.s. Full Text
* Exact Value: 包括數字/日期/具體的一個字符串
    * Elasticsearch中的keyword
    * 在索引時，不需要做特殊的分詞處理
* Full Text: 非結構化的文本數據
    * Elasticsearch中的text

自定義分詞
* 當Elasticsearch自帶的分詞器無法滿足時，可以自定義分詞器，透過組合不同的組件實現
    * Character Filter
    * Tokenizer
    * Token Filter

Character Filters
* 在Tokenizer之前對文本進行處理，例如增加刪除及替換字符。可以配置多個Character Filters。會影響Tokenizer的position和offset資訊
* 一些自帶的Character Filters
    * HTML strip - 去除html標籤
    * Mapping - 字符串替換
    * Pattern replace - 正則匹配替換
Tokenizer
* 將原始的文本按照一定的規則切分為詞(term or token)
* Elasticsearch內置的Tokenizers
    * whitespace / standard / uax_url_email / pattern / keyword / path hierarchy
* 可以用Java開發插件，自訂Tokenizer
Token Filters
* 將Tokenizer輸出的單詞(term)，進行增加/修改/刪除
* 自帶的Token Filters
    * Lowercase / stop / synonym

### Index Template
* Index Templates - 幫助設定Mappings和Settings，並按照一定的規則，自動匹配到新創建的索引上
    * 模板僅在一個索引被新創建時才會產生作用，修改模板不會影響到已創建的索引
    * 可以設定多個索引模板，這些設定會被"merge"在一起
    * 可以指定"order"的數值，控制"merging"的過程

Index Template的工作方式
* 當一個索引被新建立時
    * 應用Elasticsearch默認的settings和mappings
    * 應用order數值低的Index Template的設定
    * 應用order數值高的Index Template的設定，之前的設定會被覆蓋
    * 應用創建索引時，用戶所指定的Settings和Mappings，並覆蓋之前模板中的設定

### Dynamic Template
* 根據Elasticsearch識別的數據類型，結合字段名稱來動態設定字段類型
    * 所有的字符串類型都設定成Keyword，或者關閉keyword字段
    * is開頭的字頭都設置成boolean
    * long_開頭的都設置成long類型

```
PUT my_test_index
{
    "mappings": {
        "dynamic_templates": [
            {
                "full_name": {
                    "path_match": "name.*",
                    "path_unmatch": "*.middle",
                    "mapping": {
                        "type": "text",
                        "copy_to": "full_name"
                    }
                }
            }
        ]
    }
}
```
* Dynamic Template是定義在某個索引的Mapping中
* Template有一個名稱
* 匹配規則是一個數組
* 為匹配到字段設置Mapping

## Search(搜尋)
### Aggregation(聚合)
* Elasticsearch除搜尋以外，提供針對ES數據進行統計分析的功能
    * 實時性高
    * Hadoop(T+1)
* 透過聚合會得到一個數據的概覽，是分析和總結全套的數據，而不是尋找單個文檔
    * XX和XX的客房數量
    * 不同的價格區間，可預訂的一般型/五星級的數量
* 高性能，只需要一條語句就能夠從Elasticsearch得到分析結果
    * 無需在客戶端自己去實現分析邏輯

集合的分類
* Bucket Aggregation - 一些列滿足特定條件的文檔的集合
* Metric Aggregation - 一些數學運算，可以對文檔字段進行統計分析
* Pipeline Aggregation - 對其他聚合結果進行二次聚合
* Matrix Aggregation - 支持對多個字段的操作並提供一個結果矩陣

Metric
* Metric會基於數據集計算結果，除了支持在字段上進行計算，同樣也支持在腳本(painless script)產生的結果之上進行計算
* 大多數Metric是數學計算，僅輸出一個值
    * min / max / sum / avg / cardinality
* 部分metric支持輸出多個數值
    * stats / percentiles / percentile_ranks

### 基於Term的查詢
* Term的重要性
    * Term是表達語意的最小單位。搜尋和利用統計語言模型進行自然語言處理都需要處理Term
* 特色
    * Term Level Query: Term Query / Range Query / Exists Query / Prefix Query / Wildcard Query
    * 在ES中，Term查詢，對輸入不做分詞。會將輸入作為一個整體，在倒排索引中查找準確的詞項，並且使用相關度評分公式為每個包含該詞項的文檔進行相關度評分，例如"Apple Store"
    * 可以透過Constant Score將查詢轉換成一個Filtering，避免評分，並利用緩存提高性能

複合查詢 - Constant Score轉為Filter
* 將Query轉成Filter，忽略TF-IDF計算，避免相關性評分的開銷
* Filter可以有效利用緩存

### 基於全文的查詢
* 基於全文本的查找
    * Match Query / Match Phrase Query / Query String Query
* 特色
    * 索引和搜尋時都會進行分詞，查詢字符串先傳遞到一個合適的分詞器，然後生成一個供查詢的詞項列表
    * 查詢時，會先對輸入的查詢進行分詞，然後每個詞項逐個進行底層的查詢，最終將結果進行合併，並為每個文檔生成一個評分。例如查"Matrix reloaded"，會查到包括Matrix或者reload的所有結果

### 結構化搜尋
* 結構化數據 & 結構化搜尋
    * 如果不需要算分，可以透過Constant Score，將查詢轉為Filtering
* 範圍查詢和Date Math
* 使用Exist查詢處理非空Null值
* 精確值 & 多值字段的精確值查找
    * Term查詢是包含，不是完全相等。針對多值字段查詢需要注意

### 相關性和相關性評分
* 相關性 - Relevance
    * 搜尋的相關性算分，描述了一個文檔和查詢語句匹配的程度。ES會對每個匹配查詢條件的結果進行算分_score
    * 評分的本質是排序，需要把最符合用戶需求的文檔排在前面。ES5之前，默認的相關性評分採用TF-IDF，現在採用BM25

詞頻TF
* Term Frequency: 檢索在一篇文檔 中出現的頻率
    * 檢索詞出現的次數除以文檔的總字數
* 度量一條查詢和結果文檔相關性的簡單方法: 簡單將搜尋中每一個詞的TF進行相加
    * TF(區塊鏈) + TF(的) + TF(應用)
* Stop Word
    * "的"在文檔中出現了很多次，但是對貢獻相關度幾乎沒有用處，不應該考慮他們的TF

逆文檔頻率IDF
* DF: 檢索詞在所有文檔中出現的頻率
    * "區塊鏈"在相對比較少的文檔中出現
    * "應用"在相對比較多的文檔中出現
    * "Stop Word"在大量的文檔中出現
* Inverse Document Frequency: 簡單說=log(全部文檔數/檢索詞出現過的文檔總數)
* TF-IDF本質上就是將TF加總變成加權後加總
    * TF(區塊鏈)*IDF(區塊鏈) + TF(的)*IDF(的) + TF(應用)*IDF(應用)

Boosting Relevance
* Boosting是控制相關度的一種手段
    * 索引、字段或查詢子條件
* 參數boost的含意
    * boost > 1, 評分的相關度相對性提升
    * 0 < boost < 1, 打分的權重相對性降低
    * boost < 0, 貢獻負分

### Query Context & Filter Context
* 高級搜尋的功能: 支持多項文本輸入，針對多個字段進行搜尋
* 搜尋引擎一般也提供基於時間/價格等條件過濾
* 在Elasticsearch中，有Query和Filter兩種不同的Context
    * Query Context: 相關性評分
    * Filter Context: 不需要算分(Yes or No)，可以利用Cache，獲得更好的性能

#### bool查詢
* 一個bool查詢，是一個或者多個查詢子句的組合
    * 總共包括4種子句，其中2種會影響評分，2種不影響評分
* 相關性並不只是全文本檢索的專利，也適用於yes|no的子句，匹配的子句越多，相關性評分越高。如果多條查詢子句被合併為一條複合查詢語句，比如bool查詢，則每個查詢子句計算得出的評分會被合併到總的相關性評分中

|  |   |
| ---- | ---- |
|  must  | 必須匹配，貢獻評分 |
| should | 選擇性匹配，貢獻評分 |
| must_not | Filter Context 查詢子句，必須不能匹配 |
| filter | Filter Context 必須匹配，但是不貢獻評分 |

bool查詢語法
```
POST /products/_search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "price" : "30" }
      },
      "filter": {
        "term" : { "avaliable" : "true" }
      },
      "must_not" : {
        "range" : {
          "price" : { "lte" : 10 }
        }
      },
      "should" : [
        { "term" : { "productID.keyword" : "JODL-X-1937-#pV7" } },
        { "term" : { "productID.keyword" : "XHDK-A-1293-#fJ3" } }
      ],
      "minimum_should_match" :1
    }
  }
}
```
* 子查詢可以任意順序出現
* 可以嵌套多個查詢
* 如果你的bool查詢中，沒有must條件，should中必須至少滿足一條查詢

#### 單字符串多字段查詢
```

PUT /blogs/_doc/1
{
    "title": "Quick brown rabbits",
    "body":  "Brown rabbits are commonly seen."
}

PUT /blogs/_doc/2
{
    "title": "Keeping pets healthy",
    "body":  "My quick brown fox eats rabbits on a regular basis."
}

POST /blogs/_search
{
    "query": {
        "bool": {
            "should": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}

POST blogs/_search
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ]
        }
    }
}


POST blogs/_search
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ],
            "tie_breaker": 0.2
        }
    }
}
```
* 評分過程
    * 查詢should語句中的兩個查詢
    * 加總兩個查詢的評分
    * 乘以匹配語句的總數
    * 除以所有語句的總數

引入Dis Max Query => 返回字段當中評分最高的作為整體的評分

可以引入tie_breaker，Tie Breaker是一個介於0-1之間的浮點數。0代表使用最佳匹配，1代表所有語句同等重要。
*   獲取最佳匹配語句的評分_score
*   將其他匹配語句評分與tie_breaker相乘
*   對以上評分求和並規範化


Multi Match(三種場景)
* 最佳字段(Best Fields)
    * 當字段之間相互競爭，又相互關聯。例如title和body這樣的字段。評分來自最佳匹配字段
* 多數字段(Most Fields)
    * 處理英文內容時，一種常見的手段是，在主字段(English Analyzer)，抽取詞幹，加入同義詞，以匹配更多的文檔。相同的文本，加入子字段(Standard Analyzer)，以提供更加精確的匹配。其他字段作為匹配文檔提高相關度的信號，匹配字段越多則越好
* 混合字段(Cross Field)
    * 對於某些實體，例如人名、地址、圖書資訊，需要在多個字段中確定資訊，單個字段只能作為整體的一部分，希望在任何這些列出的字段中找到盡可能多的詞
Multi Match Query
```
POST blogs/_search
{
  "query": {
    "multi_match": {
      "type": "best_fields",
      "query": "Quick pets",
      "fields": ["title","body"],
      "tie_breaker": 0.2,
      "minimum_should_match": "20%"
    }
  }
}
```
* Best Fields是默認類型，可以不用指定
* Minimum should match等參數可以傳遞到生成的query中

跨字段搜尋
```
{
    "street": "1",
    "city": "2",
    "country": "3",
    postcode: "4"
}
POST address/_search
{
    "query": {
        "multi_match": {
            "query": "1",
            "type": "most_fields",
            // "operator": "and",
            "fields": ["street", "city", "country", "postcode"]
        }
    }
}
```
* 無法使用Operator
* 可以用copy_to解決，但需要額外的存儲空間

### 自然語言與查詢Recall
* 當處理人類自然語言時，有些情況，儘管搜尋和原本不完全匹配，但是希望搜到一些內容
* 一些可採取的優化
    * 規一化: 清除變音符號
    * 抽取詞根: 清除單複數和時態的差異
    * 包含同義詞
    * 拼寫錯誤: 拼寫錯誤或者同音異形詞

#### 混合多語言的挑戰
* 一些具體的多語言場景
    * 不同的索引使用不同的語言 / 同一個索引中，不同的字段使用不同的語言 / 一個文檔的一個字段內混合不同的語言
* 混合語言存在的一些挑戰
    * 詞幹提取: 以色列文檔包含了希伯來語、阿拉伯語、俄語和英語
    * 不正確的詞檔頻率: 以英文為主的文章中，德文算分高(稀有)
    * 需要判斷用戶搜尋時使用的語言，語言識別(Compact Language Detector)
        * 例如根據語言查詢不同的索引

#### 分詞的挑戰
* 英文分詞: You're分成一個還是多個? Half-baked
* 中文分詞
    * 分詞標準: 實際情況須制訂不同的標準
    * 歧義 (組合型歧義、交集型歧義、真歧義)

### Search Template - 解耦程序 & 搜尋DSL
```
POST _scripts/tmdb
{
  "script": {
    "lang": "mustache",
    "source": {
      "_source": [
        "title","overview"
      ],
      "size": 20,
      "query": {
        "multi_match": {
          "query": "{{q}}",
          "fields": ["title","overview"]
        }
      }
    }
  }
}
```
* Elasticsearch的查詢語句
    * 對相關性評分 / 查詢性能都至關重要
    * 在開發初期，雖然可以明確定義查詢餐數，但是往往還不能定義最終查詢的DSL具體結構
        * 透過Search Template定義一個Contract
    * 各司其職(解耦)
        * 開發人員 / 搜尋工程師 / 性能工程師

### Index Alias 實現零停機維運
```
PUT movies-2019/_doc/1
{
  "name":"the matrix",
  "rating":5
}

PUT movies-2019/_doc/2
{
  "name":"Speed",
  "rating":3
}

POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "movies-2019",
        "alias": "movies-latest"
      }
    }
  ]
}

POST movies-latest/_search
{
  "query": {
    "match_all": {}
  }
}

POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "movies-2019",
        "alias": "movies-lastest-highrate",
        "filter": {
          "range": {
            "rating": {
              "gte": 4
            }
          }
        }
      }
    }
  ]
}

POST movies-lastest-highrate/_search
{
  "query": {
    "match_all": {}
  }
}
```

### 評分與排序
* Elasticsearch默認會以文檔的相關度評分進行排序
* 可以透過指定一個或者多個字段進行排序
* 使用相關度評分(score)排序，不能滿足某些特定條件
    * 無法針對相關度，對排序實現更多的控制

#### Function Score Query
* Function Score Query
    * 可以在查詢結束時，對每一個匹配的文檔進行一系列的重新算分，根據新生成的分數進行排序
* 提供了幾種默認的計算分值得函數
    * Weight: 為每一個文檔設置一個簡單而不被規範化的權重
    * Field Value Factor: 使用該數值來修改_score，例如將熱度和點讚數作為評分的參考因素
    * Random Score: 為每一個用戶使用一個不同(隨機的)評分結果
    * 衰減函數: 以某個字段的值為標準，距離某個值越近，得分越高
    * Script Scroe: 自定義腳本完全控制所需邏輯

##### Boost Mode & Max Boost
* Boost Mode
    * Multiply: 評分與函數值得乘積
    * Sum: 評分和函數的和
    * Min / Max: 評分與函數取最小/最大值
    * Replace: 使用函數值取代評分
* Max Boost可以將算分控制在一個最大值

### Term & Phrase Suggester
#### Elasticsearch Suggester API
* 搜尋引擎中類似的功能在Elasticsearch中是透過Suggester API實現的
* 原理: 將輸入的文本分解為Token，然後在索引的字典裡查找相似的Term並返回
* 根據不同的使用場景，Elasticsearch設計了4種類別的Suggesters
    * Term & Phrase Suggester
    * Complete & Context Suggester

#### Term Suggester
```
POST /articles/_search
{
  "size": 1,
  "query": {
    "match": {
      "body": "lucen rock"
    }
  },
  "suggest": {
    "term-suggestion": {
      "text": "lucen rock",
      "term": {
        "suggest_mode": "missing",
        "field": "body"
      }
    }
  }
}
```
* Suggester就是一種特殊類型的搜尋。"text"裡是調用時候提供的文本，通常來自於用戶介面上用戶輸入的內容
* 用戶輸入的"lucen"是一個錯誤的拼寫
* 會到指定的字段"body"上搜尋，當無法搜尋到結果時(missing)，返回建議的詞

#### Phrase Suggester
```
POST /articles/_search
{
  "suggest": {
    "my-suggestion": {
      "text": "lucne and elasticsear rock hello world ",
      "phrase": {
        "field": "body",
        "max_errors":2,
        "confidence":0,
        "direct_generator":[{
          "field":"body",
          "suggest_mode":"always"
        }],
        "highlight": {
          "pre_tag": "<em>",
          "post_tag": "</em>"
        }
      }
    }
  }
}
```
* Phrase Suggester在Term Suggester上增加了一些額外的邏輯
* 一些參數
    * Suggest Mode: missing, popular, always
    * Max Errors: 最多可以拼錯的Terms數
    * Confidence: 限制返回結果數，默認為1

### Auto Complete
#### The Completion Suggester
* Completion Suggester提供了自動完成(Auto Complete)的功能。用戶每輸入一個字符就需要及時發送一個查詢請求到後端查找匹配項
* 對性能要求比較苛刻。Elasticsearch採用了不同的數據結構，並非透過倒排索引來完成，而是將Analyze的數據編碼成FST和索引一起存放。FST會被ES整個加載進記憶體，速度很快
* FST只能用於前綴查找

#### 使用Completion Suggester的一些步驟
```
PUT articles
{
  "mappings": {
    "properties": {
      "title_completion":{
        "type": "completion"
      }
    }
  }
}
```
* 定義Mapping，使用"completion" type
* 索引數據
* 運行"suggest"查詢，得到搜尋建議

#### Context Suggester
* Completion Suggester的擴展
* 可以在搜尋中加入更多的上下文資訊，例如，輸入"star"
    * 咖啡相關: 建議"Starbucks"
    * 電影相關: "star wars"

#### 實現Context Suggester
* 可以定義兩種類型的Context
    * Category - 任意的字符串
    * Geo - 地理位置資訊
* 實現Context Suggester的具體步驟
    * 訂製一個Mapping
    * 索引數據，並且為每個文檔加入Context資訊
    * 結合Context進行Suggestion查詢

### 精準度和召回率
* 精準度
    * Completion > Phrase > Term
* 召回率
    * Term > Phrase > Completion
* 性能
    * Completion > Phrase > Term

## 配置跨集群搜尋
* 單集群 - 當水平擴展時，節點數不能無限增加
    * 當集群的meta資訊(節點、索引和集群狀態)過多，會導致更新壓力變大，單個Active Master會成為性能瓶頸，導致整個集群無法正常工作
* 早期版本，透過Tribe Node可以實現多集群訪問的需求，但還是存在一定的問題
    * Tribe Node會以Client Node的方式加入每個集群。集群中Master節點的任務變更需要Tribe Node的回應才能繼續
    * Tribe Node不保存Cluster State資訊，一旦重啟，初始化很慢
    * 當多個集群存在索引重名的情況時，只能設置一種Prefer規則

### 跨集群搜尋
* 早期Tribe Node的方案存在一定的問題，現已被Deprecated
* Elasticsearch5.3引入了跨集群搜尋的功能(Cross Cluster Search)
    * 允許任何節點扮演federated節點，以輕量的方式，將搜尋請求進行代理
    * 不需要以Client Node的形式加入其他集群
```
//在每個集群上設置
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster0": {
          "seeds": [
            "127.0.0.1:9300"
          ],
          "transport.ping_schedule": "30s"
        },
        "cluster1": {
          "seeds": [
            "127.0.0.1:9301"
          ],
          "transport.compress": true,
          "skip_unavailable": true
        },
        "cluster2": {
          "seeds": [
            "127.0.0.1:9302"
          ]
        }
      }
    }
  }
}

#cURL
curl -XPUT "http://localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'

curl -XPUT "http://localhost:9201/_cluster/settings" -H 'Content-Type: application/json' -d'
{"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'

curl -XPUT "http://localhost:9202/_cluster/settings" -H 'Content-Type: application/json' -d'
{"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'


#查詢
GET /users,cluster1:users,cluster2:users/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 20,
        "lte": 40
      }
    }
  }
}
```

## 剖析分布式查詢及相關性評分

### Query階段
* 使用者發出搜索請求到ES節點，節點收到請求後，會以Coordinating節點的身分，在6個主副分片中隨機選擇3個分片發送查詢請求
* 被選中的分片執行查詢進行排序，然後每個分片都會返回From+Size個排序後的Document ID和排序值給Coordinating節點

### Fetch階段
* Coordinating Node會將Query階段，從每個分片獲取的排序後Document ID列表重新進行排序，選取From到From+Size個Document ID
* 以multi get請求的方式，到相應的分片獲取詳細的Document數據。

### Query Then Fetch潛在的問題
* 性能問題
    * 每個分片上需要查的Documeny數量 = from + size
    * 最終協調節點需要處理: number_of_shard * (from + size)
    * 深度分頁
* 相關性評分
    * 每個分片都基於自己的分片上的數據進行相關度計算，這會導致評分偏離，特別是數據量很少時。相關性評分在分片之間是相互獨立，當Document總數很少的情況下，如果主分片大於1，主分片數越多，相關性評分會越不準

### 解決評分不準的方法
* 當數據量不大的時候，將主分片數設置為 1
* 當數據量夠多時，只要保證文件均勻分佈在各個分片上, 結果一班就不會出現偏差

* 使用DFS Query Then Fetch
    * 在搜尋的URL 中指定參數_search?search_type=dfs_query_then_fetch
    * 到每個分片把各分片的詞頻和文檔頻率進行蒐集, 然後完整的進行一次相關性算分，耗費更加多的CPU和記憶體，執行性能低下，一般不建議使用.
```
DELETE message

PUT message
{
  "settings": {
    "number_of_shards": 20
  }
}

POST message/_bulk
{"create":{}}
{"content": "good"}
{"create":{}}
{"content": "good morning"}
{"create":{}}
{"content": "good morning everyone"}

// 此時評分是有問題的，所有Document的評分都是0.2876821
POST message/_search
{
  "query": {
    "term": {
      "content": {
        "value": "good"
      }
    }
  }
  //,"explain": true
}


// 使用 DFS Query Then Fetch正確獲得評分 
// 這種方式的性能不好，通常較少使用
POST message/_search?search_type=dfs_query_then_fetch
{
  "query": {
    "term": {
      "content": {
        "value": "good"
      }
    }
  }
}
```

## 排序及Doc Values & Fielddata

### 排序
* ES預設採用相關性算分對結果進行降序排序
* 可透過設定sort參數自行設定排序，若不指定_score，則算分為null

### 排序的過程
* 排序是針對字段原始內容進行的，倒排索引無法發揮作用
* 需要用到正排索引，透過文檔Id和字段快速得到字段原始內容
* ES有兩種實作方式
    * Fielddata : 預設是關閉的, 可透過修改Mapping 中欄位的fielddata: true來開啟，但必須了解這邊儲存的是分詞後的結果，若這不是想要的效果，那可以為該text欄位加一個keyword子欄位，然後直接使用該子欄位
    ```
    {
    "properties": {
            "屬性": {
                "type": "text",            // 以text類型為例
                "fielddata": true        // keyword 類型的默認開啟, 因此無需特別設置
            }
        }
    }
    ```

    * Doc Values (列式儲存，對Text類型無效)
        * 列式儲存，對text類型無效
        * 預設是開啟的，可透過修改Mapping 中欄位的doc_values: false來關閉
        * 重新打開需要重建索引
        * 關閉Doc Values 的好處:1.增加索引的速度 2.減少磁碟空間
        * 什麼情況要關閉:當明確不需要該欄位參與排序及聚合分析時

### Doc Values vs Field Data
|  | Doc Values | Field Data |
| :----: | :----: | :----: |
| 何時建立 | 索引時和倒排索引一起創建 | 搜索時動態創建 |
| 建立位置 | 硬碟文件 | JVM Heap |
| 優點 | 避免大量占用記憶體 | 索引速度快，不占用額外的硬碟空間 |
| 缺點 | 降低索引速度，占用額外硬碟空間 | 文檔過多時，動態創建開銷大，占用過多JVM Heap |
| 默認值 | ES 2.x之後 | ES 1.x及之前 |

## Reference
[極客時間](https://time.geekbang.org/course/detail/100030501-102655)

[Cerebro](https://github.com/lmenezes/cerebro)