---
title: elasticsearch01
date: 2017-04-21 03:49:13
tags:
categories: ElasticSearch
---

## 
Elasticsearch是一个基于Apache Lucene(TM)的开源搜索引擎。无论在开源还是专有领域，Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的搜索引擎库。
但是，Lucene只是一个库。想要使用它，你必须使用Java来作为开发语言并将其直接集成到你的应用中，更糟糕的是，Lucene非常复杂，你需要深入了解检索的相关知识来理解它是如何工作的。
Elasticsearch也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。
不过，Elasticsearch不仅仅是Lucene和全文搜索，我们还能这样去描述它：
* 分布式的实时文件存储，每个字段都被索引并可被搜索
* 分布式的实时分析搜索引擎
* 可以扩展到上百台服务器，处理PB级结构化或非结构化数据
而且，所有的这些功能被集成到一个服务里面，你的应用可以通过简单的RESTful API、各种语言的客户端甚至命令行与之交互。

可以用传统的关系数据库来作对比：
```
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
```
## 如何安装
1.安装jdk，我自己机器上的是jdk7
2.下载安装包
``` 
wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.4/elasticsearch-2.3.4.tar.gz
```
3.解压
```
tar -zxvf elasticsearch-2.3.4.tar.gz
```
4.启动服务
```
./bin/elasticsearch
```
5.测试是否成功
```
curl 'http://localhost:9200/?pretty'
```

如果输出下面的结果就大功告成了
```
{
  "count" : 3,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  }
}
```

## 如何使用

###让我们建立一个员工目录

假设我们刚好在Megacorp工作，这时人力资源部门出于某种目的需要让我们创建一个员工目录，这个目录用于促进人文关怀和用于实时协同工作

###索引员工文档
所以为了创建员工目录，我们将进行如下操作：
*为每个员工的文档(document)建立索引，每个文档包含了相应员工的所有信息。
*每个文档的类型为employee。
*employee类型归属于索引megacorp。
*megacorp索引存储在Elasticsearch集群中。

我们看到path:/megacorp/employee/1包含三部分信息：
| 名字 | 说明 |
| ------------- | ------------- |
| megacorp | 索引名 |
| employee | 类型名 |
| 1 | 这个员工的ID |

使用RESTful API，通过9200端口的与Elasticsearch进行通信
``` cpp
curl -XPUT 'localhost:9200/megacorp/employee/1' -d '
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}'

curl -XPUT 'localhost:9200/megacorp/employee/2' -d '
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}'

curl -XPUT 'localhost:9200/megacorp/employee/3' -d '
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}'
```
