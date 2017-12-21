## ELK 搭建（5.4.3）

### 一. ElasticSearch安装

1. 下载 https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.4.3.zip

2. 安装、配置

   - 进入 elasticsearch-5.4.3/bin，运行 `./elasticsearch`。

   - 测试，`curl localhost:9200`

     ```json
     {
       "name" : "9j2Sdvb",
       "cluster_name" : "elc-search",
       "cluster_uuid" : "LDB5IlrATaq1w8O0eDNdMg",
       "version" : {
         "number" : "5.4.3",
         "build_hash" : "eed30a8",
         "build_date" : "2017-06-22T00:34:03.743Z",
         "build_snapshot" : false,
         "lucene_version" : "6.5.1"
       },
       "tagline" : "You Know, for Search"
     }
     ```

### 二. Logstash安装

1. 下载 https://artifacts.elastic.co/downloads/logstash/logstash-5.4.3.zip

2. 安装、配置

   - 准备 logstach.conf 配置文件

     ```json
     input { stdin { } }
     output {
       elasticsearch { hosts => ["localhost:9200"] }
       stdout { codec => rubydebug }
     }
     ```

   - 进入 logstash-5.4.3/bin，运行 `./logstash -f ../conf/logstach-simple.conf`。

### 三. Kibana安装

1. 下载 https://artifacts.elastic.co/downloads/kibana/kibana-5.4.3-darwin-x86_64.tar.gz
2. 安装、配置
   - 配置 `config/kibana.yml`，将 elasticsearch.url 指向真正的 Elasticsearch 实例。
   - 进入 kibana-5.4.3-darwin-x86_64/bin，运行 `./kibana`。
   - 在浏览器中，访问 http://localhost:5601

### 四. X-Pack安装

​	X-pack是一个集成了如下功能的扩展包：安全性 security，警报 alerting，监视 monitoring，报告 reporting，图形化分析 graph exploration, 和机器学习 machine learning。

- Elasticsearch下安装，进入bin目录 `./elasticsearch-plugin install x-pack` 。

- Kibana下安装，进入bin目录

   `./kibana-plugin install x-pack` 

  `·/kibana-plugin install file:///some/local/path/x-pack-5.4.3.zip-5.4.3.zip`

#### 1. 安全

