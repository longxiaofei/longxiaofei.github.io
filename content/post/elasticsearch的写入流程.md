---
title: "Elasticsearch的写入流程"
date: 2021-10-05T08:50:14+08:00
draft: false
categories: ["Elasticsearch"]
tags: ["Elasticsearch", "数据库"]
---

> 粗略整理了一下Elasticsearch写入数据的流程, 有不对的地方希望指正, 下面Elasticsearch简称为es。

## 提出的问题
* es数据写入操作响应成功后，es随即崩溃，数据会丢吗?
* es为什么是准实时的?

## es单分片单副本的写入流程

### 写入流程

如图所示（该图的序号没有先后顺序的意义)

![](/img/es-write/0.png)

1. 客户插入(索引)一个文档

2. 对数据进行格式化
    * 对文档数据进行处理和验证

3. 将操作数据日志写入translog内存缓冲区
    * 这一步骤在2之后进行
    * 在7.x默认配置下，1、2、3、6步骤是同步执行的，只有当6步骤执行完后，才会向客户端返回成功的响应。
    * 如果想要提高写入的效率，并且接受在发生故障丢失一部分数据，可以将6步骤设为异步定时批量执行，具体说明见文档：[文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/index-modules-translog.html)

4. refresh操作
    * 将缓冲区里的数据，在内存中生成新的segment。
    * 默认1s进行一次refresh操作，可以配置这个时间间隔。[文档](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-refresh.html)
    * es的search是对segment中的数据进行检索，所以在默认情况下，新创建的文档最慢也需要1s时间内才能被检索到，这也就是为什么"es搜索为什么是准实时的?"的原因。[文档](https://www.elastic.co/guide/en/elasticsearch/reference/master/near-real-time.html#img-pre-refresh)
    * 如果直接通过文档ID获取文档，是可以从translog中查询数据，所以通过文档ID获取数据是实时的。[文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-get.html)

5. flush操作
    * 将内存中的segment写入磁盘，同时删除磁盘中的translog文件，并修改checkpoint。
    * 默认是每当translog大于512M时，进行一次flush操作，可以进行配置。[文档0](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-flush.html) | [文档1](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-translog.html)

6. 将内存中translog写入磁盘
    * 默认为同步。相关说明在3步骤中已说明，不再赘述。

7. merge操作
    * 将磁盘中小的segment进行合并，并"真正删除"已删除的文档。[文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-merge.html)
    * 防止小的segment过多，影响检索效率。

### 总结
1. es的search是通过查询segment，而segment是通过refresh操作生成的，所以es的搜索实时性，取决于refresh操作的时间间隔。
2. es写入请求成功返回时，es崩溃。当6步骤为同步的话，数据已记录在磁盘的translog中，es重启，会基于translog恢复数据，所以数据不会丢失。当6步骤为异步的话，es重启，会丢失没有在磁盘的translog中的数据。


## es多分片多副本的写入流程

如图所示

![](/img/es-write/1.png)

### 写入流程
1. 客户端写请求发送至node-1
* node根据"路由规则"计算出数据应该存在分片shard-0上。
* 通过集群维护的分片元数据查找得到shard-0主分片位于node0节点上。
* 将请求转发给node0节点

2. 转发请求至node0
* 接收到写请求，处理写入流程，与"单分片单副本的写入流程"一致。
* 将写入操作同时转发给两个从分片。

3. 主分片同步数据至从分片
* 接收到写请求，处理写入流程，与"单分片单副本的写入流程"一致。

### 总结
1. 这三个步骤是同步执行的，只有当所有分片成功处理完"写入请求"后，客户端才会收到写入成功的响应。
2. 注意3步骤中，转发至多个副本分片是并发进行的，所以所有分片成功处理完"写入请求"的时间为`主分片成功写入数据的时间+max(每个副本分片成功写入数据的时间)`
