## ElasticSearch

### 一. 概述

Elasticsearch是一个基于[Apache Lucene](https://lucene.apache.org/core/)实时分布式存储、搜索和分析引擎，具有以下功能

- 分布式的实时文件存储，**每个字段都被索引**并可被搜索

- 分布式的实时分析搜索引擎

- 集群，可以扩展到上百台服务器，处理PB级结构化或非结构化数据

- ```markdown
  Relational DB -> Databases -> Tables -> Rows -> Columns
  Elasticsearch -> Indices   -> Types  -> Documents -> Fields
  ```


### 二. 节点和集群

#### 1. 集群（cluster）

- 一组具有相同`cluster.name`的节点集合，他们协同工作，共享数据并提供故障转移和扩展功能。

- 集群中一个节点会被选举为**主节点(master)**，它将临时管理集群级别的一些变更，例如新建或删除索引、增加或移除节点等。

  - 主节点不参与文档级别的变更或搜索，这意味着在流量增长的时候，该主节点不会成为集群的瓶颈。
  - 任何节点都可以成为主节点。

- 集群健康，三种状态：`green`、`yellow`或`red`。

  | 颜色       | 意义                    |
  | -------- | --------------------- |
  | `green`  | 所有主要分片和复制分片都可用        |
  | `yellow` | 所有主要分片可用，但不是所有复制分片都可用 |
  | `red`    | 不是所有的主要分片都可用          |



#### 2. 节点（node）

一个运行着的Elasticsearch实例，可修改 **elasticsearch.yml** 中的 ***node.name***。

#### 3. 分片（shard）

### 三. 索引

Elasticsearch 中的数据库。只是一个用来指向一个或多个**分片(shards)**的**“逻辑命名空间(logical namespace)”**.

- **索引（名词）**，一个**索引(index)**就像是传统关系数据库中的**数据库**，它是相关文档存储的地方，index的复数是**indices **或**indexes**。
- **索引（动词）**，**「索引一个文档」**表示把一个文档存储到**索引（名词）**里，以便它可以被检索或者查询。这很像SQL中的`INSERT`关键字，差别是，如果文档已经存在，新的文档将覆盖旧的文档。
- **倒排索引**， 传统数据库为特定列增加一个索引，例如B-Tree索引来加速检索。Elasticsearch和Lucene使用一种叫做**倒排索引(inverted index)**的数据结构来达到相同目的。即由属性值来确定记录的位置。

### 五. 参考资料

- https://www.elastic.co/
- https://es.xiaoleilu.com/