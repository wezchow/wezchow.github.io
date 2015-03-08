---
layout:         post
title:          "MongoDB Map-Reduce"
date:           2015-03-09
categories:     MongoDB
tags:           MongoDB Map-Reduce
image:          /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---


<br/>
看过 PyMongo 的 Map-Reduce 之后，老夫觉得还是需要看一眼 MongoDB 源生支持的 M-R 是怎样的。所以花了点时间把这篇简介翻了一下。翻完以后才觉得有些浪费时间。干货比较少，感觉都是在凑字数的重复描述简单的几件事情。Anyway， 就当继续巩固一下。

#Map-Reduce
Map-reduce 是一种数据处理的范式，它专为大数据背景下的数据聚合场景服务。对于 map-reduce 相关操作，MongoDB 提供了 mapReduce 数据库操作命令。

先来观察下面的 map-reduce 操作：

<img src="http://wnono.com/assets/images/2015-03-09-mongodb_mapreduce.png" />

在这个 map-reduce 操作中，MongoDB 对于每一个文档在输入阶段使用了 map 功能, 当然这里所指的文档是在满足查询条件的情况下。 依据不同的输入，Map 函数会弹射对应的键值对 (Key-value paris)。对于那些有多个值的键， MongoDB 则会进一步应用 reduce 功能，它主要是为收集和聚合数据服务。最后 MongoDB 会把结果保存在一个新的数据集中。除了这种操作，我们还可以把这个结果集重复使用在其他数据聚合的操作中。

在 MongoDB 中所有的 map-reduce 功能都是基于 JavaScript 语言并运行在 mongod 这个进程上。Map-reduce 操作可以从一个单个数据集中获取输入数据，并且在实际执行 Map 操作前，随心所欲的针对数据进行排序或者删减 (Sorting and limiting)。mapReduce 方法可以把计算结果以文档的形式返回，或者也可以写入数据集中。所有输入和输出的数据集都可以来自，或保存至数据库集群。

>注意： 对于所有的聚合操作来说，聚合流水线功能 (Aggregation Pipeline) 提供了更好的性能和更统一的接口。不管如何，map-reduce 功能则提供了一些 Aggregation Pipeline 并不具备的灵活性。

## Map-Reduce JavaScript Functions
在 MongoDB 中，map-reduce 操作可以使用自定义的 JavaScript 函数来 map 或者联结 (associate) key 中的数据。如果一个 key 中有多个 values 被 mapped，则 reduce 操作会把这些值转化为该 key 的一个对象。

使用自定义的 JavaScript 函数使一种更为灵活的 map-reduce 操作成为可能。试想一下，当我们需要处理一个文档，map 功能可以创建多个键值映射或者干脆不映射。Map-reduce 操作同样也可以通过使用自定义 JavaScript 函数来对其他 M-R 操作的结果进行最终处理，比如进行额外的计算等等。


## Map-Reduce Behavior
In MongoDB, the map-reduce operation can write results to a collection or return the results inline. If you write map-reduce output to a collection, you can perform subsequent map-reduce operations on the same input collection that merge replace, merge, or reduce new results with previous results. See mapReduce and Perform Incremental Map-Reduce for details and examples.

When returning the results of a map reduce operation inline, the result documents must be within the BSON Document Size limit, which is currently 16 megabytes. For additional information on limits and restrictions on map-reduce operations, see the mapReduce reference page.

MongoDB supports map-reduce operations on sharded collections. Map-reduce operations can also output the results to a sharded collection. See Map-Reduce and Sharded Collections.
